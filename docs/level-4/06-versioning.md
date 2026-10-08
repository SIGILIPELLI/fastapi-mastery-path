---
description: "Evolve a FastAPI API without breaking clients: additive vs breaking changes, v1 and v2 routers side by side, deprecation headers and an OpenAPI diff check."
---

# 06 · Versioning and Evolving an API

Once someone depends on your API, every change is either **additive** (old clients keep
working) or **breaking** (some client somewhere stops working). Most of the skill in API
evolution is making changes additive; versioning is the tool for the rest. This lesson
shows how to tell the two apart automatically, how to run two versions side by side in
FastAPI, and how to retire the old one politely.

## Additive vs breaking

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

Responses and requests break in opposite directions: you can usually add to a response
and remove from a request, but not the reverse.

## Two versions side by side

The bookshop's v1 sent prices as float dollars and the author as one string. v2 fixes
both — a breaking change, so it gets a new version while v1 keeps working:

```python
from fastapi import APIRouter, FastAPI, Response
from pydantic import BaseModel

BOOK = {"id": 1, "title": "Dune", "price_cents": 999,
        "author_first": "Frank", "author_last": "Herbert"}

class BookV1(BaseModel):
    id: int
    title: str
    price: float                      # v1 sent dollars as a float
    author: str

class Author(BaseModel):
    first: str
    last: str

class BookV2(BaseModel):
    id: int
    title: str
    price_cents: int                  # v2: integer cents
    author: Author                    # v2: structured

v1 = APIRouter(prefix="/v1", tags=["v1"], deprecated=True)
v2 = APIRouter(prefix="/v2", tags=["v2"])

@v1.get("/books/{book_id}", response_model=BookV1)
def get_book_v1(book_id: int, response: Response):
    response.headers["Deprecation"] = "@1798761600"               # 2027-01-01T00:00:00Z
    response.headers["Sunset"] = "Fri, 01 Oct 2027 00:00:00 GMT"
    response.headers["Link"] = '</v2/books/1>; rel="successor-version"'
    return BookV1(id=BOOK["id"], title=BOOK["title"], price=BOOK["price_cents"] / 100,
                  author=f'{BOOK["author_first"]} {BOOK["author_last"]}')

@v2.get("/books/{book_id}", response_model=BookV2)
def get_book_v2(book_id: int):
    return BookV2(id=BOOK["id"], title=BOOK["title"], price_cents=BOOK["price_cents"],
                  author=Author(first=BOOK["author_first"], last=BOOK["author_last"]))

app = FastAPI(title="Bookshop", version="2.0.0")
app.include_router(v1)
app.include_router(v2)
```

```text
v1: {'id': 1, 'title': 'Dune', 'price': 9.99, 'author': 'Frank Herbert'}
    headers: {'deprecation': '@1798761600', 'sunset': 'Fri, 01 Oct 2027 00:00:00 GMT', 'link': '</v2/books/1>; rel="successor-version"'}
v2: {'id': 1, 'title': 'Dune', 'price_cents': 999, 'author': {'first': 'Frank', 'last': 'Herbert'}}
v1 op deprecated in OpenAPI: True
```

Both versions read **the same storage**. Each version is a different *representation*
of the same data — a translation layer, not a fork of the codebase. Keep business logic
version-free and put the differences in schemas and thin endpoint functions.

### Telling clients the old version is going away

- `deprecated=True` on the router marks every v1 operation as deprecated in OpenAPI;
  `/docs` shows them struck through and generated clients flag them.
- **`Deprecation`** (RFC 9745) says the resource is deprecated, optionally since when —
  here as a structured-field date, `@` followed by Unix seconds.
- **`Sunset`** (RFC 8594) gives the date after which it may stop working.
- **`Link: <...>; rel="successor-version"`** points at the replacement.

Headers only help clients that read them. In practice you also announce in a changelog,
email API-key holders, and **measure** v1 traffic per client (lesson 4's metrics, labelled
by version or API key) so you know who still needs a nudge before the sunset date.

## Where to put the version

| Style | Example | Notes |
|---|---|---|
| URL path | `/v2/books/1` | visible, cache-friendly, easy to route at the proxy; the most common |
| header | `Accept: application/vnd.bookshop.v2+json` or `Api-Version: 2` | clean URLs; harder to test in a browser; caches must `Vary` on it |
| query | `/books/1?version=2` | easy, but mixes with real parameters |
| date-based | `Api-Version: 2026-10-01` | each client pinned to the version current when they started; more machinery |

Path versioning with routers, as above, is the simplest to operate in FastAPI. Version
the **whole API** (`/v2/...`) rather than individual endpoints, so clients don't juggle
mixed versions.

## Worked example: catching breaking changes in CI

Humans miss breaking changes in review. The OpenAPI document describes the contract, so
compare the old and new documents automatically. A deliberately small checker:

```python
def schema_props(spec, name):
    s = spec["components"]["schemas"][name]
    return s.get("properties", {}), set(s.get("required", []))

def breaking_changes(old, new):
    problems = []
    for name in old["components"]["schemas"]:
        if name not in new["components"]["schemas"]:
            problems.append(f"schema {name} removed"); continue
        op, oreq = schema_props(old, name); np_, nreq = schema_props(new, name)
        for prop in op:
            if prop not in np_:
                problems.append(f"{name}.{prop} removed")
            elif op[prop].get("type") != np_[prop].get("type"):
                problems.append(f"{name}.{prop} type {op[prop].get('type')} -> {np_[prop].get('type')}")
        for prop in nreq - oreq:
            if prop not in op:
                problems.append(f"{name}.{prop} added as required (breaks old clients sending requests)")
    for path in old["paths"]:
        if path not in new["paths"]:
            problems.append(f"path {path} removed")
    return problems
```

Comparing an old `BookIn` (`title: str, price: float`, plus a search endpoint) with a new
one that changed `price` to `int`, added a required `isbn`, and dropped the search
endpoint:

```text
BREAKING: BookIn.price type number -> integer
BREAKING: BookIn.isbn added as required (breaks old clients sending requests)
BREAKING: path /v1/books/search removed
```

And a purely additive change (an optional `subtitle` field and an optional `limit`
query parameter):

```text
additive change problems: []
```

In CI: commit the current `openapi.json` (generate it with
`json.dump(app.openapi(), f, indent=2)`), regenerate on each pull request, and fail the
build on any breaking change unless the change also bumps the major version. This checker
only covers the cases above and treats every schema as both request and response; for
real projects, a dedicated OpenAPI diff tool (several open-source ones exist) handles
parameters, enums, nested schemas and request-vs-response direction properly. None was
installed for this lesson.

## Renaming a field without a new version

Sometimes you can make a "breaking" change additively. To rename `price` to
`price_cents` in a response, send **both** for a while:

```python
from pydantic import computed_field

class BookV1Transitional(BaseModel):
    id: int
    title: str
    price_cents: int

    @computed_field(deprecated="use price_cents")
    @property
    def price(self) -> float:
        return self.price_cents / 100
```

On Pydantic 2.14.0, dumping one instance gave
`{'id': 1, 'title': 'Dune', 'price_cents': 999, 'price': 9.99}`, and the serialization
schema marked the old field
`{'deprecated': True, 'readOnly': True, 'title': 'Price', 'type': 'number'}` — so
generated clients flag it. One side effect: serializing emitted a Python
`DeprecationWarning: use price_cents` on the *server* every time (visible with warnings
enabled), which you may want to filter in production logs.

Old clients keep reading `price`; new ones switch to `price_cents`; later, when metrics
show nobody reads the old field (or a sunset date passes), remove it in the next major
version.

## How It Actually Works

Including two routers with different prefixes creates two independent sets of routes in
one app; the dependency system, middleware and lifespan are shared. Each route has its
own response model, so FastAPI generates separate schemas (`BookV1`, `BookV2`) under
`components/schemas` — one OpenAPI document describing both versions. You could also
mount two separate `FastAPI` sub-applications (lesson 8) to get two separate documents.

`deprecated=True` on an `APIRouter` is a default copied to each route, which sets
`"deprecated": true` on the operation. It has no runtime effect; the headers are what
clients see at runtime.

## Common mistakes

- **Breaking changes without a version bump** — "it's only a type change".
- **Forking the codebase per version** instead of translating representations.
- **Versioning single endpoints**, leaving clients with a patchwork.
- **No sunset plan**: v1 lives forever, with its bugs and security issues (API9 in
  lesson 5's table).
- **Removing a version before measuring who still uses it.**
- **Adding enum values** without warning clients that switch exhaustively over them.

## Exercise

1. Add `/v2/books` (list) to the example with cursor pagination while `/v1/books` keeps
   offset pagination. Share the data access function.
2. Add a middleware that counts requests per API version and per API key, and expose the
   counts on `/metrics`.
3. Extend `breaking_changes` to detect removed query parameters and newly required ones.
   Test it with two versions of an endpoint.
4. Write the changelog entry and the email you'd send to v1 users announcing the sunset.
