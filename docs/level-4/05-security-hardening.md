---
description: "Harden a FastAPI API against the OWASP API Security Top 10: mass assignment, body-size limits, SSRF, path traversal, debug leaks and secure defaults."
---

# 05 · Security Hardening Against the OWASP API Top 10

The OWASP **API Security Top 10 (2023 edition)** is a list of the most common ways APIs
are actually broken. Several items were covered earlier in this course; this lesson maps
the whole list to FastAPI and demonstrates — with attacks that succeeded against
deliberately unsafe endpoints — the ones not yet covered.

!!! warning "Scope"
    Every attack here was run against a local test app built for the purpose. Only test
    systems you own or are explicitly authorised to test.

## The list, mapped to this course

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

## API3: mass assignment

The examples in this lesson share one test app; its imports were:

```python
import ipaddress, socket
from pathlib import Path
from urllib.parse import urlsplit
from fastapi import FastAPI, HTTPException
from fastapi.responses import FileResponse, JSONResponse
from pydantic import BaseModel, ConfigDict
```

An update model that happens to contain a privileged field:

```python
class UserUpdateLoose(BaseModel):          # extra="ignore" (default) but has is_admin!
    display_name: str | None = None
    is_admin: bool = False

class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")
    display_name: str | None = None
```

Sent `{"display_name": "Ada L.", "is_admin": true}`:

```text
API3 loose : {'id': 1, 'display_name': 'Ada L.', 'is_admin': True}
API3 strict: 422 extra_forbidden ['body', 'is_admin']
```

The loose endpoint made Ada an administrator. It's an easy mistake when one model is
reused for "fields users can edit" and "fields admins can edit". The rules: a separate
input model per permission level, containing **only** fields that caller may set, with
`extra="forbid"` so an attempt is visible as a 422 rather than silently ignored. The
other half of API3 — returning properties the caller shouldn't see — is what response
models prevent (Level 1 lesson 5).

## API4: limit request bodies

Level 1 lesson 8 found that nothing limits the size of a JSON or upload body by default.
The real fix is at the proxy (`client_max_body_size`, lesson 3), but a defence in the app
is cheap:

```python
class BodyLimit:
    def __init__(self, app, max_bytes: int):
        self.app, self.max = app, max_bytes
    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        declared = dict(scope["headers"]).get(b"content-length")
        if declared is not None and int(declared) > self.max:
            return await JSONResponse({"detail": "Request body too large"}, 413)(scope, receive, send)
        seen = 0
        async def limited_receive():
            nonlocal seen
            message = await receive()
            if message["type"] == "http.request":
                seen += len(message.get("body", b""))
                if seen > self.max:
                    raise HTTPException(413, "Request body too large")
            return message
        await self.app(scope, limited_receive, send)

app.add_middleware(BodyLimit, max_bytes=1024)
```

```text
API4 small : 200
API4 big   : 413 {'detail': 'Request body too large'}
API4 chunked (no content-length): 413 {"detail":"Request body too large"}
```

The middleware rejects an over-large `Content-Length` before reading anything, and also
counts bytes as they arrive, which catches chunked requests that don't declare a length.
Pick limits per route group in a real app (uploads need more than JSON endpoints).

The rest of API4 is limits everywhere else: maximum page sizes, maximum list lengths in
models (`Field(max_length=...)`), string lengths, timeouts on outgoing calls, and rate
limits on expensive operations.

## API7: server-side request forgery

Any feature that fetches a URL supplied by a user — link previews, webhooks, "import
from URL" — can be pointed at things only the *server* can reach: `localhost` admin
ports, other internal services, or the cloud metadata endpoint at `169.254.169.254`
(which on many clouds hands out credentials). A guard that resolves the host and refuses
non-public addresses:

```python
def resolve_public(url: str) -> str:
    parts = urlsplit(url)
    if parts.scheme not in ("http", "https") or not parts.hostname:
        raise ValueError("only http(s) URLs with a host")
    infos = socket.getaddrinfo(parts.hostname, parts.port or (443 if parts.scheme == "https" else 80))
    for *_, sockaddr in infos:
        ip = ipaddress.ip_address(sockaddr[0])
        if not ip.is_global:
            raise ValueError(f"{parts.hostname} resolves to non-public address {ip}")
    return url
```

```text
SSRF blocked: http://127.0.0.1:8000/admin -> 127.0.0.1 resolves to non-public address 127.0.0.1
SSRF blocked: http://169.254.169.254/latest/meta-data/ -> 169.254.169.254 resolves to non-public address 169.254.169.254
SSRF blocked: http://localhost/ -> localhost resolves to non-public address ::1
SSRF blocked: http://[::1]/ -> ::1 resolves to non-public address ::1
SSRF blocked: http://10.1.2.3/ -> 10.1.2.3 resolves to non-public address 10.1.2.3
SSRF blocked: file:///etc/passwd -> only http(s) URLs with a host
SSRF blocked: http://0x7f000001/ -> 0x7f000001 resolves to non-public address 127.0.0.1
```

The last line is why string checks (`"127.0.0.1" in url`) fail: `0x7f000001` is
127.0.0.1 written in hex, and the resolver accepts it. Resolving and checking the actual
address catches it. `ip.is_global` excludes loopback, private, link-local and other
reserved ranges.

This guard still has a gap: **DNS rebinding**. The hostname is resolved once for the
check, and again by the HTTP client for the real request — an attacker's DNS server can
answer differently the second time. Robust defences connect to the IP you checked (with
the original `Host` header), disable redirects or re-check every redirect target, and
run URL-fetching from a network location that can't reach internal services at all.
None of that is shown here.

## Path traversal

Serving files by name, two ways:

```python
@app.get("/covers-unsafe/{name:path}")
def cover_unsafe(name: str):
    return FileResponse(Path("files/covers") / name)

BASE = Path("files/covers").resolve()
def safe_path(name: str) -> Path:
    p = (BASE / name).resolve()
    if not p.is_relative_to(BASE) or not p.is_file():
        raise HTTPException(404, "Not found")
    return p

@app.get("/covers/{name:path}")
def cover(name: str):
    return FileResponse(safe_path(name))
```

Test files: `files/covers/dune.txt`, plus a `files/.env` containing `SECRET=1` one level
up.

```text
path dune.txt     unsafe -> 200 'cover bytes\n'           safe -> 200
path ../.env      unsafe -> 404 '{"detail":"Not Found'    safe -> 404
path ..%2F.env    unsafe -> 200 'SECRET=1\n'              safe -> 404
path /etc/hosts   unsafe -> 200 '##\n# Host Database\n#'  safe -> 404
```

- `../.env` was harmless only by accident: the HTTP client normalised `..` out of the URL
  before sending it. An attacker's client doesn't have to.
- `..%2F.env` — the slash URL-encoded — sailed through routing, was decoded into
  `../.env`, and the unsafe endpoint served the secrets file.
- `/etc/hosts`: `Path("files/covers") / "/etc/hosts"` is `/etc/hosts`, because joining
  an absolute path discards the base. The unsafe endpoint served a system file.

`safe_path` resolves the final path (following `..` and symlinks) and requires it to be
inside the base directory with `is_relative_to`. Better still, don't use client-supplied
names at all: look up a database record by ID and serve the storage name you generated
(Level 1 lesson 8).

## API8: misconfiguration — debug mode

```python
dbg = FastAPI(debug=True)

@dbg.get("/boom")
def boom():
    raise RuntimeError("db password is hunter2")
```

```text
debug=True 500 body: text/plain; charset=utf-8 contains secret: True 4724 chars
```

With `debug=True`, a crash returns the full traceback to the client — 4,724 characters
including the exception message with the "password". Other misconfigurations to check
before every release:

- `debug=False` (the default) — drive it from settings, and fail startup if
  `environment == "prod"` and debug is on;
- CORS origins explicit, never `"*"` with credentials (Level 3 lesson 4);
- docs disabled or protected on internal APIs, if that's your policy (Level 3 lesson 7);
- `TrustedHostMiddleware` and correct proxy trust (lesson 3);
- security headers on responses — for a JSON API, at least
  `X-Content-Type-Options: nosniff` and, behind HTTPS, `Strict-Transport-Security`;
  usually set at the proxy;
- secrets from the environment or a secret manager, never in the image or repository.

## How It Actually Works

**Mass assignment** works because `model_dump()` on the loose model includes every
declared field, and `dict.update` writes them all; Pydantic did exactly what the model
said. Security comes from what the model *doesn't* declare.

**The body limit** wraps the ASGI `receive` callable. FastAPI reads the body by awaiting
`receive()` until `more_body` is false; the wrapper counts bytes on each message and
raises once the limit is passed, so reading stops early. Raising `HTTPException` from
inside `receive` works because it propagates up through the route handler to
FastAPI's exception handling, which produced the 413 response.

**Path traversal** depends on two facts: Starlette's `:path` converter decodes
percent-encoded slashes into the parameter value, and `pathlib`'s `/` operator lets an
absolute right-hand side replace the left. `Path.resolve()` collapses `..` and follows
symlinks, so the containment check is done on the real target.

## Common mistakes

- **One model for every role's updates** — mass assignment.
- **Limits only in the app** (or only at the proxy). Have both.
- **String-matching URLs for SSRF**, or forgetting redirects and DNS rebinding.
- **Building file paths from user input** without resolving and containing them.
- **`debug=True` anywhere outside a laptop.**
- **Trusting third-party API responses**: validate them with a Pydantic model and set
  timeouts, exactly as you treat client input.

## Exercise

1. Add the body-limit middleware to the Level 3 bookmarks project with different limits
   for `/auth/*` (4 KB) and everything else (64 KB). Test both.
2. Add `POST /bookmarks/preview` that fetches the title of a URL with `httpx`. Use
   `resolve_public`, disable redirects, set a 3-second timeout and a 1 MB response cap,
   and test it against `http://127.0.0.1` URLs (it must refuse).
3. Write a test that tries `..%2F`, `%2e%2e%2f`, absolute paths and a symlink pointing
   outside the base against `safe_path`.
4. Add a startup check that refuses to start in production with `debug=True`, wildcard
   CORS, or a JWT secret shorter than 32 bytes.
