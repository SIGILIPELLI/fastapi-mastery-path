---
description: "Level 1 project: build a complete in-memory notes API with routers, validated models, PATCH, search, pagination, custom errors and a pytest suite."
---

# 10 · Project — An In-Memory Notes API

Time to put Level 1 together. You'll build a notes API with create, read, update, delete,
search and pagination; clean validation; correct status codes; a domain exception mapped
to 404; and a test suite. Data lives in memory — Level 2 replaces the store with a
database without touching the endpoints much, which is the point of the structure used
here.

## What you're building

| Method | Path | Purpose | Success |
|--------|------|---------|---------|
| `GET` | `/notes` | search + paginate (`q`, `tag`, `pinned`, `limit`, `offset`) | 200 |
| `POST` | `/notes` | create | 201 + `Location` |
| `GET` | `/notes/{note_id}` | fetch one | 200 / 404 |
| `PATCH` | `/notes/{note_id}` | change only the sent fields | 200 / 404 / 422 |
| `DELETE` | `/notes/{note_id}` | remove | 204 / 404 |

Requirements, each traceable to a lesson:

- Titles 1–120 characters, whitespace stripped; body up to 10,000 characters; up to 10
  tags, each lower-cased and limited to letters, digits and hyphens (lesson 6).
- Unknown fields are rejected, so client typos surface (lesson 6).
- Responses always use the `Note` model; list responses use a page envelope with
  `total` (lesson 5).
- `PATCH` changes only the fields sent, and sending `null` doesn't erase a field
  (lesson 4).
- Pinned notes come first, then most recently updated (lesson 3's query parameters).
- Missing notes are a domain exception, translated to 404 in one place (lesson 7).

## Layout

```text
notes/
├── notes_api/
│   ├── __init__.py
│   ├── main.py
│   ├── schemas.py
│   ├── store.py
│   └── routers/
│       ├── __init__.py
│       └── notes.py
└── tests/
    ├── __init__.py
    └── test_notes.py
```

## Schemas

```python
# notes_api/schemas.py
from datetime import datetime
from typing import Annotated

from pydantic import BaseModel, ConfigDict, Field, AfterValidator


def _clean_tag(v: str) -> str:
    v = v.strip().lower()
    if not v.replace("-", "").isalnum():
        raise ValueError("tags may contain only letters, digits and hyphens")
    return v


Tag = Annotated[str, Field(min_length=1, max_length=20), AfterValidator(_clean_tag)]


class NoteBase(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)


class NoteCreate(NoteBase):
    title: str = Field(min_length=1, max_length=120)
    body: str = Field(default="", max_length=10_000)
    tags: list[Tag] = Field(default_factory=list, max_length=10)
    pinned: bool = False


class NoteUpdate(NoteBase):
    title: str | None = Field(default=None, min_length=1, max_length=120)
    body: str | None = Field(default=None, max_length=10_000)
    tags: list[Tag] | None = Field(default=None, max_length=10)
    pinned: bool | None = None


class Note(BaseModel):
    id: int
    title: str
    body: str
    tags: list[str]
    pinned: bool
    created_at: datetime
    updated_at: datetime


class NotePage(BaseModel):
    items: list[Note]
    total: int
    limit: int
    offset: int
```

Design notes:

- `NoteBase` holds the shared config so create and update models can't drift apart.
- `NoteUpdate` makes every field optional but keeps the same constraints, so a `PATCH`
  can't set a 500-character title that `POST` would reject.
- `Note` (the output model) has no `extra="forbid"`: it's built by our own code, and
  output models should be tolerant.
- The `Tag` type puts the length constraint *before* the `AfterValidator`, so cleaning
  only runs on strings that already passed the length check.

## The store

```python
# notes_api/store.py
"""A small in-memory repository. It knows nothing about HTTP."""
from datetime import datetime, timezone
from itertools import count
from threading import Lock

from notes_api.schemas import Note, NoteCreate, NoteUpdate


class NoteNotFound(Exception):
    def __init__(self, note_id: int):
        self.note_id = note_id


class NoteStore:
    def __init__(self) -> None:
        self._notes: dict[int, Note] = {}
        self._ids = count(1)
        self._lock = Lock()

    def create(self, data: NoteCreate) -> Note:
        now = datetime.now(timezone.utc)
        with self._lock:
            note = Note(id=next(self._ids), created_at=now, updated_at=now, **data.model_dump())
            self._notes[note.id] = note
        return note

    def get(self, note_id: int) -> Note:
        try:
            return self._notes[note_id]
        except KeyError:
            raise NoteNotFound(note_id) from None

    def update(self, note_id: int, changes: NoteUpdate) -> Note:
        with self._lock:
            current = self.get(note_id)
            patch = changes.model_dump(exclude_unset=True, exclude_none=True)
            updated = Note.model_validate({**current.model_dump(), **patch,
                                           "updated_at": datetime.now(timezone.utc)})
            self._notes[note_id] = updated
        return updated

    def delete(self, note_id: int) -> None:
        with self._lock:
            self.get(note_id)
            del self._notes[note_id]

    def search(self, q: str | None, tag: str | None, pinned: bool | None,
               limit: int, offset: int) -> tuple[list[Note], int]:
        notes = list(self._notes.values())
        if q:
            ql = q.lower()
            notes = [n for n in notes if ql in n.title.lower() or ql in n.body.lower()]
        if tag:
            notes = [n for n in notes if tag.lower() in n.tags]
        if pinned is not None:
            notes = [n for n in notes if n.pinned == pinned]
        notes.sort(key=lambda n: (not n.pinned, -n.updated_at.timestamp(), -n.id))
        return notes[offset:offset + limit], len(notes)


store = NoteStore()
```

The store raises `NoteNotFound` rather than an `HTTPException`; it could be used from a
CLI unchanged. The lock matters because `def` endpoints run in a thread pool (Level 2
lesson 3), so two requests really can call `create` at the same moment. `count()` alone
is not a safe ID generator across threads without it.

`update` uses `exclude_unset=True` *and* `exclude_none=True`: a client that sends
`{"title": null}` changes nothing rather than erasing the title. Then it re-validates
the merged record against `Note`, so the stored object is guaranteed to be valid.

## The router

```python
# notes_api/routers/notes.py
from typing import Annotated

from fastapi import APIRouter, Query, Request, Response, status

from notes_api import store as store_module
from notes_api.schemas import Note, NoteCreate, NotePage, NoteUpdate

router = APIRouter(prefix="/notes", tags=["notes"])


def _store():
    return store_module.store


@router.get("", response_model=NotePage)
def list_notes(
    q: Annotated[str | None, Query(max_length=100)] = None,
    tag: Annotated[str | None, Query(max_length=20)] = None,
    pinned: bool | None = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
):
    items, total = _store().search(q, tag, pinned, limit, offset)
    return NotePage(items=items, total=total, limit=limit, offset=offset)


@router.post("", response_model=Note, status_code=status.HTTP_201_CREATED)
def create_note(data: NoteCreate, request: Request, response: Response):
    note = _store().create(data)
    response.headers["Location"] = str(request.url_for("get_note", note_id=note.id))
    return note


@router.get("/{note_id}", response_model=Note)
def get_note(note_id: int):
    return _store().get(note_id)


@router.patch("/{note_id}", response_model=Note)
def update_note(note_id: int, changes: NoteUpdate):
    return _store().update(note_id, changes)


@router.delete("/{note_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_note(note_id: int):
    _store().delete(note_id)
```

Every endpoint is three lines or fewer: parse (FastAPI does it), call the store, return.
`_store()` reads `store_module.store` at call time so tests can swap in a fresh store —
Level 2 replaces this with dependency injection, which is the proper tool.

## The app

```python
# notes_api/main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from notes_api.routers import notes
from notes_api.store import NoteNotFound

app = FastAPI(title="Notes API", version="1.0.0")
app.include_router(notes.router)


@app.exception_handler(NoteNotFound)
async def note_not_found(request: Request, exc: NoteNotFound):
    return JSONResponse(status_code=404, content={"detail": f"Note {exc.note_id} not found"})


@app.get("/health", include_in_schema=False)
def health():
    return {"status": "ok"}
```

## Running it

```bash
fastapi dev notes_api/main.py
```

A session with `curl` against `fastapi run notes_api/main.py --port 8707`:

```bash
curl -s -i -X POST localhost:8707/notes -H 'content-type: application/json' \
     -d '{"title":"Read Dune","tags":["Books"],"pinned":true}'
```

```text
HTTP/1.1 201 Created
date: Thu, 08 Oct 2026 16:28:13 GMT
server: uvicorn
content-length: 155
content-type: application/json
location: http://localhost:8707/notes/1

{"id":1,"title":"Read Dune","body":"","tags":["books"],"pinned":true,"created_at":"2026-10-08T16:28:13.635244Z","updated_at":"2026-10-08T16:28:13.635244Z"}
```

```bash
curl -s 'localhost:8707/notes?tag=books'
curl -s -X PATCH localhost:8707/notes/1 -H 'content-type: application/json' -d '{"body":"chapter 3"}'
curl -s -o /dev/null -w "%{http_code}\n" -X DELETE localhost:8707/notes/1
curl -s localhost:8707/notes/1
```

```text
{"items":[{"id":1,"title":"Read Dune","body":"","tags":["books"],"pinned":true,"created_at":"2026-10-08T16:28:13.635244Z","updated_at":"2026-10-08T16:28:13.635244Z"}],"total":1,"limit":20,"offset":0}
{"id":1,"title":"Read Dune","body":"chapter 3","tags":["books"],"pinned":true,"created_at":"2026-10-08T16:28:13.635244Z","updated_at":"2026-10-08T16:28:13.658652Z"}
204
{"detail":"Note 1 not found"}
```

The tag `"Books"` was stored as `"books"`; the `PATCH` kept the title and tags and moved
`updated_at`; the `Location` header was built with `url_for`, so it's correct whatever
host and port the app runs on.

## Tests

```python
# tests/test_notes.py
import pytest
from fastapi.testclient import TestClient

from notes_api import store as store_module
from notes_api.main import app


@pytest.fixture
def client():
    store_module.store = store_module.NoteStore()   # fresh, empty store per test
    return TestClient(app)


def make(client, **fields):
    payload = {"title": "Shopping", **fields}
    r = client.post("/notes", json=payload)
    assert r.status_code == 201, r.text
    return r.json()


def test_create_returns_201_location_and_cleaned_tags(client):
    r = client.post("/notes", json={"title": "  Ideas ", "tags": [" Work ", "q4-plans"]})
    assert r.status_code == 201
    body = r.json()
    assert body["title"] == "Ideas"
    assert body["tags"] == ["work", "q4-plans"]
    assert r.headers["location"].endswith(f"/notes/{body['id']}")


def test_rejects_unknown_fields_and_bad_tags(client):
    r = client.post("/notes", json={"title": "x", "colour": "red", "tags": ["no spaces!"]})
    assert r.status_code == 422
    kinds = {(e["type"], tuple(e["loc"])) for e in r.json()["detail"]}
    assert ("extra_forbidden", ("body", "colour")) in kinds
    assert ("value_error", ("body", "tags", 0)) in kinds


def test_get_missing_note_is_404(client):
    r = client.get("/notes/999")
    assert r.status_code == 404
    assert r.json() == {"detail": "Note 999 not found"}


def test_patch_changes_only_sent_fields(client):
    note = make(client, body="milk", tags=["home"])
    r = client.patch(f"/notes/{note['id']}", json={"pinned": True})
    assert r.status_code == 200
    updated = r.json()
    assert updated["pinned"] is True
    assert updated["body"] == "milk" and updated["tags"] == ["home"]
    assert updated["updated_at"] >= note["updated_at"]


def test_patch_null_title_does_not_erase_it(client):
    note = make(client)
    r = client.patch(f"/notes/{note['id']}", json={"title": None})
    assert r.status_code == 200
    assert r.json()["title"] == "Shopping"


def test_delete_then_get(client):
    note = make(client)
    assert client.delete(f"/notes/{note['id']}").status_code == 204
    assert client.get(f"/notes/{note['id']}").status_code == 404
    assert client.delete(f"/notes/{note['id']}").status_code == 404


def test_search_filters_and_pins_first(client):
    make(client, title="Groceries", body="eggs", tags=["home"])
    make(client, title="Sprint notes", tags=["work"], pinned=True)
    make(client, title="Garden", body="buy eggs and seeds", tags=["home"])
    r = client.get("/notes", params={"q": "eggs"})
    assert [n["title"] for n in r.json()["items"]] == ["Garden", "Groceries"]
    r = client.get("/notes", params={"tag": "home", "limit": 1})
    page = r.json()
    assert page["total"] == 2 and len(page["items"]) == 1
    r = client.get("/notes")
    assert r.json()["items"][0]["title"] == "Sprint notes"


@pytest.mark.parametrize("params", [{"limit": 0}, {"limit": 101}, {"offset": -1}])
def test_pagination_bounds(client, params):
    assert client.get("/notes", params=params).status_code == 422
```

```bash
python -m pytest -q
```

```text
..........                                                               [100%]
10 passed in 0.28s
```

(Eight test functions; the parametrised one counts three times.) Each test gets a fresh
`NoteStore` through the fixture, so they don't depend on each other's data or order.

## How It Actually Works

Follow one request, `PATCH /notes/1` with `{"pinned": true}`, through the pieces:

1. The router matches `/notes/{note_id}` for `PATCH`; FastAPI converts `"1"` to `1` and
   validates the body against `NoteUpdate`. `extra="forbid"` and the per-field
   constraints run in Pydantic's Rust core; on failure the 422 is sent and the store is
   never touched.
2. `NoteUpdate` remembers that only `pinned` was set (`model_fields_set == {"pinned"}`).
3. Because `update_note` is a plain `def`, FastAPI runs it in a worker thread. The store
   takes its lock, so a concurrent `DELETE` can't remove the note between the `get` and
   the write.
4. The store merges and re-validates into a new `Note`. Replacing the object instead of
   mutating it means a response being serialized on another thread never sees a
   half-updated note.
5. The returned `Note` is validated against `response_model=Note` and serialized; the
   timezone-aware `datetime`s become ISO 8601 strings ending in `Z`.
6. If the ID didn't exist, `NoteNotFound` propagates out of the endpoint, and the
   handler registered in `main.py` turns it into the 404.

What this design *doesn't* handle — and Level 2 does — is persistence (restart and it's
gone), multiple worker processes (each would have its own store), and swapping the store
cleanly in tests (we reached into a module attribute).

## Common mistakes

- **Mutating stored objects in place** from several threads. Build a new one and swap it.
- **Skipping re-validation after a merge**, letting a `PATCH` create a record `POST`
  would reject.
- **Tests that share the global store** and pass or fail depending on order.
- **Forgetting the page envelope**: returning a bare list makes it impossible to add
  `total` later without breaking clients.
- **`Location` built by string concatenation**, which breaks behind a proxy or prefix.

## Exercise

1. Add `POST /notes/{note_id}/archive` and `.../unarchive`, plus an `archived` filter
   on the list (default: hide archived notes). Write tests first.
2. Add `sort` with values `updated` (default), `created` and `title`, validated with a
   `Literal`. Pinned notes should still come first.
3. Add an `If-Match`-style safety check: `PATCH` accepts an optional `expected_updated_at`
   query parameter and returns 409 if the note has changed since. Test it with two
   successive patches.
4. Run the app with `fastapi run --workers 2` (Level 4 covers workers), create a note,
   and fetch it a few times. Explain the inconsistent results you see.
