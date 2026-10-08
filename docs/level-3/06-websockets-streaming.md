---
description: "Real-time FastAPI: authenticated WebSocket endpoints with a broadcast room, JSON Lines from yield endpoints, and Server-Sent Events."
---

# 06 · WebSockets & Streaming Responses

A normal endpoint computes a whole response and sends it. Some features don't fit that
shape: a chat, live progress for a long job, a feed of price updates. FastAPI offers two
families of tools:

- **Streaming responses** — one HTTP request, a response that arrives in pieces over
  time (JSON Lines, Server-Sent Events). Server → client only, plain HTTP, works through
  most proxies.
- **WebSockets** — an upgraded connection where both sides can send messages at any
  time. Needed when the client also talks continuously.

Prefer streaming when data flows one way; it's simpler to deploy, cache-friendly to
reason about, and reconnects automatically in the browser's `EventSource`.

## Streaming JSON Lines from a `yield` endpoint

On the version used here (FastAPI 0.143.0), an endpoint can simply be an async generator
with a declared item type:

```python
import asyncio
from collections.abc import AsyncIterable
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Progress(BaseModel):
    step: int
    of: int

@app.get("/jobs/42/lines")
async def lines() -> AsyncIterable[Progress]:
    for i in range(1, 4):
        await asyncio.sleep(0.2)
        yield Progress(step=i, of=3)
```

Under Uvicorn, with `curl -N` (no buffering):

```text
HTTP/1.1 200 OK
server: uvicorn
content-type: application/jsonl
transfer-encoding: chunked

{"step":1,"of":3}
{"step":2,"of":3}
{"step":3,"of":3}
```

Each item was validated against `Progress`, serialized, and sent as one line as soon as
it was yielded. `transfer-encoding: chunked` means there's no `Content-Length` — the
server doesn't know the total size in advance.

!!! note "Version-specific"
    Yield-based endpoints and `fastapi.sse` are recent additions to FastAPI. On older
    versions you return a `StreamingResponse` wrapping a generator yourself, and use a
    third-party package (such as `sse-starlette`) for SSE. Check
    `python -c "import fastapi.sse"` to see whether your version has it.

## Server-Sent Events

SSE is a standard text format for event streams that browsers consume with the built-in
`EventSource` API:

```python
from fastapi.sse import EventSourceResponse, ServerSentEvent

@app.get("/jobs/42/events", response_class=EventSourceResponse)
async def events() -> AsyncIterable[ServerSentEvent]:
    for i in range(1, 4):
        await asyncio.sleep(0.2)
        yield ServerSentEvent(data={"step": i, "of": 3}, event="progress", id=str(i))
    yield ServerSentEvent(data={"done": True}, event="done")
```

```text
HTTP/1.1 200 OK
server: uvicorn
content-type: text/event-stream; charset=utf-8
cache-control: no-cache
x-accel-buffering: no
transfer-encoding: chunked

event: progress
data: {"step": 1, "of": 3}
id: 1

event: progress
data: {"step": 2, "of": 3}
id: 2
...
event: done
data: {"done": true}
```

Each event is a block of `field: value` lines ended by a blank line. Two headers were
added for you: `cache-control: no-cache`, and `x-accel-buffering: no`, which tells nginx
not to buffer the stream (a buffering proxy would hold events back until the response
ended, defeating the point). Timing confirmed the stream: the first bytes arrived after
**0.0015 s**, the whole response after **0.61 s** — three 0.2 s sleeps.

In a browser:

```javascript
const es = new EventSource("/jobs/42/events");
es.addEventListener("progress", (e) => console.log(JSON.parse(e.data)));
es.addEventListener("done", () => es.close());
```

If the connection drops, `EventSource` reconnects by itself and sends a
`Last-Event-ID` header with the last `id` it saw, so the server can resume.
(`EventSource` reconnects after a `done` too, unless you call `close()`.)

## WebSockets

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws/echo")
async def echo(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            text = await ws.receive_text()
            await ws.send_text(f"echo: {text}")
    except WebSocketDisconnect as e:
        print(f"echo client left, code={e.code}")
```

The test client can drive WebSockets too:

```python
with client.websocket_connect("/ws/echo") as ws:
    ws.send_text("hello")
    print(ws.receive_text())
```

```text
echo: hello
echo client left, code=1000
```

Code 1000 is a normal close (the `with` block ending). Every WebSocket endpoint needs
that `try/except WebSocketDisconnect`: a client vanishing is the normal end of a
WebSocket's life, not an error.

## Worked example: an authenticated chat room

Browsers can't set an `Authorization` header on a WebSocket, so tokens usually travel in
the query string (or a cookie, or the first message). Here, a query token, and a room
that broadcasts to everyone connected:

```python
from typing import Annotated
from fastapi import Query, WebSocketException, status

TOKENS = {"tok-ada": "ada", "tok-bob": "bob"}

class Room:
    def __init__(self):
        self.members: dict[str, WebSocket] = {}

    async def join(self, user: str, ws: WebSocket):
        await ws.accept()
        self.members[user] = ws
        await self.broadcast({"event": "joined", "user": user})

    def leave(self, user: str):
        self.members.pop(user, None)

    async def broadcast(self, msg: dict):
        for name, ws in list(self.members.items()):
            try:
                await ws.send_json(msg)
            except Exception:
                self.leave(name)

room = Room()

@app.websocket("/ws/chat")
async def chat(ws: WebSocket, token: Annotated[str | None, Query()] = None):
    user = TOKENS.get(token or "")
    if user is None:
        raise WebSocketException(code=status.WS_1008_POLICY_VIOLATION, reason="invalid token")
    await room.join(user, ws)
    try:
        while True:
            data = await ws.receive_json()
            text = str(data.get("text", ""))[:500]
            await room.broadcast({"event": "message", "user": user, "text": text})
    except WebSocketDisconnect:
        room.leave(user)
        await room.broadcast({"event": "left", "user": user})
```

Two clients through the test client:

```text
bad token -> WebSocketDisconnect 1008 invalid token
ada got: {'event': 'joined', 'user': 'ada'}
ada got: {'event': 'joined', 'user': 'bob'}
bob got: {'event': 'joined', 'user': 'bob'}
ada got: {'event': 'message', 'user': 'bob', 'text': 'hi ada'}
bob got: {'event': 'message', 'user': 'bob', 'text': 'hi ada'}
ada got: {'event': 'left', 'user': 'bob'}
```

And against a real Uvicorn server with the `websockets` client library:

```text
ada: {"event":"joined","user":"ada"}
ada: {"event":"message","user":"ada","text":"anyone here?"}
bad token: InvalidStatus server rejected WebSocket connection: HTTP 403
```

with these server log lines:

```text
INFO:     127.0.0.1:52265 - "WebSocket /ws/chat?token=tok-ada" [accepted]
INFO:     127.0.0.1:52266 - "WebSocket /ws/chat?token=nope" 403
INFO:     connection rejected (403 Forbidden)
```

The test client reported the rejection as close code 1008, while the real client saw an
**HTTP 403**: the exception was raised *before* `accept()`, so the handshake itself was
refused and no WebSocket ever existed. That's the right place to reject — but test the
real behaviour with a real client at least once.

Note `[:500]` on the text and `list(...)` around the members while broadcasting: input
limits and safe iteration matter even more for long-lived connections.

## How It Actually Works

**Streaming**: FastAPI wraps your generator in a streaming response. Starlette sends
`http.response.start` (status and headers) immediately, then one `http.response.body`
message with `more_body=True` per chunk, and a final empty one. Uvicorn writes each as
an HTTP/1.1 chunk. Because headers go first, the status code is fixed before your first
`yield` — an exception halfway through can't turn into a 500; the stream just breaks.
Send errors as an event in the stream when the client needs to know.

**WebSockets**: the client sends an HTTP `GET` with `Upgrade: websocket`. Uvicorn turns
that into an ASGI scope of type `"websocket"` and delivers a `websocket.connect` event.
Your endpoint's `accept()` sends `websocket.accept`, and the server completes the
handshake with `101 Switching Protocols`. If the app closes or raises before accepting,
Uvicorn answers the handshake with HTTP 403 instead. After that, `receive_*` and
`send_*` exchange `websocket.receive`/`websocket.send` events, and a client close
arrives as `websocket.disconnect`, which Starlette raises as `WebSocketDisconnect`.

The `Room` here lives in one process's memory. With several workers or servers, Ada and
Bob may be connected to different processes that can't see each other's `members`. Real
deployments put a pub/sub layer (Redis, NATS, PostgreSQL `LISTEN/NOTIFY`) between
processes: each process subscribes, and broadcasts go through the broker.

## Common mistakes

- **No `WebSocketDisconnect` handling**, leaving dead connections in your registry.
- **Blocking calls inside a WebSocket loop.** One `time.sleep` stalls every connection
  in the process (Level 2 lesson 3).
- **Authenticating after `accept()`**, or not at all. Reject during the handshake.
- **In-memory rooms with multiple workers.**
- **Proxies buffering streams** — configure them (`x-accel-buffering`, or
  `proxy_buffering off` for nginx), and set proxy timeouts long enough for idle
  WebSockets or send periodic pings.
- **Using WebSockets for one-way updates** that SSE would handle with less code.
- **Unbounded message sizes** from clients.

## Exercise

1. Add a `GET /jobs/{id}/events` stream that sends a `heartbeat` comment every 15 seconds
   while waiting for progress, and ends with a `done` event.
2. Support `Last-Event-ID`: read the header and skip events the client has already seen.
3. Extend the chat room with named rooms (`/ws/rooms/{name}`), per-room membership and a
   limit of 50 members per room.
4. Write a test using the test client that connects three users, disconnects one
   abruptly, and asserts the others receive a `left` event.
