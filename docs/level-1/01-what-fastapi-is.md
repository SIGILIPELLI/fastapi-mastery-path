---
description: "What FastAPI actually is: an ASGI app built on Starlette and Pydantic, and how type hints become validation, conversion and OpenAPI docs."
---

# 01 · What FastAPI Is: ASGI, Starlette, Pydantic and One Request

FastAPI is a Python framework for building HTTP APIs. You write ordinary functions, add
type hints to their parameters, and FastAPI uses those hints for three jobs at once:

1. **Parsing and converting** incoming data (the string `"42"` in a URL becomes the
   integer `42`).
2. **Validating** it, and answering with a structured `422` error when it's wrong.
3. **Documenting** it, by generating an OpenAPI schema and an interactive docs page.

That's the whole pitch, and it's why the framework feels small. Most of what you'll learn
in this course is how those three jobs work in detail, and what FastAPI deliberately leaves
to you: database access, background job queues, deployment and security policy.

## The three layers

FastAPI is not one big library. It's a thin layer that joins two others:

| Layer | What it does | You notice it when… |
|-------|--------------|----------------------|
| **ASGI server** (Uvicorn) | Owns the socket, speaks HTTP/1.1 or WebSocket, turns bytes into an event dictionary | you start the app, tune workers, sit behind a proxy |
| **Starlette** | Routing, request/response objects, middleware, WebSockets, background tasks, test client | you add middleware, stream a response, mount a sub-app |
| **Pydantic** | Data validation and serialization from type hints | you define models, write validators, read a 422 error |
| **FastAPI** | Reads your function signatures, wires Starlette and Pydantic together, dependency injection, OpenAPI | you write `@app.get(...)`, `Depends(...)` |

You can confirm the layering yourself. `FastAPI` is literally a subclass of `Starlette`:

```python
import fastapi, starlette, pydantic
from fastapi import FastAPI

app = FastAPI()
print("versions:", fastapi.__version__, starlette.__version__, pydantic.VERSION)
print("MRO:", [c.__name__ for c in type(app).__mro__][:3])
```

Output from the run used for this lesson:

```text
versions: 0.143.0 1.7.0 2.14.0
MRO: ['FastAPI', 'Starlette', 'object']
```

Your version numbers will differ — FastAPI releases often and still uses `0.x` numbering,
so pin versions in real projects (Level 4 lesson 9 covers upgrading).

## ASGI without any framework

To understand what FastAPI sits on, write an ASGI app by hand. ASGI (Asynchronous Server
Gateway Interface) is a specification: an application is an `async` callable that takes
three arguments.

```python
# raw_asgi.py — a complete ASGI application with no framework at all.
async def app(scope, receive, send):
    if scope["type"] != "http":
        return
    body = f"{scope['method']} {scope['path']}\n".encode()
    await send({"type": "http.response.start", "status": 200,
                "headers": [(b"content-type", b"text/plain")]})
    await send({"type": "http.response.body", "body": body})
```

- `scope` is a dict describing the connection: `type` (`"http"`, `"websocket"` or
  `"lifespan"`), `method`, `path`, `query_string` (bytes), `headers` (a list of byte
  pairs), client and server addresses.
- `receive` is an awaitable that yields events from the client, such as chunks of the
  request body.
- `send` is an awaitable you call with events going back: first the status line and
  headers, then one or more body chunks.

Run it with Uvicorn and send it a request:

```bash
uvicorn raw_asgi:app --port 8701
curl -i 'localhost:8701/hello?x=1'
```

```text
HTTP/1.1 200 OK
date: Thu, 08 Oct 2026 16:12:29 GMT
server: uvicorn
content-type: text/plain
transfer-encoding: chunked

GET /hello
```

Notice what you *didn't* get: no query-string parsing (`?x=1` is still raw bytes in
`scope["query_string"]`), no JSON, no routing, no error handling. Everything FastAPI does
is built on top of exactly this interface.

## The same idea, in FastAPI

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/books/{book_id}")
def read_book(book_id: int, q: str | None = None):
    return {"book_id": book_id, "q": q}
```

From the signature alone FastAPI decided that:

- `book_id` appears in the path template, so it's a **path parameter**, and it must be an
  `int`.
- `q` doesn't appear in the path and has a simple type, so it's a **query parameter**;
  because it has a default of `None` it's optional.
- The return value is a dict, so it gets serialized to JSON.

You can exercise it without starting a server, using the test client (more in Level 2
lesson 8):

```python
from fastapi.testclient import TestClient
from main import app

c = TestClient(app)
r = c.get("/books/42?q=dune"); print(r.status_code, r.json())
r = c.get("/books/forty-two"); print(r.status_code, r.json())
```

```text
200 {'book_id': 42, 'q': 'dune'}
422 {'detail': [{'type': 'int_parsing', 'loc': ['path', 'book_id'], 'msg': 'Input should be a valid integer, unable to parse string as an integer', 'input': 'forty-two'}]}
```

That `422` response is generated entirely by Pydantic. `loc` tells the client *where* the
problem was (`path` → `book_id`), `type` is a stable machine-readable code, and `input`
echoes what was received. You wrote no validation code.

!!! note "A warning you may see"
    On the versions used here, importing the test client printed
    `StarletteDeprecationWarning: Using httpx with starlette.testclient is deprecated; install httpx2 instead.`
    Installing the `httpx2` package made the warning go away. Older Starlette versions
    don't print it. If your versions differ, follow whatever the warning text says.

## Worked example: what the framework registered

The app object keeps a list of routes. Printing them shows that FastAPI added four routes
of its own:

```python
print([(type(r).__name__, r.path) for r in app.routes])
print(sorted(app.openapi()["paths"]))
```

```text
[('Route', '/openapi.json'), ('Route', '/docs'), ('Route', '/docs/oauth2-redirect'), ('Route', '/redoc'), ('APIRoute', '/books/{book_id}')]
['/books/{book_id}']
```

- `/openapi.json` serves the machine-readable description of your API.
- `/docs` serves Swagger UI, an interactive page that reads `/openapi.json`.
- `/redoc` is an alternative read-only documentation page.
- Your endpoint is an `APIRoute`, FastAPI's subclass of Starlette's `Route` that knows
  about parameters, dependencies and response models.

The built-in routes are plain Starlette `Route`s, which is why they don't appear in the
OpenAPI `paths` themselves.

## How It Actually Works

When you decorate a function with `@app.get("/books/{book_id}")`, FastAPI does most of its
work **once, at import time**, not on every request:

1. It calls `inspect.signature()` on your function and walks each parameter.
2. For each parameter it decides where the value comes from. The rules, in order: a name
   in the path template → path; a Pydantic model or something marked `Body()` → JSON body;
   anything declared with `Depends()` → a dependency; a simple scalar type → query string.
   Explicit markers (`Query()`, `Header()`, `Cookie()`, `Form()`, `File()`) override the
   defaults.
3. It builds a **dependant** tree, a description of everything this endpoint needs, and
   Pydantic validators for each parameter.
4. It wraps your function in a Starlette-compatible request handler and adds an
   `APIRoute` to the router.

Then, per request:

1. Uvicorn parses the HTTP bytes and calls `app(scope, receive, send)`.
2. Starlette's middleware stack runs (error handling, any middleware you've added), then
   the router compares `scope["path"]` against each route's compiled regex in order.
3. The matching `APIRoute` collects raw values: path params from the regex match, query
   params from `scope["query_string"]`, headers, cookies, and — if the endpoint needs a
   body — awaits `receive()` until the whole body has arrived and parses it.
4. Everything is validated with Pydantic in one pass. If anything fails, all errors are
   collected and FastAPI raises `RequestValidationError`, which its default handler turns
   into the `422` you saw.
5. Your function is called. If it's `async def` it is awaited on the event loop; if it's a
   plain `def`, FastAPI runs it in a thread pool so it can't block the loop (Level 2
   lesson 3 explains why this matters).
6. The return value is converted to JSON-compatible data (using a response model if you
   declared one) and wrapped in a `JSONResponse`, which calls `send()` twice — headers,
   then body — just like the raw ASGI app.

The OpenAPI schema is generated lazily the first time something calls `app.openapi()` and
then cached on the app object.

## Where FastAPI fits, and where it doesn't

FastAPI is a good fit for JSON APIs, internal services, ML model serving, and backends
for single-page or mobile apps. It is a weaker fit when you want a batteries-included
web application with server-rendered pages, an admin back office and an ORM chosen for
you — that's the space [Django](https://sigilipelli.github.io/django-mastery-path/)
occupies. FastAPI can render templates (it bundles Jinja2 support from Starlette), but
it's not its focus.

Things FastAPI doesn't include, which you'll add in later lessons:

- a database layer (we use SQLAlchemy 2.x in Level 2);
- migrations (Alembic);
- user accounts and password storage (built by hand in Level 3, using the security
  utilities FastAPI does provide);
- a job queue for work that outlives a request (Level 4 lesson 7).

## Common mistakes

- **Thinking type hints are only documentation.** In FastAPI they change behaviour.
  Changing `book_id: int` to `book_id: str` changes what's accepted and what the docs say.
- **Confusing the framework with the server.** `FastAPI()` creates an ASGI app; it doesn't
  listen on a port. Something else (Uvicorn, usually via `fastapi dev` or `fastapi run`)
  does.
- **Doing heavy work at request time that could be done at import time** — or the
  reverse: opening a database connection at import time, which breaks tests and
  multi-worker deployments. Level 3 lesson 5 covers the lifespan hook for this.
- **Ignoring the 0.x version.** Minor releases have changed defaults before. Pin
  `fastapi==X.Y.Z` in production and read release notes when upgrading.

## Exercise

1. Save the raw ASGI app above and run it with Uvicorn. Change it so that it returns JSON
   containing the raw `query_string` and the list of header names from `scope`. What
   type are the header names?
2. Write the FastAPI version of `/books/{book_id}` and use the test client to send
   `/books/3.5`, `/books/-1` and `/books/007`. Record which succeed and what `book_id`
   comes back. Can you explain `007`?
3. Add a second parameter `limit: int = 10` and look at `app.openapi()["paths"]` to see
   how it's described. Which fields in the schema came from your default value?
