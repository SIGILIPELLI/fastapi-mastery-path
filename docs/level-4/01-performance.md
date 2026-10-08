---
description: "Measure FastAPI performance: load tests with ab, pyinstrument in the right event loop, catching blocking calls, and why response models were faster."
---

# 01 · Performance: Finding Blocking Code and Measuring Throughput

"FastAPI is fast" is true of the framework's overhead. Whether *your* API is fast depends
on what your endpoints do, and the only way to know is to measure. This lesson measures
a handful of endpoints, profiles the slowest, catches a blocking call with asyncio's
debug mode, and shows one surprising result about response models.

!!! note "About the numbers"
    All figures below come from one run on one 8-core laptop, with the load generator on
    the same machine as the server. They're useful for **comparing endpoints against each
    other**, not as absolute benchmarks. Re-run them on your hardware before drawing
    conclusions.

## The endpoints under test

```python
import hashlib, json, time
from fastapi import FastAPI
from fastapi.responses import Response
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    id: int
    name: str
    price_cents: int
    tags: list[str]

ITEMS = [{"id": i, "name": f"Item {i}", "price_cents": i * 7 % 5000, "tags": ["a", "b", "c"]}
         for i in range(1000)]
ITEM_MODELS = [Item(**d) for d in ITEMS]

@app.get("/ping")
async def ping():
    return {"ok": True}

@app.get("/items-model", response_model=list[Item])
async def items_model():
    return ITEMS                                  # dicts, validated by the response model

@app.get("/items-annotated")
async def items_annotated() -> list[Item]:
    return ITEM_MODELS                            # already models

@app.get("/items-raw")
async def items_raw():
    return ITEMS                                  # no response model at all

@app.get("/items-prebuilt")
async def items_prebuilt():
    return Response(content=json.dumps(ITEMS), media_type="application/json")

@app.get("/cpu")
def cpu():
    h = b"x"
    for _ in range(200_000):
        h = hashlib.sha256(h).digest()
    return {"h": h.hex()[:8]}

@app.get("/sneaky-block")
async def sneaky_block():
    time.sleep(0.2)
    return {"ok": True}
```

Each list endpoint returns the same 1,000 items — 68,463 bytes of JSON.

## Load testing with `ab`

ApacheBench (`ab`) ships with macOS and most Linux distributions. `-n` is the total
number of requests, `-c` the concurrency:

```bash
uvicorn app:app --port 8713 --log-level warning
ab -q -n 400 -c 10 http://127.0.0.1:8713/items-model
```

Results with one Uvicorn worker:

| Endpoint | Requests/s | Mean time per request (c=10) |
|---|---|---|
| `/ping` (n=2000, c=20) | 11,575 | — |
| `/items-annotated` (returns models) | 2,446 | 4.1 ms |
| `/items-prebuilt` (`json.dumps` yourself) | 1,625 | 6.2 ms |
| `/items-model` (dicts + `response_model`) | 804 | 12.4 ms |
| `/items-raw` (**no** response model) | 156 | 64.2 ms |

Turn off access logging (`--log-level warning`) for load tests, or you'll partly be
measuring the terminal.

### The surprise: a response model made it faster

The endpoint with **no** response model was the slowest by far — about 5× slower than
the same data with `response_model=list[Item]`, and about 15× slower than returning
model instances. Intuition says validation should cost time. The profile says why it
doesn't.

## Profiling — in the right place

A first attempt profiled 20 requests made through `TestClient`:

```text
0.2611 <module>  l4_01_prof.py:1
`- 0.2611 TestClient.get  httpx2/_client.py:1096
         0.2581 _TestClientTransport.handle_request  starlette/testclient.py:223
         |- 0.2506 BlockingPortal.call  anyio/from_thread.py:325
         |  `- 0.1321 Future.result  concurrent/futures/_base.py:424
         |     `- 0.1321 Condition.wait  threading.py:337
         |        `- 0.1321 lock.acquire  <built-in>
...
```

Useless: all the time is spent *waiting on a lock*, because `TestClient` runs the app in
another thread (Level 2 lesson 8) and the profiler only sampled the calling thread. The
fix is to call the app in the **same** event loop, with `httpx.AsyncClient` and
`ASGITransport`, and pyinstrument's async mode:

```python
import asyncio
from httpx import ASGITransport, AsyncClient
from pyinstrument import Profiler

async def main():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://t") as c:
        await c.get("/items-raw")                       # warm up
        p = Profiler(interval=0.0005, async_mode="enabled")
        p.start()
        for _ in range(20):
            await c.get("/items-raw")
        p.stop()
        print(p.output_text(unicode=False, color=False))

asyncio.run(main())
```

```text
0.4598 main  l4_01_prof2.py:6
`- 0.4598 AsyncClient.get  httpx/_client.py:1751
      [32 frames hidden]  httpx, fastapi, starlette, <built-in>
         0.4239 jsonable_encoder  fastapi/encoders.py:129
         |- 0.3179 jsonable_encoder  fastapi/encoders.py:129
         |  |- 0.1020 [self]  fastapi/encoders.py
         |  |- 0.0944 jsonable_encoder  fastapi/encoders.py:129
         |  |  `- 0.0236 is_dataclass  dataclasses.py:1477
         |  `- 0.0555 is_dataclass  dataclasses.py:1477
         ...
         0.0169 JSONResponse.render  starlette/responses.py:196
         `- 0.0169 dumps  json/__init__.py:185
```

(pyinstrument 5.1.3.) **92%** of the time (0.424 of 0.460 s) was in `jsonable_encoder`,
recursing through every dict, list and string, checking each value's type in Python
(`is_dataclass` shows up as a hotspot). The actual JSON encoding (`dumps`) took 0.017 s.
With a response model, FastAPI hands the data to Pydantic's compiled serializer instead,
which does the same walk in Rust — and when the values are already model instances, the
validation step has very little to do.

The lesson isn't "always return models" so much as **"profile before guessing"**, and
**declare response models** — they are a correctness feature (Level 1 lesson 5) that, on
this version, also happened to be the fast path.

## Catching blocking calls

The `/sneaky-block` endpoint is `async def` with a `time.sleep` inside — Level 2
lesson 3's classic bug. In a large codebase it's rarely that obvious: a blocking call
hides three functions deep in a library. asyncio's **debug mode** logs any callback that
holds the loop too long (100 ms by default):

```bash
PYTHONASYNCIODEBUG=1 uvicorn app:app --port 8714 --loop asyncio
```

One request to `/sneaky-block` produced:

```text
Executing <Task finished name='Task-5' coro=<RequestResponseCycle.run_asgi() done, defined at .../uvicorn/protocols/http/httptools_impl.py:420> result=None created at .../uvicorn/protocols/http/httptools_impl.py:309> took 0.210 seconds
```

`--loop asyncio` matters: debug mode is a feature of the standard asyncio loop, and
Uvicorn otherwise uses `uvloop` when it's installed. Debug mode slows everything down;
use it in development and in a staging load test, not in production. The message points
at the request task, not the guilty line — combine it with the profiler, or with
`asyncio`'s `slow_callback_duration` lowered, to narrow it down.

## Worked example: CPU-bound work and workers

`/cpu` hashes in a loop for about 70 milliseconds — pure CPU in a `def` endpoint.
With 8 concurrent clients:

| Setup | Requests/s for `/cpu` |
|---|---|
| 1 worker | 14.35 |
| 4 workers (`--workers 4`) | 43.22 |

With one worker, eight concurrent clients got no more throughput than one: a separate
`ab -c 1` run gave **14.40** requests/s at 69 ms per request, the same as the 14.35 at
`-c 8`. The thread pool ran eight requests "at once", but CPython threads take turns
holding the GIL for Python bytecode, and hashing 32-byte inputs in a Python loop is
almost all bytecode, so they simply queued.
**Processes** scale CPU-bound work, roughly 3× here for 4 workers on 8 cores, with the
load generator competing for the same CPUs. The same 4 workers lifted `/items-model` from
804 to 2,701 requests/s. Level 4 lesson 2 covers running workers in production.

## How It Actually Works

When an endpoint has no response model, FastAPI calls `jsonable_encoder(value)`, a
pure-Python recursive function that converts anything (dataclasses, Pydantic models,
datetimes, enums, sets…) into JSON-compatible primitives, checking the type of every
value it visits. Then `JSONResponse` calls `json.dumps`. That's flexible and slow for
large payloads.

With a response model, the source read in Level 1 lesson 5 showed `serialize_response`
calling `field.validate(...)` and then `field.serialize(...)` — or, when FastAPI decides
to, `field.serialize_json(...)`, which produces the JSON bytes directly from
`pydantic-core`. Both validation and serialization are compiled code. For data that's
already a model instance, validation is little more than an isinstance check per item.

`ab` opens `-c` connections and sends requests as fast as responses come back, so
"requests per second" is a throughput figure under that fixed concurrency. Latency
percentiles (shown in `ab`'s full output) matter more for user experience than the mean;
look at the 95th and 99th percentiles before and after a change.

## Common mistakes

- **Optimising without measuring**, or measuring through `TestClient` in a profiler.
- **Benchmarking with access logs on**, or with debug mode on.
- **Large responses without a response model**, paying for `jsonable_encoder`.
- **Returning huge payloads at all.** Pagination (Level 2 lesson 9) beats any serializer.
- **Blocking calls in `async def`**, found only under production load. Run a staging
  load test with `PYTHONASYNCIODEBUG=1`.
- **Expecting threads to scale CPU-bound Python.** Use workers or move the work out of
  the request (Level 4 lesson 7).
- **Comparing numbers across machines** or across runs with different background load.

## Exercise

1. Reproduce the table on your machine. Then add an endpoint that returns `ITEM_MODELS`
   with `response_model=list[Item]` *and* `response_model_exclude_none=True`, and measure
   the cost of the option.
2. Profile `/items-model` with the async profiler and find where its time goes.
3. Add a blocking `requests.get` (to a local endpoint) inside an `async def` dependency
   and find it using debug mode.
4. Run `/cpu` with 1, 2, 4 and 8 workers and plot requests/s. Where does it stop scaling,
   and why?
