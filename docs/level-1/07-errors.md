---
description: "Return useful errors from FastAPI: HTTPException with headers, domain exceptions with handlers, custom 404/405/422 shapes and safe handling of crashes."
---

# 07 · Errors: HTTPException, Validation Errors & Handlers

An API's error responses are part of its interface. Clients branch on them, retry on
them, show them to users. This lesson covers the three kinds of error a FastAPI app
produces, and how to shape each one deliberately:

1. **Errors you raise on purpose** — "no such book", "not logged in".
2. **Validation errors** produced by Pydantic before your code runs.
3. **Crashes** — bugs, timeouts, a database that went away.

## Raising `HTTPException`

```python
from fastapi import FastAPI, HTTPException, status

app = FastAPI()
BOOKS = {1: "Dune"}

@app.get("/books/{book_id}")
def get_book(book_id: int):
    if book_id not in BOOKS:
        raise HTTPException(status_code=404, detail=f"Book {book_id} not found")
    return {"id": book_id, "title": BOOKS[book_id]}
```

With FastAPI's default handler, `GET /books/2` returned `404` and
`{'detail': 'Book 2 not found'}`. It's `raise`, not `return`: raising works from any
depth — a helper function, a dependency — and stops the request immediately.

`detail` can be any JSON-serializable value, so `detail={"code": "not_found", "id": 2}`
is fine too. Some errors need headers; HTTP says a 401 should tell the client how to
authenticate:

```python
@app.get("/admin")
def admin():
    raise HTTPException(status.HTTP_401_UNAUTHORIZED, detail="Not authenticated",
                        headers={"WWW-Authenticate": "Bearer"})
```

```text
GET /admin -> 401 {'error': 'Not authenticated', 'status': 401} {'www-authenticate': 'Bearer'}
```

(The body shape here comes from the custom handler later in this lesson.)

## Domain exceptions and handlers

`HTTPException` everywhere ties your business logic to HTTP. A cleaner design raises
exceptions that describe *what went wrong in your domain*, and converts them to HTTP in
one place:

```python
from fastapi import Request
from fastapi.responses import JSONResponse

STOCK = {1: 0}

class OutOfStock(Exception):
    def __init__(self, book_id: int):
        self.book_id = book_id

@app.exception_handler(OutOfStock)
async def out_of_stock_handler(request: Request, exc: OutOfStock):
    return JSONResponse(status_code=409, content={
        "type": "out_of_stock", "title": "Book is out of stock",
        "book_id": exc.book_id, "path": request.url.path})

@app.post("/books/{book_id}/buy")
def buy(book_id: int):
    if STOCK.get(book_id, 0) == 0:
        raise OutOfStock(book_id)
    return {"ok": True}
```

```text
POST /books/1/buy -> 409 {'type': 'out_of_stock', 'title': 'Book is out of stock', 'book_id': 1, 'path': '/books/1/buy'}
```

The `buy` function — and any service code it calls — now has no idea what a status code
is. The same `OutOfStock` could be raised from a CLI script or a background job and
handled differently there. The `type`/`title` shape is borrowed from RFC 9457 "Problem
Details", which Level 4 lesson 8 uses for a consistent error contract.

## Customising validation errors

Override the handler for `RequestValidationError` to change the 422 shape, or just to
observe it. Reusing FastAPI's default handler keeps the standard body:

```python
import logging
from fastapi.exceptions import RequestValidationError
from fastapi.exception_handlers import request_validation_exception_handler

log = logging.getLogger("bookshop")

@app.exception_handler(RequestValidationError)
async def log_validation(request: Request, exc: RequestValidationError):
    log.info("validation failed on %s: %d error(s)", request.url.path, len(exc.errors()))
    return await request_validation_exception_handler(request, exc)
```

```text
INFO bookshop: validation failed on /books/abc: 1 error(s)
GET /books/abc -> 422 {'detail': [{'type': 'int_parsing', 'loc': ['path', 'book_id'], ...}]}
```

Logging validation failures is useful: a sudden spike usually means a client shipped a
bug. Be careful not to log `exc.body` or the `input` values blindly — they can contain
passwords or personal data.

## Customising 404 and 405 for the whole app

Requests for routes that don't exist never reach your code; Starlette's router raises
its own `HTTPException`. To change *those* responses, register the handler for
**Starlette's** class, which FastAPI's `HTTPException` inherits from:

```python
from starlette.exceptions import HTTPException as StarletteHTTPException

@app.exception_handler(StarletteHTTPException)
async def http_errors(request: Request, exc: StarletteHTTPException):
    return JSONResponse(status_code=exc.status_code,
                        content={"error": exc.detail, "status": exc.status_code},
                        headers=getattr(exc, "headers", None))
```

Results from the run:

```text
GET /books/2     -> 404 {'error': 'Book 2 not found', 'status': 404}
GET /nope        -> 404 {'error': 'Not Found', 'status': 404}
DELETE /books/1  -> 405 {'error': 'Method Not Allowed', 'status': 405} {'allow': 'GET'}
```

One handler now covers your own `raise HTTPException(...)`, unknown paths and wrong
methods. Note the `Allow: GET` header on the 405 — the router knows which methods the
path does support, and passing `exc.headers` through preserves that. A handler
registered for FastAPI's `HTTPException` alone would miss the router's 404s and 405s.

## Crashes

```python
@app.get("/crash")
def crash():
    return {"ratio": 1 / 0}
```

The client receives `500` with the plain-text body `Internal Server Error`. Nothing about
the exception leaks, which is what you want. To return JSON instead and log it your way,
register a handler for `Exception`:

```python
@app.exception_handler(Exception)
async def unhandled(request: Request, exc: Exception):
    log.exception("unhandled error on %s", request.url.path)
    return JSONResponse(status_code=500, content={"error": "internal_error"})
```

Running under Uvicorn, `curl -i /crash` gave `500` with `content-type: application/json`
and the 26-byte body `{"error":"internal_error"}`. The server log contained **two**
tracebacks: one from `log.exception` and a second one after
`ERROR:    Exception in ASGI application`. That's by design — see below — and it means
you don't need to log the traceback yourself unless you want it in a specific format
or logger.

## Worked example: picking status codes for a "buy" endpoint

| Situation | Status | Why |
|---|---|---|
| unknown book ID | 404 | the resource doesn't exist |
| quantity is `0` or `"lots"` | 422 | invalid input, let validation do it |
| book exists, none in stock | 409 | request is valid, current *state* forbids it |
| user not logged in | 401 + `WWW-Authenticate` | missing/invalid credentials |
| logged in, but account suspended | 403 | identity known, not allowed |
| payment provider timed out | 502 or 503 | upstream failure, retry may work |
| your code raised `KeyError` | 500 | a bug |

A client can treat 4xx as "don't retry without changing something" and 5xx (other than
501) as "maybe retry later". Returning 500 for out-of-stock, or 200 with
`{"error": ...}`, breaks that contract.

## How It Actually Works

Starlette builds the app as a stack of ASGI layers. From outside in:

1. **`ServerErrorMiddleware`** — the outermost layer. It catches any exception that
   escapes everything else. If you registered a handler for `Exception` (or status
   `500`), it's called here; otherwise it sends the plain-text 500. Either way, it then
   **re-raises** the exception so the server can log it — which is the second traceback
   you saw, logged by Uvicorn as `Exception in ASGI application`.
2. Any user middleware you've added (Level 3 lesson 4).
3. **`ExceptionMiddleware`** — holds every other handler you registered, keyed by
   exception class or status code. When an exception arrives it walks the exception's
   MRO (`OutOfStock` → `Exception` → …) to find the most specific handler.
4. The router and your endpoint.

The tracebacks in this run also showed the same exception-handling wrapper applied
again around each route, so a handled exception from inside a route doesn't have to
unwind through other layers first.

`RequestValidationError` is raised by FastAPI's route handler after it fails to
validate parameters, before your function is called. FastAPI registers default handlers
for it and for `HTTPException` when the app is created; your
`@app.exception_handler(...)` calls replace those entries.

Because exception handlers run *outside* your endpoint, they're the right place for
cross-cutting translation, but they can't see local variables. Put anything a handler
needs on the exception object, like `OutOfStock.book_id`.

## Common mistakes

- **`return HTTPException(...)`** instead of `raise`. Tested: the client got status
  200 with the body `{"status_code":404,"detail":"x","headers":null}` — the exception
  object serialized as data.
- **Catching `Exception` inside endpoints** and returning `{"error": str(e)}`. You leak
  internals and turn bugs into 200s. Let unexpected exceptions reach the 500 handler.
- **Registering for `fastapi.HTTPException` and expecting it to catch router 404s.**
  Use `starlette.exceptions.HTTPException`.
- **Dropping `exc.headers`** in a custom handler, which loses `WWW-Authenticate` and
  `Allow`.
- **Putting secrets or stack traces in `detail`.** Anything in `detail` goes to the
  client.
- **Inconsistent shapes**: `{"detail": ...}` from some endpoints, `{"error": ...}` from
  others. Pick one shape and use handlers to enforce it.

## Exercise

1. Create domain exceptions `BookNotFound` and `DuplicateIsbn`, raise them from a small
   `catalogue.py` module with no FastAPI imports, and map them to 404 and 409 with
   handlers.
2. Write a single error shape `{"error": {"code": ..., "message": ...}}` and make 404s,
   405s, 409s and 422s all use it. For 422, include a list of `{"field": ..., "message": ...}`
   built from `exc.errors()`.
3. Add the `Exception` handler and confirm with `curl -i` that a crash returns JSON
   without any internal detail. Find both tracebacks in the server log.
4. Write down, for an endpoint `POST /orders`, which situations produce 400, 401, 403,
   404, 409, 422 and 503 in your design.
