---
description: "Accept JSON request bodies with Pydantic models: nested models, constraints, partial updates with exclude_unset, embedded values and content-type rules."
---

# 04 · Request Bodies with Pydantic Models

Path and query parameters are fine for identifiers and filters. Anything larger — a new
book, an order, a settings object — arrives as a **request body**, almost always JSON. In
FastAPI you describe the body's shape with a Pydantic model and take it as a parameter.

## A model is the contract

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()

class Author(BaseModel):
    name: str
    born: int | None = None

class BookIn(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    year: int = Field(ge=1450, le=2100)
    price: float = Field(gt=0)
    tags: list[str] = []
    author: Author

BOOKS = {1: {"title": "Dune", "year": 1965, "price": 9.99, "tags": [],
             "author": {"name": "Frank Herbert", "born": 1920}}}

@app.post("/books", status_code=201)
def create(book: BookIn):
    new_id = max(BOOKS) + 1
    BOOKS[new_id] = book.model_dump()
    return {"id": new_id, **BOOKS[new_id]}
```

Because `book` is typed as a `BaseModel` subclass, FastAPI reads it from the JSON body.
Inside the function, `book` is a fully validated `BookIn` instance: `book.author.name` is
guaranteed to be a string, `book.year` an integer in range.

A valid request:

```python
good = {"title": "Neuromancer", "year": 1984, "price": 12.5,
        "author": {"name": "William Gibson"}, "isbn": "ignored?"}
```

```text
POST /books -> 201 {'id': 2, 'title': 'Neuromancer', 'year': 1984, 'price': 12.5, 'tags': [], 'author': {'name': 'William Gibson', 'born': None}}
```

Three things happened that you didn't write code for:

- `tags` and `author.born` were filled from their defaults.
- The nested `author` dict became an `Author` instance.
- The unknown `isbn` key was **silently dropped**. That's Pydantic's default
  (`extra="ignore"`). Lesson 6 shows how to forbid extras instead, which catches client
  typos.

## Every error at once

Send a body that breaks four rules:

```python
{"title": "", "year": 1300, "price": -1, "author": {}}
```

```text
POST /books -> 422 {'detail': [
  {'type': 'string_too_short', 'loc': ['body', 'title'], 'msg': 'String should have at least 1 character', 'input': '', 'ctx': {'min_length': 1}},
  {'type': 'greater_than_equal', 'loc': ['body', 'year'], 'msg': 'Input should be greater than or equal to 1450', 'input': 1300, 'ctx': {'ge': 1450}},
  {'type': 'greater_than', 'loc': ['body', 'price'], 'msg': 'Input should be greater than 0', 'input': -1, 'ctx': {'gt': 0.0}},
  {'type': 'missing', 'loc': ['body', 'author', 'name'], 'msg': 'Field required', 'input': {}}]}
```

(Line breaks added for reading.) Pydantic doesn't stop at the first problem; it reports
all of them, each with a `loc` path into the body. A form on the client side can map
`['body', 'author', 'name']` straight onto the right input field.

## Conversion is lax by default

```python
{"title": "X", "year": "1999", "price": "4.50", "author": {"name": "A"}}
```

```text
POST /books -> 201 {'id': 3, 'title': 'X', 'year': 1999, 'price': 4.5, ...}
```

The strings `"1999"` and `"4.50"` were accepted and converted. That's convenient for
clients but can hide bugs; lesson 6 covers strict mode for when you want `"1999"` to be
an error.

## Broken JSON and wrong content types

Malformed JSON gets its own error type, with the character offset in `loc`:

```text
POST /books -> 422 {'detail': [{'type': 'json_invalid', 'loc': ['body', 17], 'msg': 'JSON decode error', 'input': {}, 'ctx': {'error': 'Expecting property name enclosed in double quotes'}}]}
```

The body was `{"title": "Oops",` — the parser hit end-of-input at offset 17 while
expecting another key.

The `Content-Type` header matters. The same valid JSON bytes were sent three ways:

| Content-Type | Result |
|---|---|
| *(none)* | `422 model_attributes_type` — body treated as a plain string |
| `text/plain` | `422 model_attributes_type` |
| `application/vnd.api+json` | `201` — any `+json` type is parsed as JSON |
| `application/x-www-form-urlencoded` with `title=Dune` | `422 model_attributes_type` |

So clients must send `Content-Type: application/json` (`curl` needs
`-H 'content-type: application/json'` when you use `-d`; `httpx` and `requests` set it
for you with `json=`). Older FastAPI versions were more lenient about a missing header;
don't rely on either behaviour — send the header.

## Body + path + query together

FastAPI sorts parameters by where they come from, so you can mix them freely:

```python
class BookPatch(BaseModel):
    title: str | None = None
    year: int | None = None
    price: float | None = None

@app.patch("/books/{book_id}")
def patch(book_id: int, changes: BookPatch, notify: bool = False):
    stored = BOOKS[book_id]
    stored.update(changes.model_dump(exclude_unset=True))
    return {"id": book_id, "notify": notify, **stored}
```

`book_id` is in the path template, `changes` is a model (body), and `notify` is a simple
type not in the path (query).

## Worked example: partial updates with `exclude_unset`

A `PATCH` should change only the fields the client sent. Every field in `BookPatch` is
optional, so after validation you can't tell "client sent nothing" from "client sent
`null`" just by looking at values. `model_dump(exclude_unset=True)` can tell, because
Pydantic records which fields were explicitly provided:

```text
PATCH /books/1?notify=true  body {"price": 7.99}
  exclude_unset: {'price': 7.99}
  full dump:     {'title': None, 'year': None, 'price': 7.99}
-> 200 {'id': 1, 'notify': True, 'title': 'Dune', 'year': 1965, 'price': 7.99, ...}
```

Using the full dump would have overwritten the title and year with `None`. Now look at
what happens when a client sends `null` on purpose:

```text
PATCH /books/1  body {"price": null}
  exclude_unset: {'price': None}
-> 200 {'id': 1, ..., 'price': None, ...}
```

The stored book now has no price — even though `BookIn` requires `price > 0`. The patch
model's `float | None` allowed `null`, and nothing re-checked the combined result. Two
fixes, depending on what you want:

- If `null` should never be allowed for a field, keep the type non-nullable but still
  optional to send, and reject explicit `None` with a validator (lesson 6).
- Better: after merging, validate the result against the full model:
  `BookIn.model_validate({**stored, **changes.model_dump(exclude_unset=True)})`. That
  turns the merged record back into something guaranteed to be valid. Run against the
  `{"price": null}` patch, it raised a `ValidationError` whose first error was type
  `float_type` at `('price',)` — catch it and return a 422.

## Single values and multiple bodies

A lone scalar parameter would be read from the query string. To take it from the body,
use `Body()`. With `embed=True` the client wraps it in an object:

```python
from typing import Annotated
from fastapi import Body

@app.post("/books/{book_id}/rating")
def rate(book_id: int, stars: Annotated[int, Body(ge=1, le=5, embed=True)]):
    return {"book_id": book_id, "stars": stars}
```

```text
POST /books/1/rating  body {"stars": 4}  -> 200 {'book_id': 1, 'stars': 4}
POST /books/1/rating  body 4             -> 422 {'detail': [{'type': 'missing', 'loc': ['body', 'stars'], ...}]}
```

When an endpoint takes **two** models, FastAPI automatically expects each under its
parameter name:

```python
@app.post("/transfer")
def transfer(book: BookPatch, author: Author):
    return {"book": book, "author": author}
```

```text
POST /transfer  body {"book": {"title": "T"}, "author": {"name": "N"}}
-> 200 {'book': {'title': 'T', 'year': None, 'price': None}, 'author': {'name': 'N', 'born': None}}
```

## How It Actually Works

When FastAPI analyses the endpoint at startup, it collects every body parameter. If there's
exactly one and it isn't `embed=True`, the whole JSON document is validated against that
one model. If there are several (or one embedded), FastAPI builds a **combined model** on
the fly with one field per parameter and validates the body against that — which is why
`/transfer` expects `{"book": …, "author": …}`.

Per request, the route reads the body only if the endpoint declares body parameters:

1. It awaits the full body from the ASGI `receive` channel. (FastAPI doesn't stream JSON
   bodies; the whole thing is in memory before parsing.)
2. If the content type is JSON (`application/json` or any `*/*+json`), it parses with
   `json.loads`. A `JSONDecodeError` becomes the `json_invalid` error you saw, with the
   decoder's position. Otherwise the raw bytes are passed on, and validating bytes
   against a model fails with `model_attributes_type`.
3. The parsed value goes to Pydantic's core validator, written in Rust, which walks the
   model's schema, converts types, applies constraints and collects every error.
4. On success your function receives model instances. Each instance carries a
   `model_fields_set` attribute: the names of fields that were present in the input.
   `exclude_unset=True` simply filters the dump to those names.

## Common mistakes

- **Using the full `model_dump()` for a PATCH** and wiping fields the client didn't send.
- **Assuming extra keys are rejected.** By default they're dropped. If a client sends
  `"titel"` instead of `"title"`, a model with a default for `title` will happily accept
  it.
- **Reusing the create model for updates.** Required fields on create are optional on
  update; use separate models (`BookIn`, `BookPatch`) and later a shared base class.
- **Using `GET` with a body.** HTTP doesn't forbid it, but many proxies, caches and client
  libraries drop or reject it. Use query parameters for `GET`.
- **Forgetting `Content-Type: application/json`** in `curl` or hand-written clients.
- **No size limits.** `list[str]` with no `max_length` accepts a million tags. Lesson 6
  and Level 4 lesson 5 cover limits.

## Exercise

1. Add `POST /authors` that takes an `Author` with `name` (1–100 chars) and `born`
   (optional, between 1000 and the current year). Return it with a generated ID.
2. Write a `PATCH /authors/{id}` that uses `exclude_unset=True` and then re-validates the
   merged record against the full model. Show with the test client that
   `{"name": null}` now returns a 422.
3. Send the same body with and without a `Content-Type` header using `curl -d` and
   explain the difference you see.
4. Make `POST /books` reject unknown keys and confirm that `isbn` now produces an
   `extra_forbidden` error. (Look ahead to lesson 6 for `model_config`.)
