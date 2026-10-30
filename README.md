# n8n Automation Projects

Two practice projects built with n8n: a customer support chatbot, and a demo of a savings and loan automation system. All data and rules are invented.

## Project 1: Customer Support Bot

A customer support chatbot for a fictional gadget shop, Nabil Gadgets. It answers questions from the shop's information and hands anything it can't answer to a human.

### How it works

1. **Trigger:** the Chat Trigger node starts the workflow whenever a customer sends a message.
2. **AI Agent:** replies using the shop information in its system message, with a Gemini model.
3. **Memory:** the Simple Memory node lets the bot remember the last 15 messages, so follow-up questions like "how long will that take?" make sense.
4. **Escalation rule:** if the bot can't find a shop-related answer in the shop info, it replies with the single word ESCALATE.
5. **IF check:** an IF node checks whether the reply contains ESCALATE.
   - **True:** the customer gets a handoff message and `escalated` is set to true.
   - **False:** the bot's reply goes to the customer and `escalated` is set to false.
6. **Logging:** every conversation is saved to a data table called `support logs`.

### Data table: `support logs`

| column | what it stores |
|---|---|
| time | When the chat happened |
| customer_message | What the customer typed |
| bot_reply | What the bot answered |
| escalated | Whether it was handed to a human |

### How to run it

1. Import `support-bot.json` into n8n (Workflows → Import from file).
2. Create your own Gemini credential (the workflow doesn't include any keys).
3. Create the data table `support logs` with the four columns above.
4. Open the chat and send "hello".

### What I learned

- **Network errors don't all mean the same thing.** `fetch failed` meant n8n never reached the API, and `Request timed out` meant the request went out but no reply came back in time. Both pointed to a weak connection, not a broken workflow.
- **An error that comes back from a server means my connection works.** `insufficient balance` and "too many requests" were account limits, not network or workflow faults, so I check which kind of error it is before changing anything.

## Project 2: Savings & Loan Automation (demo)

A demo of how a cooperative's savings and loan scheme can be automated in n8n. It uses invented rules and fake data (about 100 imagined members), and it's meant to show the approach, not to run a real organization. It has five workflows that share state through data tables.

### The rules (stored in a `settings` table, never hardcoded)

| rule | value |
|---|---|
| Monthly contribution | ₦50,000, exactly, once per member per month |
| Minimum saving period before borrowing | 90 days |
| Maximum loan | ₦150,000 |
| Interest | 5% flat, charged once |
| Repayment period | 60 days |
| Late penalty | 8%, applied once |

### Data tables

| table | what it stores |
|---|---|
| `members` | member_id, names, phone, first_contribution_date, status |
| `contributions` | member_id, amount, date, month |
| `loans` | member_id, amount, interest, total_due, loan_date, due_date, status, penalty_applied, amount_paid |
| `statements` | member_id, names, month, months_paid, total_saved, loan_due, generated_on |
| `settings` | the rules above, in one row |

### Workflow 1: `loan-request.json`

1. A member submits their ID and the amount they want.
2. The workflow looks up the member, any active loan they have, and the rules.
3. An IF node checks three things: saved for at least 90 days, no active loan, and amount within the limit.
4. **Approved:** it calculates interest, total due, and due date, then saves the loan.
5. **Rejected:** it replies with the reason.

### Workflow 2: `repay-loan.json`

1. A member submits their ID.
2. The workflow finds their active loan. If there isn't one, it says so and stops.
3. It checks whether today is past the due date.
4. It adds the 8% penalty only if the loan is late **and** `penalty_applied` is false. If the daily job already charged the penalty, the stored total is used as it is.
5. It marks the loan repaid and records the amount paid.

### Workflow 3: `penalty-check.json`

A scheduled workflow that runs once a day with no one clicking anything.

1. A Schedule Trigger starts it.
2. It finds loans that are active and not yet penalized.
3. A filter keeps only the loans past their due date.
4. It reads the penalty percentage from `settings`.
5. It calculates the new total (total due plus 8%).
6. It updates the loan with the new total and sets `penalty_applied` to true.

Because of the `penalty_applied` flag, running it twice in a row changes nothing, so a loan can never be penalized twice.

### Workflow 4: `record-contribution.json`

1. A treasurer submits a member ID and an amount.
2. The workflow looks up the member, any payment already recorded this month, and the rules.
3. An IF node checks three things: the member exists, no payment exists for this month, and the amount equals the monthly contribution.
4. **Valid:** it saves the payment with today's date and the month (like `2026-10`).
5. **Invalid:** it replies with every rule that failed, not just the first.

### Workflow 5: `monthly-statements.json`

A scheduled workflow that builds one statement per member, once a month.

1. A Schedule Trigger starts it.
2. It reads all contributions, all active loans, and all members. The first two lookups run once (Execute Once), so nothing is multiplied.
3. For each member, it counts their payments, adds up what they saved, and finds any loan they still owe.
4. It saves one statement row per member. The save is an upsert on member and month, so a rerun updates the statement instead of duplicating it.

### How the five workflows connect

The workflows never call each other. They share state through the data tables:

| workflow | started by | what it does to the data |
|---|---|---|
| loan request | a member's form | creates a loan |
| penalty check | the clock, daily | adds the penalty once, sets the flag |
| repay loan | a member's form | closes the loan |
| record contribution | a treasurer's form | adds a payment |
| monthly statements | the clock, monthly | summarizes payments and loans per member |

### How to run it

1. Import the five JSON files into n8n (Workflows → Import from file).
2. Create the data tables with the columns above.
3. Add one row to `settings` with the rule values.
4. Add a few fake members, some with a first contribution date over 90 days ago and some recent.
5. Publish `penalty-check` and `monthly-statements` if you want them to run on their own.

### Known limitations

- The loan request rejection names only the first rule that failed. The contribution rejection names all of them.
- Repayment is full payment only, with no installments.
- The scheduled workflows only run while n8n is running, so a real deployment needs an always-on server.
- Statements are labeled with the month the job runs in, so a run on the 1st labels the new month. A real client would decide whether to run it at month end.
- Everything here is demo data with invented rules, not a production system.

### What I learned

- **A field can't be used in the node that creates it.** Calculations that depend on each other need two nodes, or the result comes out as NaN or undefined.
- **A true/false value saved as text is dangerous.** The text "false" counts as true, so yes/no fields must be set to Boolean.
- **A flag makes a scheduled job safe to repeat.** `penalty_applied` stops a daily job (and the repay workflow) from charging the same loan twice.
- **An Update row should list only the columns it changes,** and filter on a unique `id`, or it can overwrite the whole row.
- **Node names must match exactly, including invisible spaces.** An extra trailing space in a node's name made `$('Find month payment')` fail with "Referenced node doesn't exist". Picking the name from the autocomplete list, instead of typing it, avoids the mismatch.
- **A node runs once for every item it receives.** To get one item per member, the earlier lookups need Execute Once, or the counts multiply.
- **Upsert makes a repeated job safe.** Matching on member and month updates a row if it exists and creates it if it doesn't, so running the statements twice leaves one row per member per month.