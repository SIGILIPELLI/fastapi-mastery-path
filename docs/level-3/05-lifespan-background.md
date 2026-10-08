---
description: "Manage startup and shutdown with FastAPI's lifespan handler and lifespan state, and use BackgroundTasks correctly: timing, failures and their limits."
---

# 05 · Lifespan Events & Background Tasks

Two things happen outside the request/response cycle in almost every app:

- **Once per process**, at startup and shutdown: open a connection pool, load a model,
  create a shared HTTP client — and close them cleanly.
- **After a response**, small follow-up work the client shouldn't wait for: send a
  confirmation email, write an audit record.

FastAPI handles the first with a **lifespan** handler and the second with
**`BackgroundTasks`**. Both are simple; both have limits worth measuring.

## The lifespan handler

```python
import time
from contextlib import asynccontextmanager
from typing import Annotated
import httpx
from fastapi import Depends, FastAPI, Request

log: list[str] = []

@asynccontextmanager
async def lifespan(app: FastAPI):
    log.append("startup: create shared resources")
    client = httpx.AsyncClient(timeout=5)
    model = {"name": "price-model", "loaded_at": time.time()}
    yield {"http": client, "model": model}         # lifespan state
    log.append("shutdown: close shared resources")
    await client.aclose()

app = FastAPI(lifespan=lifespan)

def get_model(request: Request) -> dict:
    return request.state.model

@app.get("/model")
async def model_info(model: Annotated[dict, Depends(get_model)]):
    return {"model": model["name"]}
```

- Code before `yield` runs **once**, before the server accepts any request.
- The dict you `yield` is **lifespan state**: each key becomes available on
  `request.state` in every request. Wrapping access in a dependency (`get_model`) keeps
  endpoints clean and lets tests override it.
- Code after `yield` runs once at shutdown, after the server stops accepting requests.

Under Uvicorn this is the `Waiting for application startup.` /
`Application startup complete.` pair in the logs. In tests, the lifespan runs only when
`TestClient` is used as a context manager:

```text
before TestClient context: []
inside context: ['startup: create shared resources']
{'model': 'price-model'}
after context: shutdown: close shared resources
```

A test that creates `TestClient(app)` without `with` never runs startup, and any
dependency reading `request.state.model` fails. That's a common source of confusing test
errors.

If startup raises, the app doesn't start. A lifespan that raised
`RuntimeError("database unreachable")` made `with TestClient(app):` raise that same
`RuntimeError`. In production, Uvicorn logs it and exits — which is what you want: a
process that can't reach its database shouldn't accept traffic.

!!! note "`@app.on_event`"
    Older code uses `@app.on_event("startup")` and `@app.on_event("shutdown")`. On the
    version used here, registering one printed
    `DeprecationWarning: on_event is deprecated, use lifespan event handlers instead.`
    The lifespan form keeps setup and teardown in one function, so a resource opened at
    startup is visibly closed at shutdown.

### What belongs in lifespan

| Good fit | Why |
|---|---|
| database engine / connection pool | one per process, reused by every request |
| `httpx.AsyncClient` for calling other APIs | connection pooling, keep-alive |
| ML model or large lookup table | expensive to load, read-only afterwards |
| cache or message-broker client | long-lived connection |

| Bad fit | Why |
|---|---|
| running migrations | every worker would run them; do it in deployment (Level 2 lesson 6) |
| per-user or per-request state | lifespan state is shared by all requests |
| long blocking work in an `async` lifespan | delays startup and blocks the loop; offload or precompute |

Lifespan state is **per process**. With four workers (Level 4 lesson 2) you get four
engines, four clients and four copies of the model — size pools and memory accordingly.

## Background tasks

```python
import asyncio
from fastapi import BackgroundTasks

def send_email(to: str, subject: str):
    time.sleep(0.3)                        # pretend SMTP
    log.append(f"email sent to {to}: {subject}")

async def audit(event: str):
    await asyncio.sleep(0.05)
    log.append(f"audit: {event}")

@app.post("/orders")
async def place_order(background: BackgroundTasks):
    background.add_task(send_email, "ada@example.com", "Order confirmed")
    background.add_task(audit, "order placed")
    log.append("endpoint returning")
    return {"status": "accepted"}
```

Declare a `BackgroundTasks` parameter, add functions with their arguments, return. Tasks
run **after the response has been sent**, in the order added. `def` tasks run in the
thread pool; `async def` tasks run on the event loop.

## Worked example: what the client actually waits for

Under a real Uvicorn server:

```text
$ curl -s -o /dev/null -w "POST /orders: total %{time_total}s\n" -X POST localhost:8710/orders
POST /orders: total 0.013481s

GET /log immediately afterwards:
["startup: create shared resources","endpoint returning"]

GET /log half a second later:
[..., "endpoint returning","email sent to ada@example.com: Order confirmed","audit: order placed"]
```

The client had its answer in 13 ms; the 0.3 s "email" finished afterwards. That's the
point of background tasks.

The **test client** behaves differently. The same request through `TestClient`:

```text
POST /orders 200 {'status': 'accepted'} client saw response after 0.36s
log: ['endpoint returning', 'email sent to ada@example.com: Order confirmed', 'audit: order placed']
```

It waited for the background tasks before returning. That's convenient for asserting
that a task ran, but it means latency tests through the test client include background
work. Measure latency against a real server.

## When a background task fails

```python
def broken_task():
    raise RuntimeError("background task failed")

@app.post("/orders-broken")
def broken(background: BackgroundTasks):
    background.add_task(broken_task)
    background.add_task(audit, "never runs?")
    return {"status": "accepted"}
```

The client got `200 {"status":"accepted"}`. The server logged
`ERROR:    Exception in ASGI application ... RuntimeError: background task failed`. And
`audit` **never ran** — the log stayed empty: one failing task stops the ones after it.

So: the client was told "accepted", the email wasn't sent, nothing retried it, and if the
process had been restarting during a deploy, even a successful task could have been cut
off. Background tasks have **no persistence, no retries and no visibility**. They're
appropriate for best-effort work where losing one occasionally is acceptable — a cache
warm-up, an analytics ping. For anything that must happen (payment capture, order
emails customers depend on), write a job to a durable queue instead: Level 4 lesson 7.

Wrap best-effort tasks so one failure doesn't take out the rest and the failure is
logged with context:

```python
import logging
logger = logging.getLogger("tasks")

def safe(fn):
    def wrapper(*args, **kwargs):
        try:
            return fn(*args, **kwargs)
        except Exception:
            logger.exception("background task %s failed", fn.__name__)
    return wrapper

background.add_task(safe(send_email), "ada@example.com", "Order confirmed")
```

(For `async` tasks, write the equivalent `async def` wrapper.)

## How It Actually Works

**Lifespan** is part of the ASGI spec. When Uvicorn starts, before opening the socket to
traffic, it calls the app with `scope["type"] == "lifespan"` and sends a
`lifespan.startup` event. Starlette's router runs your context manager up to `yield`,
merges the yielded dict into the lifespan scope's `state` (`scope["state"].update(...)`
in `starlette/routing.py`), and replies `lifespan.startup.complete` (or
`lifespan.startup.failed` if it raised, which makes Uvicorn exit). The *server* then
gives every request a shallow copy of that dict — Uvicorn's HTTP protocol code builds
each request scope with `"state": self.app_state.copy()` — and that's what
`request.state` reads. Shallow means the objects are shared: every request sees the
same `httpx.AsyncClient`, but assigning `request.state.x = ...` in one request doesn't
leak into another. On shutdown, Uvicorn sends `lifespan.shutdown`, and the code after `yield` runs.

**Background tasks**: `BackgroundTasks` is a list of `(func, args, kwargs)`. FastAPI
attaches it to the response object it builds. Starlette's `Response.__call__` sends the
headers and body through ASGI `send`, and only then runs each task in sequence —
`await`ing coroutines, and sending sync functions to the thread pool. The response
bytes have left the app at that point, so the client gets them; but the task still runs
inside the same request handling, which is why an exception propagates to Uvicorn's
error log, and why the test client, which waits for the ASGI call to finish completely,
waits for the tasks too.

## Common mistakes

- **Creating clients or engines per request** instead of once in lifespan.
- **`TestClient(app)` without `with`**, so startup never runs.
- **Running migrations or other one-off jobs in lifespan**, multiplied by every worker.
- **Using `BackgroundTasks` for work that must not be lost.**
- **Heavy CPU work in an `async def` background task**, which blocks the loop for
  everyone. Make it `def` (thread pool) or move it out of the web process.
- **Passing a request-scoped database session into a background task.** By the time the
  task runs, the dependency's teardown may have closed it. Pass IDs, and open a new
  session inside the task.

## Exercise

1. Move the database engine creation from Level 2's bookshelf `db.py` into a lifespan
   handler, expose the sessionmaker through lifespan state, and update `get_db` to use
   it. Make the tests still pass.
2. Add a `/health` endpoint that reports whether the shared `httpx.AsyncClient` is open,
   and confirm it's closed after shutdown in a test.
3. Add a background task that writes an audit row to the database. Pass only the order
   ID, open its own session, and wrap it with `safe`.
4. Measure, against a real Uvicorn process, the latency of an endpoint with and without
   a 0.5 s background task. Then measure it through `TestClient` and explain the
   difference.
