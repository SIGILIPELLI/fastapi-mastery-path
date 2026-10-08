---
description: "Declare path and query parameters with types, Annotated, Path() and Query() constraints, enums, lists and query models, and avoid route-order bugs."
---

# 03 · Path & Query Parameters

Every HTTP request carries data in its URL before it carries anything else. FastAPI
splits that data into two kinds:

- **Path parameters** are part of the resource's identity: `/books/42`.
- **Query parameters** modify how you look at a resource: `/books?limit=5&in_stock=true`.

A useful rule of thumb when designing an API: if removing the value would make the URL
point at a *different thing*, it belongs in the path. If it filters, sorts, paginates or
toggles, it belongs in the query.

## Path parameters

A name in braces in the path template becomes a path parameter. Its type hint controls
conversion and validation:

```python
from typing import Annotated
from fastapi import FastAPI, Path

app = FastAPI()

@app.get("/books/{book_id}")
def get_book(book_id: Annotated[int, Path(ge=1, description="Catalogue ID")]):
    return {"book_id": book_id}
```

`Annotated[int, Path(...)]` reads as "an `int`, with this extra metadata". `Path(ge=1)`
adds a constraint: greater than or equal to 1. A request for `/books/0`:

```text
422 {'detail': [{'type': 'greater_than_equal', 'loc': ['path', 'book_id'], 'msg': 'Input should be greater than or equal to 1', 'input': '0', 'ctx': {'ge': 1}}]}
```

The `ctx` key carries the constraint value, so a client can build its own message
without parsing English text.

!!! tip "Why `Annotated`?"
    Older FastAPI code writes `book_id: int = Path(ge=1)`. That still works, but the
    `Annotated` form is what the FastAPI docs now recommend: the default value slot stays
    free for a real default, the function can be called normally from other Python code,
    and the same `Annotated` type can be reused across endpoints as a type alias.

### Constraining to a fixed set of values

Use an `Enum` (or `Literal`) when only certain values make sense:

```python
from enum import Enum

class Genre(str, Enum):
    scifi = "scifi"
    fantasy = "fantasy"
    crime = "crime"

@app.get("/genres/{genre}")
def by_genre(genre: Genre):
    return {"genre": genre, "is_enum": isinstance(genre, Genre), "value": genre.value}
```

```text
/genres/fantasy -> 200 {'genre': 'fantasy', 'is_enum': True, 'value': 'fantasy'}
/genres/horror -> 422 {'detail': [{'type': 'enum', 'loc': ['path', 'genre'], 'msg': "Input should be 'scifi', 'fantasy' or 'crime'", 'input': 'horror', 'ctx': {'expected': "'scifi', 'fantasy' or 'crime'"}}]}
```

Inside the function you get a real `Genre` member, not a string. Inheriting from `str`
makes it serialize back to `"fantasy"` in JSON, and the allowed values show up as a
dropdown in `/docs`.

### Paths that contain slashes

By default a path parameter matches one segment — it stops at the next `/`. To capture
the rest of the path, use Starlette's `:path` converter:

```python
@app.get("/files/{file_path:path}")
def get_file(file_path: str):
    return {"file_path": file_path}
```

```text
/files/covers/2024/dune.jpg -> 200 {'file_path': 'covers/2024/dune.jpg'}
```

Never pass a value like this straight to `open()`. `../../etc/passwd` is also a valid
path. Level 4 lesson 5 shows how to resolve and confine it.

## Query parameters

Any simple-typed parameter that isn't in the path template is a query parameter.
A default value makes it optional; no default makes it required.

```python
from fastapi import Query

@app.get("/books")
def search(
    q: Annotated[str | None, Query(min_length=2, max_length=50)] = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    in_stock: bool = False,
    tag: Annotated[list[str], Query()] = [],
):
    return {"q": q, "limit": limit, "in_stock": in_stock, "tag": tag}
```

A request that uses all four:

```text
/books?q=du&limit=5&in_stock=yes&tag=classic&tag=desert -> 200 {'q': 'du', 'limit': 5, 'in_stock': True, 'tag': ['classic', 'desert']}
```

### How booleans are parsed

Query strings are always text, so FastAPI has to decide what counts as true. These are
the real results for `?in_stock=<value>`:

| Value | Result |
|-------|--------|
| `true`, `1`, `on`, `yes` | `True` |
| `false`, `0`, `off`, `no` | `False` |
| `maybe` | `422 bool_parsing` |

Matching is case-insensitive: `TRUE` and `Yes` also gave `True` in a follow-up run.

### Lists need `Query()`

`tag: list[str] = []` *without* `Query()` would make FastAPI treat `tag` as a request
body, because a list isn't a "simple" type. Wrapping it in `Query()` says "repeat the key
in the query string": `?tag=classic&tag=desert`. A mutable default `[]` is safe here
because the default is copied for each request: an endpoint that appended to `tag` and
was called twice returned `['x']` both times, not `['x', 'x']`.

### Required query parameters

```python
@app.get("/required")
def required(page: int):
    return {"page": page}
```

```text
/required -> 422 {'detail': [{'type': 'missing', 'loc': ['query', 'page'], 'msg': 'Field required', 'input': None}]}
```

## Query parameters as a model

When an endpoint takes many query parameters, group them in a Pydantic model and mark
it with `Query()`:

```python
from typing import Literal
from pydantic import BaseModel, Field

class BookFilter(BaseModel):
    model_config = {"extra": "forbid"}
    q: str | None = None
    limit: int = Field(20, ge=1, le=100)
    order_by: Literal["title", "year"] = "title"

@app.get("/books")
def books(f: Annotated[BookFilter, Query()]):
    return f
```

```text
/books?limit=3&order_by=year -> 200 {'q': None, 'limit': 3, 'order_by': 'year'}
/books?order_by=price -> 422 {'detail': [{'type': 'literal_error', 'loc': ['query', 'order_by'], 'msg': "Input should be 'title' or 'year'", ...}]}
/books?colour=red -> 422 {'detail': [{'type': 'extra_forbidden', 'loc': ['query', 'colour'], 'msg': 'Extra inputs are not permitted', 'input': 'red'}]}
```

(The second line is trimmed.) `extra: "forbid"` turns unknown query keys into errors,
which catches typos like `?limt=5` that would otherwise be silently ignored. Query models
were added in FastAPI 0.115; on older versions you'd use a dependency instead (Level 2
lesson 1).

## Worked example: the route-order trap

Routes are matched **in the order they were registered**. Here `/users/me` is defined
after the parameterised route:

```python
@app.get("/users/{user_id}")
def user(user_id: int):
    return {"user_id": user_id}

@app.get("/users/me")
def me():
    return {"user": "current"}
```

```text
/users/me 422 {'detail': [{'type': 'int_parsing', 'loc': ['path', 'user_id'], 'msg': 'Input should be a valid integer, unable to parse string as an integer', 'input': 'me'}]}
```

The first route's pattern matched `/users/me` (the regex accepts any segment), so FastAPI
tried to validate `"me"` as an integer and failed — it never tried the second route.
Swap the order so the fixed path comes first, and both work:

```text
/users/me -> 200 {'user': 'current'}
/users/alice -> 200 {'user_id': 'alice'}
```

(That second output comes from a version where `user_id` is a `str`.)

## How It Actually Works

When a route is registered, Starlette compiles its path template into a regular
expression. `{book_id}` becomes a named group matching `[^/]+`; `{file_path:path}`
becomes `.*`. Starlette's own converters (`int`, `float`, `path`, `uuid`) can change the
regex, but FastAPI usually leaves the type work to Pydantic: the regex captures a string,
and Pydantic converts and validates it afterwards.

That two-step design explains the route-order trap. Matching happens on the regex, which
knows nothing about your `int` hint. Once a route matches, the router stops looking.
Validation failure doesn't send it back to try other routes — it produces a 422.

For the query string, Starlette parses `scope["query_string"]` into a multi-dict, where a
key can appear several times. FastAPI pulls a single value for scalar parameters (the
last occurrence — `?limit=5&limit=7` gave `7`) and all values for `list[...]` parameters, then validates them with the
same Pydantic machinery used for bodies. Validation runs in Pydantic's **lax** mode, which
is why `"5"` becomes `5` and `"yes"` becomes `True`. In strict mode those would be errors.

All the constraints you wrote (`ge`, `le`, `min_length`, the enum values) are also
written into the OpenAPI schema. This is the real entry for `limit`:

```json
{
 "name": "limit",
 "in": "query",
 "required": false,
 "schema": {
  "type": "integer",
  "maximum": 100,
  "minimum": 1,
  "default": 20,
  "title": "Limit"
 }
}
```

One declaration drives validation, conversion and documentation, so they can't drift
apart.

## Common mistakes

- **Fixed routes after parameterised ones**, as shown above. Put `/users/me` before
  `/users/{user_id}`.
- **Forgetting `Query()` on list parameters**, which turns them into a request body.
- **Using `str` for IDs that are really integers or UUIDs.** You lose validation and
  push errors deeper into your code. Use `int` or `uuid.UUID`.
- **No upper bound on `limit`.** `?limit=10000000` should be a 422, not a query that
  loads your whole table.
- **Treating `in_stock=false` as falsy text.** With a `bool` hint you get a real
  `False`; with a `str` hint you get the non-empty (and therefore truthy) string `"false"`.

## Exercise

1. Build `GET /authors/{author_id}/books` where `author_id` is an int ≥ 1 and the
   endpoint accepts `year_from` and `year_to` (optional ints) and `sort` (`"title"` or
   `"year"`). Return the parsed values.
2. Add a check that returns 422 when `year_from > year_to`. (Hint: a query model with a
   validator is one way; you'll meet validators properly in lesson 6.)
3. Register `/authors/top` and `/authors/{author_id}` in the wrong order, observe the
   failure with the test client, then fix it.
4. Look at `/openapi.json` and find where your `Literal` values were documented.
