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