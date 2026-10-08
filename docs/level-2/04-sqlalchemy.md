---
description: "Use SQLAlchemy 2.x with FastAPI: typed models, a session-per-request dependency, CRUD, IntegrityError to 409, expire_on_commit and the SQL it emits."
---

# 04 · SQLAlchemy 2.x with FastAPI

FastAPI has no database layer of its own, which is a feature: you choose. The most widely
used choice in Python is **SQLAlchemy**, and its 2.x API fits FastAPI well — typed models,
explicit sessions, and `select()` statements you can read. This lesson wires a
synchronous SQLAlchemy setup into FastAPI with SQLite; lesson 5 does the async version.

The examples ran on **SQLAlchemy 2.1.4**. The 2.0-style API used here (`Mapped`,
`mapped_column`, `select()`, `Session.scalars`) is the same in every 2.x release. If you
know SQL already, the [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/)
explains what the generated statements do.

```bash
pip install sqlalchemy
```

## Models

```python
from sqlalchemy import ForeignKey, String, create_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship, sessionmaker

engine = create_engine("sqlite:///shop.db")
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    books: Mapped[list["Book"]] = relationship(back_populates="author")

class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(13), unique=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped[Author] = relationship(back_populates="books")

Base.metadata.create_all(engine)
```

- `Mapped[int]` tells both SQLAlchemy and your type checker the attribute's type.
  `Mapped[str | None]` would make the column nullable; plain `Mapped[str]` is
  `NOT NULL`.
- `unique=True` on `isbn` creates a database constraint. That's the real guard against
  duplicates; a Python check alone has a race (two requests check, both see nothing, both
  insert).
- `create_all` is fine for a demo. Real projects use migrations (lesson 6).

ORM models and Pydantic models are **separate on purpose**. The ORM model describes a
table; the Pydantic models describe what the API accepts and returns. They often look
alike at first and then diverge (password hashes, internal flags, computed fields).

## A session per request

```python
from typing import Annotated
from fastapi import Depends
from sqlalchemy.orm import Session

def get_db():
    with SessionLocal() as session:
        yield session

DB = Annotated[Session, Depends(get_db)]
```

One session per request, closed when the request ends. Following lesson 2's advice, the
dependency doesn't commit — endpoints commit explicitly, so a failed commit becomes an
ordinary error before the response is sent. If the endpoint raises before committing,
closing the session rolls back the open transaction.

## Pydantic schemas that read ORM objects

```python
from pydantic import BaseModel, ConfigDict, Field

class BookIn(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    isbn: str = Field(pattern=r"^\d{13}$")
    author_id: int

class BookOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    isbn: str
    author_id: int
```

`from_attributes=True` lets `BookOut.model_validate(book_orm_object)` read attributes
instead of dict keys. FastAPI's response validation reads attributes anyway (Level 1
lesson 5), but setting it makes the schema usable outside FastAPI too — in a service
function or a test.

## CRUD endpoints

```python
from fastapi import FastAPI, HTTPException, status
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError

app = FastAPI()

@app.post("/books", response_model=BookOut, status_code=status.HTTP_201_CREATED)
def create_book(data: BookIn, db: DB):
    if db.get(Author, data.author_id) is None:
        raise HTTPException(422, "Unknown author_id")
    book = Book(**data.model_dump())
    db.add(book)
    try:
        db.commit()
    except IntegrityError:
        db.rollback()
        raise HTTPException(409, "A book with this ISBN already exists")
    return book

@app.get("/books/{book_id}", response_model=BookOut)
def get_book(book_id: int, db: DB):
    book = db.get(Book, book_id)
    if book is None:
        raise HTTPException(404, "Book not found")
    return book

@app.get("/books", response_model=list[BookOut])
def list_books(db: DB, q: str | None = None, limit: int = 20):
    stmt = select(Book).order_by(Book.title).limit(limit)
    if q:
        stmt = stmt.where(Book.title.ilike(f"%{q}%"))
    return db.scalars(stmt).all()

@app.delete("/books/{book_id}", status_code=204)
def delete_book(book_id: int, db: DB):
    book = db.get(Book, book_id)
    if book is None:
        raise HTTPException(404, "Book not found")
    db.delete(book)
    db.commit()
```

(There's also a small `POST /authors` endpoint in the test file.) The endpoints are
`def`, not `async def`, because the SQLite driver and this `Session` are blocking —
lesson 3's rule. A run through the test client:

```text
POST /authors?name=Frank%20Herbert -> 201 {'id': 1, 'name': 'Frank Herbert'}
POST /books -> 201 {'id': 1, 'title': 'Dune', 'isbn': '9780441172719', 'author_id': 1}
POST /books -> 201 {'id': 2, 'title': 'Dune Messiah', 'isbn': '9780593098233', 'author_id': 1}
POST /books -> 409 {'detail': 'A book with this ISBN already exists'}
POST /books -> 422 {'detail': 'Unknown author_id'}
GET /books?q=mess -> 200 [{'id': 2, 'title': 'Dune Messiah', 'isbn': '9780593098233', 'author_id': 1}]
GET /books/1 -> 200 {'id': 1, 'title': 'Dune', 'isbn': '9780441172719', 'author_id': 1}
DELETE /books/2 -> 204
GET /books/2 -> 404 {'detail': 'Book not found'}
```

The duplicate ISBN was caught by the database's unique constraint and translated to a
409. `ilike` made the search case-insensitive (`mess` matched `Messiah`).

## Worked example: reading the SQL

Turn on SQL logging with `create_engine(..., echo=True)` or, more selectively:

```python
import logging
logging.basicConfig()
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)
```

`GET /books?q=dune&limit=5` produced:

```text
BEGIN (implicit)
SELECT books.id, books.title, books.isbn, books.author_id
FROM books
WHERE lower(books.title) LIKE lower(?) ORDER BY books.title
 LIMIT ? OFFSET ?
[cached since 0.006893s ago] ('%dune%', 5, 0)
ROLLBACK
```

- The search value is a **bound parameter** (`?`), not pasted into the SQL. The f-string
  builds the *pattern* `%dune%`, which is passed as data. SQL injection isn't possible
  here — but `%` and `_` typed by the user are still wildcards; escape them if that
  matters.
- On SQLite, `ilike` is emulated with `lower(...) LIKE lower(...)`. On PostgreSQL it
  becomes the native `ILIKE`.
- `[cached since ...]` means SQLAlchemy reused the compiled statement from an earlier
  request.
- The read ended with `ROLLBACK`: closing a session that never committed rolls back. For
  a read that's harmless; it just ends the transaction.

### What `expire_on_commit` does

The `sessionmaker` above sets `expire_on_commit=False`. The difference shows in the SQL
for `POST /books`. With `False`:

```text
BEGIN (implicit)
SELECT authors.id, authors.name FROM authors WHERE authors.id = ?
INSERT INTO books (title, isbn, author_id) VALUES (?, ?, ?)
COMMIT
```

With SQLAlchemy's default `True`, the same request added a second round-trip:

```text
COMMIT
BEGIN (implicit)
SELECT books.id, books.title, books.isbn, books.author_id FROM books WHERE books.id = ?
ROLLBACK
```

(Parameter lines trimmed.) After a commit, the default marks every loaded object as
stale; reading `book.title` for the response triggered a reload. In a request that
commits and then returns the object, that reload is usually wasted. With async
SQLAlchemy (lesson 5) it's worse than wasted: an implicit reload can't run outside an
`await`, so `expire_on_commit=False` is effectively required there.

## How It Actually Works

- The **engine** owns a connection pool. For a file-based SQLite URL on this version it
  was a `QueuePool`, and SQLAlchemy passed `check_same_thread=False` to the driver
  automatically — which is why sessions created in one worker thread and used in another
  didn't complain. (Older tutorials add `connect_args={"check_same_thread": False}` by
  hand; it's harmless but unnecessary on SQLAlchemy 2.x.) An in-memory URL
  (`sqlite://`) got a `SingletonThreadPool` instead — one connection *per thread*, which
  means each thread sees a different, empty in-memory database. Lesson 8 deals with that
  in tests.
- A **session** is a unit of work. It checks a connection out of the pool when it first
  needs one (`BEGIN (implicit)`), tracks the objects you load or add in its identity map
  (so `db.get(Book, 1)` twice in one session runs one query), and on `commit()` flushes
  pending changes as SQL in dependency order — authors before books — then commits.
- `db.get()` checks the identity map before querying. `db.scalars(select(...))` always
  queries, then returns ORM objects from the first column of each row.
- On `close()`, the session returns its connection to the pool, rolling back anything
  uncommitted.

## Common mistakes

- **`async def` endpoints with a sync `Session`.** Every query blocks the event loop.
  Use `def` (or async SQLAlchemy).
- **Sharing one global session** across requests. Sessions aren't thread-safe and their
  identity map grows forever. One per request.
- **Checking uniqueness only in Python.** Use a unique constraint and handle
  `IntegrityError`.
- **Forgetting `rollback()` after a failed commit** if you keep using the session; it
  stays in a failed state until rolled back.
- **Returning ORM objects without a response model.** It can look fine: a test endpoint
  that returned `db.get(Book, 1)` with no response model sent
  `{"id":1,"author_id":1,"title":"Dune","isbn":"9780441172719"}` — every loaded column,
  in whatever order the object stored them. Add a `password_hash` column to a model and
  it goes out too. Always declare the output schema.
- **Building SQL with f-strings** around user input in `text()` queries. Use bound
  parameters.

## Exercise

1. Add `PATCH /books/{id}` using a `BookPatch` schema and `exclude_unset=True`, setting
   attributes on the ORM object with `setattr`. Make a duplicate ISBN return 409.
2. Add `GET /authors/{id}/books` using `select(Book).where(Book.author_id == id)`. Turn
   on SQL logging and read the query.
3. Change the engine to `expire_on_commit=True` and confirm the extra `SELECT` in the log.
   Then access `book.author.name` in the response and count the queries.
4. Write the IntegrityError handling as an app-level exception handler (Level 1
   lesson 7) instead of a `try` in the endpoint. What do you lose?
