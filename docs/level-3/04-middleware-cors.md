---
description: "Write FastAPI middleware (function, class and pure ASGI), get the ordering right, and configure CORS, GZip and TrustedHost, avoiding the wildcard trap."
---

# 04 · Middleware & CORS

Dependencies run for the endpoints that declare them. **Middleware** runs for *every*
request that enters the app — including 404s, CORS preflights and requests that fail
validation — wrapping the whole thing like layers of an onion. It's the right tool for
cross-cutting concerns that don't belong to any one endpoint: request IDs, timing,
compression, host checks and CORS.

## Function middleware

```python
import time, uuid
from fastapi import FastAPI, Request

app = FastAPI()

@app.middleware("http")
async def timing_and_request_id(request: Request, call_next):
    request_id = request.headers.get("x-request-id") or uuid.uuid4().hex[:12]
    request.state.request_id = request_id
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    response.headers["Server-Timing"] = f"app;dur={(time.perf_counter() - start) * 1000:.1f}"
    return response
```

Code before `await call_next(request)` runs on the way in; code after runs on the way
out. `request.state` is a per-request namespace that endpoints can read
(`request.state.request_id`). A request with `X-Request-ID: abc123` came back with:

```text
{'x-request-id': 'abc123', 'server-timing': 'app;dur=0.8', ...}
```

`Server-Timing` is a standard header that browser dev tools display in the network panel.
Level 4 lesson 4 extends the request ID into structured logs.

## Order: the last one added is the outermost

The full test app had two function middlewares (`inner`, registered first, and the
timing one, registered second), a pure ASGI middleware, and Starlette's GZip, CORS and
TrustedHost middleware added with `app.add_middleware(...)`. Each layer recorded when it
ran:

```text
order: ['outer before', 'inner before', 'endpoint', 'inner after', 'outer after', 'asgi saw status 200']
middleware stack: ['TrustedHostMiddleware', 'CORSMiddleware', 'GZipMiddleware', 'PureASGILogger', 'BaseHTTPMiddleware', 'BaseHTTPMiddleware']
```

`app.user_middleware` lists the stack **outermost first**. Every `add_middleware` (and
every `@app.middleware`) inserts at the front, so the last one registered wraps all the
others. The practical rules:

- Register **TrustedHost and CORS last**, so they're outermost: a bad host or a CORS
  preflight is answered before any other work.
- Register **GZip** outside anything that inspects the response body, so those layers
  see uncompressed content.
- Error-handling middleware that should catch everything goes outside the layers it
  protects.

## Pure ASGI middleware

`@app.middleware("http")` is implemented with Starlette's `BaseHTTPMiddleware`
(visible in the stack above). It's convenient, but it works on the whole `Response`
object. For streaming responses, WebSockets, or anything performance-sensitive, write a
pure ASGI middleware — a class wrapping the app, like the raw ASGI app in Level 1
lesson 1:

```python
class PureASGILogger:
    """Pure ASGI middleware: sees every message, never buffers the body."""
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)

        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                order.append(f"asgi saw status {message['status']}")
            await send(message)

        await self.app(scope, receive, send_wrapper)

app.add_middleware(PureASGILogger)
```

It intercepts the `send` channel: it can read or modify the status and headers in the
`http.response.start` message, and pass body chunks through untouched. It ran last in
the log above because it was registered after the function middlewares and so sat
*outside* them — the response passed through it after `outer after`.

## Compression and host checks

```python
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

app.add_middleware(GZipMiddleware, minimum_size=500)
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["api.example.com", "testserver"])
```

The `/books` response was 739 bytes uncompressed. The test client advertises gzip, so it
received `content-encoding: gzip`; with `Accept-Encoding: identity` the same response came
back uncompressed with `content-length: 739`. Responses under `minimum_size` aren't
compressed — it isn't worth the CPU.

`TrustedHostMiddleware` rejected a request with `Host: attacker.example`:

```text
400 Invalid host header
```

That matters because code that builds absolute URLs from the request (password-reset
links, `Location` headers, `url_for`) uses the `Host` header, and an attacker controls
it. ("testserver" is in the list only because that's the test client's host.)

## CORS

Browsers enforce the **same-origin policy**: JavaScript on `https://shop.example.com`
can't read responses from `https://api.example.com` unless the API says it may.
**Cross-Origin Resource Sharing** is how the API says so, through response headers.

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://shop.example.com"],
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
    allow_credentials=True,
    max_age=600,
)
```

For "non-simple" requests (anything with an `Authorization` header or a JSON content
type, for example), the browser first sends a **preflight** `OPTIONS` request. From the
allowed origin:

```text
OPTIONS /books  Origin: https://shop.example.com
                Access-Control-Request-Method: POST
                Access-Control-Request-Headers: authorization,content-type
200 OK
  access-control-allow-origin: https://shop.example.com
  access-control-allow-methods: GET, POST
  access-control-allow-headers: Accept, Accept-Language, Authorization, Content-Language, Content-Type
  access-control-allow-credentials: true
  access-control-max-age: 600
  vary: Origin, Access-Control-Request-Method, Access-Control-Request-Headers, Access-Control-Request-Private-Network
```

(Header order rearranged for reading.) The browser caches that answer for `max_age`
seconds. From an origin that isn't listed, or for a method that isn't allowed:

```text
OPTIONS from https://evil.example        -> 400 Disallowed CORS origin
OPTIONS with method DELETE               -> 400 Disallowed CORS method
```

### CORS is not access control

Now the important part. A plain `GET` from each origin:

```text
https://shop.example.com   200  allow-origin: https://shop.example.com
https://evil.example       200  allow-origin: None
https://shop.example.com/  200  allow-origin: None
```

The request from `evil.example` got a **200** — the endpoint ran and produced the data.
The only difference is the missing `Access-Control-Allow-Origin` header, which makes the
*browser* refuse to show the response to the page's JavaScript. `curl`, scripts and
servers ignore CORS entirely. CORS protects users' browsers from malicious pages; it
doesn't protect your API. Authentication and authorization (lessons 1–3) do that.

The third line shows exact matching: an origin is scheme + host + port, with **no
trailing slash**. Configure `"https://shop.example.com/"` and nothing will ever match.
(Level 2 lesson 7 noted that `AnyHttpUrl` adds that trailing slash when parsing
settings.)

## Worked example: the wildcard-with-credentials trap

`allow_origins=["*"]` looks like the easy fix for CORS errors in development. Here's what
it did with and without credentials, for a request from `https://evil.example`:

```text
allow_credentials=False -> allow-origin: *                    allow-credentials: None
allow_credentials=True  -> allow-origin: https://evil.example allow-credentials: true
```

With credentials off, `*` is fairly harmless: browsers won't send cookies to a wildcard
response. With credentials **on**, the middleware didn't send `*` — browsers forbid that
combination — it **reflected the requesting origin** and allowed credentials. Any website
a logged-in user visits can now make authenticated requests to your API with their
cookies and read the responses. Never combine `"*"` with `allow_credentials=True`; list
your real origins (from settings), or use `allow_origin_regex` with a carefully anchored
pattern.

## How It Actually Works

`app.add_middleware(cls, **options)` records the class and options in
`app.user_middleware`. The stack is built when the app first handles a request. In
FastAPI 0.143.0's `build_middleware_stack`, the full list, outermost first, is:

```python
[ServerErrorMiddleware, ExceptionTelemetryMiddleware]   # FastAPI's own outer layers
+ self.user_middleware                                    # yours, last-added first
+ [ExceptionMiddleware, AsyncExitStackMiddleware]         # handlers, file cleanup
```

and the app is built by wrapping the router in each of those, in reverse. So your
middleware sits *inside* `ServerErrorMiddleware` (the 500 handler from Level 1 lesson 7)
but *outside* `ExceptionMiddleware` — meaning a `raise HTTPException` in an endpoint has
already been turned into a response by the time your middleware sees it. Each layer is
an ASGI app holding a reference to the next.

`BaseHTTPMiddleware` converts the ASGI conversation into `Request`/`Response` objects so
your function can use `call_next`. To do that, it runs the inner app in a separate task
and streams its response back through a memory channel — extra machinery on every
request, and a source of subtle issues with background tasks and context variables in
some versions. Pure ASGI middleware avoids it by just wrapping `send`.

`CORSMiddleware` inspects the `Origin` header. If absent, it's not a CORS request and
passes straight through. If it's an `OPTIONS` with `Access-Control-Request-Method`, it's a
preflight: the middleware answers it **itself** without calling your app (so a preflight
never needs an endpoint). Otherwise it calls the app and adds the allow headers to the
response when the origin is allowed — which is why the disallowed `GET` still reached
the endpoint.

## Common mistakes

- **`allow_origins=["*"]` with `allow_credentials=True`.**
- **Treating CORS as security** for the API itself.
- **Trailing slashes in origins.**
- **CORS added inside other middleware**, so a failure in an inner layer produces a
  response without CORS headers, and the browser reports a confusing CORS error instead
  of the real one. Keep CORS outermost.
- **Reading the request body in `BaseHTTPMiddleware`** — it's consumed and can break the
  endpoint, or force the whole body into memory. Use a pure ASGI middleware if you must.
- **Heavy work in middleware** that runs for every request, including health checks.
- **Building absolute URLs from an unchecked `Host` header.**

## Exercise

1. Write a pure ASGI middleware that adds `X-Request-ID` (reusing the incoming header if
   it's a safe, short alphanumeric string, otherwise generating one) and stores it in
   `scope["state"]`.
2. Configure CORS from settings: `cors_origins: list[str]` validated to have no trailing
   slash and an `https://` scheme in production.
3. Reproduce the wildcard-with-credentials behaviour, then write a test that fails if
   anyone ever configures it.
4. Register two middlewares in the "wrong" order (GZip inside one that logs response body
   size) and observe what the logger sees.
