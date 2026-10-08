---
description: "Advanced FastAPI structure: a consistent RFC 9457 error contract, custom APIRoute classes, mounted sub-applications with their own docs, and static files."
---

# 08 · Advanced Patterns: Sub-Applications, Custom Routes & Error Contracts

As an API grows, three structural problems show up: errors come back in different shapes
from different places; some behaviour belongs to a *group* of routes but not the whole
app; and parts of the system (an admin API, static assets) want different settings or
documentation. FastAPI has a tool for each — and each has a sharp edge, which this lesson
finds by testing.

## A single error contract: Problem Details

Level 1 lesson 7 noted that FastAPI's errors come in several shapes — `{"detail": "..."}`
for `HTTPException`, `{"detail": [...]}` for validation, plain text for crashes. Clients
end up special-casing each. **RFC 9457, "Problem Details for HTTP APIs"**, defines one
shape with its own media type, `application/problem+json`:

```json
{"type": "about:blank", "title": "Not Found", "status": 404, "detail": "No book 2", "instance": "/books/2"}
```

`type` is a URI identifying the problem kind (`about:blank` means "just the HTTP status"),
`title` a short summary, `status` the HTTP code, `detail` this occurrence, `instance`
where it happened; you may add your own members. Installing it for every error the app
can produce:

```python
PROBLEM = "application/problem+json"

def problem(status: int, title: str, detail: str | None = None, type_: str = "about:blank", **extra):
    body = {"type": type_, "title": title, "status": status}
    if detail: body["detail"] = detail
    body.update(extra)
    return JSONResponse(body, status_code=status, media_type=PROBLEM)

def install_problem_handlers(app: FastAPI):
    @app.exception_handler(StarletteHTTPException)
    async def http_problem(request: Request, exc: StarletteHTTPException):
        resp = problem(exc.status_code, title=_title(exc.status_code), detail=str(exc.detail), instance=request.url.path)
        for k, v in (exc.headers or {}).items(): resp.headers[k] = v
        return resp
    @app.exception_handler(RequestValidationError)
    async def validation_problem(request: Request, exc: RequestValidationError):
        errors = [{"field": ".".join(str(p) for p in e["loc"][1:]), "message": e["msg"], "type": e["type"]} for e in exc.errors()]
        return problem(422, "Validation failed", type_="https://errors.example.com/validation", errors=errors, instance=request.url.path)

def _title(code: int) -> str:
    from http import HTTPStatus
    return HTTPStatus(code).phrase
```

```text
404    : 404 application/problem+json {'type': 'about:blank', 'title': 'Not Found', 'status': 404, 'detail': 'No book 2', 'instance': '/books/2'}
422    : 422 {'type': 'https://errors.example.com/validation', 'title': 'Validation failed', 'status': 422, 'errors': [{'field': 'book_id', 'message': 'Input should be a valid integer, unable to parse string as an integer', 'type': 'int_parsing'}], 'instance': '/books/abc'}
405    : 405 {'type': 'about:blank', 'title': 'Method Not Allowed', 'status': 405, 'detail': 'Method Not Allowed', 'instance': '/books/1'} GET
no route: 404 {'type': 'about:blank', 'title': 'Not Found', 'status': 404, 'detail': 'Not Found', 'instance': '/nowhere'}
```

Every error — your own `HTTPException`, validation, wrong method (with its `Allow: GET`
header preserved), unknown path — now has one shape. Add the `Exception` handler from
Level 4 lesson 4 (returning a 500 problem with the request ID) to cover crashes too, and
document the shape once in OpenAPI with a shared `responses=` entry.

## Custom route classes

Middleware (Level 3 lesson 4) wraps the *whole app* and sees raw ASGI. A **custom
`APIRoute` subclass** wraps individual route handlers and works with `Request` and
`Response` objects — and it can be applied to just one router:

```python
class TimedRoute(APIRoute):
    def get_route_handler(self) -> Callable:
        original = super().get_route_handler()
        async def handler(request: Request) -> Response:
            start = time.perf_counter()
            response = await original(request)
            response.headers["X-Handler-Time-ms"] = f"{(time.perf_counter() - start) * 1000:.2f}"
            response.headers["X-Route"] = self.path
            return response
        return handler

books = APIRouter(prefix="/books", route_class=TimedRoute)

@books.get("/{book_id}")
def get_book(book_id: int):
    if book_id != 1: raise HTTPException(404, f"No book {book_id}")
    return {"id": 1, "title": "Dune"}

books = APIRouter(prefix="/books", route_class=TimedRoute)

@books.get("/{book_id}")
def get_book(book_id: int):
    if book_id != 1:
        raise HTTPException(404, f"No book {book_id}")
    return {"id": 1, "title": "Dune"}
```

```text
ok     : 200 {'id': 1, 'title': 'Dune'} {'x-handler-time-ms': '0.91', 'x-route': '/books/{book_id}'}
```

The route knows things middleware doesn't easily know, like its own path template
(`self.path`) and its response model. Typical uses: timing or auditing one group of
endpoints, reading the request body once for signature verification (webhooks), or
converting a specific exception type for one router. The handler only sees successful
responses here — the 404 above was raised inside `original(request)` and turned into a
response by the app's exception handlers further out, so it had no timing header.

## Sub-applications

`app.mount(path, other_asgi_app)` hands everything under a prefix to another complete
ASGI application — here a second `FastAPI` app with its own title and docs:

```python
admin_app = FastAPI(title="Bookshop Admin", docs_url="/docs", openapi_url="/openapi.json")
@admin_app.get("/stats")
def stats():
    return {"books": 1}

main = FastAPI(title="Bookshop API")
install_problem_handlers(main)
main.include_router(books)
main.mount("/admin", admin_app)
main.mount("/static", StaticFiles(directory="static"), name="static")
```

```text
sub-app: 200 {'books': 1}
main paths: ['/books/{book_id}']
admin paths: ['/stats'] | admin docs: 200
static : 200 text/css; charset=utf-8 True /static/site.css
```

- The main app's OpenAPI document doesn't include `/admin/...`; the admin app has its own
  at `/admin/openapi.json` and its own Swagger UI at `/admin/docs`. That's the main reason
  to mount instead of including a router: separate documentation for separate audiences
  (and the option to disable one of them).
- `StaticFiles` serves a directory, with `ETag` headers for caching; `url_path_for("static",
  path="site.css")` builds `/static/site.css`.

## Worked example: the sub-app doesn't share your error contract

```text
sub-app 404 (own handlers): 404 application/json {'detail': 'Not Found'}
```

A request for `/admin/nope` got FastAPI's *default* error shape, not a problem document.
A mounted app is a separate application: **its own exception handlers, its own
middleware, its own dependency overrides, its own lifespan** — none of the main app's
apply inside it. Run `install_problem_handlers(admin_app)` as well, add middleware to
both, and remember in tests that `main.dependency_overrides` doesn't reach the admin
app's routes. If you don't need separate docs or settings, prefer `include_router`,
which shares everything.

## How It Actually Works

**Exception handlers** are looked up by class through the MRO (Level 1 lesson 7), so one
handler for `starlette.exceptions.HTTPException` covers FastAPI's subclass and the
router's own 404/405. `RequestValidationError` isn't an `HTTPException`, hence its own
handler.

**Route classes**: when FastAPI builds the app, each `APIRoute` creates its ASGI handler
by calling `get_route_handler()`, which returns the async function that solves
dependencies, calls your endpoint and serializes the result. Overriding it lets you wrap
that function. `route_class=` on a router makes every route added to it an instance of
your class.

**Mounting** adds a `Mount` route. When a request matches its prefix, Starlette builds a
child scope with the prefix appended to `root_path` (`"root_path": root_path +
matched_path` in `starlette/routing.py`) and calls the sub-application with it. A test
endpoint in the admin app reported `{'path': '/admin/where', 'root_path': '/admin'}` —
`path` keeps the full value, and the sub-app's router matches only the part after
`root_path`. It's the same `root_path` mechanism as the reverse-proxy lesson, which is
why the admin app's docs know to load `/admin/openapi.json`, and why the sub-app's routes
are written without the prefix.

## Common mistakes

- **Several error shapes** across an API, or documenting one and returning another.
- **Assuming mounted apps inherit handlers, middleware or overrides.**
- **Mounting for code organisation** when `include_router` is what you need.
- **Reading the body in a route class or middleware** without making it available to the
  endpoint again (Starlette caches `await request.body()` on the `Request` object, but
  only for that object).
- **Changing response status or body in a route class** for errors — they're produced by
  exception handlers outside it.
- **Serving user uploads with `StaticFiles`** from a directory that also contains code
  or configuration.

## Exercise

1. Add a `500` problem handler that includes the request ID, and a shared OpenAPI
   `responses` entry describing the problem schema for 4xx/5xx on every route.
2. Write a `SignedWebhookRoute` route class that verifies an HMAC signature header over
   the raw body before calling the endpoint, and returns a 401 problem if it's wrong.
3. Install the problem handlers on the admin sub-app and write a test that asserts every
   404 in both apps has the `application/problem+json` content type.
4. Mount a second version of the API as a sub-app (`/v2`) and compare that with the router
   approach from lesson 6. Which would you choose, and why?
