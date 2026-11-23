---
title: "FastAPI Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free FastAPI cheat sheet: 19 quick-reference tables from every level of the FastAPI Mastery Path, each linked to its full lesson."
---

# FastAPI Cheat Sheet

Quick-reference tables from **19 lessons** of the free FastAPI Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What FastAPI Is: ASGI, Starlette, Pydantic and One Request](level-1/01-what-fastapi-is.md)

*The three layers*

| Layer | What it does | You notice it when… |
|-------|--------------|----------------------|
| **ASGI server** (Uvicorn) | Owns the socket, speaks HTTP/1.1 or WebSocket, turns bytes into an event dictionary | you start the app, tune workers, sit behind a proxy |
| **Starlette** | Routing, request/response objects, middleware, WebSockets, background tasks, test client | you add middleware, stream a response, mount a sub-app |
| **Pydantic** | Data validation and serialization from type hints | you define models, write validators, read a 422 error |
| **FastAPI** | Reads your function signatures, wires Starlette and Pydantic together, dependency injection, OpenAPI | you write `@app.get(...)`, `Depends(...)` |

### [02 · Installing FastAPI &amp; Your First App](level-1/02-first-app.md)

*`fastapi run`: production mode*

| | `fastapi dev` | `fastapi run` |
|---|---|---|
| Auto-reload | on | off |
| Default host | `127.0.0.1` | `0.0.0.0` (all interfaces) |
| Intended for | your laptop | containers and servers |
| Workers | 1 | 1 unless you pass `--workers N` |

### [03 · Path &amp; Query Parameters](level-1/03-path-query-params.md)

*How booleans are parsed*

| Value | Result |
|-------|--------|
| `true`, `1`, `on`, `yes` | `True` |
| `false`, `0`, `off`, `no` | `False` |
| `maybe` | `422 bool_parsing` |

### [04 · Request Bodies with Pydantic Models](level-1/04-request-bodies.md)

*Broken JSON and wrong content types*

| Content-Type | Result |
|---|---|
| *(none)* | `422 model_attributes_type` — body treated as a plain string |
| `text/plain` | `422 model_attributes_type` |
| `application/vnd.api+json` | `201` — any `+json` type is parsed as JSON |
| `application/x-www-form-urlencoded` with `title=Dune` | `422 model_attributes_type` |

### [05 · Response Models &amp; Status Codes](level-1/05-response-models.md)

*Status codes*

| Code | Meaning | Typical use |
|------|---------|-------------|
| 200 OK | success with a body | `GET`, most `PUT`/`PATCH` |
| 201 Created | a new resource exists | `POST` that creates |
| 204 No Content | success, nothing to say | `DELETE` |
| 400 / 422 | the request was wrong | bad input (422 is FastAPI's default) |
| 404 Not Found | no such resource | unknown ID |
| 409 Conflict | the state forbids it | duplicate email |

### [06 · Pydantic v2 in Depth: Fields, Validators &amp; Serialization](level-1/06-pydantic-v2.md)

*Reading the pieces*

| Tool | What it's for |
|------|---------------|
| `Field(...)` | constraints (`gt`, `max_length`, `decimal_places`), defaults, `default_factory`, aliases, docs (`description`, `examples`) |
| `Annotated[T, ...]` | attach validators and constraints to a type, then reuse that type anywhere (`ISBN`, `Tag`) |
| `BeforeValidator` | runs on the raw input, *before* type conversion — good for normalising |
| `AfterValidator` | runs on the already-converted value — good for rules |
| `@field_validator` | the same, written as a method; handy when the rule belongs to one model |
| `@model_validator` | rules that involve several fields |
| `@computed_field` | derived values included in output and schema |
| `@field_serializer` | control how a field is written out |
| `model_config` | model-wide behaviour (`extra`, `strict`, `str_strip_whitespace`, …) |

### [07 · Errors: HTTPException, Validation Errors &amp; Handlers](level-1/07-errors.md)

*Worked example: picking status codes for a "buy" endpoint*

| Situation | Status | Why |
|---|---|---|
| unknown book ID | 404 | the resource doesn't exist |
| quantity is `0` or `"lots"` | 422 | invalid input, let validation do it |
| book exists, none in stock | 409 | request is valid, current *state* forbids it |
| user not logged in | 401 + `WWW-Authenticate` | missing/invalid credentials |
| logged in, but account suspended | 403 | identity known, not allowed |
| payment provider timed out | 502 or 503 | upstream failure, retry may work |
| your code raised `KeyError` | 500 | a bug |

### [08 · Forms, File Uploads, Headers &amp; Cookies](level-1/08-forms-files-headers.md)

*File uploads*

| Attribute | Meaning |
|-----------|---------|
| `filename` | the name **the client claims** |
| `content_type` | the type **the client claims** |
| `size` | bytes received, when known |
| `file` | a `SpooledTemporaryFile` — in memory while small, on disk when large |
| `read()`, `seek()`, `write()`, `close()` | async wrappers around `file` |

### [10 · Project — An In-Memory Notes API](level-1/10-project-notes-api.md)

*What you're building*

| Method | Path | Purpose | Success |
|--------|------|---------|---------|
| `GET` | `/notes` | search + paginate (`q`, `tag`, `pinned`, `limit`, `offset`) | 200 |
| `POST` | `/notes` | create | 201 + `Location` |
| `GET` | `/notes/{note_id}` | fetch one | 200 / 404 |
| `PATCH` | `/notes/{note_id}` | change only the sent fields | 200 / 404 / 422 |
| `DELETE` | `/notes/{note_id}` | remove | 204 / 404 |

## Level 2 · Intermediate

### [03 · `async def` vs `def`: The Event Loop and the Threadpool](level-2/03-async-vs-sync.md)

*The decision rule*

| Your code… | Write | Why |
|---|---|---|
| awaits async libraries only (`httpx.AsyncClient`, `asyncpg`, async SQLAlchemy) | `async def` | no thread hop, high concurrency |
| calls any blocking library (`requests`, sync SQLAlchemy, `boto3`, file I/O) | `def` | keeps the loop free |
| is mostly async but has one blocking call | `async def` + `await run_in_threadpool(fn, ...)` | offload just that call |
| is CPU-heavy (image resizing, big JSON, ML inference) | `def`, or a process pool / job queue | threads still share the GIL for pure-Python work |
| does trivial work (return a constant, read settings) | either | `async def` avoids a thread hop |

### [09 · Pagination, Filtering &amp; Sorting](level-2/09-pagination-filtering.md)

*Which to use*

| | Offset | Cursor |
|---|---|---|
| Jump to page 37 | yes | no |
| Total count | natural (but costs a query) | usually omitted |
| Stable under inserts/deletes | no | yes |
| Fast deep in the list | no | yes, with an index |
| Change sort order mid-stream | easy | the cursor is tied to one sort |

## Level 3 · Advanced

### [02 · JWT Access Tokens &amp; Refresh Tokens](level-3/02-jwt.md)

*What a JWT is*

| Claim | Meaning |
|-------|---------|
| `sub` | subject — who the token is about |
| `exp` | expiry time; PyJWT rejects the token after it |
| `iat` | issued-at time |
| `iss` | issuer — who created it |
| `jti` | a unique token ID, useful for revocation lists |

### [05 · Lifespan Events &amp; Background Tasks](level-3/05-lifespan-background.md)

*What belongs in lifespan*

| Good fit | Why |
|---|---|
| database engine / connection pool | one per process, reused by every request |
| `httpx.AsyncClient` for calling other APIs | connection pooling, keep-alive |
| ML model or large lookup table | expensive to load, read-only afterwards |
| cache or message-broker client | long-lived connection |

### [09 · Caching, ETags &amp; Rate Limiting](level-3/09-caching-rate-limiting.md)

*ETags and conditional GET*

| Header | Meaning | Use for |
|---|---|---|
| `no-store` | don't keep it at all | sensitive data |
| `private, max-age=0, must-revalidate` | keep, but revalidate each time (ETag) | per-user data that changes |
| `public, max-age=300` | anyone may cache it for 5 minutes | public catalogue data |
| `public, max-age=31536000, immutable` | cache "forever" | versioned static files |

### [10 · Project — A Multi-User Authenticated API](level-3/10-project-auth-api.md)

*Requirements*

| Area | Rule |
|---|---|
| Sign-up | lower-case username 3–30 chars (`a-z0-9_`); password 12–128 chars, not on a common-password list; stored as Argon2id |
| Login | OAuth2 password form; identical errors for "no user" and "wrong password"; constant-cost check; 5 attempts burst, 5/minute per username |
| Tokens | 15-minute access JWT with scopes; 7-day refresh JWT, **rotated** on use; reuse of an old refresh token revokes the whole family; logout revokes the family |
| Bookmarks | users see and change only their own; an owner may mark a bookmark public; anyone can list a user's public bookmarks |
| Admin | needs the `admin` scope *and* an admin account; can list and disable users; disabled users lose access immediately |
| Data | async SQLAlchemy on SQLite via aiosqlite; schema via Alembic's async template; UTC timestamps |

## Level 4 · Master

### [01 · Performance: Finding Blocking Code and Measuring Throughput](level-4/01-performance.md)

*Load testing with `ab`*

| Endpoint | Requests/s | Mean time per request (c=10) |
|---|---|---|
| `/ping` (n=2000, c=20) | 11,575 | — |
| `/items-annotated` (returns models) | 2,446 | 4.1 ms |
| `/items-prebuilt` (`json.dumps` yourself) | 1,625 | 6.2 ms |
| `/items-model` (dicts + `response_model`) | 804 | 12.4 ms |
| `/items-raw` (**no** response model) | 156 | 64.2 ms |

### [05 · Security Hardening Against the OWASP API Top 10](level-4/05-security-hardening.md)

*The list, mapped to this course*

| # | Risk | Where it's handled |
|---|---|---|
| API1 | Broken Object Level Authorization | Level 3 lesson 3 — ownership in the query |
| API2 | Broken Authentication | Level 3 lessons 1–2 — Argon2, timing, token checks |
| API3 | Broken Object Property Level Authorization | below — mass assignment; Level 1 lesson 5 for over-exposure |
| API4 | Unrestricted Resource Consumption | below — body limits; Level 2 lesson 9 (page sizes), Level 3 lesson 9 (rate limits) |
| API5 | Broken Function Level Authorization | Level 3 lesson 3 — role/scope dependencies on routers |
| API6 | Unrestricted Access to Sensitive Business Flows | rate limits + business rules (e.g. one coupon per account) |
| API7 | Server-Side Request Forgery | below |
| API8 | Security Misconfiguration | below — debug mode, CORS (Level 3 lesson 4), docs exposure |
| API9 | Improper Inventory Management | Level 4 lesson 6 — versioning and retiring old APIs |
| API10 | Unsafe Consumption of APIs | validate third-party responses with Pydantic; timeouts (Level 2 lesson 3) |

### [06 · Versioning and Evolving an API](level-4/06-versioning.md)

*Additive vs breaking*

| Change | Response (server → client) | Request (client → server) |
|---|---|---|
| add an optional field | safe | safe (if it has a default) |
| add a **required** field | safe | **breaking** — old clients don't send it |
| remove a field | **breaking** — clients may read it | safe-ish (old clients still send it; reject or ignore) |
| rename a field | **breaking** | **breaking** |
| change a type (`float` → `int`, string → object) | **breaking** | **breaking** |
| add an endpoint | safe | — |
| remove an endpoint | **breaking** | — |
| add a new enum value | can break clients with exhaustive `switch`es | safe |
| tighten validation (shorter max length) | — | **breaking** |

### [10 · Capstone — A Production-Ready Event Booking API](level-4/10-capstone.md)

*Requirements*

| Area | Rule |
|---|---|
| Events | organisers (scope `events:write`) create events with a timezone-aware start time and a capacity; anyone can read them |
| Bookings | attendees (scope `bookings`) book 1–10 seats; sold out → 409; event started → 409; cancel returns seats; cancel is idempotent |
| Retries | an `Idempotency-Key` header makes a booking request safe to retry: same key, same booking |
| Auth | tokens come from an identity provider; the API verifies signature, expiry, issuer and audience — it never handles passwords |
| Errors | every error is `application/problem+json` with a `request_id` |
| Ops | JSON access logs; `X-Request-ID` on every response; 16 KB body limit; `/health/live` and `/health/ready`; docs off in prod |
| Config | the app refuses to start in prod with a short secret, SQLite, or wildcard CORS |

---

Go deeper with the full [FastAPI Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
