# Tips for building on Twigg

Twigg stores the conversation and builds every model call, so most of what you
would normally write (a messages table, history trimming, provider adapters,
token accounting) should not be written at all. Before adding any of those,
check whether Twigg already does it. The live version of these tips is at
https://twigg.ai/tips.txt.

## Structure

1. **Use namespaces to structure chats.** A namespace is a path you choose, such
   as `platform/org/user/project`. Build it from the app's own ids so it mirrors
   how the app is organised. Prefer short hashes of those ids over raw values:
   namespaces show up in URLs and logs. Segments are lowercase `a-z 0-9 _ -`,
   at most 64 bytes each and 255 for the whole path, so a truncated lowercase
   hex hash fits well. A namespace is fixed when the chat is created.

2. **Do not store chat ids** if the app's database already models the structure
   the namespace mirrors. List a user's or project's chats with
   `GET /api/v1/chats?namespace=<prefix>`; the filter covers the namespace and
   everything beneath it, so `org/user` returns every project's chats for that
   user, newest first.

3. **Address chats by your own id.** If a conversation already has an id (a
   ticket, a phone number, a thread id), set it as the chat's `external_id` and
   use the `/api/v1/chats/external/{external_id}/...` routes.
   `POST .../external/{external_id}/responses` creates the chat on first use
   (describe it in the optional `chat` object), so there is no create step to
   write. Prefix ids per channel (`whatsapp:`, `sms:`). Percent-encode them in
   the path. Hash them if they identify a person.

4. **Use `user_metadata` instead of a side table.** Each chat carries up to 1 KB
   of your JSON (a ticket ref, a UI flag, a status). Set it on create, merge into
   it with PATCH (a key sent as `null` is removed), read it back from list and
   get. Turns can carry their own `user_metadata` too. For data attached to one
   message, such as your UI's message id, send `client_data` on the part: it is
   stored encrypted, never sent to a model, and returned from history.

5. **Use titles.** Set `title` on create or PATCH it later; list responses return
   it, so a chat sidebar needs nothing but the list call.

6. **Namespaces are not a security boundary.** The API key reaches every chat in
   the organisation. Check in your own backend that the caller may access a
   namespace or chat before calling Twigg, and never ship the key to a browser or
   mobile client. To stream to a browser, forward the stream from your server
   (the TypeScript SDK's `toResponse()` does this and strips cost and config
   warnings).

## Configuration

7. **Put the system prompt and tools in configuration, not code.** Publish them
   per namespace and every turn under that namespace gets them without sending
   anything. Configuration at `platform` applies to everything beneath it;
   instructions and tools combine down the path with `append` or `replace`. Use
   the request's `system` and `tools` only for what varies per request (a
   request `system` is appended, not stored, so send it every turn).

8. **Keep configuration in the repo.** `@twigg/cli` reads `twigg.config.yaml`
   (or `.json`, `.ts`); run `npx twigg config plan` then `apply`, from CI if you
   like. `npx twigg config pull` writes what is configured today.

9. **A typo in a namespace is not an error;** it silently resolves to the
   organisation defaults. Build namespaces from one helper function, and check
   what a chat will receive with `GET /api/v1/config/resolved?namespace=...`
   (or `npx twigg config resolve --scope ...`).

10. **Do not write history trimming.** Context budgets and retention policies
    decide what is replayed to the model; compaction summarises old history when
    it no longer fits. Defaults are sensible. Reach for retention only when large
    tool results or attachments are replayed long after they matter.

## Turns

11. **Send only what just happened:** one `prompt`, or the `tool_result`s
    answering the model's tool calls. Never the transcript.

12. **Pick the model per turn.** `model` is a field on each request, not a
    property of the chat, so switching provider mid-conversation is a one-field
    change. Use the catalogue `name`, never the provider's model string, and
    never copy `done.model_served` into a request. A router such as
    `telluvian/auto` picks a model per prompt; routers are listed in the
    catalogue.

13. **Your code runs the tools.** When `done.pending_tool_calls` is non-empty,
    run them and send the results as the next turn's input. The SDKs run this
    loop for you. Any open call left unanswered is closed on the next
    submission as abandoned.

14. **No streaming needed? Send `"stream": false`** and get one JSON object
    with the text, pending tool calls, usage and cost. Much less code than
    parsing server-sent events, and the right choice for webhooks and bots.

15. **Send an `idempotency_key` with every turn,** derived from the incoming
    message (its id in your system or the platform it came from). A retry with
    the same key never runs the turn twice; with `"stream": false` it returns
    the original result marked `replayed: true`. Never generate a fresh key to
    retry an uncertain submission. Keys allow letters, digits and `-._~` only.

16. **A dropped connection cancels nothing.** The turn finishes server-side and
    is stored and billed. Recover with `GET /api/v1/runs/{run_id}` or read
    `/history` instead of resending. To stop a turn on purpose, call `/abort`
    with the `run_id` from the `run` event.

17. **One turn at a time per chat.** A second submission while one runs gets
    `409 chat_busy` with `Retry-After`; wait and retry rather than writing your
    own lock.

18. **Tools that run elsewhere** (a workflow, a queue) can be overtaken by a new
    user message, which closes the open calls as abandoned. Set
    `wait_for_pending_tool_calls` (seconds, 1 to 600) to get
    `409 tool_calls_pending` instead while a recent call is unanswered.

19. **A turn failed?** Retry it with `retry_of: <run_id>`, `input: []` and a new
    idempotency key, rather than appending the prompt again.

## Reading and cleanup

20. **Render a chat UI from `GET /api/v1/chats/{chat_id}/history`.** It returns
    everything stored, regardless of what retention replayed to the model.
    Without a cursor it opens at the newest end; page forward with
    `after_ordinal` to pick up only what is new. Page with the ordinals you were
    given, never by arithmetic: deletes leave gaps.

21. **Moving from another system?** `POST .../external/{external_id}/import`
    loads an existing conversation (up to 500 parts) into an empty chat. It
    repairs broken tool-call pairs and is safe to retry: a chat that already has
    history answers `skipped`.

22. **Delete by structure.** `DELETE /api/v1/chats?namespace_prefix=org/user`
    removes every chat for that user: account deletion in one call. Pair it
    with `DELETE /api/v1/config?namespace_prefix=...` to drop that subtree's
    configuration. Both are irreversible; confirm with the user first.

23. **Usage and cost are already tracked.**
    `GET /api/v1/usage?range=7d&namespace=...` reports per namespace
    (descendants included, `group_by=namespace` for a breakdown; it lags by a
    few minutes), and each `done` event
    carries `usage` and `cost`. If the app bills its own users, the namespace
    structure from tip 1 is the billing key.

## Billing facts worth knowing

- Twigg is prepaid in USD. A turn the balance cannot cover returns `402`.
- `cost` is a decimal string; `null` in `done` means settlement is pending.
  `twigg.runs.waitForSettlement(runId)` / `wait_for_settlement` gives the final
  figure.
- Compaction is its own model call and its own billed line.
- An organisation can bring its own provider keys in the console; Twigg then
  charges only its margin on those models.
