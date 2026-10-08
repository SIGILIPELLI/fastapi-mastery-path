---
description: "Level 2 of FastAPI: dependency injection, yield dependencies, async vs sync, SQLAlchemy 2.x, Alembic, settings, testing, pagination and a project."
---

# Level 2 · Intermediate <span class="level-badge">Real Data</span>

Goal: turn a toy API into one backed by a real database, configured per environment, and
covered by fast, isolated tests — and understand the concurrency model well enough not
to freeze your own server.

Three ideas carry this level:

1. **Everything an endpoint needs is a dependency.** Sessions, settings, the current
   user, pagination policy. Declared once, injected everywhere, replaced in tests.
2. **Know where your code runs.** `async def` runs on the event loop; `def` runs in a
   40-thread pool. Blocking the loop stalls every request in the process — lesson 3
   measures by how much.
3. **The database is the source of truth.** Constraints live in the schema, schema
   changes live in migrations, and tests run against a real database engine rather than
   mocks.

## Modules

1. [Dependency Injection with Depends](01-dependency-injection.md) — function and class dependencies, chains, router-wide checks, caching, overrides
2. [Yield Dependencies, Caching & Overrides](02-yield-dependencies.md) — setup/teardown, `scope="function"` vs `"request"`, the commit-after-response trap
3. [`async def` vs `def`](03-async-vs-sync.md) — measured: blocking the loop, the thread-pool limit, offloading
4. [SQLAlchemy 2.x with FastAPI](04-sqlalchemy.md) — typed models, session per request, CRUD, `IntegrityError`, reading the SQL
5. [Async SQLAlchemy, Relationships & N+1 Queries](05-async-sqlalchemy.md) — `AsyncSession`, `MissingGreenlet`, `selectinload` vs `joinedload`
6. [Database Migrations with Alembic](06-alembic.md) — autogenerate, naming conventions, backfilling `NOT NULL` columns, SQLite batch mode
7. [Configuration with pydantic-settings](07-settings.md) — env vars, `.env`, `SecretStr`, JSON lists, caching and overrides
8. [Testing with TestClient & pytest](08-testing.md) — in-memory SQLite done right, rollback-per-test isolation, coverage
9. [Pagination, Filtering & Sorting](09-pagination-filtering.md) — bounded filter models, safe sorting, offset vs cursor pagination
10. [Project — A Bookshelf API with a Database and Tests](10-project-bookshelf.md) — everything above in one app, plus a timezone bug found and fixed

## What you need before starting

- **Level 1 of this course**, or equivalent: request bodies, response models, routers,
  exception handlers.
- **Basic SQL**: `SELECT`, `JOIN`, `WHERE`, indexes, constraints. The
  [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) covers it.
- **pytest basics**: fixtures and `assert`. The
  [Python Testing Mastery Path](https://sigilipelli.github.io/python-testing-mastery-path/)
  goes deeper.
- Install: `pip install "fastapi[standard]" "sqlalchemy[asyncio]" aiosqlite alembic pydantic-settings pytest`.

## How the examples were checked

Everything in this level was run on Python 3.14.7 with **FastAPI 0.143.0**,
**SQLAlchemy 2.1.4**, **Alembic 1.20.0**, **pydantic-settings 2.15.0**, **aiosqlite
0.22.1** and **pytest 9.1.1**, against SQLite (file and in-memory). The concurrency
measurements in lesson 3 were taken against a real Uvicorn 0.54.0 process with
concurrent `httpx` requests on one laptop; they show the shape of the effect, not
benchmark numbers. Several failures are shown exactly as they happened — a missing
`greenlet` package, an Alembic migration that left a temporary table behind, three
different in-memory SQLite test failures — because you'll meet them too.

!!! note "What wasn't run"
    No PostgreSQL server was available, so PostgreSQL-specific behaviour (transactional
    DDL, `ILIKE`, sequence behaviour after rollback, timezone-aware columns) is described
    from SQLAlchemy's and PostgreSQL's documented behaviour and marked as such in the
    lessons. Wherever SQLite and PostgreSQL differ in a way that matters, the lesson
    says so.
