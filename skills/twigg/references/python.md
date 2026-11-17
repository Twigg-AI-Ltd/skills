# Python: `twigg`

```bash
pip install twigg
```

Python 3.10+, sync (`Twigg`) and async (`AsyncTwigg`) with the same methods,
built on `httpx` and `pydantic`. Reads `TWIGG_API_KEY`. Full README:
https://github.com/Twigg-AI-Ltd/sdk/blob/main/python/README.md

Model names below are examples. Take real ones from `twigg.models.list()` or
https://api.twigg.ai/v1/catalogue/models.md.

## Smoke test

```python
from twigg import Twigg

twigg = Twigg(model="claude-opus-5")  # default model for every turn
chat = twigg.chats.open(namespace="dev/smoke", title="Smoke test")
reply = chat.send("Say hello in five words.")
print(reply.text, reply.cost)
```

## Quick reference

```python
from twigg import Twigg, tool

twigg = Twigg(model="gpt-6-luna")
models = twigg.models.list()

chat = twigg.chats.open(namespace="acme/user-99")  # new chat
again = twigg.chat(chat_id)  # existing chat, no request
by_phone = twigg.chat_by_external_id("whatsapp:<hash>", chat={"namespace": "support"})

reply = chat.send("Hi")  # .text .cost .usage .tool_calls .stop_reason .run_id
with chat.stream("Hi") as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    result = stream.final()
outcome = chat.run_tools("Hi", tools=[my_tool]).final()

chats = list(twigg.chats.list(namespace="acme"))  # newest first, all pages
transcript = list(chat.history(after_ordinal=-1))  # whole transcript, oldest first

acme = twigg.namespace("acme")
acme.config.instructions.publish(mode="append", body="Answer briefly.")
run = twigg.runs.wait_for_settlement(reply.run_id)  # final cost
```

Async:

```python
from twigg import AsyncTwigg

async with AsyncTwigg() as twigg:
    chat = await twigg.chats.open(namespace="acme/user-99")
    reply = await chat.send("Hi")
```

## Tools

```python
from twigg import tool


@tool
def get_order(order_id: str) -> dict:
    """Look up an order by number."""
    return db.orders.find(order_id)


outcome = chat.run_tools("Where is order 1042?", tools=[get_order]).final()
print(outcome.stop_reason, outcome.result.text)
```

- The schema comes from type hints and the docstring, or from a single pydantic
  model parameter.
- Input is validated first; invalid input and raised exceptions go back to the
  model as error results (`on_tool_error="throw"` raises instead).
- Return a string, any JSON value, or `ToolResult` for media, an error flag or
  a trust level. Annotate a parameter `ToolContext` to see the call.
- `handlers={"name": fn}` answers tools published to the namespace config.
- `idempotency_key="job-7"` lets a restarted loop recover finished turns.
- On `AsyncTwigg`, a turn's tools run concurrently (`def` tools in threads).

## Streaming in a web server (FastAPI)

```python
from fastapi.responses import StreamingResponse


@app.post("/api/chat")
async def chat_route(body: ChatIn, user=Depends(current_user)):
    # check user may use body.chat_id before this line
    chat = twigg.chat(body.chat_id)

    async def gen():
        async with chat.stream(body.text) as stream:
            async for text in stream.text_stream:
                yield text

    return StreamingResponse(gen(), media_type="text/plain")
```

(`twigg` here is an `AsyncTwigg`.) Forward only text to the browser; keep cost
and config warnings on the server.

## Stopping and dropped connections

- `stream.abort()` stops the turn on Twigg.
- `stream.close()` stops reading but the turn finishes and lands in history.
  Safe to call from another thread or task.
- If the connection drops, `final()` rebuilds the result from the run and
  history (`result.recovered` is true).

## Chats by your own id

```python
chat = twigg.chat_by_external_id(f"whatsapp:{hash_of(sender)}", chat={"namespace": "support"})
reply = chat.send(text, idempotency_key=message_id)
```

Import a prior conversation into an empty chat with `chat.import_history(parts)`.

## Managing chats

```python
chat.update(title="Refund", user_metadata={"tier": "gold", "trial": None})
usage = chat.get(include="usage").usage
for run in chat.runs():
    print(run.model, run.status, run.cost)

job = twigg.namespace("swarm").child("run-abc")
job.chats.delete_all()  # irreversible
job.config.delete()
```

## Attachments

```python
from twigg import media

chat.send(["What is wrong with this chart?", media.from_file("chart.png")])
```

`media.check(model, items)` lists what a model would reject.

## Errors

Every failure is a `TwiggError` (`status`, `code`, `message`, `hint`,
`details`). Catch by kind: `NotFoundError`, `InsufficientBalanceError`,
`ValidationError`, `ConflictError`, `IdempotencyConflictError`, `StreamError`,
`APIConnectionError`, `APITimeoutError`. `chat_busy` is a `ConflictError`;
wait `details["retry_after_secs"]`. Safe retries are built in.
