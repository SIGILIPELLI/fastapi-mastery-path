---
description: "Control what a FastAPI API returns with response models and return types: filtering secrets, exclude_none, status codes, 204s and validation errors."
---

# 05 · Response Models & Status Codes

Lesson 4 was about what clients may send. This lesson is about what you send back, which
matters more than it first appears: an endpoint that returns "whatever the function
returned" will one day return a password hash, an internal cost price, or a field you
renamed without telling anyone.

A **response model** is a declared output shape. FastAPI uses it to:

1. **filter** the returned data down to the declared fields;
2. **validate** it, so a bug in your code produces an error on your side rather than a
   malformed payload on the client's;
3. **document** it in OpenAPI, so clients and generated SDKs know exactly what to expect.

## Separate input, output and storage models

The classic case is a user account:

```python
from datetime import datetime, timezone
from fastapi import FastAPI, status
from pydantic import BaseModel, EmailStr

app = FastAPI()

class UserIn(BaseModel):          # what a client sends to sign up
    email: EmailStr
    password: str

class UserOut(BaseModel):         # what any client may see
    id: int
    email: EmailStr

class UserDB(UserOut):            # what we store
    password_hash: str
    created_at: datetime

USERS: dict[int, UserDB] = {}

@app.post("/users", response_model=UserOut, status_code=status.HTTP_201_CREATED)
def create_user(user: UserIn):
    db = UserDB(id=len(USERS) + 1, email=user.email,
                password_hash="not-a-real-hash:" + user.password[::-1],
                created_at=datetime(2026, 10, 8, tzinfo=timezone.utc))
    USERS[db.id] = db
    return db          # a UserDB, filtered down to UserOut
```

(The "hash" here is a deliberately fake placeholder for the example; Level 3 lesson 1
uses a real password hasher.) The function returns the full `UserDB` object, but the
client receives only the `UserOut` fields:

```text
POST /users -> 201 {'id': 1, 'email': 'ada@example.com'}
```

Now the same object from an endpoint *without* a response model:

```python
@app.get("/users-leaky/{user_id}")
def leaky(user_id: int):
    return USERS[user_id]
```

```text
GET /users-leaky/1 -> 200 {'id': 1, 'email': 'ada@example.com', 'password_hash': 'not-a-real-hash:!terc3s', 'created_at': '2026-10-08T00:00:00Z'}
```

Every field went out, including the hash. This is one of the most common real-world API
data leaks, and a response model prevents it structurally.

`EmailStr` (it needs the `email-validator` package, included in `fastapi[standard]`)
validates on the way in too:

```text
POST /users -> 422 {'detail': [{'type': 'value_error', 'loc': ['body', 'email'], 'msg': 'value is not a valid email address: An email address must have an @-sign.', ...}]}
```

## Return type annotations

Instead of `response_model=`, you can annotate the return type:

```python
@app.get("/users/{user_id}")
def get_user(user_id: int) -> UserOut:
    return USERS[user_id]
```

```text
GET /users/1 -> 200 {'id': 1, 'email': 'ada@example.com'}
```

FastAPI treats the annotation as the response model. It still filters: a `UserDB` was
returned, a `UserOut` was sent. Which form to use?

- Use the **return annotation** when the function really returns that type; type
  checkers like mypy can then verify your code.
- Use **`response_model=`** when the function returns something else on purpose (a dict,
  an ORM object, a richer internal model) and you want FastAPI to convert it. If both
  are present, `response_model` wins.

## When the response is wrong, it's your bug

```python
@app.get("/books-bad")
def bad() -> Book:
    return {"id": "not-an-int", "title": "Dune"}
```

The client gets a bare `500 Internal Server Error`, and the server raises:

```text
fastapi.exceptions ResponseValidationError
1 validation error:
  {'type': 'int_parsing', 'loc': ('response', 'id'), 'msg': 'Input should be a valid integer, unable to parse string as an integer', 'input': 'not-an-int'}

  Endpoint: GET /books-bad
```

This is deliberately different from request validation. A bad request is the client's
fault (422, with details). A bad response is the server's fault (500, details only in
your logs, because they could expose internals).

## Trimming the output

```python
class Book(BaseModel):
    id: int
    title: str
    subtitle: str | None = None
    tags: list[str] = []

@app.get("/books/{book_id}", response_model=Book, response_model_exclude_none=True)
def book(book_id: int):
    return {"id": book_id, "title": "Dune", "subtitle": None,
            "tags": ["classic"], "internal_cost": 3.2}
```

```text
GET /books/7 -> 200 {'id': 7, 'title': 'Dune', 'tags': ['classic']}
```

`internal_cost` was filtered because it isn't in `Book`; `subtitle` was dropped because
it was `None`. The related options are `response_model_exclude_unset` (omit fields that
were never set) and `response_model_exclude_defaults`. Use them sparingly: a client that
sometimes sees `"subtitle": null` and sometimes no key at all has to handle both. A
stable shape is usually kinder.

## Status codes

Pick the code that tells the client what happened:

| Code | Meaning | Typical use |
|------|---------|-------------|
| 200 OK | success with a body | `GET`, most `PUT`/`PATCH` |
| 201 Created | a new resource exists | `POST` that creates |
| 204 No Content | success, nothing to say | `DELETE` |
| 400 / 422 | the request was wrong | bad input (422 is FastAPI's default) |
| 404 Not Found | no such resource | unknown ID |
| 409 Conflict | the state forbids it | duplicate email |

Set the default per route with `status_code=` (an int, or a named constant from
`fastapi.status` for readability). For `204`, return nothing:

```python
@app.delete("/books/{book_id}", status_code=204)
def delete(book_id: int):
    return None
```

```text
DELETE /books/7 -> 204 ''
```

The body was empty and no `content-length` header was sent, as HTTP requires for 204.

## Worked example: an upsert that picks its status at runtime

`PUT /books/{id}` creates the book if it doesn't exist (201) or replaces it (200). The
status isn't known until the function runs, so inject the `Response` object and set it:

```python
from fastapi import Response

@app.put("/books/{book_id}", response_model=Book)
def upsert(book_id: int, book: Book, response: Response):
    created = book_id not in (1, 2)        # pretend IDs 1 and 2 exist
    response.status_code = 201 if created else 200
    response.headers["Location"] = f"/books/{book_id}"
    return book
```

```text
PUT /books/9 -> 201 {'id': 9, 'title': 'New', 'subtitle': None, 'tags': []} {'location': '/books/9'}
PUT /books/1 -> 200 {'id': 1, 'title': 'Old', 'subtitle': None, 'tags': []} {'location': '/books/1'}
```

The `Response` parameter is a temporary object; FastAPI copies its status code and
headers onto the real response it builds from your return value.

## How It Actually Works

At startup, FastAPI turns the response model into a Pydantic **field** used purely for
output. Per request, after your function returns:

1. `serialize_response` (in `fastapi/routing.py`) calls `field.validate(...)` on the
   return value with the location prefix `("response",)` — that's where the
   `('response', 'id')` in the error above comes from. Validation reads attributes as
   well as dict keys, so a plain object works too: a class instance with `id`, `name`
   and `secret` attributes, returned from an endpoint with a two-field response model,
   came out as `{"id":1,"name":"x"}`. For a `def` endpoint, this validation is run in
   the thread pool, like the endpoint itself.
2. If there are errors, it raises `ResponseValidationError`. That isn't handled by the
   422 handler, so it surfaces as a 500.
3. Otherwise the validated value is serialized with the `exclude_*` options applied —
   on the version read for this lesson, optionally straight to JSON bytes by Pydantic —
   and sent with the route's status code (or the one you set on the injected
   `Response`).

Filtering is a consequence of validation: the output model simply has no slot for
`password_hash`, so it never makes it into the dump.

In OpenAPI, the response is documented as a reference to a named schema:

```json
{
 "description": "Successful Response",
 "content": {
  "application/json": {
   "schema": {
    "$ref": "#/components/schemas/UserOut"
   }
  }
 }
}
```

If you return a `Response` object directly (for example `JSONResponse(...)`), FastAPI
skips all of this — no validation, no filtering. That's an escape hatch, not a default.

## Common mistakes

- **No response model on endpoints that return stored records.** Sooner or later an
  internal field leaks.
- **One model for everything.** `User` used for input, output and storage means the
  password is either missing from storage or present in output. Split them.
- **Returning a `JSONResponse` "to be safe"**, which bypasses filtering entirely.
- **`200` for creation and `200` with `{"error": ...}` for failures.** Clients and
  monitoring rely on status codes; use them.
- **A body on 204.** Return `None`; don't return `{}`.
- **Leaning on `exclude_none` to hide data.** It hides `None` values, not sensitive ones.
  Filtering belongs in the model's field list.

## Exercise

1. Add `UserPublic` (only `id` and a display name) and `UserPrivate` (adds `email`).
   Create `GET /users/{id}` returning `UserPublic` and `GET /me` returning `UserPrivate`.
2. Deliberately return a dict with a wrong type from an endpoint that has a response
   model. Confirm the client sees a 500 and find the `ResponseValidationError` in the
   server logs.
3. Implement `DELETE /users/{id}` returning 204, and 404 when the user doesn't exist
   (look ahead to lesson 7 for `HTTPException`).
4. Open `/docs` and compare how the schema for `POST /users` describes the request body
   and the response. Which model appears in each?
