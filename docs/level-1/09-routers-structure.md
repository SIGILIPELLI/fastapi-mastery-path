---
description: "Split a FastAPI app into APIRouter modules with prefixes and tags, lay out a package that scales, and avoid circular imports and route clashes."
---

# 09 · APIRouter & Project Structure

A single `main.py` is perfect for the first hundred lines. After that, books, authors,
users and orders all end up in one file, every change touches it, and imports start to
tangle. `APIRouter` lets you split endpoints into modules, each a mini-app with its own
prefix, tags and defaults, and then plug them into the main app.

## A layout that scales

```text
bookshop/
├── app/
│   ├── __init__.py
│   ├── main.py            # creates the FastAPI app, includes routers
│   ├── schemas.py         # Pydantic models
│   ├── store.py           # data access (a dict now, a database in Level 2)
│   └── routers/
│       ├── __init__.py
│       ├── authors.py
│       └── books.py
└── tests/
```

The rule that keeps this healthy: **dependencies point one way**. Routers import schemas
and the store; the store imports schemas; `main.py` imports routers. Nothing imports
`main.py`. That single direction is what prevents circular imports.

Here are the files used for this lesson.

```python
# app/schemas.py
from pydantic import BaseModel


class BookIn(BaseModel):
    title: str
    author_id: int


class Book(BookIn):
    id: int


class Author(BaseModel):
    id: int
    name: str
```

```python
# app/store.py
"""In-memory data store. Replaced by a database in Level 2."""
from app.schemas import Author, Book

AUTHORS: dict[int, Author] = {1: Author(id=1, name="Frank Herbert")}
BOOKS: dict[int, Book] = {1: Book(id=1, title="Dune", author_id=1)}
```

```python
# app/routers/books.py
from fastapi import APIRouter, HTTPException, status

from app import store
from app.schemas import Book, BookIn

router = APIRouter(prefix="/books", tags=["books"])


@router.get("", response_model=list[Book])
def list_books():
    return list(store.BOOKS.values())


@router.get("/{book_id}", response_model=Book,
            responses={404: {"description": "No such book"}})
def get_book(book_id: int):
    if book_id not in store.BOOKS:
        raise HTTPException(404, "Book not found")
    return store.BOOKS[book_id]


@router.post("", response_model=Book, status_code=status.HTTP_201_CREATED)
def create_book(data: BookIn):
    if data.author_id not in store.AUTHORS:
        raise HTTPException(422, "Unknown author_id")
    book = Book(id=max(store.BOOKS, default=0) + 1, **data.model_dump())
    store.BOOKS[book.id] = book
    return book
```

```python
# app/routers/authors.py
from fastapi import APIRouter, HTTPException

from app import store
from app.schemas import Author, Book

router = APIRouter(prefix="/authors", tags=["authors"])


@router.get("/{author_id}", response_model=Author)
def get_author(author_id: int):
    if author_id not in store.AUTHORS:
        raise HTTPException(404, "Author not found")
    return store.AUTHORS[author_id]


@router.get("/{author_id}/books", response_model=list[Book])
def author_books(author_id: int):
    return [b for b in store.BOOKS.values() if b.author_id == author_id]
```

```python
# app/main.py
from fastapi import FastAPI

from app.routers import authors, books

app = FastAPI(title="Bookshop API")
app.include_router(books.router, prefix="/api/v1")
app.include_router(authors.router, prefix="/api/v1")


@app.get("/health", include_in_schema=False)
def health():
    return {"status": "ok"}
```

Run it from the `bookshop/` directory with `fastapi dev app/main.py`; the CLI finds the
`app` package and imports `app.main:app`.

Note `from app import store` followed by `store.BOOKS`, rather than
`from app.store import BOOKS`. Both work for a dict you mutate, but the module form
always reads the current attribute, which matters when a test replaces `store.BOOKS`
with a fresh dict.

## Prefixes stack

The router says `/books`; `include_router` adds `/api/v1`. The final paths:

```text
[{'title': 'Dune', 'author_id': 1, 'id': 1}]                                         # GET  /api/v1/books
{'title': 'Children of Dune', 'author_id': 1, 'id': 2}                               # POST /api/v1/books
[{'title': 'Dune', 'author_id': 1, 'id': 1}, {'title': 'Children of Dune', ...}]      # GET  /api/v1/authors/1/books
```

(The comments were added; the last line is trimmed.) The same router could be included
twice with different prefixes — `/api/v1` and `/api/v2` — which Level 4 lesson 6 uses
for versioning.

### The trailing slash

The list route is declared with an empty path `""`, so the full path is `/api/v1/books`
with no trailing slash. A request to `/api/v1/books/`:

```text
followed by the test client: 200
not followed:                307  location: http://testserver/api/v1/books
```

Starlette redirects to the slash-less version (`redirect_slashes=True` by default). It
works, but it's an extra round-trip, and some clients refuse to repeat a `POST` body
after a redirect. Pick one style, document it, and have clients use it exactly.

## Tags and per-router defaults

`tags=["books"]` groups the endpoints in `/docs`. The tags on the generated operations:

```text
['books', 'books', 'books', 'authors', 'authors']
```

`/health` was registered with `include_in_schema=False`, so it doesn't appear in OpenAPI
at all (`"/health" in spec["paths"]` was `False`) — useful for probe endpoints that
clients shouldn't call.

`APIRouter` also accepts defaults applied to every route in it:

```python
router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(require_admin)],          # runs before every endpoint (Level 2)
    responses={401: {"description": "Not logged in"}},
)
```

Putting an auth dependency on the router rather than on each endpoint means a new admin
endpoint can't be added without it by accident.

## Reversing URLs

Each route has a name — the function name by default. Build URLs from names instead of
hard-coding strings:

```python
print(app.url_path_for("get_book", book_id=7))
```

```text
/api/v1/books/7
```

The prefixes are included. Inside an endpoint, `request.url_for("get_book", book_id=7)`
returns an absolute URL, which is what you want in a `Location` header. Function names
must then be unique across the app — two routers each with a `get_item` function would
collide.

## Worked example: adding a route after including the router

In FastAPI 0.143.0 (used for this lesson), `app.routes` doesn't contain copies of the
router's routes. Printing route types showed two wrapper objects:

```text
starlette.routing Route /openapi.json
starlette.routing Route /docs
starlette.routing Route /docs/oauth2-redirect
starlette.routing Route /redoc
fastapi.routing _IncludedRouter None
fastapi.routing _IncludedRouter None
fastapi.routing APIRoute /health
```

Because the wrapper refers to the live router, a route added to `books.router` **after**
`include_router` was still reachable and appeared in OpenAPI:

```python
@books.router.get("/stats/summary")
def late():
    return "added after include"
```

```text
'added after include'
True
```

Older FastAPI releases copied each route into the app at `include_router` time, so the
same code would have produced a 404 there. Two lessons from this:

- **Register all routes before including the router.** It works on every version.
- **Don't write code that walks `app.routes` expecting `APIRoute` objects**; it depends
  on internals that changed. Use `app.openapi()["paths"]` or `url_path_for` instead.

A first attempt at this experiment used the path `/late`, which returned a 422 — it was
added after `/{book_id}`, so `/books/late` matched that route first and `"late"` failed
integer validation. Route order (lesson 3) applies inside routers too.

## How It Actually Works

An `APIRouter` is a collection of routes plus default settings (prefix, tags,
dependencies, responses, response class). Calling `app.include_router(router, prefix=...)`
records the router along with an **include context** — the combined prefix, tags and
dependencies from every level of inclusion. Routers can include other routers, and the
contexts combine, which is how `/api/v1` + `/books` + `/{book_id}` becomes one path.

On the version read for this lesson, the included router is represented by an
`_IncludedRouter` object that computes the "effective" routes (each original route with
the combined context applied) lazily, caches them, and recomputes when the original
router's route list changes — the source tracks a routes version number for this. When a
request arrives, the app's router walks its route list in order; an included router
contributes its effective routes at its position in that list, so registration order
still decides which route wins.

Prefixes must start with `/` and must not end with `/`. Both `APIRouter(prefix=...)` and
`include_router(..., prefix=...)` check this immediately, raising
`AssertionError: A path prefix must start with '/'` or
`AssertionError: A path prefix must not end with '/', as the routes will start with '/'`. Route paths inside a prefixed router can be `""`, as the list
endpoint shows.

## Common mistakes

- **Circular imports**: a router importing from `main.py` (to get `app` or a setting).
  Move shared things into their own module — `config.py`, `deps.py` — that both import.
- **Duplicate prefixes**: `APIRouter(prefix="/books")` *and*
  `include_router(..., prefix="/books")` gives `/books/books`.
- **Mixed trailing-slash styles**, leading to redirects and confused clients.
- **Duplicate function names** across routers, which breaks `url_path_for`.
- **A `routers/` directory full of business logic.** Keep endpoints thin: parse, call a
  function in a service or store module, return. That's what makes the store swappable
  for a database next level.
- **Relying on `app.routes` internals** for tooling, as shown above.

## Exercise

1. Recreate the layout above. Add a `routers/reviews.py` with `GET /books/{id}/reviews`
   and `POST /books/{id}/reviews`, using a prefix of `/books/{book_id}/reviews` on the
   router. Does a path parameter work inside a router prefix?
2. Include the books router a second time under `/api/v2`. Look at `/docs` and the
   `operationId`s. What problem do duplicate operation IDs cause for client generators?
3. Add `include_in_schema=False` to a `/debug/routes` endpoint that returns the sorted
   keys of `app.openapi()["paths"]`. (Hint: you'll need the `Request` object to reach
   `request.app`.)
4. Deliberately create a circular import between `main.py` and a router, read the
   `ImportError`, and fix it by moving the shared object into a new module.
