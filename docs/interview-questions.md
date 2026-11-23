---
title: "FastAPI Interview Questions and Answers (179 Questions)"
description: "179 FastAPI interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the FastAPI Mastery Path."
---

# FastAPI Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free FastAPI Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What FastAPI Is: ASGI, Starlette, Pydantic and One Request](level-1/01-what-fastapi-is.md)

**Q1.** What three jobs do type hints do in FastAPI?

**Answer:** Validate input, convert/serialize data and generate OpenAPI documentation

One source of truth.

**Q2.** What's FastAPI built on?

**Answer:** Starlette (ASGI) and Pydantic

ASGI server needed (Uvicorn).

**Q3.** Where are interactive docs served by default?

**Answer:** /docs (Swagger UI) and /redoc

Generated from your code.

**Q4.** What does `uvicorn main:app --reload` do?

**Answer:** Runs the ASGI app and restarts on code changes (development)

main:app = module:variable.

### [02 · Installing FastAPI &amp; Your First App](level-1/02-first-app.md)

**Q5.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
@app.get('/ping')
def ping():
    return {'ok': True}
c=TestClient(app)
r=c.get('/ping')
print(r.status_code, r.json(), c.get('/nope').status_code)
```

**Answer:** 200 {'ok': True} 404

Dicts become JSON; unknown routes give 404. Verified by running it with FastAPI's TestClient.

**Q6.** What does an endpoint decorator like @app.get('/items') define?

**Answer:** The HTTP method and path that triggers the function

Path operation.

**Q7.** What's the default status code of a successful GET?

**Answer:** 200

Use status_code= to change.

**Q8.** What does `async def` allow in an endpoint?

**Answer:** Using await for non-blocking I/O

Both def and async def work.

### [03 · Path &amp; Query Parameters](level-1/03-path-query-params.md)

**Q9.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
@app.get('/items/{item_id}')
def item(item_id: int, q: str | None = None, limit: int = 10):
    return {'id': item_id, 'q': q, 'limit': limit}
c=TestClient(app)
print(c.get('/items/5?q=hi').json())
r=c.get('/items/abc')
print(r.status_code, r.json()['detail'][0]['type'])
```

**Answer:** {'id': 5, 'q': 'hi', 'limit': 10}
422 int_parsing

Path params are converted and invalid ones give a 422 validation error. Verified by running it with FastAPI's TestClient.

**Q10.** How does FastAPI tell path parameters from query parameters?

**Answer:** Path parameters appear in the path template; other simple-typed arguments are query parameters

Pydantic models are body.

**Q11.** What makes a query parameter optional?

**Answer:** A default value (e.g. = None)

No default means required.

**Q12.** How do you constrain a query value to 1-100?

**Answer:** Query(ge=1, le=100)

Validated automatically.

### [04 · Request Bodies with Pydantic Models](level-1/04-request-bodies.md)

**Q13.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
class Item(BaseModel):
    name: str
    price: float = Field(gt=0)
@app.post('/items', status_code=201)
def create(item: Item):
    return item
c=TestClient(app)
r=c.post('/items', json={'name':'pen','price':1.5})
print(r.status_code, r.json())
r=c.post('/items', json={'name':'pen','price':0})
print(r.status_code, r.json()['detail'][0]['loc'])
```

**Answer:** 201 {'name': 'pen', 'price': 1.5}
422 ['body', 'price']

Invalid bodies are rejected before your function runs. Verified by running it with FastAPI's TestClient.

**Q14.** What status code means 'created'?

**Answer:** 201

Set it on POST creating resources.

**Q15.** What does FastAPI do with a Pydantic model parameter?

**Answer:** Reads the JSON body and validates it against the model

Body parameter.

**Q16.** What does `Field(gt=0)` do?

**Answer:** Adds a validation constraint that the number must be greater than 0

Appears in OpenAPI.

### [05 · Response Models &amp; Status Codes](level-1/05-response-models.md)

**Q17.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
class UserIn(BaseModel):
    name: str
    password: str
class UserOut(BaseModel):
    name: str
@app.post('/users', response_model=UserOut)
def make(u: UserIn):
    return u
c=TestClient(app)
print(c.post('/users', json={'name':'ada','password':'s3cret'}).json())
```

**Answer:** {'name': 'ada'}

response_model filters out fields not in the output model. Verified by running it with FastAPI's TestClient.

**Q18.** Why use a separate output model?

**Answer:** To avoid leaking internal fields like password hashes

Control the contract.

**Q19.** What does `response_model_exclude_unset=True` do?

**Answer:** Omits fields that weren't explicitly set

Useful for PATCH-style output.

**Q20.** What happens if the returned data doesn't fit the response model?

**Answer:** FastAPI raises a server error (500) because the response is invalid

Output is validated too.

### [06 · Pydantic v2 in Depth: Fields, Validators &amp; Serialization](level-1/06-pydantic-v2.md)

**Q21.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
class P(BaseModel):
    id: int
    tags: list[str] = []
p=P(id='7', tags=('a','b'))
print(p.id, p.tags, p.model_dump())
try:
    P(id='x')
except Exception as e:
    print(type(e).__name__)
```

**Answer:** 7 ['a', 'b'] {'id': 7, 'tags': ['a', 'b']}
ValidationError

Pydantic coerces compatible values (lax mode) and raises ValidationError otherwise. Verified by running it with FastAPI's TestClient.

**Q22.** What did Pydantic v2 rename model_dump from?

**Answer:** dict() (and json() became model_dump_json())

Migration.

**Q23.** What does a field_validator do?

**Answer:** Runs custom validation/normalization on a field

@field_validator('name').

**Q24.** What does `model_config = ConfigDict(from_attributes=True)` allow?

**Answer:** Building models from ORM objects/attributes

Formerly orm_mode.

### [07 · Errors: HTTPException, Validation Errors &amp; Handlers](level-1/07-errors.md)

**Q25.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
@app.get('/items/{i}')
def get(i: int):
    if i != 1:
        raise HTTPException(status_code=404, detail='not found')
    return {'i': i}
c=TestClient(app)
r=c.get('/items/2')
print(r.status_code, r.json())
```

**Answer:** 404 {'detail': 'not found'}

HTTPException produces a JSON error. Verified by running it with FastAPI's TestClient.

**Q26.** What status does request validation failure return by default?

**Answer:** 422 Unprocessable Content/Entity

Pydantic error list in detail.

**Q27.** How do you map a domain exception to HTTP?

**Answer:** @app.exception_handler(MyError) returning a JSONResponse

Keep endpoints clean.

**Q28.** Why keep a consistent error shape?

**Answer:** Clients branch on it and show messages

Part of the API contract.

### [08 · Forms, File Uploads, Headers &amp; Cookies](level-1/08-forms-files-headers.md)

**Q29.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
@app.post('/login')
def login(username: str = Form(), password: str = Form()):
    return {'user': username}
@app.get('/h')
def h(x_token: str = Header()):
    return {'token': x_token}
c=TestClient(app)
print(c.post('/login', data={'username':'ada','password':'x'}).json())
print(c.get('/h', headers={'X-Token':'abc'}).json())
```

**Answer:** {'user': 'ada'}
{'token': 'abc'}

Header names convert underscores to hyphens. Verified by running it with FastAPI's TestClient.

**Q30.** Which content type do HTML forms use?

**Answer:** application/x-www-form-urlencoded (or multipart/form-data for files)

Needs python-multipart.

**Q31.** How do you accept an uploaded file?

**Answer:** UploadFile with File()

Streams to disk for large files.

**Q32.** Why prefer UploadFile over bytes for big files?

**Answer:** It spools to disk instead of holding everything in memory

Memory safety.

### [09 · APIRouter &amp; Project Structure](level-1/09-routers-structure.md)

**Q33.** What's APIRouter for?

**Answer:** Grouping related endpoints into modules included in the app

app.include_router().

**Q34.** What does `prefix='/books'` do?

**Answer:** Prepends the prefix to every route in the router

Combine with tags.

**Q35.** How do you organize a growing app?

**Answer:** Separate routers, schemas, services and dependencies modules

Maintainable.

**Q36.** What are tags used for?

**Answer:** Grouping endpoints in the OpenAPI docs

Documentation.

### [10 · Project — An In-Memory Notes API](level-1/10-project-notes-api.md)

**Q37.** What does the notes API combine?

**Answer:** CRUD, validation, status codes, errors, search and pagination

Level 1 project.

**Q38.** Which status for a successful delete with no body?

**Answer:** 204

No content.

**Q39.** How do you test the API?

**Answer:** TestClient requests against the app

In-process.

**Q40.** How should a missing note be handled?

**Answer:** Raise a domain exception mapped to a 404 handler

Consistent errors.

## Level 2 · Intermediate

### [01 · Dependency Injection with Depends](level-2/01-dependency-injection.md)

**Q41.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
def get_db():
    return {'n': 1}
@app.get('/x')
def x(db: dict = Depends(get_db)):
    return db
c=TestClient(app)
print(c.get('/x').json())
app.dependency_overrides[get_db]=lambda: {'n': 99}
print(c.get('/x').json())
```

**Answer:** {'n': 1}
{'n': 99}

Overrides swap dependencies in tests. Verified by running it with FastAPI's TestClient.

**Q42.** What does Depends() do?

**Answer:** Declares a dependency FastAPI resolves and injects per request

Reusable logic.

**Q43.** What can dependencies be?

**Answer:** Functions or classes (callables) with their own parameters and dependencies

A tree.

**Q44.** Why use dependency overrides in tests?

**Answer:** Replace databases/auth with test doubles without changing app code

app.dependency_overrides.

### [02 · Yield Dependencies, Caching &amp; Overrides](level-2/02-yield-dependencies.md)

**Q45.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
events=[]
def res():
    events.append('open')
    try:
        yield 'r'
    finally:
        events.append('close')
@app.get('/y')
def y(r: str = Depends(res)):
    events.append('handler')
    return r
c=TestClient(app)
c.get('/y')
print(events)
```

**Answer:** ['open', 'handler', 'close']

Code after yield runs after the response is handled. Verified by running it with FastAPI's TestClient.

**Q46.** What are yield dependencies for?

**Answer:** Setup and cleanup around a request, like DB sessions

try/finally.

**Q47.** Where should session commit/rollback live?

**Answer:** In the dependency or service layer consistently

Single place.

**Q48.** What happens to cleanup code if the handler raises?

**Answer:** The finally block still runs

Resource safety.

### [03 · `async def` vs `def`: The Event Loop and the Threadpool](level-2/03-async-vs-sync.md)

**Q49.** Where does a plain `def` endpoint run?

**Answer:** In a thread pool, so blocking code doesn't freeze the event loop

FastAPI offloads it.

**Q50.** What's the danger of blocking code in `async def`?

**Answer:** It blocks the event loop and stalls all requests

Use await or def.

**Q51.** When use async def?

**Answer:** When awaiting async libraries (httpx, async DB drivers)

Match the stack.

**Q52.** How do you run blocking code from async def?

**Answer:** await run_in_threadpool(...) / anyio.to_thread

Offload.

### [04 · SQLAlchemy 2.x with FastAPI](level-2/04-sqlalchemy.md)

**Q53.** What does SQLAlchemy 2.x provide?

**Answer:** An ORM and SQL toolkit with typed models and sessions

Mapped columns.

**Q54.** Why one session per request?

**Answer:** Isolates work and ensures cleanup via a dependency

Avoid sharing.

**Q55.** What's a model vs a schema in this stack?

**Answer:** Model = database table class; schema = Pydantic API shape

Separate concerns.

**Q56.** What does `db.commit()` do?

**Answer:** Persists the transaction's changes

Handle errors with rollback.

### [05 · Async SQLAlchemy, Relationships &amp; N+1 Queries](level-2/05-async-sqlalchemy.md)

**Q57.** What's async SQLAlchemy used with?

**Answer:** An async driver and AsyncSession from create_async_engine

asyncpg/aiosqlite.

**Q58.** What must you avoid with async sessions?

**Answer:** Lazy-loading relationships implicitly; eager load with selectinload/joinedload

MissingGreenlet error.

**Q59.** Why await session methods?

**Answer:** They perform I/O

Async API.

**Q60.** When is sync SQLAlchemy fine?

**Answer:** When def endpoints run in the threadpool; it's simple and often fast enough

Choose deliberately.

### [06 · Database Migrations with Alembic](level-2/06-alembic.md)

**Q61.** Why use Alembic?

**Answer:** Versioned schema migrations; create_all ignores changes to existing tables

alembic revision --autogenerate.

**Q62.** What does `alembic upgrade head` do?

**Answer:** Applies all pending migrations

Run in deployment.

**Q63.** Why review autogenerated migrations?

**Answer:** They can miss or mis-detect changes (renames become drop/add)

Verify before applying.

**Q64.** Where do migrations belong?

**Answer:** In version control with the code

Reproducible.

### [07 · Configuration with pydantic-settings](level-2/07-settings.md)

**Q65.** What does this print?

```python
from functools import lru_cache
from pydantic import BaseModel
import os
class Settings(BaseModel):
    db_url: str = 'sqlite://'
    debug: bool = False
os.environ['DEBUG']='true'
s=Settings(db_url='postgres://x', debug=os.environ['DEBUG'])
print(s.db_url, s.debug)
```

**Answer:** postgres://x True

Pydantic coerces 'true' to a bool (pydantic-settings reads env vars automatically). Verified by running it with FastAPI's TestClient.

**Q66.** Where should secrets come from?

**Answer:** Environment variables or a secret manager, not source control

12-factor.

**Q67.** What's pydantic-settings for?

**Answer:** Typed settings loaded from env vars and .env files

BaseSettings.

**Q68.** Why cache settings with lru_cache?

**Answer:** Avoid re-reading env files on every request

Dependency returning a singleton.

### [08 · Testing with TestClient &amp; pytest](level-2/08-testing.md)

**Q69.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
@app.get('/me')
def me(user=Depends(lambda: 'real')):
    return {'user': user}
c=TestClient(app)
from fastapi.testclient import TestClient as T2
def fake(): return 'test'
print(c.get('/me').json())
import inspect
print(callable(c.get))
```

**Answer:** {'user': 'real'}
True

Dependencies resolve normally unless overridden. Verified by running it with FastAPI's TestClient.

**Q70.** What's TestClient?

**Answer:** An in-process client calling the ASGI app without a server

Fast tests.

**Q71.** How do you isolate DB tests?

**Answer:** Use a test database/transaction and override the session dependency

Rollback after each test.

**Q72.** What should tests assert besides status codes?

**Answer:** Response bodies, side effects and error shapes

Behavior.

### [09 · Pagination, Filtering &amp; Sorting](level-2/09-pagination-filtering.md)

**Q73.** Why paginate list endpoints?

**Answer:** Bound response sizes and database load

limit/offset or cursors.

**Q74.** Why prefer cursor (keyset) pagination at scale?

**Answer:** Stable and fast: it avoids large OFFSET scans and skipped/duplicated rows

Order by an indexed key.

**Q75.** How do you validate pagination params?

**Answer:** Query(ge=1, le=100) for limit and ge=0 for offset

Protect the DB.

**Q76.** What must sorting be restricted to?

**Answer:** An allow-list of columns to avoid injection and slow queries

Validate sort fields.

### [10 · Project — A Bookshelf API with a Database and Tests](level-2/10-project-bookshelf.md)

**Q77.** What does the bookshelf project combine?

**Answer:** SQLAlchemy models, Alembic, dependencies, pagination, settings and tests

Level 2 project.

**Q78.** How do you model authors and books?

**Answer:** One-to-many relationship with foreign keys and Pydantic schemas

Relational.

**Q79.** How do you rate a book?

**Answer:** Validated rating field (e.g. 1-5) via Pydantic constraints

Validation.

**Q80.** What proves it works?

**Answer:** Tests with overridden DB dependency and migrations applied

Automate.

## Level 3 · Advanced

### [01 · Password Hashing &amp; the OAuth2 Password Flow](level-3/01-password-auth.md)

**Q81.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
import hashlib, hmac, os
salt=b'fixedsalt'
def h(p): return hashlib.pbkdf2_hmac('sha256', p.encode(), salt, 1000).hex()
stored=h('s3cret')
print(hmac.compare_digest(stored, h('s3cret')), hmac.compare_digest(stored, h('wrong')), len(stored))
```

**Answer:** True False 64

Salted slow hash; compare in constant time (use bcrypt/argon2 in production). Verified by running it with FastAPI's TestClient.

**Q82.** Why hash passwords?

**Answer:** Never store plaintext; hashes limit damage from database leaks

Use slow salted algorithms.

**Q83.** What does OAuth2PasswordBearer do?

**Answer:** Extracts the bearer token and documents it in OpenAPI

Dependency.

**Q84.** What should login errors reveal?

**Answer:** A generic message, not whether the username exists

Avoid enumeration.

### [02 · JWT Access Tokens &amp; Refresh Tokens](level-3/02-jwt.md)

**Q85.** What is a JWT?

**Answer:** A signed token carrying claims such as sub and exp

Stateless.

**Q86.** What must you verify on each JWT?

**Answer:** Signature, expiry (exp) and issuer/audience if used

Reject tampered tokens.

**Q87.** Why short access-token lifetimes?

**Answer:** Limit damage from theft; use refresh tokens

Rotation.

**Q88.** Where should the signing secret live?

**Answer:** In settings/env, never in the repo

Rotate if leaked.

### [03 · Authorization: Scopes, Roles &amp; Ownership](level-3/03-authorization.md)

**Q89.** What's authorization vs authentication?

**Answer:** AuthZ decides what an authenticated user may do

Roles/scopes.

**Q90.** How implement role checks?

**Answer:** A dependency that checks the current user's role and raises 403

Reusable.

**Q91.** What status for a logged-in user lacking permission?

**Answer:** 403 Forbidden

401 means not authenticated.

**Q92.** Why enforce object ownership?

**Answer:** Roles don't stop users reading others' data (IDOR)

Check owner.

### [04 · Middleware &amp; CORS](level-3/04-middleware-cors.md)

**Q93.** What does CORS control?

**Answer:** Which browser origins may call your API from JavaScript

Browser-enforced.

**Q94.** What's wrong with allow_origins=['*'] with credentials?

**Answer:** Unsafe and not allowed together; list specific origins

Be explicit.

**Q95.** What is middleware?

**Answer:** Code wrapping every request/response (timing, headers, logging)

@app.middleware('http').

**Q96.** What order do middlewares run?

**Answer:** Last added wraps outermost; consider ordering carefully

Stack.

### [05 · Lifespan Events &amp; Background Tasks](level-3/05-lifespan-background.md)

**Q97.** What is lifespan for?

**Answer:** Startup/shutdown logic like creating and closing connection pools

Replaces on_event.

**Q98.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
import asyncio
log=[]
@app.post('/n')
def n(bg: __import__('fastapi').BackgroundTasks):
    bg.add_task(lambda: log.append('sent'))
    log.append('responded')
    return {'ok': True}
c=TestClient(app)
print(c.post('/n').json(), log)
```

**Answer:** {'ok': True} ['responded', 'sent']

BackgroundTasks run after the response is sent. Verified by running it with FastAPI's TestClient.

**Q99.** When are BackgroundTasks insufficient?

**Answer:** Long, heavy or must-survive-restart jobs: use a queue like Celery/ARQ

Same process.

**Q100.** What belongs in lifespan?

**Answer:** Resources shared across requests (HTTP clients, DB engine)

Created once.

### [06 · WebSockets &amp; Streaming Responses](level-3/06-websockets-streaming.md)

**Q101.** What do WebSockets provide?

**Answer:** Persistent bidirectional communication between client and server

Chat, live updates.

**Q102.** How do you stream a large response?

**Answer:** StreamingResponse with a generator

Memory friendly.

**Q103.** What should WebSocket handlers do on disconnect?

**Answer:** Catch WebSocketDisconnect and clean up

Resource hygiene.

**Q104.** What's Server-Sent Events good for?

**Answer:** One-way server-to-client event streams over HTTP

Simpler than WebSockets.

### [07 · Shaping the OpenAPI Schema](level-3/07-openapi.md)

**Q105.** What generates OpenAPI?

**Answer:** FastAPI from your path operations and models

/openapi.json.

**Q106.** How do you improve docs?

**Answer:** summary, description, tags, examples and response models

Document errors too.

**Q107.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Path, Header, Form
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
@app.get('/a', tags=['x'], summary='Do A')
def a(): return 1
spec=app.openapi()
print(spec['paths']['/a']['get']['summary'], spec['paths']['/a']['get']['tags'])
```

**Answer:** Do A ['x']

The spec is generated from decorators. Verified by running it with FastAPI's TestClient.

**Q108.** Why is OpenAPI valuable?

**Answer:** Client SDK generation, contract testing and docs

Contract.

### [08 · Async Tests &amp; Isolated Test Databases](level-3/08-async-testing.md)

**Q109.** How do you test async endpoints?

**Answer:** httpx.AsyncClient with ASGITransport and pytest-anyio/asyncio

Same app, async client.

**Q110.** Why use async tests?

**Answer:** To await async dependencies/DB and test concurrency behavior

Match the stack.

**Q111.** What's a fixture for the client?

**Answer:** An async fixture yielding the AsyncClient

Setup/teardown.

**Q112.** How do you avoid flaky async tests?

**Answer:** No sleeps; await events/conditions and isolate state

Deterministic.

### [09 · Caching, ETags &amp; Rate Limiting](level-3/09-caching-rate-limiting.md)

**Q113.** What can you cache in an API?

**Answer:** Expensive read results with sensible TTLs and keys (Redis)

Invalidate correctly.

**Q114.** What does rate limiting protect?

**Answer:** Resources and fairness from abuse or bursts

429 Too Many Requests.

**Q115.** Where should rate limits be enforced?

**Answer:** At the edge/gateway and/or per-user in the app with a shared store

Multiple workers need shared counters.

**Q116.** Which status for rate limited?

**Answer:** 429

Include Retry-After.

### [10 · Project — A Multi-User Authenticated API](level-3/10-project-auth-api.md)

**Q117.** What does the auth API project combine?

**Answer:** Password hashing, JWTs, roles, CORS, rate limiting and tests

Level 3 project.

**Q118.** How do you refresh tokens safely?

**Answer:** Rotate refresh tokens, store them hashed and revoke on reuse

Detect theft.

**Q119.** What do you log?

**Answer:** Auth events without passwords or tokens

Security logging.

**Q120.** How do you test auth?

**Answer:** Success, wrong password, expired token and forbidden role

Negative tests.

## Level 4 · Master

### [01 · Performance: Finding Blocking Code and Measuring Throughput](level-4/01-performance.md)

**Q121.** What gives the biggest FastAPI performance win?

**Answer:** Avoiding blocking calls and N+1 queries, and right-sizing workers

Measure first.

**Q122.** What serializer speedup exists?

**Answer:** orjson/ORJSONResponse or Pydantic v2's Rust core

Already fast.

**Q123.** What should you profile?

**Answer:** Endpoint latency, DB queries and CPU hotspots

Evidence.

**Q124.** What does connection pooling do?

**Answer:** Reuses database connections to cut latency and load

Tune pool size.

### [02 · Running in Production: Workers, Gunicorn &amp; Containers](level-4/02-deployment.md)

**Q125.** How is FastAPI served in production?

**Answer:** An ASGI server like Uvicorn/Gunicorn with Uvicorn workers behind a proxy

Multiple workers.

**Q126.** Why containerize?

**Answer:** Reproducible environments and easy scaling

Docker.

**Q127.** What does a health endpoint enable?

**Answer:** Orchestrators to check liveness/readiness

/health.

**Q128.** What belongs in the image?

**Answer:** App, locked dependencies and a non-root user

Least privilege.

### [03 · Behind a Reverse Proxy: root_path, Forwarded Headers &amp; HTTPS](level-4/03-reverse-proxy.md)

**Q129.** What does a reverse proxy (Nginx/Traefik) provide?

**Answer:** TLS termination, buffering, routing and static files

In front of Uvicorn.

**Q130.** What does `--proxy-headers`/root_path handle?

**Answer:** Correct client IPs and URL prefixes behind a proxy

X-Forwarded-*.

**Q131.** Why set timeouts at the proxy?

**Answer:** Prevent slow clients tying up workers

Protection.

**Q132.** Where terminate TLS?

**Answer:** At the proxy/load balancer in most setups

Certificates.

### [04 · Observability: Structured Logs, Request IDs &amp; Metrics](level-4/04-observability.md)

**Q133.** What are the pillars of observability?

**Answer:** Logs, metrics and traces

Together.

**Q134.** What does structured logging provide?

**Answer:** Searchable fields like request id and user id

JSON logs.

**Q135.** What does a request id enable?

**Answer:** Correlating logs across services for one request

Middleware.

**Q136.** What's OpenTelemetry for?

**Answer:** Standard traces/metrics instrumentation across services

Vendor-neutral.

### [05 · Security Hardening Against the OWASP API Top 10](level-4/05-security-hardening.md)

**Q137.** Which header protects against clickjacking?

**Answer:** X-Frame-Options / CSP frame-ancestors

Security headers.

**Q138.** What validates all inputs?

**Answer:** Pydantic models with constraints plus size limits

Never trust clients.

**Q139.** How should errors be exposed?

**Answer:** Without stack traces or internals in production

Log internally.

**Q140.** How handle dependency vulnerabilities?

**Answer:** Pin, scan and update dependencies regularly

pip-audit.

### [06 · Versioning and Evolving an API](level-4/06-versioning.md)

**Q141.** How do you version an API?

**Answer:** URL prefix (/v1) or header, keeping old versions stable for clients

Compatibility.

**Q142.** What's a breaking change?

**Answer:** Removing/renaming fields or changing semantics

Needs a new version.

**Q143.** How do you deprecate?

**Answer:** Announce, mark deprecated in OpenAPI and set a sunset date

Communicate.

**Q144.** Why additive changes first?

**Answer:** They don't break existing clients

Expand then contract.

### [07 · Long-Running Work: Job Queues and Polling](level-4/07-job-queues.md)

**Q145.** When use a job queue (Celery/ARQ/RQ)?

**Answer:** For long-running, retryable or scheduled work outside the request

Reliable processing.

**Q146.** What must tasks be?

**Answer:** Idempotent, since delivery is at-least-once

Retries.

**Q147.** What's a broker?

**Answer:** Message transport such as Redis/RabbitMQ

Between app and workers.

**Q148.** How do you report task status?

**Answer:** Store state/results and expose a status endpoint

Polling or webhooks.

### [08 · Advanced Patterns: Sub-Applications, Custom Routes &amp; Error Contracts](level-4/08-advanced-patterns.md)

**Q149.** What does Annotated enable for dependencies?

**Answer:** Reusable typed dependency aliases (e.g. CurrentUser)

Cleaner signatures.

**Q150.** What does the repository/service pattern separate?

**Answer:** Data access and business logic from HTTP handlers

Testable.

**Q151.** What is a custom APIRoute for?

**Answer:** Wrapping handlers with cross-cutting behavior

Advanced.

**Q152.** Why keep endpoints thin?

**Answer:** Logic lives in services and is easier to test and reuse

Layered design.

### [09 · Upgrading: Pydantic v1 to v2 and FastAPI Releases](level-4/09-upgrading.md)

**Q153.** What's a safe upgrade path for FastAPI/Pydantic?

**Answer:** Read release notes, pin versions, run tests and fix deprecations incrementally

CI.

**Q154.** What changed in Pydantic v2?

**Answer:** Rust core, new API names (model_dump, field_validator) and stricter behavior

Migration tool bump-pydantic.

**Q155.** Why pin dependencies?

**Answer:** Reproducible builds and controlled upgrades

Lock files.

**Q156.** How do you detect breakage early?

**Answer:** Tests and CI against new versions in a branch

Canary upgrades.

### [10 · Capstone — A Production-Ready Event Booking API](level-4/10-capstone.md)

**Q157.** What does the capstone integrate?

**Answer:** Auth, database, migrations, tests, observability, deployment and CI

Production API.

**Q158.** How do you prove production readiness?

**Answer:** Tests, load checks, monitoring, runbooks and rollback

Operate it.

**Q159.** What documentation helps?

**Answer:** README, architecture notes, API docs and runbooks

Onboarding.

**Q160.** What guards regressions?

**Answer:** CI running tests, lint and type checks on every change

Automation.

## Mixed questions across all levels

**Q161.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Header, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
@app.get('/items/{i}')
def get(i: int, q: str | None = None):
    return {'i': i, 'q': q}
c=TestClient(app)
print(c.get('/items/3?q=x').json(), c.get('/items/abc').status_code)
```

**Answer:** {'i': 3, 'q': 'x'} 422

Path params are converted; invalid ones give 422. Verified by running it with FastAPI's TestClient.

**Q162.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Header, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
class U(BaseModel):
    name: str
    pw: str
class Out(BaseModel):
    name: str
@app.post('/u', response_model=Out)
def mk(u: U): return u
c=TestClient(app)
print(c.post('/u', json={'name':'a','pw':'x'}).json())
```

**Answer:** {'name': 'a'}

response_model filters the output. Verified by running it with FastAPI's TestClient.

**Q163.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Header, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
@app.get('/n')
def n():
    raise HTTPException(status_code=404, detail='nope')
c=TestClient(app)
r=c.get('/n')
print(r.status_code, r.json())
```

**Answer:** 404 {'detail': 'nope'}

HTTPException → JSON error. Verified by running it with FastAPI's TestClient.

**Q164.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Header, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
ev=[]
def dep():
    ev.append('in')
    try:
        yield 1
    finally:
        ev.append('out')
@app.get('/d')
def d(x=Depends(dep)):
    ev.append('handler'); return x
c=TestClient(app); c.get('/d'); print(ev)
```

**Answer:** ['in', 'handler', 'out']

Cleanup runs after the handler. Verified by running it with FastAPI's TestClient.

**Q165.** What does this print?

```python
from fastapi import FastAPI, HTTPException, Depends, Query, Header, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field
app = FastAPI()
def real(): return 'real'
@app.get('/w')
def w(x=Depends(real)): return x
c=TestClient(app)
app.dependency_overrides[real]=lambda: 'fake'
print(c.get('/w').json())
```

**Answer:** fake

Overrides are how tests replace dependencies. Verified by running it with FastAPI's TestClient.

**Q166.** A client sends price=-5 to an endpoint whose model has Field(gt=0). What happens?

**Answer:** FastAPI returns 422 before your function runs

Validation errors list location and type.

**Q167.** A def endpoint calls a blocking library and an async def one calls time.sleep(2). Which is the problem?

**Answer:** The async def one blocks the event loop; def endpoints run in a thread pool

Use await or run_in_threadpool.

**Q168.** The model gains a column but the table isn't updated and queries fail. What did you skip?

**Answer:** A migration (Alembic); create_all won't alter existing tables

Version your schema.

**Q169.** Where should the database URL and JWT secret come from?

**Answer:** Environment variables/settings, never committed

pydantic-settings.

**Q170.** How do you give each request its own DB session and guarantee cleanup?

**Answer:** A yield dependency with try/finally

Dependency lifecycle.

**Q171.** A list endpoint returns 2 million rows. What do you add?

**Answer:** Pagination with validated limit/offset (or cursors) and an index on the sort key

Bound response size.

**Q172.** How should passwords be stored for a login endpoint?

**Answer:** As salted, slow hashes (bcrypt/argon2), compared in constant time

Never reversible.

**Q173.** A JWT's signature is valid but exp is in the past. What do you do?

**Answer:** Reject it (401); always verify expiry as well as the signature

Short lifetimes.

**Q174.** A logged-in user requests an admin endpoint. Which status do you return?

**Answer:** 403 Forbidden

401 means unauthenticated.

**Q175.** A browser app on another origin gets blocked calling your API. What do you configure?

**Answer:** CORSMiddleware with explicit allowed origins (not '*' with credentials)

CORS is browser-enforced.

**Q176.** An email must be sent after the response, but it takes 20 s and must survive a crash. What do you use?

**Answer:** A job queue (Celery/ARQ) rather than BackgroundTasks

Durable, retryable.

**Q177.** Behind Nginx, client IPs and redirects look wrong. What do you set?

**Answer:** Proxy headers handling (--proxy-headers / forwarded-allow-ips) and root_path if mounted

X-Forwarded-* headers.

**Q178.** A release renames a response field. How do you avoid breaking clients?

**Answer:** Version the API (/v1 → /v2), keep v1 working and deprecate with notice

Additive changes first.

**Q179.** A Pydantic v1 codebase must move to v2. What's the safest approach?

**Answer:** Pin versions, read the migration guide, update APIs (model_dump, field_validator) and rely on tests

CI protects upgrades.

## Practical coding questions

### A validated notes API

Complete create, read and list_notes: create stores a note and returns only id and title (the response model hides the body), read raises the NotFound domain exception handled as a JSON 404, and list_notes paginates with validated limit/offset. The script exercises the API with TestClient.

Starter code:

```
from fastapi import FastAPI, Depends, Query, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field

app = FastAPI()

class NoteIn(BaseModel):
    title: str = Field(min_length=1, max_length=50)
    body: str = ""

class NoteOut(BaseModel):
    id: int
    title: str

class NotFound(Exception):
    def __init__(self, note_id):
        self.note_id = note_id

@app.exception_handler(NotFound)
def not_found(request: Request, exc: NotFound):
    return JSONResponse(status_code=404, content={"error": "note_not_found", "id": exc.note_id})

class Store:
    def __init__(self):
        self.notes = {}
        self.next_id = 1

def get_store():
    return STORE

STORE = Store()

@app.post("/notes", response_model=NoteOut, status_code=201)
def create(note: NoteIn, store: Store = Depends(get_store)):
    # assign the next id, store {id, title, body}; the response model must hide the body
    return {"id": 0, "title": ""}

@app.get("/notes/{note_id}", response_model=NoteOut)
def read(note_id: int, store: Store = Depends(get_store)):
    # raise NotFound(note_id) when the note doesn't exist
    return {"id": 0, "title": ""}

@app.get("/notes", response_model=list[NoteOut])
def list_notes(limit: int = Query(2, ge=1, le=10), offset: int = Query(0, ge=0), store: Store = Depends(get_store)):
    # notes ordered by id, sliced with offset and limit
    return []

c = TestClient(app)
r = c.post("/notes", json={"title": "a", "body": "secret body"})
print(r.status_code, r.json())
c.post("/notes", json={"title": "b"})
c.post("/notes", json={"title": "c"})
print([n["title"] for n in c.get("/notes").json()], [n["title"] for n in c.get("/notes?limit=2&offset=2").json()])
r = c.get("/notes/99")
print(r.status_code, r.json())
r = c.post("/notes", json={"title": ""})
print(r.status_code, r.json()["detail"][0]["loc"])
print(c.get("/notes?limit=0").status_code)
```

Expected output:

```text
201 {'id': 1, 'title': 'a'}
['a', 'b'] ['c']
404 {'error': 'note_not_found', 'id': 99}
422 ['body', 'title']
422
```

??? success "Show a solution"

    ```
    from fastapi import FastAPI, Depends, Query, Request
    from fastapi.responses import JSONResponse
    from fastapi.testclient import TestClient
    from pydantic import BaseModel, Field

    app = FastAPI()

    class NoteIn(BaseModel):
        title: str = Field(min_length=1, max_length=50)
        body: str = ""

    class NoteOut(BaseModel):
        id: int
        title: str

    class NotFound(Exception):
        def __init__(self, note_id):
            self.note_id = note_id

    @app.exception_handler(NotFound)
    def not_found(request: Request, exc: NotFound):
        return JSONResponse(status_code=404, content={"error": "note_not_found", "id": exc.note_id})

    class Store:
        def __init__(self):
            self.notes = {}
            self.next_id = 1

    def get_store():
        return STORE

    STORE = Store()

    @app.post("/notes", response_model=NoteOut, status_code=201)
    def create(note: NoteIn, store: Store = Depends(get_store)):
        nid = store.next_id
        store.next_id += 1
        store.notes[nid] = {"id": nid, "title": note.title, "body": note.body}
        return store.notes[nid]

    @app.get("/notes/{note_id}", response_model=NoteOut)
    def read(note_id: int, store: Store = Depends(get_store)):
        if note_id not in store.notes:
            raise NotFound(note_id)
        return store.notes[note_id]

    @app.get("/notes", response_model=list[NoteOut])
    def list_notes(limit: int = Query(2, ge=1, le=10), offset: int = Query(0, ge=0), store: Store = Depends(get_store)):
        items = [store.notes[k] for k in sorted(store.notes)]
        return items[offset: offset + limit]

    c = TestClient(app)
    r = c.post("/notes", json={"title": "a", "body": "secret body"})
    print(r.status_code, r.json())
    c.post("/notes", json={"title": "b"})
    c.post("/notes", json={"title": "c"})
    print([n["title"] for n in c.get("/notes").json()], [n["title"] for n in c.get("/notes?limit=2&offset=2").json()])
    r = c.get("/notes/99")
    print(r.status_code, r.json())
    r = c.post("/notes", json={"title": ""})
    print(r.status_code, r.json()["detail"][0]["loc"])
    print(c.get("/notes?limit=0").status_code)
    ```

    The response model keeps `body` out of responses, a domain exception handler gives a consistent 404 shape, and Query(ge=, le=) validates pagination before your code runs. Verified by running the solution with FastAPI's TestClient.

### Authentication, roles and ownership

Implement current_user (401 for missing or unknown Bearer tokens), require_role (a dependency factory that raises 403) and the document endpoint's ownership rule (return 404, not 403, for documents the caller may not see).

Starter code:

```
from fastapi import FastAPI, Depends, Header, HTTPException
from fastapi.testclient import TestClient

app = FastAPI()

USERS = {"tok-ada": {"name": "ada", "role": "admin"}, "tok-bo": {"name": "bo", "role": "user"}}
DOCS = {1: {"owner": "ada", "text": "ada's doc"}, 2: {"owner": "bo", "text": "bo's doc"}}

def current_user(authorization: str | None = Header(default=None)):
    # 401 when the header is missing/not Bearer or the token is unknown; otherwise return the user dict
    return {"name": "", "role": ""}

def require_role(role):
    # return a dependency that depends on current_user and raises 403 unless user["role"] == role
    def checker(user=Depends(current_user)):
        return user
    return checker

@app.get("/docs/{doc_id}")
def read_doc(doc_id: int, user=Depends(current_user)):
    # 404 if missing; 404 (not 403) if the doc belongs to someone else and the user isn't an admin
    return {}

@app.get("/admin/stats")
def stats(user=Depends(require_role("admin"))):
    return {"docs": len(DOCS)}

c = TestClient(app)
def show(r): print(r.status_code, r.json()["detail"] if r.status_code >= 400 else r.json())
show(c.get("/docs/1"))
show(c.get("/docs/1", headers={"Authorization": "Bearer nope"}))
show(c.get("/docs/1", headers={"Authorization": "Bearer tok-ada"}))
show(c.get("/docs/1", headers={"Authorization": "Bearer tok-bo"}))
show(c.get("/docs/2", headers={"Authorization": "Bearer tok-ada"}))
show(c.get("/admin/stats", headers={"Authorization": "Bearer tok-bo"}))
show(c.get("/admin/stats", headers={"Authorization": "Bearer tok-ada"}))
```

Expected output:

```text
401 missing token
401 invalid token
200 {'owner': 'ada', 'text': "ada's doc"}
404 no such doc
200 {'owner': 'bo', 'text': "bo's doc"}
403 forbidden
200 {'docs': 2}
```

??? success "Show a solution"

    ```
    from fastapi import FastAPI, Depends, Header, HTTPException
    from fastapi.testclient import TestClient

    app = FastAPI()

    USERS = {"tok-ada": {"name": "ada", "role": "admin"}, "tok-bo": {"name": "bo", "role": "user"}}
    DOCS = {1: {"owner": "ada", "text": "ada's doc"}, 2: {"owner": "bo", "text": "bo's doc"}}

    def current_user(authorization: str | None = Header(default=None)):
        if not authorization or not authorization.startswith("Bearer "):
            raise HTTPException(status_code=401, detail="missing token", headers={"WWW-Authenticate": "Bearer"})
        user = USERS.get(authorization.removeprefix("Bearer "))
        if user is None:
            raise HTTPException(status_code=401, detail="invalid token")
        return user

    def require_role(role):
        def checker(user=Depends(current_user)):
            if user["role"] != role:
                raise HTTPException(status_code=403, detail="forbidden")
            return user
        return checker

    @app.get("/docs/{doc_id}")
    def read_doc(doc_id: int, user=Depends(current_user)):
        doc = DOCS.get(doc_id)
        if doc is None:
            raise HTTPException(status_code=404, detail="no such doc")
        if doc["owner"] != user["name"] and user["role"] != "admin":
            raise HTTPException(status_code=404, detail="no such doc")   # don't leak existence
        return doc

    @app.get("/admin/stats")
    def stats(user=Depends(require_role("admin"))):
        return {"docs": len(DOCS)}

    c = TestClient(app)
    def show(r): print(r.status_code, r.json()["detail"] if r.status_code >= 400 else r.json())
    show(c.get("/docs/1"))
    show(c.get("/docs/1", headers={"Authorization": "Bearer nope"}))
    show(c.get("/docs/1", headers={"Authorization": "Bearer tok-ada"}))
    show(c.get("/docs/1", headers={"Authorization": "Bearer tok-bo"}))
    show(c.get("/docs/2", headers={"Authorization": "Bearer tok-ada"}))
    show(c.get("/admin/stats", headers={"Authorization": "Bearer tok-bo"}))
    show(c.get("/admin/stats", headers={"Authorization": "Bearer tok-ada"}))
    ```

    Authentication (who are you) is separate from authorization (what may you do): roles gate endpoints, ownership gates objects, and returning 404 for others' documents avoids revealing they exist. Verified by running the solution with FastAPI's TestClient.

---

More free practice: the [FastAPI Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
