---
description: "Read HTML form data, file uploads with UploadFile, headers and cookies in FastAPI, with size and type checks and secure cookie settings."
---

# 08 · Forms, File Uploads, Headers & Cookies

JSON isn't the only way data reaches an API. Browsers submit HTML forms as
`application/x-www-form-urlencoded`, uploads come as `multipart/form-data`, and a lot of
important information travels in headers and cookies. FastAPI reads all of these with the
same pattern you already know: a type hint plus a marker.

Form and file parsing needs the `python-multipart` package. It's included in
`fastapi[standard]`; with plain `fastapi`, declaring a `Form()` or `File()` parameter
raises an error that tells you to install it.

## Form fields

```python
from typing import Annotated
from fastapi import FastAPI, Form, Response

app = FastAPI()

@app.post("/login")
def login(username: Annotated[str, Form()],
          password: Annotated[str, Form(min_length=8)],
          response: Response):
    response.set_cookie("session", "abc123", httponly=True, secure=True,
                        samesite="lax", max_age=3600)
    return {"user": username}
```

(Hard-coded session value for illustration only; Level 3 builds real authentication.)

```text
login -> 200 {'user': 'ada'}
  set-cookie: session=abc123; HttpOnly; Max-Age=3600; Path=/; SameSite=lax; Secure
login short pw -> 422 {'detail': [{'type': 'string_too_short', 'loc': ['body', 'password'], 'msg': 'String should have at least 8 characters', ...}]}
```

Form fields are validated like any other parameter, and their errors use `loc`
`['body', ...]` because form data is part of the body. Sending the same data as JSON
fails — the endpoint expects form encoding:

```text
login as json -> 422 {'detail': [{'type': 'missing', 'loc': ['body', 'username'], ...}, {'type': 'missing', 'loc': ['body', 'password'], ...}]}
```

An endpoint can't accept both JSON and form data for the same parameter. If you need both,
make two endpoints or read the raw `Request` yourself.

### A form as a model

As with query parameters, a group of form fields can be a model:

```python
from pydantic import BaseModel

class Review(BaseModel):
    rating: int
    text: str = ""

@app.post("/reviews")
def review(data: Annotated[Review, Form()]):
    return data
```

```text
review form model -> 200 {'rating': 5, 'text': 'Great'}
```

The form sent `rating=5` as text and it arrived as an `int`.

## File uploads

```python
import hashlib
from fastapi import UploadFile, HTTPException

MAX_COVER = 2 * 1024 * 1024
ALLOWED = {"image/png", "image/jpeg"}

@app.post("/books/{book_id}/cover")
async def upload_cover(book_id: int, cover: UploadFile,
                       caption: Annotated[str, Form()] = ""):
    if cover.content_type not in ALLOWED:
        raise HTTPException(415, f"Unsupported type {cover.content_type}")
    size, digest = 0, hashlib.sha256()
    while chunk := await cover.read(64 * 1024):
        size += len(chunk)
        if size > MAX_COVER:
            raise HTTPException(413, "Cover too large")
        digest.update(chunk)
    return {"book_id": book_id, "filename": cover.filename,
            "type": cover.content_type, "bytes": size,
            "sha256": digest.hexdigest()[:12], "caption": caption,
            "file_obj": type(cover.file).__name__}
```

The tests, sent with the test client's `files=` argument:

```text
cover     -> 200 {'book_id': 1, 'filename': 'dune.png', 'type': 'image/png', 'bytes': 5008, 'sha256': 'af42feb61d20', 'caption': 'First edition', 'file_obj': 'SpooledTemporaryFile'}
cover gif -> 415 {'detail': 'Unsupported type image/gif'}
cover big -> 413 {'detail': 'Cover too large'}
```

What `UploadFile` gives you:

| Attribute | Meaning |
|-----------|---------|
| `filename` | the name **the client claims** |
| `content_type` | the type **the client claims** |
| `size` | bytes received, when known |
| `file` | a `SpooledTemporaryFile` — in memory while small, on disk when large |
| `read()`, `seek()`, `write()`, `close()` | async wrappers around `file` |

Reading in chunks keeps memory flat however large the file is, and lets you stop as soon
as the limit is crossed. `await cover.read()` with no argument loads the whole file into
memory, which is fine for small files you've already size-checked.

!!! warning "Never trust `filename` or `content_type`"
    Both come from the client. `filename` can be `../../app/main.py`; `content_type` can
    say `image/png` for an executable. For real security, generate your own storage name
    (a UUID or the hash), check the file's actual bytes (for PNG, the first eight bytes
    are `\x89PNG\r\n\x1a\n`), and store uploads outside any directory your app serves
    or executes from. Level 4 lesson 5 returns to this.

### Several files, or raw bytes

```python
@app.post("/bulk")
async def bulk(files: list[UploadFile]):
    return [f.filename for f in files]

@app.post("/raw")
def raw(blob: Annotated[bytes, File()]):
    return {"len": len(blob)}
```

```text
bulk -> 200 ['a.txt', 'b.txt']
raw  -> 200 {'len': 5}
```

`bytes` with `File()` reads the whole upload into memory — convenient for tiny files,
dangerous for large ones. Prefer `UploadFile`.

## Headers

```python
from fastapi import Header, Cookie

@app.get("/whoami")
def whoami(user_agent: Annotated[str | None, Header()] = None,
           x_request_id: Annotated[str | None, Header()] = None,
           accept_language: Annotated[list[str] | None, Header()] = None,
           session: Annotated[str | None, Cookie()] = None):
    return {"ua": user_agent, "req_id": x_request_id,
            "lang": accept_language, "session": session}
```

```text
whoami -> 200 {'ua': 'demo/1.0', 'req_id': 'r-1', 'lang': ['en-GB'], 'session': 'abc123'}
```

Header names in HTTP use hyphens and are case-insensitive. Python names can't contain
hyphens, so FastAPI converts underscores: `x_request_id` reads `X-Request-ID`. If a
header genuinely contains underscores (some proxies drop those, so avoid them), pass
`Header(convert_underscores=False)`. A `list[str]` type collects repeated headers.

## Cookies

Reading uses `Cookie()`, as above. Writing uses `response.set_cookie(...)`. The attributes
matter more than the value:

| Attribute | Effect | Recommendation for session cookies |
|-----------|--------|------------------------------------|
| `httponly=True` | JavaScript can't read it | always |
| `secure=True` | only sent over HTTPS | always in production |
| `samesite="lax"` | not sent on most cross-site requests | default choice; `"strict"` for sensitive apps |
| `max_age` / `expires` | lifetime | short, and renew on activity |
| `path`, `domain` | where it's sent | leave defaults unless you need to share |

A cookie set with `secure=True` won't be sent back by a browser over plain
`http://localhost` in some browsers, which surprises people during development. The test
client isn't a browser and doesn't enforce this.

## Worked example: a profile form with an avatar

Combine everything: a form with text fields and an optional file, a header for
idempotency, and a cookie for the session.

```python
from uuid import uuid4

@app.post("/profile")
async def update_profile(
    display_name: Annotated[str, Form(min_length=1, max_length=50)],
    session: Annotated[str, Cookie()],
    idempotency_key: Annotated[str | None, Header()] = None,
    avatar: UploadFile | None = None,
):
    stored_as = None
    if avatar is not None:
        if avatar.content_type not in ALLOWED:
            raise HTTPException(415, "Avatar must be PNG or JPEG")
        stored_as = f"{uuid4()}.img"          # our name, not the client's
        # ...stream chunks to object storage under stored_as...
    return {"display_name": display_name, "avatar": stored_as,
            "idempotency_key": idempotency_key}
```

Run with the test client:

```text
no cookie          -> 422 {'detail': [{'type': 'missing', 'loc': ['cookie', 'session'], 'msg': 'Field required', 'input': None}]}
cookie + header    -> 200 {'display_name': 'Ada', 'avatar': None, 'idempotency_key': 'k1'}
cookie + PNG file  -> 200 {'display_name': 'Ada', 'avatar': '0a695317-bc94-4b03-851f-300109b3a31a.img', 'idempotency_key': None}
```

`session` has no default, so the request without the cookie got a 422 at
`['cookie', 'session']`. In a real app you'd turn that into a 401 with a dependency
(Level 2 lesson 1).

## How It Actually Works

When FastAPI sees any `Form()` or `File()` parameter, it knows the endpoint consumes a
form body. Per request it calls Starlette's `request.form()`, which picks a parser by
content type:

- `application/x-www-form-urlencoded` is parsed in memory as `key=value&...`.
- `multipart/form-data` is fed through `python-multipart`'s streaming parser. Each file
  part is written into a `SpooledTemporaryFile` that switches from memory to a real
  temporary file once it passes **1 MB** (`spool_max_size = 1024 * 1024` in the Starlette
  source used here).

Starlette also applies defaults to protect against abusive forms: `max_files=1000`,
`max_fields=1000` and `max_part_size` of 1 MB for each *non-file* field. Exceed one and
the request fails with a 400. Both were triggered in testing:

```text
username of 1 MB + 10 bytes -> 400 {"detail":"Field exceeded maximum size of 1024KB."}
1,003 form fields           -> 400 {"detail":"Too many fields. Maximum number of fields is 1000."}
```

Note what's missing: there is **no default limit on a
file's total size**. The whole upload is received and spooled before your function even
runs — your chunked size check limits what you *keep*, not what the server accepts. Real
limits on request size belong in front of the app, in the reverse proxy
(`client_max_body_size` in nginx), as covered in Level 4 lesson 3.

After the response is sent, FastAPI closes the uploaded files, which deletes the
temporary files.

Headers come from `scope["headers"]`, a list of `(bytes, bytes)` pairs with lower-cased
names; FastAPI looks up the converted name. Cookies are parsed from the `cookie` header.

## Common mistakes

- **Missing `python-multipart`** when not using `fastapi[standard]`.
- **Saving uploads under `cover.filename`**: path traversal and overwrites. Generate the
  name.
- **Trusting `content_type`.** It's a claim. Check the bytes.
- **`bytes = File()` for large uploads**, loading the whole file into memory.
- **Assuming your size check stops large uploads.** The body was already received. Limit
  at the proxy.
- **Session cookies without `HttpOnly` and `Secure`.**
- **Mixing JSON body models with `Form()` in one endpoint.** A request body has one
  encoding; FastAPI will expect form data and your JSON model field won't be filled the
  way you expect.

## Exercise

1. Build `POST /books/{id}/sample` that accepts a PDF upload up to 5 MB. Verify the first
   five bytes are `%PDF-` and reject anything else with 415, regardless of
   `content_type`.
2. Store accepted files under `uploads/<sha256>.pdf` and return the hash. Upload the same
   file twice; what should the second response be?
3. Add `GET /whoami` that returns the `X-Forwarded-For` header as a list. Send it with two
   values and check what you get.
4. Write a `/logout` endpoint that deletes the session cookie with
   `response.delete_cookie(...)`, and inspect the `set-cookie` header it produces.
