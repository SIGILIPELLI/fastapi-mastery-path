---
description: "Test FastAPI apps with TestClient and pytest: dependency overrides, an in-memory SQLite database, rollback-per-test isolation and coverage."
---

# 08 · Testing with TestClient & pytest

FastAPI apps are unusually easy to test, because every request goes through one ASGI
callable you can call in-process, and every external resource arrives through a
dependency you can override. This lesson builds a test setup for a database-backed app
that is fast, isolated (no test sees another's data) and touches no real files — and
shows the three failures you hit on the way, because you will hit them too.

Tools: `pytest` (9.1.1 here), FastAPI's `TestClient`, and optionally `pytest-cov`
(7.1.0). For pytest itself in depth, see the
[Python Testing Mastery Path](https://sigilipelli.github.io/python-testing-mastery-path/).

## The app under test

```python
# app/db.py
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

engine = create_engine("sqlite:///bookshop.db")
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


def get_db():
    with SessionLocal() as session:
        yield session
```

```python
# app/main.py
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import String, select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.orm import Mapped, Session, mapped_column

from app.db import Base, get_db


class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    isbn: Mapped[str] = mapped_column(String(13), unique=True)


class BookIn(BaseModel):
    title: str = Field(min_length=1)
    isbn: str = Field(pattern=r"^\d{13}$")


class BookOut(BookIn):
    model_config = ConfigDict(from_attributes=True)
    id: int


DB = Annotated[Session, Depends(get_db)]
app = FastAPI()


@app.post("/books", response_model=BookOut, status_code=201)
def create_book(data: BookIn, db: DB):
    book = Book(**data.model_dump())
    db.add(book)
    try:
        db.commit()
    except IntegrityError:
        db.rollback()
        raise HTTPException(409, "duplicate ISBN")
    return book


@app.get("/books", response_model=list[BookOut])
def list_books(db: DB):
    return db.scalars(select(Book).order_by(Book.id)).all()
```

Note that `app/db.py` creates an engine for `bookshop.db` at import time but never
connects; SQLAlchemy engines are lazy. The tests below never create that file.

## Failure 1: an in-memory database with no tables

The obvious first attempt: point an override at an in-memory SQLite engine and create the
tables on it.

```python
engine = create_engine("sqlite://")
TestSession = sessionmaker(bind=engine, expire_on_commit=False)
Base.metadata.create_all(engine)

def override():
    with TestSession() as s:
        yield s

def test_create_naive():
    app.dependency_overrides[get_db] = override
    ...
```

```text
E       sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) no such table: books
1 failed in 0.90s
```

The tables *were* created — on a different connection. As lesson 4 found, an in-memory
SQLite URL uses `SingletonThreadPool`: one connection per thread, and each in-memory
connection is its own separate database. `create_all` ran on the test's main thread; the
`def` endpoint ran in a worker thread and got a brand-new, empty database.

The fix is `StaticPool`, which hands every caller the **same single connection**.

## Failure 2: SQLite's thread check

With `poolclass=StaticPool` alone:

```text
E       sqlite3.ProgrammingError: SQLite objects created in a thread can only be used in that same thread. The object was created in thread id 8358969216 and this is thread id 6162198528.
```

Python's `sqlite3` refuses cross-thread use by default. SQLAlchemy disables that check
automatically for *file* databases (lesson 4), but not for this in-memory one, so pass
`connect_args={"check_same_thread": False}`. It's safe here because tests make one
request at a time.

## Failure 3: rollback didn't isolate the tests

The plan for isolation: open a connection, begin a transaction, give the app a session
bound to that connection in **savepoint mode** (so the endpoint's `commit()` only
releases a savepoint), and roll the outer transaction back after the test. With only the
two fixes above, three tests failed:

```text
E       AssertionError: assert [{'title': 'D...19', 'id': 1}] == []
E       AssertionError: assert 409 == 201
E       AssertionError: assert ['Dune', 'B', 'A'] == ['B', 'A']
```

Books from earlier tests were still there: the rollback hadn't undone them. The `sqlite3`
driver manages transactions on its own — it decides when to emit `BEGIN` and commits
around some statements — which breaks the `SAVEPOINT` nesting. SQLAlchemy's documented
recipe is to switch the driver's handling off and have SQLAlchemy emit `BEGIN` itself.
With that, every test passed. On PostgreSQL none of this is needed; the same fixture works
as written.

## The working fixtures

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine, event
from sqlalchemy.orm import Session
from sqlalchemy.pool import StaticPool

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
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

How the three fixtures fit together:

- **`engine`** (session-scoped): one in-memory database for the whole run, tables created
  once. Fast.
- **`db`** (per test): a connection with an outer transaction that's always rolled back.
  `join_transaction_mode="create_savepoint"` makes the session's `commit()` and
  `rollback()` operate on savepoints inside that outer transaction, so the endpoint's
  error handling (rollback after `IntegrityError`) behaves exactly as in production.
- **`client`** (per test): overrides `get_db` to yield *the same session the test has*,
  so a test can seed data through `db` and the app sees it, then clears the override.
  Using `TestClient` as a context manager also runs the app's lifespan startup and
  shutdown (Level 3 lesson 5).

## The tests

```python
# tests/test_books.py
import pytest

from app.main import Book

DUNE = {"title": "Dune", "isbn": "9780441172719"}


def test_create_book(client):
    r = client.post("/books", json=DUNE)
    assert r.status_code == 201
    assert r.json() == {"id": 1, **DUNE}


def test_database_is_empty_again(client):
    # The previous test's book was rolled back.
    assert client.get("/books").json() == []


def test_duplicate_isbn_is_409_and_session_still_usable(client, db):
    assert client.post("/books", json=DUNE).status_code == 201
    r = client.post("/books", json={"title": "Dune (copy)", "isbn": DUNE["isbn"]})
    assert r.status_code == 409
    assert [b.title for b in db.query(Book)] == ["Dune"]


@pytest.mark.parametrize("payload, field", [
    ({"title": "", "isbn": "9780441172719"}, "title"),
    ({"title": "X", "isbn": "123"}, "isbn"),
    ({"isbn": "9780441172719"}, "title"),
])
def test_validation_errors(client, payload, field):
    r = client.post("/books", json=payload)
    assert r.status_code == 422
    assert r.json()["detail"][0]["loc"] == ["body", field]


def test_seeded_data_via_session(client, db):
    db.add_all([Book(title="B", isbn="1" * 13), Book(title="A", isbn="2" * 13)])
    db.flush()
    titles = [b["title"] for b in client.get("/books").json()]
    assert titles == ["B", "A"]
```

```text
tests/test_books.py::test_create_book PASSED                             [ 14%]
tests/test_books.py::test_database_is_empty_again PASSED                 [ 28%]
tests/test_books.py::test_duplicate_isbn_is_409_and_session_still_usable PASSED [ 42%]
tests/test_books.py::test_validation_errors[payload0-title] PASSED       [ 57%]
tests/test_books.py::test_validation_errors[payload1-isbn] PASSED        [ 71%]
tests/test_books.py::test_validation_errors[payload2-title] PASSED       [ 85%]
tests/test_books.py::test_seeded_data_via_session PASSED                 [100%]
============================== 7 passed in 0.03s ===============================
```

Thirty milliseconds for seven database tests. Notable details:

- `test_create_book` asserts `"id": 1` — reliable only because every test starts from an
  empty, rolled-back table. (SQLite reuses the rolled-back ID. PostgreSQL sequences
  don't roll back, so on PostgreSQL don't assert on exact IDs.)
- `test_duplicate_isbn...` proves the session is still usable after the endpoint's
  `rollback()` — the savepoint mode at work.
- `test_seeded_data_via_session` uses `db.flush()`, not `commit()`, to make the rows
  visible to the app's queries on the same connection.

## Worked example: measuring what's covered

```bash
pip install pytest-cov
python -m pytest -q --cov=app --cov-report=term-missing
```

```text
Name              Stmts   Miss  Cover   Missing
-----------------------------------------------
app/__init__.py       0      0   100%
app/db.py             9      2    78%   13-14
app/main.py          32      0   100%
-----------------------------------------------
TOTAL                41      2    95%
```

The uncovered lines 13–14 are the body of the real `get_db` — the one the tests
override. That's expected, and a good reminder that overrides mean the production
dependency itself needs at least one test elsewhere (or a smoke test against the real
configuration). Coverage tells you what didn't run, not whether what ran was checked.

## How It Actually Works

`TestClient` is a subclass of an HTTP client (`httpx`'s API, delivered here through the
`httpx2` package — see Level 1 lesson 1's note) whose transport, instead of opening a
socket, calls your ASGI app directly. For each request it builds the `scope` dict,
supplies `receive` with the request body, collects the `send` messages into a response,
and returns it. Your app can't tell the difference between this and Uvicorn, except
that the host is `testserver`.

The test client runs the app in an event loop on a separate thread (via AnyIO's
blocking portal), so your synchronous test code can call `client.get()` without
`await`. `def` endpoints then run in the AnyIO worker thread pool, as in production —
which is exactly why the thread-related SQLite failures above appeared. Tests that
exercise the real threading model are a feature.

`dependency_overrides` is consulted during dependency resolution (lesson 1): when
FastAPI is about to call `get_db`, it finds `override_get_db` in the dict and calls that
instead, with the same caching and yield semantics.

## Common mistakes

- **In-memory SQLite without `StaticPool`** — "no such table".
- **Forgetting `app.dependency_overrides.clear()`**, so one test's override leaks into
  the next. Do it in the fixture's teardown, not at the end of the test body.
- **Tests that depend on order** (test B passes only after test A created data). The
  rollback fixture prevents it; check by running a single test on its own.
- **Testing against the dev database file**, which leaves junk and fails on other
  machines.
- **Mocking the ORM** instead of using a real (in-memory or containerised) database. You
  end up testing your mocks.
- **Asserting exact error message text** from Pydantic. Assert `type` and `loc`, which
  are stable.
- **Assuming SQLite behaves like production.** It differs on types, constraints and
  concurrency. Run at least part of the suite against the real database engine in CI.

## Exercise

1. Recreate the app and conftest. Delete the two `event.listens_for` functions and
   confirm which tests fail and why.
2. Add `PATCH /books/{id}` and write tests for success, 404, 409 on a duplicate ISBN,
   and a 422.
3. Add a test that proves isolation: create a book in one test, and in another test
   (run on its own with `pytest -k`) assert the table is empty.
4. Override `get_settings` (lesson 7) in a fixture and write a test for an endpoint
   whose behaviour depends on a setting.
