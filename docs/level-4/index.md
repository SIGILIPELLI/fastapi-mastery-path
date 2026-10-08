---
description: "Level 4 of FastAPI: profiling, workers, reverse proxies, observability, OWASP hardening, versioning, job queues, upgrades and a booking API capstone."
---

# Level 4 · Master <span class="level-badge">Production</span>

Goal: run a FastAPI service the way production demands — measured, observable, hardened,
able to evolve without breaking clients, and correct when many processes serve it at
once.

Three ideas carry this level:

1. **Measure before and after.** Lesson 1's most useful result — that removing a
   response model made an endpoint five times *slower* — is the opposite of what
   intuition said. Profile, load-test, and trust the numbers from your own hardware.
2. **Production is many processes behind a proxy.** Workers, forwarded headers,
   `root_path`, graceful shutdown, per-process state: most "works on my machine" bugs
   live in the gap between `fastapi dev` and this.
3. **Correctness lives in the database.** Atomic `UPDATE ... WHERE`, unique constraints
   for idempotency, leases for job claiming — the capstone sells exactly 50 seats out of
   200 concurrent requests across four processes because of a single SQL statement, not
   because of locks in Python.

## Modules

1. [Performance: Finding Blocking Code and Measuring Throughput](01-performance.md) — `ab`, pyinstrument in the right event loop, asyncio debug mode, workers for CPU work
2. [Running in Production: Workers, Gunicorn & Containers](02-deployment.md) — the supervisor, crash recovery, graceful shutdown, a Dockerfile
3. [Behind a Reverse Proxy](03-reverse-proxy.md) — forwarded headers, who to trust, `X-Forwarded-For` spoofing, `root_path`
4. [Observability: Structured Logs, Request IDs & Metrics](04-observability.md) — JSON logs, the error-path request-ID bug, Prometheus, FastAPI's built-in OpenTelemetry
5. [Security Hardening Against the OWASP API Top 10](05-security-hardening.md) — mass assignment, body limits, SSRF, path traversal, debug leaks
6. [Versioning and Evolving an API](06-versioning.md) — additive vs breaking, v1/v2 routers, deprecation headers, an OpenAPI diff check
7. [Long-Running Work: Job Queues and Polling](07-job-queues.md) — a durable job table, atomic claims, leases, retries, idempotency
8. [Advanced Patterns](08-advanced-patterns.md) — RFC 9457 problem details, custom route classes, mounted sub-apps
9. [Upgrading: Pydantic v1 to v2 and FastAPI Releases](09-upgrading.md) — silent behaviour changes, codemod pitfalls, warnings as errors
10. [Capstone — A Production-Ready Event Booking API](10-capstone.md) — everything above, with a multi-process overbooking test

## What you need before starting

- **Levels 1–3** of this course. The capstone reuses async SQLAlchemy, Alembic, settings,
  JWT verification, scopes and the async test fixtures.
- **Comfort with a terminal**: processes, signals, environment variables. The
  [Shell Mastery Path](https://sigilipelli.github.io/shell-mastery-path/) and
  [Server Ops Mastery Path](https://sigilipelli.github.io/server-ops-mastery-path/) cover
  the background.
- For containers and orchestration, see the
  [Docker Mastery Path](https://sigilipelli.github.io/docker-mastery-path/) and
  [Kubernetes Mastery Path](https://sigilipelli.github.io/kubernetes-mastery-path/); for
  the HTTP design side of versioning, the
  [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/).

## How the examples were checked

Everything ran on one 8-core laptop with Python 3.14.7, **FastAPI 0.143.0**,
**Uvicorn 0.54.0**, **Gunicorn 26.2.0**, **SQLAlchemy 2.1.4**, **Alembic 1.20.0**,
**prometheus-client 0.26.0**, **OpenTelemetry SDK 1.45.1**, **pyinstrument 5.1.3** and
**bump-pydantic 0.8.0**, with SQLite 3.53.4. Load tests used ApacheBench (`ab`) on the
same machine as the server; worker, crash and shutdown behaviour was observed on real
Uvicorn processes with `ps`, `kill` and `curl`.

!!! note "What wasn't run"
    Docker, nginx, PostgreSQL, Redis and a real tracing backend weren't available. The
    Dockerfile (lesson 2) and nginx configuration (lesson 3) are marked as unbuilt and
    untested; proxy behaviour was simulated by sending forwarded headers directly to
    Uvicorn; PostgreSQL-specific statements (`FOR UPDATE SKIP LOCKED`, row-lock
    behaviour) are described from documented behaviour. Throughput numbers are one
    machine's and are for comparing alternatives, not for capacity planning.
