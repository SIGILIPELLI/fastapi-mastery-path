---
description: "Level 2 project: a bookshelf API with SQLAlchemy, Alembic, pydantic-settings, injected pagination, a repository layer and an isolated pytest suite."
---

# 10 · Project — A Bookshelf API with a Database and Tests

This project pulls Level 2 together: a personal bookshelf API where you track authors and
books, mark books as to-read, reading or read, and rate the ones you've finished. It uses
everything from this level — dependency injection, a yield-based session, SQLAlchemy 2.x,
Alembic, typed settings, pagination — and a test suite that runs in about a tenth of a
second against an isolated database.

## Requirements

- Authors have unique names. Books have a title, a **valid ISBN-13** (hyphens allowed on
  input; the check digit is verified), a status (`to_read`, `reading`, `read`) and an
  optional 1–5 rating.
- Only books with status `read` may have a rating — enforced on create **and** on
  update, including updates that change the status of an already-rated book.
- Book responses embed their author. The list endpoint filters by status, author and
  title text, and paginates with a default and maximum page size taken from settings.
- Duplicate ISBNs and author names are 409s; unknown IDs are 404s; rule violations are
  422s.
- The schema is managed by Alembic; tests never touch the real database file.

## Layout

```text
bookshelf/
├── alembic.ini
├── migrations/
│   ├── env.py
│   └── versions/daa83be20d93_initial_schema.py
├── app/
│   ├── config.py       settings
│   ├── db.py           engine, Base, get_db
│   ├── models.py       ORM tables
│   ├── schemas.py      Pydantic request/response models
│   ├── repository.py   database operations + domain errors
│   ├── deps.py         reusable dependencies (DB, settings, pagination)
│   ├── routers/        authors.py, books.py
│   └── main.py         app, routers, exception handlers
└── tests/
    ├── conftest.py
    └── test_bookshelf.py
```

The dependency direction is `routers → repository → models/schemas → db → config`.
Routers know HTTP; the repository knows SQL; neither knows about the other's concerns.

## Settings and database

```python
# app/config.py
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="BOOKSHELF_", env_file=".env", extra="ignore")

    database_url: str = "sqlite:///bookshelf.db"
    page_size_default: int = 20
    page_size_max: int = 100


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

```python
# app/db.py
from collections.abc import Iterator

from sqlalchemy import MetaData, create_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker

from app.config import get_settings

NAMING = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING)


engine = create_engine(get_settings().database_url)
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)


def get_db() -> Iterator[Session]:
    with SessionLocal() as session:
        yield session
```

The naming convention is in place from the first migration (lesson 6's hard lesson).

## Models

```python
# app/models.py
from datetime import datetime, timezone

from sqlalchemy import CheckConstraint, ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db import Base


def _now() -> datetime:
    return datetime.now(timezone.utc)


class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)
    books: Mapped[list["Book"]] = relationship(back_populates="author")


class Book(Base):
    __tablename__ = "books"
    __table_args__ = (CheckConstraint("rating BETWEEN 1 AND 5", name="rating_range"),)

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(13), unique=True)
    status: Mapped[str] = mapped_column(String(10), default="to_read")
    rating: Mapped[int | None]
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"), index=True)
    added_at: Mapped[datetime] = mapped_column(default=_now)
    author: Mapped[Author] = relationship(back_populates="books")
```

The rating range is a database `CHECK` constraint as well as a Pydantic rule: the
database is the last line of defence against a bug or a manual SQL edit. The
"rated only if read" rule spans two columns and lives in Python.

## Schemas

```python
# app/schemas.py
from datetime import datetime, timezone
from typing import Annotated, Literal

from pydantic import AfterValidator, BaseModel, ConfigDict, Field, model_validator

Status = Literal["to_read", "reading", "read"]


def _isbn13(v: str) -> str:
    digits = v.replace("-", "").replace(" ", "")
    if len(digits) != 13 or not digits.isdigit():
        raise ValueError("ISBN must be 13 digits")
    total = sum(int(d) * (1 if i % 2 == 0 else 3) for i, d in enumerate(digits[:12]))
    if (10 - total % 10) % 10 != int(digits[12]):
        raise ValueError("ISBN check digit is wrong")
    return digits


ISBN = Annotated[str, AfterValidator(_isbn13)]


def _assume_utc(v: datetime) -> datetime:
    # SQLite stores DATETIME without an offset; we only ever write UTC.
    return v.replace(tzinfo=timezone.utc) if v.tzinfo is None else v


UTCDateTime = Annotated[datetime, AfterValidator(_assume_utc)]


class Strict(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)


class AuthorIn(Strict):
    name: str = Field(min_length=1, max_length=100)


class AuthorOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str


class BookIn(Strict):
    title: str = Field(min_length=1, max_length=200)
    isbn: ISBN
    author_id: int
    status: Status = "to_read"
    rating: int | None = Field(None, ge=1, le=5)

    @model_validator(mode="after")
    def rating_needs_read(self):
        if self.rating is not None and self.status != "read":
            raise ValueError("only books with status 'read' can be rated")
        return self


class BookUpdate(Strict):
    title: str | None = Field(None, min_length=1, max_length=200)
    status: Status | None = None
    rating: int | None = Field(None, ge=1, le=5)


class BookOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    isbn: str
    status: Status
    rating: int | None
    added_at: UTCDateTime
    author: AuthorOut


class BookPage(BaseModel):
    items: list[BookOut]
    total: int
    limit: int
    offset: int
```

## Repository

```python
# app/repository.py
"""Database operations. Raises domain errors; knows nothing about HTTP."""
from sqlalchemy import func, select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import Session, joinedload

from app.models import Author, Book
from app.schemas import AuthorIn, BookIn, BookUpdate


class NotFound(Exception):
    def __init__(self, what: str, id_: int):
        self.what, self.id = what, id_


class Conflict(Exception):
    def __init__(self, message: str):
        self.message = message


class InvalidChange(Exception):
    def __init__(self, message: str):
        self.message = message


def _commit(db: Session, conflict_message: str) -> None:
    try:
        db.commit()
    except IntegrityError:
        db.rollback()
        raise Conflict(conflict_message) from None


def create_author(db: Session, data: AuthorIn) -> Author:
    author = Author(name=data.name)
    db.add(author)
    _commit(db, f"Author {data.name!r} already exists")
    return author


def list_authors(db: Session) -> list[Author]:
    return list(db.scalars(select(Author).order_by(Author.name)))


def get_book(db: Session, book_id: int) -> Book:
    book = db.scalar(select(Book).options(joinedload(Book.author)).where(Book.id == book_id))
    if book is None:
        raise NotFound("Book", book_id)
    return book


def create_book(db: Session, data: BookIn) -> Book:
    if db.get(Author, data.author_id) is None:
        raise NotFound("Author", data.author_id)
    book = Book(**data.model_dump())
    db.add(book)
    _commit(db, f"A book with ISBN {data.isbn} already exists")
    return get_book(db, book.id)


def update_book(db: Session, book_id: int, changes: BookUpdate) -> Book:
    book = get_book(db, book_id)
    patch = changes.model_dump(exclude_unset=True)
    new_status = patch.get("status", book.status)
    new_rating = patch.get("rating", book.rating)
    if new_rating is not None and new_status != "read":
        raise InvalidChange("only books with status 'read' can be rated")
    for field, value in patch.items():
        setattr(book, field, value)
    _commit(db, "Update conflicts with existing data")
    return book


def delete_book(db: Session, book_id: int) -> None:
    db.delete(get_book(db, book_id))
    db.commit()


def search_books(db: Session, *, status: str | None, author_id: int | None, q: str | None,
                 limit: int, offset: int) -> tuple[list[Book], int]:
    stmt = select(Book)
    if status:
        stmt = stmt.where(Book.status == status)
    if author_id:
        stmt = stmt.where(Book.author_id == author_id)
    if q:
        stmt = stmt.where(Book.title.ilike(f"%{q}%"))
    total = db.scalar(select(func.count()).select_from(stmt.subquery()))
    rows = db.scalars(stmt.options(joinedload(Book.author))
                      .order_by(Book.added_at.desc(), Book.id.desc())
                      .limit(limit).offset(offset))
    return list(rows), total
```

Notes:

- `_commit` turns any `IntegrityError` into a `Conflict`, after rolling back so the
  session stays usable.
- `create_book` re-reads the book through `get_book` so the response has the author
  loaded with `joinedload` — one query, no lazy loading.
- `update_book` checks the rating rule against the **merged** state. A `PATCH` with only
  `{"status": "reading"}` on a rated book is rejected, which a check on the patch body
  alone would miss.

## Dependencies and routers

```python
# app/deps.py
from typing import Annotated

from fastapi import Depends, Query
from sqlalchemy.orm import Session

from app.config import Settings, get_settings
from app.db import get_db

DB = Annotated[Session, Depends(get_db)]
SettingsDep = Annotated[Settings, Depends(get_settings)]


class Pagination:
    def __init__(self, settings: SettingsDep,
                 limit: Annotated[int | None, Query(ge=1)] = None,
                 offset: Annotated[int, Query(ge=0, le=10_000)] = 0):
        self.limit = min(limit or settings.page_size_default, settings.page_size_max)
        self.offset = offset


PageDep = Annotated[Pagination, Depends()]
```

`Pagination` is a class dependency that itself depends on settings, so the page-size
policy lives in configuration and tests can change it.

```python
# app/routers/authors.py
from fastapi import APIRouter, status

from app import repository as repo
from app.deps import DB
from app.schemas import AuthorIn, AuthorOut

router = APIRouter(prefix="/authors", tags=["authors"])


@router.post("", response_model=AuthorOut, status_code=status.HTTP_201_CREATED)
def create_author(data: AuthorIn, db: DB):
    return repo.create_author(db, data)


@router.get("", response_model=list[AuthorOut])
def list_authors(db: DB):
    return repo.list_authors(db)
```

```python
# app/routers/books.py
from typing import Annotated

from fastapi import APIRouter, Query, status

from app import repository as repo
from app.deps import DB, PageDep
from app.schemas import BookIn, BookOut, BookPage, BookUpdate, Status

router = APIRouter(prefix="/books", tags=["books"])


@router.get("", response_model=BookPage)
def list_books(db: DB, page: PageDep, status: Status | None = None,
               author_id: int | None = None,
               q: Annotated[str | None, Query(max_length=50)] = None):
    items, total = repo.search_books(db, status=status, author_id=author_id, q=q,
                                     limit=page.limit, offset=page.offset)
    return BookPage(items=items, total=total, limit=page.limit, offset=page.offset)


@router.post("", response_model=BookOut, status_code=status.HTTP_201_CREATED)
def create_book(data: BookIn, db: DB):
    return repo.create_book(db, data)


@router.get("/{book_id}", response_model=BookOut)
def get_book(book_id: int, db: DB):
    return repo.get_book(db, book_id)


@router.patch("/{book_id}", response_model=BookOut)
def update_book(book_id: int, changes: BookUpdate, db: DB):
    return repo.update_book(db, book_id, changes)


@router.delete("/{book_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_book(book_id: int, db: DB):
    repo.delete_book(db, book_id)
```

```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from app import repository as repo
from app.routers import authors, books

app = FastAPI(title="Bookshelf API", version="1.0.0")
app.include_router(authors.router)
app.include_router(books.router)


@app.exception_handler(repo.NotFound)
async def not_found(request: Request, exc: repo.NotFound):
    return JSONResponse(status_code=404, content={"detail": f"{exc.what} {exc.id} not found"})


@app.exception_handler(repo.Conflict)
async def conflict(request: Request, exc: repo.Conflict):
    return JSONResponse(status_code=409, content={"detail": exc.message})


@app.exception_handler(repo.InvalidChange)
async def invalid_change(request: Request, exc: repo.InvalidChange):
    return JSONResponse(status_code=422, content={"detail": exc.message})
```

## Migrations

`migrations/env.py` reads the URL from the same settings the app uses, so there's one
source of truth:

```python
import app.models  # noqa: F401  (registers tables)
from app.config import get_settings
from app.db import Base

config.set_main_option("sqlalchemy.url", get_settings().database_url)
target_metadata = Base.metadata
```

(plus `render_as_batch=True` for SQLite, as in lesson 6). Then:

```text
$ alembic revision --autogenerate -m "initial schema"
INFO  [alembic.autogenerate.compare.tables] Detected added table 'authors'
INFO  [alembic.autogenerate.compare.tables] Detected added table 'books'
INFO  [alembic.autogenerate.compare.constraints] Detected added index 'ix_books_author_id' on '('author_id',)'
Generating .../versions/daa83be20d93_initial_schema.py ...  done
$ alembic upgrade head
INFO  [alembic.runtime.migration] Running upgrade  -> daa83be20d93, initial schema
```

The resulting table, with every constraint named by the convention:

```text
CREATE TABLE books (
	id INTEGER NOT NULL,
	title VARCHAR(200) NOT NULL,
	isbn VARCHAR(13) NOT NULL,
	status VARCHAR(10) NOT NULL,
	rating INTEGER,
	author_id INTEGER NOT NULL,
	added_at DATETIME NOT NULL,
	CONSTRAINT pk_books PRIMARY KEY (id),
	CONSTRAINT ck_books_rating_range CHECK (rating BETWEEN 1 AND 5),
	CONSTRAINT fk_books_author_id_authors FOREIGN KEY(author_id) REFERENCES authors (id),
	CONSTRAINT uq_books_isbn UNIQUE (isbn)
);
CREATE INDEX ix_books_author_id ON books (author_id);
```

## Worked example: a bug found by running it

With `fastapi run app/main.py --port 8709` and `curl`, the first version of the app
produced this:

```text
POST /authors  {"name":"Ursula K. Le Guin"}
{"id":1,"name":"Ursula K. Le Guin"}

POST /books  {"title":"The Dispossessed","isbn":"978-0-06-105488-4","author_id":1}
{"id":1,"title":"The Dispossessed","isbn":"9780061054884","status":"to_read","rating":null,"added_at":"2026-10-08T16:54:04.887420Z","author":{"id":1,"name":"Ursula K. Le Guin"}}

PATCH /books/1  {"rating":5}
{"detail":"only books with status 'read' can be rated"}

PATCH /books/1  {"status":"read","rating":5}
{"id":1,...,"status":"read","rating":5,"added_at":"2026-10-08T16:54:04.887420",...}
```

The rules worked — but look at `added_at`. On create it ended in `Z` (UTC); on the next
request it had **no offset at all**. The create response used the in-memory Python
`datetime`, which was timezone-aware; the later request loaded the value from SQLite,
whose `DATETIME` column stores no offset, so SQLAlchemy returned a naive `datetime`. A
client comparing those strings, or parsing the second one as local time, would be wrong
by however many hours it is from UTC.

The fix is the `UTCDateTime` type in `schemas.py`: an `AfterValidator` that marks naive
values as UTC (safe because the app only ever writes UTC). The test that proves it:

```python
def test_timestamps_keep_utc_after_reload(client, author, db):
    created = add(client, author, EARTHSEA)
    db.expire_all()                       # force the next read to come from the database
    fetched = client.get(f"/books/{created['id']}").json()
    assert created["added_at"].endswith("Z")
    assert fetched["added_at"] == created["added_at"]
```

With the response model temporarily switched back to plain `datetime`, it failed exactly
as the curl session did:

```text
E       AssertionError: assert '2026-10-08T16:54:28.118953' == '2026-10-08T16:54:28.118953Z'
```

With `UTCDateTime`, it passes. (On PostgreSQL, a `DateTime(timezone=True)` column stores
the offset and this problem doesn't arise.) The `db.expire_all()` line matters: without
it, the session's identity map would hand back the same in-memory object and the test
would pass even with the bug.

## Tests

`tests/conftest.py` is lesson 8's fixture set, plus a settings override and an `author`
fixture:

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine, event
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

import app.models  # noqa: F401  (registers the tables on Base.metadata)
from app.config import Settings, get_settings
from app.db import Base, get_db
from app.main import app


@pytest.fixture(scope="session")
def engine():
    engine = create_engine(
        "sqlite://",
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )

    # pysqlite manages transactions itself and gets in the way of SAVEPOINTs.
    # Turn that off and let SQLAlchemy emit BEGIN (recipe from the SQLAlchemy docs).
    @event.listens_for(engine, "connect")
    def _no_pysqlite_tx(dbapi_connection, connection_record):
        dbapi_connection.isolation_level = None

    @event.listens_for(engine, "begin")
    def _emit_begin(conn):
        conn.exec_driver_sql("BEGIN")

    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()


@pytest.fixture
def db(engine):
    """A session inside a transaction that is rolled back after each test."""
    connection = engine.connect()
    outer = connection.begin()
    session = Session(bind=connection, join_transaction_mode="create_savepoint",
                      expire_on_commit=False)
    try:
        yield session
    finally:
        session.close()
        outer.rollback()
        connection.close()


@pytest.fixture
def client(db):
    def override_get_db():
        yield db
    app.dependency_overrides[get_db] = override_get_db
    app.dependency_overrides[get_settings] = lambda: Settings(
        _env_file=None, page_size_default=2, page_size_max=3)
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()


@pytest.fixture
def author(client):
    r = client.post("/authors", json={"name": "Ursula K. Le Guin"})
    assert r.status_code == 201
    return r.json()
```

```python
# tests/test_bookshelf.py
import pytest

EARTHSEA = "9780547773742"      # valid ISBN-13 check digits
DISPOSSESSED = "9780061054884"
LEFT_HAND = "9780441478125"


def add(client, author, isbn, **kw):
    payload = {"title": kw.pop("title", f"Book {isbn[-4:]}"), "isbn": isbn,
               "author_id": author["id"], **kw}
    r = client.post("/books", json=payload)
    assert r.status_code == 201, r.text
    return r.json()


def test_create_book_embeds_author(client, author):
    book = add(client, author, EARTHSEA, title="A Wizard of Earthsea")
    assert book["author"] == author
    assert book["status"] == "to_read" and book["rating"] is None


def test_isbn_hyphens_are_accepted_and_check_digit_enforced(client, author):
    book = add(client, author, "978-0-547-77374-2")
    assert book["isbn"] == EARTHSEA
    r = client.post("/books", json={"title": "X", "isbn": "9780547773743", "author_id": author["id"]})
    assert r.status_code == 422
    assert "check digit" in r.json()["detail"][0]["msg"]


def test_unknown_author_is_404(client):
    r = client.post("/books", json={"title": "X", "isbn": EARTHSEA, "author_id": 999})
    assert r.status_code == 404
    assert r.json() == {"detail": "Author 999 not found"}


def test_duplicates_are_409(client, author):
    add(client, author, EARTHSEA)
    assert client.post("/books", json={"title": "Y", "isbn": EARTHSEA, "author_id": author["id"]}).status_code == 409
    assert client.post("/authors", json={"name": author["name"]}).status_code == 409


def test_rating_rules_on_create_and_update(client, author):
    r = client.post("/books", json={"title": "X", "isbn": EARTHSEA, "author_id": author["id"], "rating": 5})
    assert r.status_code == 422
    book = add(client, author, EARTHSEA)
    assert client.patch(f"/books/{book['id']}", json={"rating": 4}).status_code == 422
    r = client.patch(f"/books/{book['id']}", json={"status": "read", "rating": 4})
    assert r.status_code == 200 and r.json()["rating"] == 4
    assert client.patch(f"/books/{book['id']}", json={"status": "reading"}).status_code == 422


def test_list_filters_and_paginates_with_settings_cap(client, author):
    add(client, author, EARTHSEA, title="A Wizard of Earthsea", status="read", rating=5)
    add(client, author, DISPOSSESSED, title="The Dispossessed")
    add(client, author, LEFT_HAND, title="The Left Hand of Darkness")
    page = client.get("/books").json()
    assert page["limit"] == 2 and page["total"] == 3 and len(page["items"]) == 2
    assert client.get("/books", params={"limit": 50}).json()["limit"] == 3
    assert [b["title"] for b in client.get("/books", params={"q": "the"}).json()["items"]] == \
        ["The Left Hand of Darkness", "The Dispossessed"]
    assert client.get("/books", params={"status": "read"}).json()["total"] == 1


def test_delete(client, author):
    book = add(client, author, EARTHSEA)
    assert client.delete(f"/books/{book['id']}").status_code == 204
    assert client.get(f"/books/{book['id']}").status_code == 404


@pytest.mark.parametrize("payload", [
    {"title": "X", "isbn": EARTHSEA, "author_id": 1, "colour": "red"},
    {"title": "X", "isbn": EARTHSEA, "author_id": 1, "status": "finished"},
    {"title": "X", "isbn": EARTHSEA, "author_id": 1, "rating": 6, "status": "read"},
])
def test_rejected_payloads(client, author, payload):
    assert client.post("/books", json=payload).status_code == 422


def test_timestamps_keep_utc_after_reload(client, author, db):
    created = add(client, author, EARTHSEA)
    db.expire_all()                       # force the next read to come from the database
    fetched = client.get(f"/books/{created['id']}").json()
    assert created["added_at"].endswith("Z")
    assert fetched["added_at"] == created["added_at"]
```

```text
$ python -m pytest -q
...........                                                              [100%]
11 passed in 0.09s
```

The ISBN constants are strings with valid ISBN-13 check digits, used as test data. The
settings override sets `page_size_default=2` and `page_size_max=3`, so pagination
behaviour can be tested with three books instead of a hundred.

## How It Actually Works

Trace `PATCH /books/1` with `{"status": "reading"}` on a rated book:

1. FastAPI resolves dependencies: `get_db` opens a session (overridden in tests to the
   savepoint session); the body is validated as `BookUpdate` (`extra="forbid"`,
   `Status` literal).
2. The router calls `repo.update_book`. `get_book` runs one `SELECT ... JOIN authors`.
3. The merged state is `status="reading"`, `rating=4`, which breaks the rule, so the
   repository raises `InvalidChange` **before** modifying anything.
4. The exception escapes the endpoint; the yield dependency's teardown closes the
   session (rolling back the empty transaction), and the handler in `main.py` returns
   the 422.

On success, `setattr` marks attributes dirty in the session; `commit()` flushes one
`UPDATE books SET ... WHERE id = ?` and commits; `expire_on_commit=False` means the
response is built from the in-memory object with no extra query.

## Common mistakes

- **Validating a PATCH body in isolation** instead of the merged state.
- **Trusting SQLite with timezones.** Test a round-trip through the database, as above.
- **A test that passes because of the identity map.** Expire or use a fresh session when
  the point is to read from the database.
- **Settings read at import time** (`create_engine(get_settings().database_url)` in
  `db.py`) — fine for the app, but the tests must override `get_db` rather than rely on
  changing the URL. That's what this project does; Level 3 moves engine creation into the
  lifespan handler.
- **Business rules only in Pydantic.** The `CHECK` constraint caught nothing in these
  tests, but it will catch the bug nobody wrote a test for.

## Exercise

1. Add `GET /authors/{id}/stats` returning counts of books by status and the average
   rating, using one aggregate query. Test it.
2. Add a `finished_on` date column with an Alembic migration. It must be set when status
   becomes `read` and cleared otherwise — enforce that in the repository and test both
   transitions.
3. Switch the list endpoint to cursor pagination on `(added_at, id)` (lesson 9). Keep
   the settings-driven page size.
4. Run the suite against PostgreSQL (a local install or a container) by making the test
   engine URL configurable. Which fixture code can you delete?
