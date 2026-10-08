---
description: "Test async FastAPI apps with httpx AsyncClient and AnyIO: async SQLAlchemy rollback isolation, the lifespan gotcha and real concurrency tests."
---

# 08 · Async Tests & Isolated Test Databases

Level 2 lesson 8 tested a synchronous app with `TestClient`. That client works for async
apps too — it runs them in an event loop on another thread — but your *test code* stays
synchronous, so you can't `await` an async session to seed data, and you can't fire
concurrent requests with `asyncio.gather`. When the app is async through and through,
write the tests async as well. This lesson builds that setup, and hits two traps on the
way.

Tools: `httpx.AsyncClient` with `ASGITransport` (httpx 0.28.1 here) and the pytest plugin
that ships with **AnyIO** (4.15.1), which runs `async def` tests. (pytest-asyncio is an
alternative plugin; it was also installed and didn't interfere.)

## The app

```python
# app/main.py
from contextlib import asynccontextmanager
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, Request
from pydantic import BaseModel, ConfigDict
from sqlalchemy import String, select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    pass


class Tag(Base):
    __tablename__ = "tags"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(30), unique=True)


class TagIn(BaseModel):
    name: str


class TagOut(TagIn):
    model_config = ConfigDict(from_attributes=True)
    id: int


@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = create_async_engine("sqlite+aiosqlite:///tags.db")
    yield {"sessionmaker": async_sessionmaker(engine, expire_on_commit=False)}
    await engine.dispose()


app = FastAPI(lifespan=lifespan)


async def get_db(request: Request):
    async with request.state.sessionmaker() as session:
        yield session


DB = Annotated[AsyncSession, Depends(get_db)]


@app.post("/tags", response_model=TagOut, status_code=201)
async def create_tag(data: TagIn, db: DB):
    tag = Tag(name=data.name.lower())
    db.add(tag)
    try:
        await db.commit()
    except IntegrityError:
        await db.rollback()
        raise HTTPException(409, "tag exists")
    return tag


@app.get("/tags", response_model=list[TagOut])
async def list_tags(db: DB):
    return (await db.scalars(select(Tag).order_by(Tag.name))).all()
```

The engine is created in the lifespan handler and exposed through lifespan state, as
lesson 5 recommends.

## Trap 1: `ASGITransport` doesn't run the lifespan

The obvious first test:

```python
@pytest.mark.anyio
async def test_without_lifespan():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        r = await c.get("/tags")
        assert r.status_code == 200
```

```text
E           KeyError: 'sessionmaker'
E           AttributeError: 'State' object has no attribute 'sessionmaker'
```

`ASGITransport` sends HTTP requests to the app, but never sends the ASGI `lifespan`
startup event — so `request.state.sessionmaker` was never set. (`TestClient` used as a
context manager *does* run the lifespan.) Two options: override the dependencies that
read lifespan state, which is what the fixtures below do, or drive the lifespan
yourself; a commonly used helper for that is the `asgi-lifespan` package's
`LifespanManager` (not used or tested here).

## The fixtures

```python
# tests/conftest.py
import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy import event
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.pool import StaticPool

from app.main import Base, app, get_db


@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"


@pytest.fixture(scope="session")
async def engine():
    engine = create_async_engine("sqlite+aiosqlite://", poolclass=StaticPool)

    # Same pysqlite savepoint fix as the sync suite, applied to the sync core.
    @event.listens_for(engine.sync_engine, "connect")
    def _no_pysqlite_tx(dbapi_connection, record):
        dbapi_connection.isolation_level = None

    @event.listens_for(engine.sync_engine, "begin")
    def _emit_begin(conn):
        conn.exec_driver_sql("BEGIN")

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest.fixture
async def db(engine):
    async with engine.connect() as conn:
        outer = await conn.begin()
        session = AsyncSession(bind=conn, join_transaction_mode="create_savepoint",
                               expire_on_commit=False)
        try:
            yield session
        finally:
            await session.close()
            await outer.rollback()


@pytest.fixture
async def client(db):
    async def override_get_db():
        yield db
    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()


@pytest.fixture
async def real_client(tmp_path):
    """Real per-request sessions against a throwaway file database.

    No shared session: each request gets its own, exactly as in production.
    Use it for tests about concurrency; it's slower and has no rollback isolation,
    so the database is a fresh temporary file per test instead.
    """
    url = f"sqlite+aiosqlite:///{tmp_path / 'test.db'}"
    engine = create_async_engine(url)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    maker = async_sessionmaker(engine, expire_on_commit=False)

    async def override_get_db():
        async with maker() as session:
            yield session
    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
    await engine.dispose()
```

- **`anyio_backend`** tells AnyIO's plugin to run tests on asyncio. Session scope lets the
  session-scoped async `engine` fixture share the same event loop.
- **`engine`**: an in-memory database with `StaticPool` (one shared connection) and the
  same pysqlite savepoint fix as Level 2 lesson 8, registered on `engine.sync_engine` —
  async engines wrap a sync core, and the events live there. `aiosqlite` connections
  aren't subject to the thread check, so `check_same_thread` isn't needed.
- **`db`**: a connection with an outer transaction, and an `AsyncSession` in savepoint
  mode; rolled back after each test.
- **`client`**: overrides `get_db` to yield that session.
- **`real_client`**: explained in trap 2.

## Tests

```python
# tests/test_tags.py
import asyncio
import pytest

pytestmark = pytest.mark.anyio


async def test_create_and_list(client):
    r = await client.post("/tags", json={"name": "SciFi"})
    assert r.status_code == 201 and r.json() == {"id": 1, "name": "scifi"}
    assert [t["name"] for t in (await client.get("/tags")).json()] == ["scifi"]


async def test_isolated_from_previous_test(client):
    assert (await client.get("/tags")).json() == []


async def test_duplicate_is_409(client):
    assert (await client.post("/tags", json={"name": "a"})).status_code == 201
    assert (await client.post("/tags", json={"name": "A"})).status_code == 409
    assert len((await client.get("/tags")).json()) == 1


async def test_concurrent_creates_with_real_sessions(real_client):
    # Ten requests at once, half of them duplicates of each other.
    names = ["x", "y", "z", "w", "v"] * 2
    results = await asyncio.gather(*[real_client.post("/tags", json={"name": n}) for n in names])
    codes = sorted(r.status_code for r in results)
    print("statuses:", codes)
    assert codes == [201] * 5 + [409] * 5
    assert len((await real_client.get("/tags")).json()) == 5
```

```text
$ python -m pytest -q -s
...statuses: [201, 201, 201, 201, 201, 409, 409, 409, 409, 409]
.
4 passed in 0.12s
```

Three more runs took 0.13 s, 0.15 s and 0.11 s, all passing. `pytestmark = pytest.mark.anyio`
marks every test in the module as async.

## Trap 2: concurrent requests and a shared session

The first version of the concurrency test used the ordinary `client` fixture:

```python
async def test_concurrent_requests_in_one_test(client):
    names = [f"t{i}" for i in range(5)]
    results = await asyncio.gather(*[client.post("/tags", json={"name": n}) for n in names])
```

```text
E               sqlalchemy.exc.IllegalStateChangeError: Method 'close()' can't be called here; method '_prepare_impl()' is already in progress and this would cause an unexpected state change to <SessionTransactionState.CLOSED: 5> ...
E           sqlalchemy.exc.InvalidRequestError: This session is provisioning a new connection; concurrent operations are not permitted ...
```

along with an `SAWarning` that `Session.add()` was called during a flush. The override
handed the **same** `AsyncSession` to five simultaneous requests, and an `AsyncSession`
must never be used by concurrent tasks (lesson 5 of Level 2). In production each request
has its own session, so this was a test-setup bug, not an app bug — but a confusing one.

The fix is a separate fixture for concurrency tests, `real_client`: a fresh temporary
**file** database per test (pytest's `tmp_path`), and a dependency that opens a new
session per request, like the real app. It can't use the rollback trick, so each test
gets its own database file instead. With it, ten simultaneous creates — five names, each
sent twice — produced exactly five 201s and five 409s: the unique constraint, not luck,
decided the winners.

Keep `real_client` for the few tests that are *about* concurrency; it's slower than the
rollback fixture and the cleanup is coarser.

## Worked example: seeding through the session

Because the test and the app share the session, async seeding is just an `await`:

```python
async def test_list_is_sorted(client, db):
    db.add_all([Tag(name="b"), Tag(name="a")])
    await db.flush()
    names = [t["name"] for t in (await client.get("/tags")).json()]
    assert names == ["a", "b"]
```

`flush()` sends the `INSERT`s inside the open transaction without committing, so the
app's `SELECT` on the same connection sees them, and the outer rollback removes them.
(Added to the suite after the run above, with `from app.main import Tag` inside the test;
the suite then reported `5 passed`.)

## How It Actually Works

AnyIO's pytest plugin looks for tests marked `anyio` (or async fixtures used by them),
and runs each one inside an event loop for the backend named by the `anyio_backend`
fixture. Async fixtures are driven in that same loop, which matters: an async engine
created in one event loop can't be used from another. Scoping `anyio_backend` to the
session keeps one loop for the session-scoped `engine`.

`ASGITransport` implements httpx's transport interface by building an ASGI HTTP scope
from each request and `await`ing `app(scope, receive, send)` directly — in the **same**
event loop as the test. That's different from `TestClient`, which starts the app in its
own loop on a separate thread. Same-loop execution is what makes `asyncio.gather` over
requests meaningful: the requests genuinely interleave at each `await` inside the app,
exactly where real concurrent requests would.

`ASGITransport` only knows HTTP scopes; lifespan and WebSocket scopes aren't part of
it. That's trap 1.

## Common mistakes

- **Expecting `ASGITransport` to run startup/shutdown.**
- **One session shared across concurrent requests** in tests.
- **Event-loop mismatches**: a session-scoped async fixture with a function-scoped loop.
  Symptoms include "attached to a different loop" errors.
- **Mixing `TestClient` and async fixtures** that share objects bound to different loops.
- **Asserting exact IDs** in tests that run concurrently or on PostgreSQL.
- **Only testing sequentially.** Race conditions (duplicate inserts, lost updates) need a
  `gather`-style test against real per-request sessions.

## Exercise

1. Add `DELETE /tags/{id}` and write async tests for 204 and 404.
2. Write a concurrency test for a "like" counter endpoint implemented as
   read-increment-write. Watch it lose updates under `gather`, then fix it with an atomic
   `UPDATE ... SET likes = likes + 1` and watch the test pass.
3. Make the session-scoped engine function-scoped instead and measure how much slower
   the suite gets.
4. Install `asgi-lifespan`, use its `LifespanManager` in a fixture, and remove the
   `get_db` override for one smoke test that exercises the real lifespan.
