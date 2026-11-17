# Plain HTTP

Base URL `https://api.twigg.ai`. Every customer endpoint is under `/api/v1` and
takes `Authorization: Bearer $TWIGG_API_KEY`. Bodies are JSON. The OpenAPI 3.1
spec is at https://api.twigg.ai/openapi.json; prefer generating a client from it
over hand-writing types. Full reference: https://twigg.ai/docs/api.md

Take `model` values from https://api.twigg.ai/v1/catalogue/models.md (no key)
or `GET /api/v1/models`. The names here are examples.

## Smoke test

```bash
curl -s https://api.twigg.ai/api/v1/chats/external/dev-smoke/responses \
  -H "Authorization: Bearer $TWIGG_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"claude-opus-5","stream":false,"idempotency_key":"smoke-1",
       "chat":{"namespace":"dev/smoke"},
       "input":[{"type":"prompt","text":"Say hello in five words."}]}'
```

## Create a chat

`POST /api/v1/chats`

```json
{ "namespace": "acme/proj-7", "title": "Support thread" }
```

Returns `201` with `id`, `namespace`, `title`, `external_id`, `user_metadata`,
`created_at`. Optional `external_id`, `description`, `user_metadata`.

## Send a turn

`POST /api/v1/chats/{chat_id}/responses`

```json
{
  "model": "claude-opus-5",
  "input": [{ "type": "prompt", "text": "What changed in the billing module?" }],
  "idempotency_key": "msg-8812"
}
```

Optional: `stream` (default `true`), `system` (appended to configured
instructions, not stored), `tools` (request-level, JSON Schema in
`input_schema`), `reasoning_effort`, `wait_for_pending_tool_calls`, `retry_of`,
`user_metadata`.

Answer a tool call with the same endpoint:

```json
{
  "model": "claude-opus-5",
  "input": [{
    "type": "tool_result",
    "tool_use_id": "toolu_01ORDER",
    "tool_name": "get_order_status",
    "text": "{\"status\":\"shipped\"}",
    "is_error": false,
    "trust": "customer_data"
  }],
  "idempotency_key": "msg-8812-tool-1"
}
```

`trust` is `trusted`, `customer_data` (default) or `untrusted`.

Attachments go on a prompt part:
`{"type":"prompt","text":"Describe this","media":[{"mime":"image/png","data_base64":"...","filename":"a.png"}]}`
(standard base64, no `data:` prefix). Check the model's `media` limits first.

## One call per message (bots, automation)

`POST /api/v1/chats/external/{external_id}/responses` creates the chat on first
use. Percent-encode the id (`+44...` is `%2B44...`). `chat` describes the chat
to create and is ignored afterwards.

```json
{
  "model": "claude-opus-5",
  "stream": false,
  "idempotency_key": "wa-msg-ABC123",
  "chat": { "namespace": "acme/support-bot" },
  "system": "The customer is on WhatsApp. Keep replies short.",
  "wait_for_pending_tool_calls": 60,
  "input": [{ "type": "prompt", "text": "Where is my order 1042?" }]
}
```

Non-streaming response: `run_id`, `chat_id`, ordered `content`, `text`,
`pending_tool_calls` (with `tool_use_id`, `name`, `arguments`), `stop_reason`,
`usage`, `cost`, and `replayed: true` when an idempotent retry returned an
earlier result.

External ids: unique per organisation, case-sensitive, printable ASCII without
spaces, up to 255 characters, no `% / \ ? #`, not starting with `.`.

## Reading the stream

`text/event-stream`; each event is `event: <name>` then `data: <json>`.

| Event | Meaning |
|---|---|
| `run` | Always first: `run_id`, `chat_id`, `closed_tool_calls`. Keep `run_id`. |
| `config_warnings` | Configuration warnings; may repeat after compaction. |
| `translation_warnings` | Model compatibility warnings before content. |
| `compacting` | Compaction the turn is waiting for. |
| `block_start` / `delta` / `block_stop` | Content blocks: text, reasoning, tool calls. |
| `done` | Terminal success: `stop_reason`, `pending_tool_calls`, `usage`, `cost` (decimal string; `null` = settlement pending), `model_served`, `compaction`. |
| `error` | Terminal failure after the stream opened: `code`, `message`, `details`, `hint`. |

Ignore event names you do not know; new ones may be added.

## Other calls

| Call | Use |
|---|---|
| `POST /api/v1/chats/{id}/abort?run_id=...` | Stop a running turn. |
| `GET /api/v1/chats/{id}/history?after_ordinal=40&include=stubs` | Stored transcript, oldest first in a page. `include` is `stubs`, `full` or `none`. |
| `GET /api/v1/runs/{run_id}` | A turn's status, input, usage, cost and manifest. |
| `GET /api/v1/chats?namespace=acme&limit=20` | List chats, newest first, namespace and beneath. |
| `PATCH /api/v1/chats/{id}` | Title, description, `user_metadata` (merge). |
| `GET /api/v1/namespaces` | Namespaces in use. |
| `GET /api/v1/usage?namespace=...&group_by=namespace` | Usage and cost per namespace. |
| `DELETE /api/v1/chats?namespace_prefix=...` | Delete a subtree's chats. Irreversible. |
| `POST /api/v1/chats/external/{id}/import` | Load up to 500 parts into an empty chat. |
| `GET /api/v1/config/resolved?namespace=...` | What a chat in that namespace receives. |

The same external-id routes exist for history, abort, get, patch and delete
under `/api/v1/chats/external/{external_id}`.

## Errors

Before the first byte, an HTTP status with
`{ "error": { "code": "...", "message": "..." } }`. After it, the same fields
arrive as the `error` event. See [errors.md](errors.md).
