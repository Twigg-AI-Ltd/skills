# Guiding someone through n8n

The person is usually not an engineer. They run an AI chatbot in n8n (WhatsApp,
Telegram, website chat, support, lead qualification) and want it to remember
conversations without a database. The full step-by-step guide, with exact button
names, is at https://twigg.ai/docs/n8n.md. Fetch it before guiding them and
follow it.

## How to guide

- One step at a time. Give one instruction, wait for them to confirm, then the next.
- Use exact button and field names in bold; they will look for those words.
- Ask first: which trigger the bot uses, n8n Cloud or self-hosted, and whether
  conversations are already stored somewhere (they can be imported).
- Never ask them to paste the API key into the chat. It goes only into n8n's
  credential form.
- Keep explanations short. They want the bot working, not the architecture.

## The outline

1. Create a Twigg account at https://twigg.ai/?utm_source=agent-skill&utm_medium=skill&utm_content=n8n and add credit.
2. Create an API key at https://twigg.ai/dashboard/api-keys?utm_source=agent-skill&utm_medium=skill&utm_content=n8n.
3. Install the **Twigg** node: it is a verified community node, under
   **More from the community** in the nodes panel (owner or admin only). On
   self-hosted n8n without verified nodes: **Settings → Community nodes →
   Install**, package `@twigg/n8n-nodes-twigg`, n8n 2.16 or later.
4. Replace the AI Agent, Chat Model and Memory nodes with one Twigg node
   (**Agent** resource).
5. Set the **Chat ID** to the conversation's id from the trigger (the sender's
   number, the Telegram chat id). Twigg uses it as the external id.
6. Test with two messages that depend on each other ("my name is Ann", then
   "what is my name?").

Storing conversations is free; the model's replies are paid from a prepaid
balance.

If something fails, use the **Troubleshooting** section of the live guide
before guessing.
