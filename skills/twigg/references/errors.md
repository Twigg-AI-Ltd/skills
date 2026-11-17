# Errors, retries and recovery

Before the first byte: an HTTP status and
`{ "error": { "code": "...", "message": "...", "details": {...}, "hint": "..." } }`.
After the stream opened: the same fields as the terminal `error` event.
With `"stream": false`, errors are HTTP statuses throughout and failures after
the turn started carry `details.run_id`.

| status | code | what to do |
|---|---|---|
| 400 | `bad_request` | Malformed request, including a bad UUID in a path. Fix the request. |
| 401 | `unauthorized` | Missing or invalid key. An API key never gets 403; a key that cannot reach something is also 401. |
| 402 | `payment_required` | Balance cannot cover the input. The user tops up in the console. |
| 404 | `not_found` | Unknown route, chat, run or model. Check the model name against the catalogue. |
| 409 | `chat_busy` | A turn is already running on the chat. Wait `Retry-After` (`details.retry_after_secs`) and resend with the same key. |
| 409 | `idempotency_key_reused` | The key belongs to another run (`details.run_id`), or to different input. Inspect that run; do not invent a new key for the same message. |
| 409 | `tool_calls_pending` | With `wait_for_pending_tool_calls`, a recent tool call is unanswered. Answer it or retry after `Retry-After`. |
| 409 | `no_active_turn` | Abort with nothing running. Harmless. |
| 409 | `external_id_taken` | Another live chat holds that external id. |
| 409 | `external_id_namespace_mismatch` | Sent a different namespace for an existing external-id chat. Prefix ids per bot, or omit `namespace`. |
| 409 | `provider_key_paused` | The organisation's own provider key is paused and fallback is off. Fix in the console. |
| 413 | `payload_too_large` | Body over the limit; base64 media counts. |
| 422 | `validation_error` | A field failed validation (uppercase namespace, tools a model cannot take, unsupported media). `details.reason` may be `fixed_content_exceeds_budget`, `current_turn_exceeds_capacity` or `checkpoint_exceeds_capacity`: instructions, tools or the current turn are too big for the window. |
| 429 | `rate_limited` | Back off and retry. |
| 500 | `internal_error` | Twigg fault. Retry with the same key. |
| 502 | `external_service_error` | The provider failed. `details.kind` and `provider_request_id` say more. Retry, or try another model. |
| 502 | `provider_key_error` | The organisation's own provider key was refused (`details.kind`, `details.key_last4`). |
| 503 | `unavailable` | Retry. |
| 504 | `response_timeout` | With `"stream": false`, the turn outlasted the request (about 280 s) but keeps running and is billed. Retry with the same key to receive the result. Without a key, do not resend. |

## Recovery rules

- **Uncertain whether a turn was submitted?** Resend with the same
  `idempotency_key`. Never generate a new one: that can run and bill the turn
  twice.
- **Stream dropped after `run`?** The turn continues. Read
  `GET /api/v1/runs/{run_id}` until it settles, then `/history`. The SDKs do
  this inside `final()`.
- **Run failed?** Retry with `input: []`, `retry_of: <run_id>`, a new key, and
  the model, request tools and request `system` again. Discard partial tool
  arguments from the failed stream.
- **Abort** with `POST /api/v1/chats/{id}/abort?run_id=...`. Name the run so a
  late abort cannot stop the next turn.
- Provider rate limits and overloads are retried by Twigg up to three times
  before anything is streamed, so a 502 that reaches you has already been
  retried upstream. A 429 `rate_limited` is Twigg's own limit: back off.

## Debugging checklist

1. Wrong behaviour, prompt or tools? `GET /api/v1/config/resolved?namespace=...`
   for the chat's namespace. A namespace typo falls back to organisation
   defaults without an error.
2. Model forgetting things? Check `config_warnings` and `done.compaction`, and
   the run's manifest via `GET /api/v1/runs/{run_id}`: it shows what was sent.
   Look for a retention rule dropping parts, or a small `max_history_tokens`.
3. Duplicated messages in history? The code is resending the transcript or
   earlier prompts.
4. Tool results ignored? The `tool_use_id` must match the call; an unanswered
   call is closed as abandoned on the next submission (`closed_tool_calls` in
   the `run` event).
5. Costs higher than expected? `GET /api/v1/usage?namespace=...&group_by=namespace`,
   then the runs; large tool results replayed every turn are the usual cause.
