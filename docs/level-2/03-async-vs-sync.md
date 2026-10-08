---
description: "When to write async def or def in FastAPI: measured effects of blocking the event loop, the 40-thread pool limit, and offloading blocking calls safely."
---

# 03 · `async def` vs `def`: The Event Loop and the Threadpool

FastAPI lets you write endpoints and dependencies as either `async def` or plain `def`,
and both "work". The choice decides *where* your code runs, and getting it wrong is the
single most common cause of a FastAPI app that's mysteriously slow under load. This
lesson measures the difference instead of asserting it.

## Two places your code can run

A Uvicorn worker process runs **one event loop** on its main thread. The loop juggles
many requests at once, but only while each of them is *waiting* — on a socket, a timer, a
database driver that supports `await`. Running Python code holds the loop; nothing else
progresses until it yields.

So FastAPI uses two strategies:

- **`async def`** — called directly on the event loop. Fast to dispatch, and perfect for
  code that `await`s. Disastrous for code that blocks.
- **`def`** — sent to a **thread pool** with `run_in_threadpool`, so blocking calls
  block only that worker thread, while the loop keeps serving other requests.

## The experiment

```python
import asyncio, threading, time
from fastapi import FastAPI
from fastapi.concurrency import run_in_threadpool

app = FastAPI()

def slow_io():               # stands in for a blocking library call
    time.sleep(0.5)
    return f"{threading.current_thread().name}-{threading.get_ident()}"

@app.get("/async-blocking")
async def async_blocking():
    time.sleep(0.5)                       # WRONG: blocks the event loop
    return {"thread": ...}

@app.get("/async-awaiting")
async def async_awaiting():
    await asyncio.sleep(0.5)              # yields to the loop while waiting
    return {"thread": ...}

@app.get("/sync-blocking")
def sync_blocking():
    time.sleep(0.5)                       # runs in a worker thread
    return {"thread": ...}

@app.get("/async-offloaded")
async def async_offloaded():
    name = await run_in_threadpool(slow_io)
    return {"thread": name}

@app.get("/ping")
async def ping():
    return {"pong": True}
```

(The `...` stand for the thread name and ID, which each endpoint returned.) `time.sleep`
stands in for any blocking call: `requests.get`, a synchronous database driver, reading
a big file, CPU-heavy work. The app ran under Uvicorn with one worker, and a script fired
**10 concurrent requests** at each endpoint with `httpx.AsyncClient`:

```text
/async-blocking    n= 10 wall= 5.07s  distinct threads=1  e.g. ['MainThread']
/async-awaiting    n= 10 wall= 0.52s  distinct threads=1  e.g. ['MainThread']
/sync-blocking     n= 10 wall= 0.53s
/async-offloaded   n= 10 wall= 0.53s
```

(The thread names on the last two lines were all `AnyIO worker thread`; the later run
below counts them by ID.)

- `async def` + `time.sleep`: **5.07 s**. Ten requests ran one after another, all on
  `MainThread`, because each one held the loop for its whole half-second.
- `async def` + `await asyncio.sleep`: 0.52 s — the waits overlapped on one thread.
- `def` + `time.sleep`: 0.53 s — ten worker threads slept in parallel.
- `async def` that offloads its blocking call: also 0.53 s.

### It hurts *other* endpoints too

The worst part of blocking the loop isn't the slow endpoint — it's everything else. With
four slow requests in flight, a trivial `/ping` took:

```text
/ping while 4x /async-blocking   took   1977 ms
/ping while 4x /async-awaiting   took      3 ms
/ping while 4x /sync-blocking    took      4 ms
```

Nearly two seconds for an endpoint that does nothing, because it had to wait for the
loop to be free. In production this shows up as health checks timing out and a load
balancer marking a healthy server as dead.

## The thread pool has a limit

`def` endpoints aren't free either. The pool is AnyIO's default thread limiter, and the
app reported its size from inside a request:

```text
{'thread': 'n/a', 'total_tokens': 40}
```

Forty threads. A second run measured what that means for a 0.5 s blocking `def`
endpoint:

```text
n= 40 wall=0.58s distinct worker threads=40
n= 41 wall=1.14s distinct worker threads=40
n= 80 wall=1.18s distinct worker threads=40
n=120 wall=1.68s distinct worker threads=40
```

Request 41 had to wait for a thread to come free, doubling the wall time. Every `def`
endpoint **and** every `def` dependency shares the same 40 tokens. (The `n=120` figure is
also affected by the client's own connection limit, so don't read it as a precise
three-batch number.) These are one laptop's numbers; the shape, not the exact
milliseconds, is the lesson.

The limit can be raised — `anyio.to_thread.current_default_thread_limiter().total_tokens`
is assignable, typically at startup — but more threads mean more memory and more
contention. Usually the better fix is to make the slow path asynchronous or to add
worker processes (Level 4 lesson 2).

## The decision rule

| Your code… | Write | Why |
|---|---|---|
| awaits async libraries only (`httpx.AsyncClient`, `asyncpg`, async SQLAlchemy) | `async def` | no thread hop, high concurrency |
| calls any blocking library (`requests`, sync SQLAlchemy, `boto3`, file I/O) | `def` | keeps the loop free |
| is mostly async but has one blocking call | `async def` + `await run_in_threadpool(fn, ...)` | offload just that call |
| is CPU-heavy (image resizing, big JSON, ML inference) | `def`, or a process pool / job queue | threads still share the GIL for pure-Python work |
| does trivial work (return a constant, read settings) | either | `async def` avoids a thread hop |

When in doubt, **`def` is the safe default**: it might cost a thread hop, but it can't
freeze the server.

## Worked example: calling another API

Two correct ways and one wrong way to call an external service:

```python
import httpx, requests

@app.get("/rates-wrong")
async def rates_wrong():
    return requests.get("https://example.com/rates").json()   # blocks the loop

@app.get("/rates-sync")
def rates_sync():
    return requests.get("https://example.com/rates", timeout=5).json()   # fine: thread pool

client = httpx.AsyncClient(timeout=5)   # create once, reuse (Level 3 lesson 5 closes it)

@app.get("/rates-async")
async def rates_async():
    r = await client.get("https://example.com/rates")
    return r.json()
```

(These weren't run against a real rates service; `example.com` is a placeholder domain.)
The async version scales to thousands of concurrent waits with no threads. The sync
version is capped by the thread pool. The wrong version serialises every request in the
process.

## How It Actually Works

When FastAPI analyses an endpoint at startup it records whether the callable is a
coroutine function (`inspect.iscoroutinefunction`, roughly). At request time:

- coroutine → `await endpoint(**values)` directly;
- otherwise → `await run_in_threadpool(endpoint, **values)`, which is Starlette's wrapper
  around `anyio.to_thread.run_sync`.

`run_sync` acquires a token from the default `CapacityLimiter` (40 tokens), hands the
function to a worker thread, and suspends the calling coroutine until the thread
finishes. While it's suspended, the event loop serves other requests. If no token is
free, the coroutine waits in a queue for one — that's the jump at request 41.

The same rule applies to dependencies, and to response-model validation for `def`
endpoints (Level 1 lesson 5 noted that `serialize_response` validates in the thread pool
for non-coroutine endpoints). A `def` dependency of an `async def` endpoint still goes to
a thread.

Why don't threads speed up CPU-bound Python? CPython's global interpreter lock lets only
one thread execute Python bytecode at a time (free-threaded builds of Python 3.13+ relax
this, but most deployments don't use them yet). Threads help when the work is *waiting* —
on I/O or in C code that releases the lock — not when it's computing in Python.

## Common mistakes

- **`async def` everywhere "because it's faster"**, then calling `requests`, sync
  SQLAlchemy or `time.sleep` inside. This is the classic FastAPI performance bug.
- **Async libraries in `def` endpoints.** You can't `await` there, so people reach for
  `asyncio.run()`. In a worker thread that starts a brand-new event loop for every call,
  and a shared async client created on the main loop can't be used from it. Make the
  endpoint `async def` instead.
- **Forgetting dependencies count.** An `async def` endpoint with a blocking `async def`
  dependency blocks the loop just the same.
- **Exhausting the thread pool** with long blocking calls (slow external APIs without
  timeouts). Always set timeouts.
- **CPU-heavy work in the request.** Move it to a process pool or a job queue (Level 4
  lesson 7).
- **Creating an `httpx.AsyncClient` per request**, which loses connection pooling.

## Exercise

1. Reproduce the experiment. Then change `/async-blocking` to use
   `await run_in_threadpool(time.sleep, 0.5)` and re-measure.
2. Find the limit: write a `def` endpoint that sleeps for 1 s and fire 39, 40, 41 and 79
   concurrent requests. Plot or tabulate the wall times.
3. Add a blocking `async def` *dependency* to `/ping` and measure `/ping`'s latency
   under concurrent load. Then turn the dependency into `def` and measure again.
4. Rewrite an endpoint that uses `requests` to use a shared `httpx.AsyncClient`. Keep
   the timeout.
