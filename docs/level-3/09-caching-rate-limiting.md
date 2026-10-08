---
description: "Make a FastAPI API cheaper and safer: Cache-Control, ETags with 304 responses, If-Match optimistic locking, a TTL cache and a token-bucket rate limiter."
---

# 09 · Caching, ETags & Rate Limiting

Two ways to make an API cheaper to run: **do less work per request** (caching, and
letting clients skip downloads they already have) and **accept fewer requests from any
one client** (rate limiting). Both are mostly HTTP features; FastAPI just gives you easy
access to the headers. This lesson implements each by hand so the mechanics are visible,
then discusses what changes in a multi-process deployment.

## ETags and conditional GET

An **ETag** is a fingerprint of a resource's current state. The client keeps it and asks
"send me this only if it changed" with `If-None-Match`; the server answers
**304 Not Modified** with no body if the fingerprint still matches.

```python
import hashlib, json
from typing import Annotated
from fastapi import FastAPI, Header, HTTPException, Response

app = FastAPI()
BOOKS = {1: {"id": 1, "title": "Dune", "price_cents": 999, "version": 1}}

def etag_for(obj: dict) -> str:
    body = json.dumps(obj, sort_keys=True, separators=(",", ":")).encode()
    return '"' + hashlib.sha256(body).hexdigest()[:16] + '"'

@app.get("/books/{book_id}")
def get_book(book_id: int, response: Response,
             if_none_match: Annotated[str | None, Header()] = None):
    book = BOOKS.get(book_id)
    if book is None:
        raise HTTPException(404, "Book not found")
    tag = etag_for(book)
    headers = {"ETag": tag, "Cache-Control": "private, max-age=0, must-revalidate"}
    if if_none_match and tag in [t.strip() for t in if_none_match.split(",")]:
        return Response(status_code=304, headers=headers)
    response.headers.update(headers)
    return book
```

```text
GET                       -> 200 "44332aef971c651b" private, max-age=0, must-revalidate 53 bytes
GET If-None-Match: <tag>  -> 304 0 bytes "44332aef971c651b"
```

The second response carried no body. For a 53-byte book that saves little; for a large
catalogue page polled every few seconds by a mobile app, it saves most of the bandwidth.
Browsers do this automatically for `GET` requests when they've seen an `ETag`.

`Cache-Control: private, max-age=0, must-revalidate` says: only the user's own client
may store it (not shared proxies — it may be user-specific), and it must check back with
the server before every reuse. Common alternatives:

| Header | Meaning | Use for |
|---|---|---|
| `no-store` | don't keep it at all | sensitive data |
| `private, max-age=0, must-revalidate` | keep, but revalidate each time (ETag) | per-user data that changes |
| `public, max-age=300` | anyone may cache it for 5 minutes | public catalogue data |
| `public, max-age=31536000, immutable` | cache "forever" | versioned static files |

The ETag here is computed from the whole object; a cheaper source in practice is the
row's version number or `updated_at`. The quotes are part of the ETag syntax.

## Worked example: `If-Match` prevents lost updates

Two staff members open the same book; both change the price; the second save silently
overwrites the first. The fix is **optimistic concurrency**: a write must say which
version it's based on.

```python
from pydantic import BaseModel

class PricePatch(BaseModel):
    price_cents: int

@app.patch("/books/{book_id}")
def patch_book(book_id: int, patch: PricePatch, response: Response,
               if_match: Annotated[str | None, Header()] = None):
    book = BOOKS[book_id]
    if if_match is None:
        raise HTTPException(428, "If-Match header required")
    if if_match != etag_for(book):
        raise HTTPException(412, "Book was modified by someone else; re-fetch and retry")
    book.update(price_cents=patch.price_cents, version=book["version"] + 1)
    response.headers["ETag"] = etag_for(book)
    return book
```

```text
PATCH no If-Match          -> 428 {'detail': 'If-Match header required'}
PATCH If-Match: <tag>      -> 200 {'id': 1, 'title': 'Dune', 'price_cents': 899, 'version': 2} "9193797af1f2fcaa"
PATCH stale If-Match: <tag>-> 412 {'detail': 'Book was modified by someone else; re-fetch and retry'}
GET If-None-Match: <old>   -> 200 {'id': 1, ..., 'price_cents': 899, 'version': 2}
```

The second writer, still holding the old ETag, got **412 Precondition Failed** instead of
clobbering the change; **428 Precondition Required** forces clients to send the header at
all. And a client revalidating with the old tag correctly received the new version.

In a database, do the check and the write in one statement so two requests can't both
pass the check: `UPDATE books SET price_cents = :p, version = version + 1 WHERE id = :id AND version = :v`,
then treat "0 rows updated" as 412. The dict version above is not safe under real
concurrency.

## A server-side TTL cache

When a result is expensive and slightly stale data is acceptable, cache it in memory for
a short time:

```python
import time

class TTLCache:
    def __init__(self, ttl: float):
        self.ttl, self.store = ttl, {}

    def get_or_set(self, key, fn, now=None):
        now = time.monotonic() if now is None else now
        hit = self.store.get(key)
        if hit and hit[1] > now:
            return hit[0], True
        value = fn()
        self.store[key] = (value, now + self.ttl)
        return value, False

cache = TTLCache(ttl=30)

@app.get("/bestsellers")
def bestsellers():
    value, hit = cache.get_or_set("bestsellers", expensive_bestsellers)  # a 0.2 s query
    return {"items": value, "cache": "hit" if hit else "miss"}
```

```text
miss 233 ms
hit 4 ms
hit 3 ms
expensive calls: 1
```

Caveats this tiny cache ignores, and real ones handle:

- **Per process.** Four workers mean four caches and four misses.
- **Stampedes.** When the entry expires under load, many requests miss at once and all
  run the expensive query. Fixes: a lock per key, or refreshing slightly before expiry.
- **Unbounded size** if keys come from user input. Bound it (an LRU), or don't key on
  arbitrary input.
- **Never cache per-user data under a shared key.** Include the user (or tenant) in the
  key, or cache only public data.

A shared cache (Redis, Memcached) solves the per-process problem. It wasn't available
for this lesson, so it isn't demonstrated here.

## Rate limiting with a token bucket

Each client has a bucket holding up to `burst` tokens, refilled at `rate` tokens per
second. A request takes one token; an empty bucket means **429 Too Many Requests**.

```python
import math
from fastapi import Depends, Request

class TokenBucket:
    def __init__(self, rate_per_sec: float, burst: int):
        self.rate, self.burst = rate_per_sec, burst
        self.state: dict[str, tuple[float, float]] = {}    # key -> (tokens, last_ts)

    def take(self, key: str, now: float) -> tuple[bool, float, int]:
        tokens, last = self.state.get(key, (self.burst, now))
        tokens = min(self.burst, tokens + (now - last) * self.rate)
        if tokens >= 1:
            self.state[key] = (tokens - 1, now)
            return True, 0.0, int(tokens - 1)
        self.state[key] = (tokens, now)
        return False, (1 - tokens) / self.rate, 0

bucket = TokenBucket(rate_per_sec=1.0, burst=3)
CLOCK = {"now": 1000.0}          # injectable clock so the test is deterministic

def rate_limit(request: Request, response: Response):
    key = request.headers.get("x-api-key") or request.client.host
    ok, retry_after, remaining = bucket.take(key, CLOCK["now"])
    response.headers["RateLimit-Limit"] = str(bucket.burst)
    response.headers["RateLimit-Remaining"] = str(remaining)
    if not ok:
        raise HTTPException(429, "Too many requests",
                            headers={"Retry-After": str(math.ceil(retry_after))})

@app.post("/token", dependencies=[Depends(rate_limit)])
def login():
    return {"ok": True}
```

With a fake clock, requests at the given times for one API key:

```text
t=0.0s -> 200 remaining=2 retry-after=None
t=0.0s -> 200 remaining=1 retry-after=None
t=0.0s -> 200 remaining=0 retry-after=None
t=0.0s -> 429 remaining=None retry-after=1
t=0.5s -> 429 remaining=None retry-after=1
t=1.0s -> 200 remaining=0 retry-after=None
t=1.0s -> 429 remaining=None retry-after=1
t=4.0s -> 200 remaining=2 retry-after=None
other key -> 200
```

A burst of three, then one per second; after an idle gap the bucket refilled to its cap
(not beyond). A different key had its own bucket. Notice `remaining=None` on the 429s:
headers set on the injected `Response` were **lost** when the `HTTPException` replaced
the response. Anything a 429 must carry — `Retry-After`, the `RateLimit-*` headers —
has to go in the exception's `headers=`.

The `RateLimit-Limit`/`RateLimit-Remaining` names follow an IETF draft that has gone
through several revisions; `Retry-After` is the long-established standard header. Check
what your clients expect.

## How It Actually Works

Conditional requests are defined in RFC 9110. The server compares the client's
validator (`If-None-Match` with ETags, or `If-Modified-Since` with dates) to the current
one; on a match a `GET` gets `304` and a body-less response, which clients satisfy from
their cache. `If-Match` is the write-side mirror: it's a *precondition*, and failing it
yields `412`. The ETag computation is entirely yours — HTTP only requires that it change
whenever the representation changes.

The token bucket stores two numbers per key and computes refill lazily on each request,
so idle keys cost nothing until they return. That "compute on access" design is also
what makes it easy to move to Redis: the read-refill-take-write sequence becomes one
atomic Lua script or transaction, so that every worker shares the same buckets. An
in-process limiter with four workers effectively allows four times the configured rate,
and resets on every deploy.

Choosing the key matters as much as the algorithm. Behind a reverse proxy,
`request.client.host` is the proxy's address unless you configure forwarded headers
(Level 4 lesson 3) — and then every client shares one bucket. Per-API-key or per-user
limits are more meaningful; per-IP limits are a coarse fallback, unfair to users behind
shared NAT.

## Common mistakes

- **Caching personalised responses publicly** (`public` on per-user data, or a shared
  cache key) — users see each other's data.
- **ETags that don't change** when the representation changes (e.g. derived from the ID
  only).
- **Check-then-write** for `If-Match` without making it atomic in the database.
- **Rate limits only in process** with several workers; or keyed by the proxy's IP.
- **Losing `Retry-After`** by setting it on the response instead of the exception.
- **Rate-limiting health checks** and getting your own instances killed by the load
  balancer.
- **Unbounded cache keys from user input** — a memory leak an attacker controls.

## Exercise

1. Derive the ETag from a `version` column in the Level 2 bookshelf, and implement the
   atomic `UPDATE ... WHERE version = :v` form of `If-Match`. Test both 200 and 412.
2. Add `If-None-Match` support to the list endpoint, with the ETag computed from the
   newest `updated_at` and the total count.
3. Make the rate limiter key on the authenticated user when there is one, and fall back
   to the client address otherwise. Apply a stricter bucket to `/token`.
4. Add a per-key lock to `TTLCache` to prevent stampedes, and write a test with
   concurrent requests that shows the expensive function runs once.
