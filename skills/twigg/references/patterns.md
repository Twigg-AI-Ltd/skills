# Patterns

Worked shapes for common integrations. Each lists what to build, what not to
build, and the calls involved. Code is in the client references.

## Web chat app (sidebar of conversations, streaming replies)

- **Namespace:** `app/<user_hash>`, or `app/<org_hash>/<user_hash>` for teams.
- **New conversation:** `chats.open({ namespace, title })`. No local chats table:
  the sidebar is `chats.list({ namespace })`, which returns titles.
- **Sending:** a server route takes `{ chatId, text }`, checks the signed-in user
  owns `chatId` (fetch the chat and compare its namespace with the user's), then
  streams the turn. In TypeScript, return `chat.stream(text).toResponse()` and
  read it in the browser with `readForwardedStream` from `@twigg/sdk/browser`.
  The API key stays on the server.
- **Rendering history:** `chat.history()` on open. Store your UI message id with
  `client_data` if you need to reconcile.
- **Titles:** set one from the first prompt, or ask a cheap model once and PATCH.
- **Do not build:** a messages table, a trimming step, a "load last N messages"
  query, a provider switch statement.

## Messaging bot (WhatsApp, Telegram, SMS, Slack)

- **Address by the platform's id:** `chatByExternalId("whatsapp:" + hash(from), { chat: { namespace: "support" } })`.
  The first message creates the chat; nothing to store.
- **One request per message.** With an SDK, `await chat.send(text, { idempotency_key })`
  and reply with `reply.text`. Over plain HTTP or from a no-code tool, send
  `"stream": false` and reply with the body's `text`.
- **Idempotency key = the platform's message id**, so webhook redeliveries are
  safe. With `"stream": false` a replay returns the original reply.
- **Busy chat:** a second message during a turn gets `409 chat_busy`; wait
  `Retry-After` and resend, or queue per chat.
- **Bot persona and tools:** publish once to the `support` namespace.
- **Separate bots:** different prefixes and namespaces (`support:`, `sales:`).

## Tool-using agent

- Define tools with the SDK's `tool()` / `@tool` and call
  `chat.runTools(prompt, { tools })` / `chat.run_tools(prompt, tools=[...])`. It
  loops until the model stops calling tools; invalid arguments and thrown errors
  go back to the model so it can correct itself.
- For long jobs, pass `idempotencyKey: "job-<id>"` so a restarted process
  recovers turns that already ran instead of re-running them.
- Tools the whole namespace should have: publish them to config and answer them
  with `handlers: { name: fn }`.
- Large tool results replayed many turns later are the main cost driver. Add a
  retention rule that truncates that tool's results past a threshold, not code.
- Mark results from untrusted sources (web pages, user uploads) with
  `trust: "untrusted"`. It sets hygiene limits; it is not an injection defence.

## Multi-tenant SaaS (your customers' users talk to an assistant)

- **Namespace:** `platform/<customer_hash>/<user_hash>[/<project_hash>]`.
- **Per-customer prompt or tools:** publish at `platform/<customer_hash>` with
  `append` on top of the platform-wide config, or `replace` to override.
- **Bill your customers:** `GET /api/v1/usage?namespace=platform/<customer_hash>`,
  or `group_by=namespace` under `platform` for every customer at once.
- **Offboard a customer or user:** `DELETE /api/v1/chats?namespace_prefix=...`
  then `DELETE /api/v1/config?namespace_prefix=...`. Irreversible.
- **Access control is yours.** Twigg sees one organisation; map the caller to
  their namespace in your backend and never take a namespace from the client.

## Batch jobs and multi-agent runs

- One namespace per run: `swarm/<run_id>/<agent_name>`. Each agent is a chat.
- Configuration at `swarm` applies to every run; per-role prompts at
  `swarm/<run_id>/<role>` if needed.
- Cost of a run: `GET /api/v1/usage?namespace=swarm/<run_id>`.
- Clean up afterwards: `namespace("swarm").child(runId).chats.deleteAll()` then
  `.config.delete()`.

## Migrating an existing app

1. Find the code that stores messages, rebuilds the prompt each call, trims or
   summarises history, and branches per provider. That is what Twigg replaces.
2. Choose the namespace and whether to address chats by an existing id
   (`external_id`) so nothing new needs storing.
3. Import existing conversations with
   `POST /api/v1/chats/external/{external_id}/import` (up to 500 parts each). It
   repairs tool-call pairs; retrying is safe.
4. Move the system prompt and tool definitions into `twigg.config.yaml` and apply
   it.
5. Replace each model call with one Twigg turn carrying only the new input.
6. Delete the old history code and table once traffic is on Twigg. Ask the user
   before dropping any data.

## Switching or comparing models

- Change `model` on any turn; the chat continues with full history on the new
  provider. Retained attachments a new model cannot take become text markers
  with a translation warning; stored bytes are untouched.
- To compare models on the same conversation, run the turn on separate chats
  imported from the same history, not on one chat.
- Check `media`, context window and price in the catalogue before switching.
