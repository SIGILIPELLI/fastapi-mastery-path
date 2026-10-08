---
description: "Paginate, filter and sort FastAPI list endpoints safely: whitelisted sorts, bounded filter models, Link headers, and cursor vs offset pagination."
---

# 09 · Pagination, Filtering & Sorting

Every list endpoint eventually needs three things: a way to narrow results (filters), a
way to order them (sorting), and a way to fetch them in pieces (pagination). They look
simple, and they're where a surprising number of API bugs and security holes live —
unbounded queries, SQL injection through a sort parameter, pages that skip or repeat
items. This lesson builds both common pagination styles against a real database and
shows where each one breaks.

The examples use 100 seeded books in an in-memory SQLite database (lesson 8's
`StaticPool` setup), with SQLAlchemy 2.x in `def` endpoints.

## A filter model with hard limits

```python
from typing import Annotated, Literal
from pydantic import BaseModel, ConfigDict, Field

class BookQuery(BaseModel):
    model_config = ConfigDict(extra="forbid")
    genre: Literal["scifi", "fantasy", "crime", "history"] | None = None
    min_price: int | None = Field(None, ge=0)
    max_price: int | None = Field(None, ge=0)
    q: str | None = Field(None, max_length=50)
    sort: Literal["title", "-title", "price", "-price", "published", "-published"] = "title"
    limit: int = Field(20, ge=1, le=100)
    offset: int = Field(0, ge=0, le=10_000)
```

Every parameter is bounded: `limit` can't exceed 100, `offset` can't exceed 10,000, the
search string can't exceed 50 characters. `sort` is a `Literal` — a closed list — with a
`-` prefix meaning descending. `extra="forbid"` turns typos into errors.

The model maps sort names to real columns through a dict, never through string
formatting:

```python
SORTABLE = {"title": Book.title, "price": Book.price_cents, "published": Book.published}

def apply_filters(stmt, f: BookQuery):
    if f.genre:
        stmt = stmt.where(Book.genre == f.genre)
    if f.min_price is not None:
        stmt = stmt.where(Book.price_cents >= f.min_price)
    if f.max_price is not None:
        stmt = stmt.where(Book.price_cents <= f.max_price)
    if f.q:
        stmt = stmt.where(Book.title.ilike(f"%{f.q}%"))
    return stmt
```

## Offset pagination with a total

```python
from fastapi import HTTPException, Query, Request, Response
from sqlalchemy import func, select

class Page(BaseModel):
    items: list[BookOut]
    total: int
    limit: int
    offset: int

@app.get("/books", response_model=Page)
def list_books(f: Annotated[BookQuery, Query()], db: DB,
               request: Request, response: Response):
    if f.min_price is not None and f.max_price is not None and f.min_price > f.max_price:
        raise HTTPException(422, "min_price must not exceed max_price")
    col = SORTABLE[f.sort.lstrip("-")]
    order = col.desc() if f.sort.startswith("-") else col.asc()
    base = apply_filters(select(Book), f)
    total = db.scalar(select(func.count()).select_from(base.subquery()))
    items = db.scalars(base.order_by(order, Book.id).limit(f.limit).offset(f.offset)).all()
    if f.offset + f.limit < total:
        nxt = request.url.include_query_params(offset=f.offset + f.limit)
        response.headers["Link"] = f'<{nxt}>; rel="next"'
    return Page(items=items, total=total, limit=f.limit, offset=f.offset)
```

`GET /books?genre=scifi&sort=-price&limit=3`:

```text
filtered: 25 [('Book 052', 2424), ('Book 048', 2276), ('Book 100', 2200)]
Link: <http://testserver/books?genre=scifi&sort=-price&limit=3&offset=3>; rel="next"
```

25 science-fiction books in total, the three most expensive first, and a `Link` header
(RFC 8288) that a client can follow without building URLs itself. `include_query_params`
kept the other filters and replaced only `offset`.

Two details matter in the `ORDER BY`:

- **The tie-breaker `Book.id`.** Many books can share a price. Without a unique final
  sort key, the database may return rows with equal prices in any order — and a
  *different* order on the next page's query, so items can repeat or vanish between
  pages.
- **The count uses the same filtered statement** as a subquery, so `total` always agrees
  with the filters.

The real SQL for `?genre=crime&q=0&limit=2&offset=4`:

```text
SELECT count(*) AS count_1 FROM (SELECT books.id AS id, books.title AS title, ... FROM books WHERE books.genre = ? AND lower(books.title) LIKE lower(?)) AS anon_1 ('crime', '%0%')
SELECT books.id, books.title, books.genre, books.price_cents, books.published FROM books WHERE books.genre = ? AND lower(books.title) LIKE lower(?) ORDER BY books.title ASC, books.id LIMIT ? OFFSET ? ('crime', '%0%', 2, 4)
```

Every user-supplied value is a bound parameter.

### What the limits rejected

```text
{'sort': 'price; DROP TABLE books'}       422 [('literal_error', ['query', 'sort'])]
{'limit': 1000}                           422 [('less_than_equal', ['query', 'limit'])]
{'min_price': 900, 'max_price': 100}      422 min_price must not exceed max_price
{'colour': 'red'}                         422 [('extra_forbidden', ['query', 'colour'])]
```

The injection attempt never got near the database: it isn't one of the allowed sort
values. Code that did `text(f"ORDER BY {sort}")` would have been exploitable.

## Worked example: offset pagination drifts

Offset pagination says "skip N rows". If rows are inserted or deleted between page
requests, N points somewhere else. Fetch page 1 sorted by title, then insert a book whose
title sorts first, then fetch page 2:

```text
offset page1: ['Book 001', 'Book 002', 'Book 003', 'Book 004', 'Book 005']
offset page2: ['Book 005', 'Book 006', 'Book 007', 'Book 008', 'Book 009']
```

`Book 005` appeared twice: the new row pushed everything down by one. A deletion would
cause the opposite — an item silently skipped. For a feed or an export, that's a real
bug. Offset paging is also slow at depth: the database has to walk and discard all the
skipped rows, which is why `offset` is capped at 10,000 here.

## Cursor (keyset) pagination

Instead of "skip N", say "give me rows after *this one*". The cursor encodes the sort
key of the last row seen:

```python
import base64, json
from sqlalchemy import tuple_

def encode_cursor(price: int, id_: int) -> str:
    return base64.urlsafe_b64encode(json.dumps([price, id_]).encode()).decode().rstrip("=")

def decode_cursor(c: str) -> tuple[int, int]:
    try:
        price, id_ = json.loads(base64.urlsafe_b64decode(c + "=" * (-len(c) % 4)))
        return int(price), int(id_)
    except Exception:
        raise HTTPException(400, "Invalid cursor")

class CursorPage(BaseModel):
    items: list[BookOut]
    next_cursor: str | None

@app.get("/books/by-price", response_model=CursorPage)
def by_price(db: DB, limit: Annotated[int, Query(ge=1, le=100)] = 20,
             cursor: str | None = None):
    stmt = select(Book).order_by(Book.price_cents, Book.id).limit(limit + 1)
    if cursor:
        p, i = decode_cursor(cursor)
        stmt = stmt.where(tuple_(Book.price_cents, Book.id) > tuple_(p, i))
    rows = db.scalars(stmt).all()
    more = len(rows) > limit
    rows = rows[:limit]
    nxt = encode_cursor(rows[-1].price_cents, rows[-1].id) if more and rows else None
    return CursorPage(items=rows, next_cursor=nxt)
```

Fetching `limit + 1` rows tells you whether there's a next page without a `COUNT`. The
same drift test, inserting a cheaper book between pages:

```text
cursor page1: [('Book 000 (new)', 1), ('Book 055', 535), ('Book 001', 537)] WzUzNywgMV0
cursor page2: [('Book 056', 572), ('Book 002', 574), ('Book 057', 609)]
bad cursor:   {'detail': 'Invalid cursor'}
```

Page 2 continued exactly where page 1 stopped; the insert (price 0, which sorts before
everything) neither shifted nor duplicated anything. The cursor `WzUzNywgMV0` is just
base64 for `[537, 1]` — opaque to clients, but not secret; never put anything in it you
wouldn't show them. The SQL:

```text
SELECT ... FROM books WHERE (books.price_cents, books.id) > (?, ?) ORDER BY books.price_cents, books.id LIMIT ? OFFSET ? (537, 1, 4, 0)
```

The row-value comparison `(price, id) > (537, 1)` means "price greater than 537, or price
equal to 537 and id greater than 1" — exactly "after this row" in that sort order. With a
composite index on `(price_cents, id)`, the database can jump straight to the position,
so page 1,000 is as fast as page 1.

### Which to use

| | Offset | Cursor |
|---|---|---|
| Jump to page 37 | yes | no |
| Total count | natural (but costs a query) | usually omitted |
| Stable under inserts/deletes | no | yes |
| Fast deep in the list | no | yes, with an index |
| Change sort order mid-stream | easy | the cursor is tied to one sort |

Admin tables with page numbers: offset, with a cap. Feeds, infinite scroll, exports,
sync APIs: cursor.

## How It Actually Works

For `Annotated[BookQuery, Query()]`, FastAPI expands the model's fields into individual
query parameters (they appear separately in OpenAPI), collects the values from the query
string, and validates them together as a model — which is why `extra="forbid"` can see
unknown keys. The cross-field check (`min_price > max_price`) is done in the endpoint
here; a `model_validator` on `BookQuery` would also work and would produce a standard
validation error shape.

On the database side, `LIMIT n OFFSET m` still makes the engine produce and discard the
first `m` rows of the sorted result. A keyset `WHERE (a, b) > (x, y)` lets the engine use
an index range scan starting at `(x, y)`. Not every database plans row-value comparisons
well — if yours doesn't, the equivalent
`WHERE a > x OR (a = x AND b > y)` works everywhere.

`select(func.count()).select_from(base.subquery())` counts the rows of the filtered query
without loading them. On large tables even that can be expensive; options include
caching the count, returning an estimate, or dropping `total` in favour of `has_more`.

## Common mistakes

- **No maximum `limit`.** `?limit=1000000` is a denial-of-service request.
- **Sorting by a raw string** from the query (`order_by(text(sort))`) — SQL injection.
  Whitelist.
- **No unique tie-breaker** in `ORDER BY`, so pagination is nondeterministic.
- **Counting with `len(query.all())`**, loading every row to count them.
- **A filter applied to the items but not the count**, so `total` lies.
- **Deep offset pagination on big tables** for exports or sync. Use a cursor.
- **Cursors that clients construct or parse.** Keep them opaque so you can change the
  encoding later.

## Exercise

1. Add `published_after` and `published_before` date filters, with a validator that
   rejects an inverted range.
2. Add a `rel="prev"` Link header for offset pagination (omit it on the first page).
3. Extend the cursor endpoint to support `sort=-price` (descending). What changes in the
   `WHERE` comparison and the `ORDER BY`?
4. Seed 100,000 rows and time `offset=90000` against the equivalent cursor query. Add an
   index on `(price_cents, id)` and time both again. Report your numbers as yours — they
   depend on the machine.
