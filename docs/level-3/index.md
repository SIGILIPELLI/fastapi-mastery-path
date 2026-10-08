---
description: "Level 3 of FastAPI: password hashing, OAuth2 and JWT, scopes and ownership, middleware, CORS, lifespan, WebSockets, OpenAPI, async tests and caching."
---

# Level 3 · Advanced <span class="level-badge">Secure & Real-Time</span>

Goal: build an API that many users can share safely — with real authentication,
fine-grained authorization, sensible cross-cutting behaviour (CORS, caching, rate
limits), real-time features — and prove it with tests that exercise the security rules.

Three ideas carry this level:

1. **Authentication is not authorization.** Knowing who someone is (lessons 1–2) tells
   you nothing about which records they may touch (lesson 3). Most real API breaches are
   the second kind.
2. **HTTP already has the answer.** `WWW-Authenticate`, CORS headers, `ETag`,
   `If-Match`, `Retry-After`, Server-Sent Events — use the protocol's mechanisms before
   inventing your own.
3. **Per-process is a limitation, not a detail.** Lifespan resources, in-memory caches,
   rate limiters and WebSocket rooms all live in one process. Know which of yours do,
   before Level 4 runs several.

## Modules

1. [Password Hashing & the OAuth2 Password Flow](01-password-auth.md) — Argon2id with pwdlib, token endpoint, bearer dependency, the timing leak
2. [JWT Access Tokens & Refresh Tokens](02-jwt.md) — claims, signature checks, `alg=none`, rotation with reuse detection
3. [Authorization: Scopes, Roles & Ownership](03-authorization.md) — `Security` and `SecurityScopes`, role factories, BOLA and how to prevent it
4. [Middleware & CORS](04-middleware-cors.md) — ordering, pure ASGI middleware, GZip, TrustedHost, the wildcard-credentials trap
5. [Lifespan Events & Background Tasks](05-lifespan-background.md) — lifespan state, what background tasks can and can't promise
6. [WebSockets & Streaming Responses](06-websockets-streaming.md) — JSON Lines, Server-Sent Events, an authenticated chat room
7. [Shaping the OpenAPI Schema](07-openapi.md) — metadata, documented errors, operation IDs, a generated TypeScript client
8. [Async Tests & Isolated Test Databases](08-async-testing.md) — `AsyncClient`, AnyIO, rollback isolation, real concurrency tests
9. [Caching, ETags & Rate Limiting](09-caching-rate-limiting.md) — 304s, `If-Match` optimistic locking, a TTL cache, a token bucket
10. [Project — A Multi-User Authenticated API](10-project-auth-api.md) — a bookmarks service with all of the above and 12 tests

## What you need before starting

- **Levels 1 and 2** of this course: dependencies, yield dependencies, async SQLAlchemy,
  settings and testing are used throughout.
- **Security basics** help: what hashing, signing and TLS are for. The
  [Cybersecurity Mastery Path](https://sigilipelli.github.io/cybersecurity-mastery-path/)
  covers the background; this level focuses on implementing it in FastAPI.
- **API design**: for status-code and resource-design conventions beyond what's covered
  here, see the [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/).
- Install: `pip install "fastapi[standard]" "pwdlib[argon2]" pyjwt "sqlalchemy[asyncio]" aiosqlite pytest`.

## How the examples were checked

Every example ran on Python 3.14.7 with **FastAPI 0.143.0**, **Starlette 1.7.0**,
**Uvicorn 0.54.0**, **pwdlib 0.3.1** (argon2-cffi 25.1.0), **PyJWT 2.15.1**,
**SQLAlchemy 2.1.4**, **httpx 0.28.1** and **AnyIO 4.15.1**. Security behaviour — 401
vs 403, rejected `alg=none` tokens, refresh-token reuse detection, CORS responses — was
checked with the test client; WebSockets, streaming and background-task timing were
also checked against a real Uvicorn process with `curl` and the `websockets` client,
because the test client behaves differently in exactly those areas (lessons 5 and 6
show how). The TypeScript client in lesson 7 was generated with openapi-typescript
7.13.0 on Node 26.

!!! note "What wasn't run"
    No Redis or other shared store was available, so distributed caching, distributed
    rate limiting and cross-process WebSocket broadcast are described, not
    demonstrated. Timing figures (Argon2's ~40 ms, the login-timing comparison) are from
    one laptop and will differ on yours; the *differences* they illustrate are the
    point.
