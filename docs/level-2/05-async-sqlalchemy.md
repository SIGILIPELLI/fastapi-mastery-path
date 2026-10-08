---
description: "Async SQLAlchemy with FastAPI: AsyncSession setup, why lazy loading fails with MissingGreenlet, and fixing N+1 queries with selectinload and joinedload."
---

# 05 · Async SQLAlchemy, Relationships & N+1 Queries

Lesson 4 used a blocking `Session` from `def` endpoints, which is correct and often good
enough. When you want `async def` endpoints that talk to the database — to serve many
concurrent requests that mostly wait on I/O — you need an async driver and SQLAlchemy's
asyncio extension. This lesson sets that up and then tackles the problem that bites
everyone who returns related objects: lazy loading and the **N+1 query** pattern.

## Install

```bash
pip install "sqlalchemy[asyncio]" aiosqlite      # SQLite; use asyncpg for PostgreSQL
```

The `[asyncio]` extra matters. Without it, the first import of
`sqlalchemy.ext.asyncio` failed in testing with:

```text
ImportError: The SQLAlchemy asyncio module requires that the Python 'greenlet' library is installed.  In order to ensure this dependency is available, use the 'sqlalchemy[asyncio]' install target:  'pip install sqlalchemy[asyncio]'
```

(`greenlet` 3.5.6 was installed by the extra. Why SQLAlchemy needs greenlets is explained
under How It Actually Works.)

## Setup

```python
from typing import Annotated
from fastapi import Depends, FastAPI
from sqlalchemy import ForeignKey, String
from sqlalchemy.ext.asyncio import AsyncAttrs, AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

engine = create_async_engine("sqlite+aiosqlite:///async.db")
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

class Base(AsyncAttrs, DeclarativeBase):
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
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped[Author] = relationship(back_populates="books")

async def get_db():
    async with SessionLocal() as session:
        yield session

DB = Annotated[AsyncSession, Depends(get_db)]
```

Differences from the sync version:

- The URL names an async driver: `sqlite+aiosqlite://` or `postgresql+asyncpg://`.
- `async_sessionmaker` and `AsyncSession`; every call that touches the database is
  awaited: `await db.scalars(...)`, `await db.get(...)`, `await db.commit()`.
- `expire_on_commit=False` is effectively mandatory: after a commit, an expired attribute
  would need a database round-trip to read, and that can't happen implicitly in async
  code (see below).
- `create_all` must run through `await conn.run_sync(Base.metadata.create_all)`, because
  metadata operations are synchronous APIs.

## The data and the response schema

The test database had 10 authors with 3 books each. The API returns each author with
their books:

```python
from pydantic import BaseModel, ConfigDict

class BookOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str

class AuthorWithBooks(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    books: list[BookOut]
```

A listener counted every SQL statement sent:

```python
from sqlalchemy import event

QUERIES: list[str] = []

@event.listens_for(engine.sync_engine, "before_cursor_execute")
def count(conn, cursor, statement, params, context, executemany):
    QUERIES.append(" ".join(statement.split()))
```

## Attempt 1: just return the authors

```python
@app.get("/lazy", response_model=list[AuthorWithBooks])
async def lazy(db: DB):
    return (await db.scalars(select(Author).order_by(Author.id))).all()
```

```text
/lazy      status=500 queries=1
      SELECT authors.id, authors.name FROM authors ORDER BY authors.id
```

A 500. The server-side `ResponseValidationError` had one entry per author, like this
(trimmed):

```text
{'type': 'get_attribute_error', 'loc': ('response', 0, 'books'),
 'msg': "Error extracting attribute: StatementError: (sqlalchemy.exc.MissingGreenlet) greenlet_spawn has not been called; can't call await_() here. Was IO attempted in an unexpected place?
 [SQL: SELECT books.id, books.title, books.author_id FROM books WHERE ? = books.author_id] ..."}
```

When the response model read `author.books`, the relationship wasn't loaded, so
SQLAlchemy tried to **lazy load** it with a new query. In async mode it can't do that
from a plain attribute access, so it raised `MissingGreenlet`. Note the SQL in the
message: one `SELECT ... WHERE ? = books.author_id` *per author*.

## The same mistake in sync code: N+1

The sync version of that code doesn't crash — which is worse, because the problem is
invisible. The same data, loaded with a sync `Session` and iterated the same way:

```text
sync lazy: queries = 11
sync selectinload: queries = 2
```

One query for the authors, plus one query **per author** for their books: N+1. With 10
authors that's 11 queries; with a page of 100 it's 101, each a network round-trip to a
real database server. This is the most common ORM performance bug, and async SQLAlchemy
at least makes it loud.

## Fix 1: `selectinload`

```python
from sqlalchemy.orm import selectinload

@app.get("/selectin", response_model=list[AuthorWithBooks])
async def selectin(db: DB):
    stmt = select(Author).options(selectinload(Author.books)).order_by(Author.id)
    return (await db.scalars(stmt)).all()
```

```text
/selectin  status=200 queries=2
      SELECT authors.id, authors.name FROM authors ORDER BY authors.id
      SELECT books.author_id, books.id, books.title FROM books WHERE books.author_id IN (?, ?, ?, ...
```

Two queries regardless of the number of authors: the second fetches all their books at
once with `IN (...)`. This is the best default for one-to-many collections.

## Fix 2: `joinedload`

```python
from sqlalchemy.orm import joinedload

@app.get("/joined", response_model=list[AuthorWithBooks])
async def joined(db: DB):
    stmt = select(Author).options(joinedload(Author.books)).order_by(Author.id)
    return (await db.scalars(stmt)).unique().all()
```

```text
/joined    status=200 queries=1
      SELECT authors.id, authors.name, books_1.id AS id_1, books_1.title, books_1.author_id FROM ...
```

One query with a `LEFT OUTER JOIN`. The result has one row per *book*, so each author
appears three times; `.unique()` collapses them back into distinct `Author` objects
(SQLAlchemy requires it for joined collections). Prefer `joinedload` for **many-to-one**
relationships (each book's single author), where the join doesn't multiply rows.
For collections, `selectinload` avoids sending the parent columns again for every child
row, and keeps `LIMIT` queries simple.

## Worked example: when you don't need objects at all

For a summary, ask the database to aggregate instead of loading everything:

```python
from sqlalchemy import func

@app.get("/counts")
async def counts(db: DB):
    stmt = (select(Author.name, func.count(Book.id).label("n"))
            .outerjoin(Book).group_by(Author.id).order_by(Author.id))
    return [{"author": n, "books": c} for n, c in (await db.execute(stmt)).all()]
```

```text
/counts    status=200 queries=1
      SELECT authors.name, count(books.id) AS n FROM authors LEFT OUTER JOIN books ON authors.id ...
      [{'author': 'Author 1', 'books': 3}, {'author': 'Author 2', 'books': 3}]
```

(Only the first two results shown.) One query, no ORM objects, a tiny result set.
`outerjoin` keeps authors with zero books in the count.

### Loading a relationship later, explicitly

Sometimes you have an object and need a relationship you didn't eager-load. In async
code, ask explicitly:

```python
a = await s.get(Author, 1)
await s.refresh(a, ["books"])           # load just that attribute
books = await a.awaitable_attrs.books   # or this, thanks to the AsyncAttrs mixin
```

Both worked in testing (`after refresh: 3`, `awaitable_attrs: 3`); plain `a.books` on a
fresh object raised the same `MissingGreenlet` error. `awaitable_attrs` exists only
because `Base` inherits `AsyncAttrs` — without the mixin it was an `AttributeError`.

## How It Actually Works

SQLAlchemy's ORM is written as synchronous code: attribute access, the unit of work, the
loading machinery. Rather than rewrite it, the asyncio extension runs that sync code
inside a **greenlet** — a lightweight coroutine that can pause in the middle of an
ordinary function call. When the ORM needs to talk to the database, the greenlet pauses,
control returns to the asyncio event loop, the async driver performs the I/O, and the
greenlet resumes with the result.

`await session.scalars(...)` sets that up: `greenlet_spawn` starts the sync ORM code in a
greenlet that's allowed to make those switches. But a plain attribute access like
`author.books` — from your endpoint, or from Pydantic reading attributes for the
response — happens outside any `greenlet_spawn`. If it needs I/O, there's no way to pause
back to the loop, hence "greenlet_spawn has not been called; can't call await_() here".

Eager-loading options decide what's loaded *inside* the awaited query:

- `selectinload` runs a second `SELECT ... WHERE parent_id IN (...)` immediately after
  the first, batching the parent IDs (in chunks for very large lists).
- `joinedload` adds a `LEFT OUTER JOIN` to the original statement and assembles the
  collections from the joined rows.
- `lazyload` (the default for relationships) emits a query on first access — which is
  N+1 in sync code and an error in async code.

You can make loading failures loud in sync code too: `relationship(..., lazy="raise")`
turns any lazy load into an exception, so N+1 can't sneak in.

## Common mistakes

- **Missing `[asyncio]` extra**, giving the greenlet `ImportError`.
- **Returning ORM objects with unloaded relationships** from async endpoints — the
  `MissingGreenlet` 500 above.
- **`expire_on_commit=True` with `AsyncSession`**, then reading attributes after
  commit.
- **Sharing an `AsyncSession` between concurrent tasks** (`asyncio.gather` over several
  queries on one session). A session is not safe for concurrent use; use one session per
  task.
- **Assuming `joinedload` + `.limit()` limits joined rows.** SQLAlchemy handles it: for
  `select(Author).options(joinedload(Author.books)).limit(2)` it compiled
  `SELECT ... FROM (SELECT authors.id, authors.name FROM authors LIMIT ? OFFSET ?) AS anon_1 LEFT OUTER JOIN books ...`,
  limiting authors in a subquery. It's correct but heavier SQL; `selectinload` keeps
  paginated collection queries simple.
- **Fixing N+1 by caching** instead of by loading strategy.
- **Async everything by default.** If your workload is a handful of quick queries per
  request, sync SQLAlchemy in `def` endpoints is simpler and perfectly fast.

## Exercise

1. Recreate the setup with 20 authors × 5 books. Count queries for `/lazy` in a *sync*
   version, then fix it with `selectinload`.
2. Add `GET /books` returning each book with its author's name (`BookWithAuthor`). Which
   loader option fits a many-to-one relationship? Count the queries.
3. Add `lazy="raise"` to `Author.books` and find which of your endpoints now fail.
4. Write `GET /authors/top` returning the three authors with the most books, using a
   single aggregate query with `order_by(func.count(Book.id).desc()).limit(3)`.
