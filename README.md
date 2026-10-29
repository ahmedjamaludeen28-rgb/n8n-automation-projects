## Customer Support Bot (n8n)

A customer support chatbot for a fictional gadget shop, Nabil gadgets. It answers questions from the shop information and hands anything it can't answer to a human.

## How it works

1. Trigger: the chat trigger node starts the workflow whenever a customer sends a message.
2. AI Agent: reads the message and replies using the shop information in it's system message. It uses the gemini model to generate answers.
3. Memory: the simple memory nodes lets the bot remember the last 15 messages, so follow-up questions like "how long will that take?" make sense.
4. Escalation rule: if the bot can't find the answer related to the shop in the shop info, it replies with a single word ESCALATE
5. IF Check: an IF node checks whether the reply contains ESCALATE
- True: the customer gets a handoff message and 'escalated' is set to True
- False: the bot's reply goes to the customer and 'escalated' is set to False
6. Logging: every conversation is saved to a data table called support logs

## Data table: `support logs`

| column | what it stores |
|---|---|
| time | When the chat happened |
| customer_message | What the customer typed |
| bot_reply | What the bot answered |
| escalated | Whether it was handed to a human |

## How to run it

1. Import support-bot.json into n8n (Workflows → Import from file).
2. Create your own Gemini credential (the bot's workflow doesn't include any keys).
3. Create the data table support logs with the four columns above.
4. Open the chat, and send "hello".

## What I learned

- Network errors don't all mean the same thing.
 `fetch failed` meant n8n never reached the API at all, and `Request timed out` meant the request went out but no reply came back in time. Both pointed to a weak connection, not a broken workflow, so now I test the connection first, then raise the model node's timeout and retries.

- An error that comes back from a server means my connection works. 
 `insufficient balance` and "too many requests" were account limits, not network or workflow faults. Now I check which kind of error it is before changing anything, and I use a free-tier model with retries turned on while testing.


 ## Project 2: Savings & Loan Automation (demo)

A demo of how a cooperative's savings and loan scheme can be automated in n8n. It uses invented rules and fake data (about 100 imagined members), and it's meant to show the approach, not to run a real organization.

### The rules (stored in a `settings` table, never hardcoded)

| rule | value |
|---|---|
| Monthly contribution | ₦50,000 |
| Minimum saving period before borrowing | 90 days |
| Maximum loan | ₦150,000 |
| Interest | 5% flat, charged once |
| Repayment period | 60 days |
| Late penalty | 8%, applied once |

### Data tables

| table | what it stores |
|---|---|
| `members` | member_id, names, phone, first_contribution_date, status |
| `loans` | member_id, amount, interest, total_due, loan_date, due_date, status, penalty_applied, amount_paid |
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

### How the three workflows connect

The workflows never call each other. They share state through the data tables:

| workflow | started by | effect on `loans` |
|---|---|---|
| loan request | a member's form | creates a loan |
| penalty check | the clock, daily | adds the penalty once, sets the flag |
| repay loan | a member's form | closes the loan |

### How to run it

1. Import the three JSON files into n8n (Workflows → Import from file).
2. Create the data tables with the columns above.
3. Add one row to `settings` with the rule values.
4. Add a few fake members, some with a first contribution date over 90 days ago and some recent.
5. Publish `penalty-check` if you want it to run daily on its own.

### Known limitations

- A rejection message names only the first rule that failed.
- Repayment is full payment only, with no installments.
- The `contributions` table exists but nothing records into it yet.
- Everything here is demo data with invented rules, not a production system.

### What I learned

- **A field can't be used in the node that creates it.** Calculations that depend on each other need two nodes, or the result comes out as NaN or undefined.
- **A true/false value saved as text is dangerous.** The text "false" counts as true, so yes/no fields must be set to Boolean.
- **A flag makes a scheduled job safe to repeat.** `penalty_applied` stops a daily job (and the repay workflow) from charging the same loan twice.
- **An Update row should list only the columns it changes,** and filter on a unique `id`, or it can overwrite the whole row.
