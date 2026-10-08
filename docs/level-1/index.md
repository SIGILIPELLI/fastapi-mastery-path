---
description: "Level 1 of FastAPI: ASGI basics, your first app, path and query parameters, Pydantic bodies and responses, errors, forms, routers and a notes API project."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: build a small but complete JSON API with FastAPI, and be able to say exactly what
happens to a request between the socket and your function — and back.

Three ideas carry you through all ten modules:

1. **The signature is the specification.** Type hints and markers like `Query()`,
   `Body()` and `Form()` decide where data comes from, how it's converted, what's
   rejected, and what the docs say. Change the signature and you've changed the API.
2. **Validate at the edge, trust inside.** Pydantic checks everything before your code
   runs, and response models check everything before it leaves. Your endpoint body can
   assume clean data.
3. **Keep endpoints thin.** Parse, call ordinary Python code, return. Business rules
   live in modules that don't import FastAPI, which makes them easy to test and easy to
   move to a database in Level 2.

## Modules

1. [What FastAPI Is: ASGI, Starlette, Pydantic and One Request](01-what-fastapi-is.md) — the layers, a raw ASGI app, what happens at import time vs request time
2. [Installing FastAPI & Your First App](02-first-app.md) — `fastapi[standard]`, `fastapi dev` vs `fastapi run`, the generated docs
3. [Path & Query Parameters](03-path-query-params.md) — `Annotated`, constraints, enums, lists, query models and the route-order trap
4. [Request Bodies with Pydantic Models](04-request-bodies.md) — nested models, every-error-at-once validation, `PATCH` with `exclude_unset`, content types
5. [Response Models & Status Codes](05-response-models.md) — filtering secrets, return annotations, 201/204, `ResponseValidationError`
6. [Pydantic v2 in Depth](06-pydantic-v2.md) — reusable `Annotated` types, validators, aliases, strict mode, serialization
7. [Errors: HTTPException, Validation Errors & Handlers](07-errors.md) — domain exceptions, custom 404/405/422 shapes, crashes
8. [Forms, File Uploads, Headers & Cookies](08-forms-files-headers.md) — `Form()`, `UploadFile`, size and type checks, cookie flags
9. [APIRouter & Project Structure](09-routers-structure.md) — routers, prefixes, tags, a layout that avoids circular imports
10. [Project — An In-Memory Notes API](10-project-notes-api.md) — CRUD, search, pagination, error handling and a pytest suite

## What you need before starting

- **Python fundamentals:** functions, classes, type hints, modules, virtual
  environments. FastAPI leans on type hints more than any other framework; if
  `list[str] | None` looks unfamiliar, work through
  [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/) first.
- **HTTP basics:** methods, status codes, headers, JSON. The
  [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/) covers
  API design in depth; this course focuses on implementing it in FastAPI.
- Python 3.10 or newer (the `X | None` syntax needs it). The examples were run on 3.14.
- `curl` or any HTTP client. Every endpoint can also be exercised from `/docs`.

## How the examples were checked

Every code sample in this level was run with **FastAPI 0.143.0**, **Pydantic 2.14.0**,
**Starlette 1.7.0** and **Uvicorn 0.54.0** on Python 3.14.7, mostly through FastAPI's
`TestClient` and, where server behaviour matters (startup output, logs, headers), under a
real Uvicorn process with `curl`. Outputs are copied from those runs, sometimes trimmed
or re-wrapped for width, and marked when they are. Timestamps and process IDs will differ
on your machine.

!!! note "About versions"
    FastAPI is still numbered `0.x` and releases frequently, so small details change. The
    concepts in this level — how parameters are sourced, how Pydantic validates, how
    response models filter — have been stable for years. Where this level found
    something version-specific (the test client's `httpx2` deprecation warning, the
    stricter `Content-Type` handling for JSON bodies, how `include_router` stores routes),
    the lesson says so. Run `pip list | grep -i fastapi` to see what you have.
