---
description: "Observe a FastAPI service: JSON logs with request IDs via contextvars, Prometheus metrics by route template, and FastAPI's built-in OpenTelemetry tracing."
---

# 04 · Observability: Structured Logs, Request IDs & Metrics

When something goes wrong in production you have three sources of truth: **logs** (what
happened, in words), **metrics** (how much and how fast, as numbers over time) and
**traces** (where the time went inside one request). This lesson wires up all three for
a FastAPI app, and fixes a request-ID bug that only shows up on the error path — the one
path where you need it most.

## Structured logs with a request ID

Plain-text log lines are for humans; JSON lines are for log systems that let you filter
by field (`request_id = "req-abc-1"`). A request ID ties together every line produced
while handling one request, and is echoed back to the client in a header so a user's
bug report can be matched to the logs.

Here is the complete working setup — logging, metrics and middleware:

```python
import json, logging, re, sys, time, uuid
from contextvars import ContextVar
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from prometheus_client import CONTENT_TYPE_LATEST, Counter, Histogram, generate_latest
from starlette.responses import Response

request_id_var: ContextVar[str] = ContextVar("request_id", default="-")

class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        entry = {"ts": round(record.created, 3), "level": record.levelname, "logger": record.name,
                 "msg": record.getMessage(), "request_id": request_id_var.get()}
        for key in ("method", "path", "route", "status", "duration_ms"):
            if hasattr(record, key):
                entry[key] = getattr(record, key)
        if record.exc_info:
            entry["exc"] = self.formatException(record.exc_info).splitlines()[-1]
        return json.dumps(entry)

handler = logging.StreamHandler(sys.stdout); handler.setFormatter(JsonFormatter())
logging.basicConfig(level=logging.INFO, handlers=[handler], force=True)
logging.getLogger("httpx2").setLevel(logging.WARNING)
log = logging.getLogger("bookshop")
access = logging.getLogger("bookshop.access")

REQUESTS = Counter("http_requests_total", "Requests", ["method", "route", "status"])
LATENCY = Histogram("http_request_duration_seconds", "Latency", ["method", "route"],
                    buckets=(0.005, 0.01, 0.05, 0.1, 0.5, 1, 5))
SAFE_ID = re.compile(r"^[A-Za-z0-9-]{1,64}$")

class ObservabilityMiddleware:
    def __init__(self, app):
        self.app = app
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        incoming = dict(scope["headers"]).get(b"x-request-id", b"").decode("latin-1")
        rid = incoming if SAFE_ID.match(incoming) else uuid.uuid4().hex
        token = request_id_var.set(rid)
        scope.setdefault("state", {})["request_id"] = rid     # survives past this layer
        start = time.perf_counter()
        status = {"code": 500}
        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                status["code"] = message["status"]
                message.setdefault("headers", []).append((b"x-request-id", rid.encode()))
            await send(message)
        try:
            await self.app(scope, receive, send_wrapper)
        finally:
            route = scope.get("route")
            template = getattr(route, "path", "unmatched")
            dur = time.perf_counter() - start
            REQUESTS.labels(scope["method"], template, str(status["code"])).inc()
            LATENCY.labels(scope["method"], template).observe(dur)
            access.info("request", extra={"method": scope["method"], "path": scope["path"],
                                           "route": template, "status": status["code"],
                                           "duration_ms": round(dur * 1000, 2)})
            request_id_var.reset(token)

app = FastAPI()
app.add_middleware(ObservabilityMiddleware)

@app.exception_handler(Exception)
async def unhandled(request: Request, exc: Exception):
    rid = getattr(request.state, "request_id", "-")
    token = request_id_var.set(rid)          # this handler runs outside our middleware
    try:
        log.exception("unhandled error")
    finally:
        request_id_var.reset(token)
    return JSONResponse(status_code=500, content={"error": "internal_error", "request_id": rid},
                        headers={"x-request-id": rid})

@app.get("/books/{book_id}")
def get_book(book_id: int):
    log.info("looking up book %s", book_id)
    if book_id == 13:
        raise RuntimeError("cursed book")
    if book_id > 100:
        raise HTTPException(404, "not found")
    return {"id": book_id}

@app.get("/metrics", include_in_schema=False)
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

Key parts:

- **`request_id_var`** is a `ContextVar`: each request (each asyncio task, and any thread
  the work is handed to by `run_in_threadpool`, which copies the context) sees its own
  value. Any log call anywhere in your code picks it up, without passing the ID around.
- **`JsonFormatter`** writes one JSON object per line, adding the request ID and any
  structured `extra=` fields.
- **`ObservabilityMiddleware`** is a pure ASGI middleware (Level 3 lesson 4). It reuses an
  incoming `X-Request-ID` only if it matches a strict pattern — a client-supplied value
  with newlines or `"` characters would otherwise be written straight into your logs —
  and generates one otherwise. It records the response status from the
  `http.response.start` message, and in `finally` logs one access line and updates the
  metrics.
- **The route template** (`/books/{book_id}`), not the raw path (`/books/7`), is used as
  the metric label and logged as `route`. Starlette stores the matched route in
  `scope["route"]`; unmatched requests are labelled `unmatched`.

## The output

```text
{"ts": 1791480462.193, "level": "INFO", "logger": "bookshop", "msg": "looking up book 7", "request_id": "req-abc-1"}
{"ts": 1791480462.193, "level": "INFO", "logger": "bookshop.access", "msg": "request", "request_id": "req-abc-1", "method": "GET", "path": "/books/7", "route": "/books/{book_id}", "status": 200, "duration_ms": 0.99}
<- 200 req-abc-1
```

The endpoint's own log line and the access line share the client's request ID, and the
response carried it back in `x-request-id`. A hostile ID was replaced:

```text
GET /nope  X-Request-ID: "bad id with spaces\nInjected: yes"
{..., "request_id": "5321a42d3062483299cf09dd3da755a5", "method": "GET", "path": "/nope", "route": "unmatched", "status": 404, ...}
<- 404 5321a42d3062483299cf09dd3da755a5
```

## Worked example: the request ID vanished on errors

The first version of the 500 handler was the obvious one:

```python
@app.exception_handler(Exception)
async def unhandled(request: Request, exc: Exception):
    log.exception("unhandled error")
    return JSONResponse(status_code=500, content={"error": "internal_error",
                                                   "request_id": request_id_var.get()})
```

and a crash produced:

```text
{"ts": 1791480462.194, "level": "INFO", "logger": "bookshop.access", "msg": "request", "request_id": "3c5d96d7583a4d9e96cbf987de0b4f6a", ..., "status": 500, ...}
{"ts": 1791480462.194, "level": "ERROR", "logger": "bookshop", "msg": "unhandled error", "request_id": "-", "exc": "RuntimeError: cursed book"}
<- 500 {'error': 'internal_error', 'request_id': '-'}
```

The error line — the one you'd search for — had request ID `-`, and so did the response
the user would paste into a bug report. Look at the order, too: the access log line came
*before* the error line.

Level 3 lesson 4 showed why. A handler for `Exception` runs in `ServerErrorMiddleware`,
the **outermost** layer, outside every user middleware. By the time it ran, our
middleware's `finally` had already logged the access line and reset the context
variable. The fix, in the code above, stores the ID in `scope["state"]` as well — the
scope dict is shared with outer layers, so `request.state.request_id` is still there —
and the handler restores the context variable while it logs:

```text
{"ts": 1791480480.917, "level": "ERROR", "logger": "bookshop", "msg": "unhandled error", "request_id": "req-500", "exc": "RuntimeError: cursed book"}
<- 500 {'error': 'internal_error', 'request_id': 'req-500'} req-500
```

Now the error, the access line, the JSON body and the response header all agree. Test
the error path of your logging deliberately; it's the one that matters.

## Metrics for Prometheus

`prometheus_client` (0.26.0 here) keeps counters and histograms in memory and renders
them as text on `/metrics`, which a Prometheus server scrapes periodically. After the
requests above:

```text
http_requests_total{method="GET",route="/books/{book_id}",status="200"} 1.0
http_requests_total{method="GET",route="/books/{book_id}",status="500"} 1.0
http_requests_total{method="GET",route="/books/{book_id}",status="404"} 1.0
http_requests_total{method="GET",route="/books/{book_id}",status="422"} 1.0
http_requests_total{method="GET",route="unmatched",status="404"} 1.0
http_request_duration_seconds_bucket{le="0.01",method="GET",route="/books/{book_id}"} 4.0
```

From those series Prometheus can compute request rates, error ratios
(`status=~"5.."`) and latency percentiles from the histogram buckets. Using the raw path
as a label instead of the template would create a new time series for every book ID —
"high cardinality", which can overwhelm a metrics system.

With several workers, each process has its own registry and a scrape hits only one of
them. `prometheus_client` has a multiprocess mode for this (it needs a shared directory
set via an environment variable); it wasn't set up for this lesson.

## Tracing: built into FastAPI 0.143

The version used for this course has **native OpenTelemetry support**: FastAPI depends
on `opentelemetry-api` and accepts a `telemetry` option. It uses the globally configured
OpenTelemetry providers by default (which do nothing until you configure an SDK), or
providers you pass in. To see what it records, spans and metrics were captured in memory
with the OpenTelemetry SDK (1.45.1):

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException
from fastapi.testclient import TestClient
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor
from opentelemetry.sdk.trace.export.in_memory_span_exporter import InMemorySpanExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import InMemoryMetricReader

exporter = InMemorySpanExporter()
tp = TracerProvider(); tp.add_span_processor(SimpleSpanProcessor(exporter))
reader = InMemoryMetricReader(); mp = MeterProvider(metric_readers=[reader])

app = FastAPI(telemetry={"tracer_provider": tp, "meter_provider": mp})

def get_user():
    return {"name": "ada"}

@app.get("/books/{book_id}")
def get_book(book_id: int, user: Annotated[dict, Depends(get_user)]):
    if book_id == 404:
        raise HTTPException(404, "nope")
    return {"id": book_id}
```

Three requests — a success, a 404 and a validation failure — produced these spans:

```text
fastapi.dependencies    parent=yes  {'code.function.name': '__main__.get_book'}
fastapi.endpoint        parent=yes  {'code.function.name': '__main__.get_book'}
fastapi.serialization   parent=yes  {'code.function.name': '__main__.get_book'}
GET /books/{book_id}    parent=no   {'server.address': 'testserver', ..., 'url.path': '/books/7', 'http.request.method': 'GET', 'http.route': '/books/{book_id}', 'http.response.status_code': 200}
fastapi.dependencies    parent=yes  ...
fastapi.endpoint        parent=yes  ...
GET /books/{book_id}    parent=no   {..., 'url.path': '/books/404', ..., 'http.response.status_code': 404}
fastapi.dependencies    parent=yes  ...
GET /books/{book_id}    parent=no   {..., 'url.path': '/books/abc', ..., 'http.response.status_code': 422}
```

(Attributes trimmed after the first.) One server span per request, named by method and
route template, with child spans for **dependency resolution**, the **endpoint**, and
**serialization** — and the spans stop where the request stopped: the 404 has no
serialization span, the 422 never reached the endpoint. The metrics reader received an
`http.server.request.duration` histogram (unit `s`) labelled by route and status, and an
`http.server.active_requests` gauge.

The source of `fastapi/telemetry/_api.py` documents the options: `tracing`, `metrics`,
`logs`, `operation_spans` (the child spans), `exclude` (a function of the ASGI scope to
skip, e.g. health checks), explicit providers, and `auto_configure`, which adds OTLP
exporters from the standard `OTEL_*` environment variables when
`FASTAPI_OTEL_AUTO_CONFIGURE=true`. Exporting to a real collector (Jaeger, Tempo, a
vendor) wasn't tested here. On older FastAPI versions the usual route is the separate
`opentelemetry-instrumentation-fastapi` package.

## How It Actually Works

`logging` calls are synchronous and run in the calling task or thread; the formatter reads
`request_id_var.get()` at format time, so it sees whatever value is current in that
context. asyncio gives each task a copy of the context at creation, and
`run_in_threadpool` runs the function with a copy too, which is why `def` endpoints log
the right ID. Setting the variable in middleware and resetting it with the token in
`finally` keeps values from leaking between requests on the same task.

`prometheus_client` metrics are module-level objects holding values in process memory;
`generate_latest()` renders the registry in the Prometheus text format. Histograms store
cumulative bucket counts — the `le="0.01"` bucket counted all four `/books/{book_id}`
requests that took under 10 ms.

FastAPI's native telemetry wraps the app (it appears in the middleware stack as
`ExceptionTelemetryMiddleware` next to `ServerErrorMiddleware`, Level 3 lesson 4) and
creates spans around the stages of the route handler that you've met throughout this
course: solving dependencies, calling the endpoint, serializing the response.

## Common mistakes

- **Raw paths as metric labels** (cardinality explosion).
- **Trusting client request IDs blindly** — log injection.
- **Request IDs missing from error logs**, as in the worked example.
- **Logging secrets**: request bodies on `/auth/token`, `Authorization` headers, full
  database URLs.
- **Per-process metrics with multiple workers** and no multiprocess setup.
- **Health checks dominating traces and metrics** — exclude them.
- **`print()` for logging**: no levels, no structure, no context.

## Exercise

1. Add the observability middleware and JSON logging to the Level 3 bookmarks project.
   Log `user_id` (not username) on authenticated requests via a second context variable
   set in `current_user`.
2. Write a test that triggers a 500 and asserts that the response body's `request_id`,
   the `x-request-id` header and the error log record all match (use pytest's `caplog`).
3. Add an `exclude` function to `FastAPI(telemetry=...)` so `/health` and `/metrics`
   produce no spans, and verify with the in-memory exporter.
4. Write a Prometheus query (on paper, or in a local Prometheus if you have one) for the
   95th-percentile latency of `/books/{book_id}` over five minutes.
