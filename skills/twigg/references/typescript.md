# TypeScript: `@twigg/sdk`

```bash
npm install @twigg/sdk
```

No runtime dependencies. Node 22+, Deno, Bun, edge runtimes. Reads
`TWIGG_API_KEY`. Full README:
https://github.com/Twigg-AI-Ltd/sdk/blob/main/typescript/packages/sdk/README.md

Model names below are examples. Take real ones from
`await twigg.models.list()` or https://api.twigg.ai/v1/catalogue/models.md.

## Smoke test

```ts
import { Twigg } from "@twigg/sdk";

const twigg = new Twigg({ model: "claude-opus-5" }); // default model for every turn
const chat = await twigg.chats.open({ namespace: "dev/smoke", title: "Smoke test" });
const reply = await chat.send("Say hello in five words.");
console.log(reply.text, reply.cost);
```

## Quick reference

```ts
import { Twigg, tool } from "@twigg/sdk";

const twigg = new Twigg({ model: "gpt-6-luna" });
const models = await twigg.models.list();

const chat = await twigg.chats.open({ namespace: "acme/user-99" }); // new chat
const again = twigg.chat(chatId);                                   // existing chat, no request
const byPhone = twigg.chatByExternalId("whatsapp:<hash>", { chat: { namespace: "support" } });

const reply = await chat.send("Hi");             // .text .cost .usage .toolCalls .stopReason .runId
const streamed = await chat.stream("Hi").on("text", (d) => process.stdout.write(d)).final();
const outcome = await chat.runTools("Hi", { tools: [myTool] }).final();

for await (const c of twigg.chats.list({ namespace: "acme" })) {}  // newest first, all pages
for await (const p of chat.history({ after_ordinal: -1 })) {}      // whole transcript, oldest first

const acme = twigg.namespace("acme");
await acme.config.instructions.publish({ mode: "append", body: "Answer briefly." });
const run = await twigg.runs.waitForSettlement(reply.runId);        // final cost
```

Per-call overrides: `chat.send("Hi", { model: "...", system: "..." })`. A
`system` here is appended after configured instructions and not stored.

## Tools

```ts
import { tool } from "@twigg/sdk";
import { z } from "zod";

const getOrder = tool({
  name: "get_order",
  description: "Look up an order by number",
  input: z.object({ orderId: z.string() }),
  run: async ({ orderId }) => db.orders.find(orderId), // string or any JSON value
});

const outcome = await chat.runTools("Where is order 1042?", { tools: [getOrder] }).final();
console.log(outcome.result.text, outcome.turns.length, outcome.stopReason);
```

- `input` is any Standard Schema that can emit JSON Schema (Zod 4.2+), or pass
  plain JSON Schema as `inputSchema`.
- Arguments are validated before `run`. Errors go back to the model as
  `is_error` results; `onToolError: "throw"` stops instead.
- Up to 8 tools run at once per turn.
- `handlers: { name: fn }` answers tools published to the namespace config.
- `stopReason` `unhandled_tool_calls`: the model called a tool with no handler;
  the calls are on `outcome.result.pendingToolCalls`.
- `idempotencyKey: "job-7"` lets a restarted loop recover finished turns.

## Streaming to a browser

Server route (Next.js style):

```ts
export async function POST(request: Request) {
  const { chatId, text } = await request.json();
  // check the signed-in user may use chatId before this line
  return twigg.chat(chatId).stream(text).toResponse();
}
```

Browser, no API key:

```ts
import { readForwardedStream } from "@twigg/sdk/browser";

const turn = readForwardedStream(await fetch("/api/chat", { method: "POST", body }));
turn.on("text", (_delta, all) => render(all));
const result = await turn.final();
```

`toReadableStream()` gives the bare stream for Hono or Express
(`Readable.fromWeb`). Only text, reasoning, tool-call blocks and `done` without
cost are forwarded; pass `toResponse({}, { events: "all" })` to forward
everything.

## Stopping and dropped connections

- `stream.abort()` stops the turn on Twigg; it ends with `stop_reason: "aborted"`.
- `stream.close()` stops reading but lets the turn finish and land in history.
- If the connection drops, `final()` waits for the run and rebuilds the result
  from history (`recovered: true`). Disable with `{ recover: false }`.

## Chats by your own id

```ts
const chat = twigg.chatByExternalId(`whatsapp:${hash(from)}`, { chat: { namespace: "support" } });
const reply = await chat.send(text, { idempotency_key: messageId });
```

Every method on the handle uses the external-id routes. Import a prior
conversation into an empty chat with `chat.importHistory(parts)`.

## Managing chats

```ts
await chat.update({ title: "Refund", user_metadata: { tier: "gold", trial: null } });
const { usage } = await chat.get({ include: "usage" });
for await (const run of chat.runs()) console.log(run.model, run.status, run.cost);

const job = twigg.namespace("swarm").child("run-abc");
await job.chats.deleteAll(); // irreversible
await job.config.delete();
```

## Attachments

```ts
import { media } from "@twigg/sdk";
await chat.send(["What is wrong with this chart?", await media.fromFile("chart.png")]);
```

`media.check(model, items)` lists what a model would reject.

## Errors

Every failure is a `TwiggError` (`status`, `code`, `message`, `hint`,
`details`). Subclasses: `AuthenticationError`, `InsufficientBalanceError`,
`NotFoundError`, `ConflictError`, `IdempotencyConflictError`, `ValidationError`,
`RateLimitError`, `UpstreamError`, `ServiceUnavailableError`, `StreamError`,
`ConnectionError`, `TimeoutError`. `chat_busy` is a `ConflictError`; wait
`details.retry_after_secs`. Safe retries are built in.

## Tests without a network

```ts
import { MockTwigg, streamBuilder } from "@twigg/sdk/testing";

const twigg = new MockTwigg().respond(
  streamBuilder().text("Let me check.").toolCall("get_order", { orderId: "1042" }).done(),
  ({ history }) => streamBuilder().text(`Read ${history.length} parts`).done(),
);
```

Use `MockTwigg` in the project's tests instead of hitting the API or mocking
`fetch` by hand.
