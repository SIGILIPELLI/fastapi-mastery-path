---
description: "FastAPI's Depends system: function and class dependencies, chains like token → user → admin, router-wide dependencies, per-request caching and overrides."
---

# 01 · Dependency Injection with Depends

Level 1's notes API had a smell: endpoints reached into a module attribute to find the
store, and tests replaced that attribute to get a clean one. Real apps have many things
like that — a database session, the current user, settings, pagination parameters, a
feature flag client. **Dependency injection** is how FastAPI hands those to your
endpoints: you declare *what* an endpoint needs, and FastAPI works out *how* to provide
it, per request.

## The simplest dependency

A dependency is any callable. Its parameters are parsed exactly like an endpoint's:

```python
from typing import Annotated
from fastapi import Depends, FastAPI, Query

app = FastAPI()

def pagination(limit: Annotated[int, Query(ge=1, le=100)] = 20,
               offset: Annotated[int, Query(ge=0)] = 0) -> dict:
    return {"limit": limit, "offset": offset}

@app.get("/books")
def list_books(page: Annotated[dict, Depends(pagination)]):
    return {"page": page}
```

`Depends(pagination)` says: before calling `list_books`, call `pagination` — reading
*its* parameters from the request — and pass the result as `page`. Every endpoint that
paginates can now share one definition, with one set of limits.

The dependency's parameters appear in OpenAPI as if they were the endpoint's own. For a
`/books` endpoint that also used the `Sorting` class below, the documented query
parameters were:

```text
['limit', 'offset', 'sort', 'desc']
```

## Classes as dependencies

A class is callable too — calling it creates an instance — so FastAPI can use its
`__init__` parameters:

```python
from dataclasses import dataclass

@dataclass
class Sorting:
    sort: str = "title"
    desc: bool = False

@app.get("/books")
def list_books(page: Annotated[dict, Depends(pagination)],
               sorting: Annotated[Sorting, Depends()]):
    return {"page": page, "sorting": sorting}
```

`Depends()` with no argument means "use the type annotation as the dependency".

```text
/books?limit=5&sort=year&desc=true -> 200 {'page': {'limit': 5, 'offset': 0}, 'sorting': {'sort': 'year', 'desc': True}}
```

A dataclass gives you attribute access and type checking in the endpoint
(`sorting.desc`) instead of dictionary keys.

## Chains: token → user → admin

Dependencies can depend on other dependencies. This is where the system earns its keep.

```python
from fastapi import Header, HTTPException

USERS = {"tok-ada": {"name": "ada", "role": "admin"},
         "tok-bob": {"name": "bob", "role": "reader"}}

def get_token(x_api_key: Annotated[str | None, Header()] = None) -> str:
    if x_api_key is None:
        raise HTTPException(401, "Missing X-API-Key", headers={"WWW-Authenticate": "ApiKey"})
    return x_api_key

def get_current_user(token: Annotated[str, Depends(get_token)]) -> dict:
    user = USERS.get(token)
    if user is None:
        raise HTTPException(401, "Invalid API key")
    return user

CurrentUser = Annotated[dict, Depends(get_current_user)]

def require_admin(user: CurrentUser) -> dict:
    if user["role"] != "admin":
        raise HTTPException(403, "Admins only")
    return user

@app.get("/me")
def me(user: CurrentUser):
    return {"user": user["name"]}
```

(A toy API-key scheme; Level 3 builds real authentication with hashed passwords and
tokens.) `CurrentUser` is a **type alias** that bundles the type and the dependency.
Endpoints now just write `user: CurrentUser`, which reads like plain type hinting.

The test runs recorded which dependencies were called for each request:

```text
/me                          -> 401 {'detail': 'Missing X-API-Key'} | calls: ['get_token']
/me  X-API-Key: nope         -> 401 {'detail': 'Invalid API key'}   | calls: ['get_token', 'get_current_user']
/me  X-API-Key: tok-bob      -> 200 {'user': 'bob', ...}           | calls: ['get_token', 'get_current_user']
```

The chain stops at the first `raise`. The endpoint function never runs for a
missing or invalid key, so it doesn't need a single `if` for authentication.

## Router-wide dependencies

When you need a dependency's *effect* (its checks) but not its *value*, put it in a
`dependencies=` list. On a router it applies to every route:

```python
from fastapi import APIRouter

admin = APIRouter(prefix="/admin", dependencies=[Depends(require_admin)])

@admin.get("/stats")
def stats(user: CurrentUser):
    return {"requested_by": user["name"]}

app.include_router(admin)
```

```text
/admin/stats  X-API-Key: tok-bob -> 403 {'detail': 'Admins only'}       | calls: ['get_token', 'get_current_user', 'require_admin']
/admin/stats  X-API-Key: tok-ada -> 200 {'requested_by': 'ada'}         | calls: ['get_token', 'get_current_user', 'require_admin']
```

Notice the `ada` request: `stats` asked for `CurrentUser` too, but `get_current_user`
appears only **once** in the call log. That's the per-request cache.

## The per-request cache

Within one request, each dependency is called at most once; every place that asks for it
receives the same result. An endpoint that took `user: CurrentUser, again: CurrentUser`
got `'same_object': True`.

Usually that's what you want — you don't want to look the user up twice. When you
genuinely need a fresh value each time, opt out:

```python
from itertools import count
n = count(1)
def ticket():
    return next(n)

@app.get("/t")
def t(a: Annotated[int, Depends(ticket)],
      b: Annotated[int, Depends(ticket)],
      c: Annotated[int, Depends(ticket, use_cache=False)]):
    return [a, b, c]
```

Two requests:

```text
[1, 1, 2] [3, 3, 4]
```

`a` and `b` shared a value; `c` got its own. The cache is per request — the second
request started fresh.

## Worked example: overriding dependencies in tests

`app.dependency_overrides` is a dict mapping a dependency to a replacement. FastAPI
checks it every time it resolves a dependency:

```python
app.dependency_overrides[get_current_user] = lambda: {"name": "test-user", "role": "admin"}
# GET /admin/stats with no header at all:
```

```text
/admin/stats -> 200 {'requested_by': 'test-user'} | calls: ['require_admin']
```

`get_token` didn't run — overriding `get_current_user` replaced it *and everything
beneath it* in the chain, so no header was needed. `require_admin` still ran, with the
fake user. This is the clean replacement for Level 1's "swap the module attribute"
trick, and it's how Level 2 lesson 8 tests database code. Always clear overrides after
a test (`app.dependency_overrides.clear()`), or they leak into the next one.

## How It Actually Works

At startup, FastAPI inspects each endpoint and builds a **dependant** tree: the endpoint
at the root, each `Depends(...)` a child, recursively. Each node records which request
parameters it needs (query, header, body…) and which sub-dependencies. The OpenAPI
generator walks the same tree, which is why dependency parameters show up in the docs.

Per request, `solve_dependencies` walks the tree depth-first:

1. For each sub-dependency, check `dependency_overrides` for a replacement; then check
   the per-request cache, keyed by the callable (and its security scopes). If cached,
   reuse the value.
2. Otherwise, recursively solve *its* sub-dependencies, then validate its own request
   parameters with Pydantic. Validation errors are **collected, not raised**, and a
   dependency whose inputs are invalid is not called. That's why `/books?limit=0` returned
   the usual 422 while the call log was empty — `pagination` never ran.
3. Call it: `await` for `async def`, run in the thread pool for `def`. Store the result
   in the cache.
4. When the tree is solved, if any errors were collected, raise one
   `RequestValidationError` with all of them. Otherwise call the endpoint with the
   values.

An `HTTPException` raised inside a dependency propagates immediately, aborting the rest
of the tree — the 401 for a missing key came before any other work.

Because the cache key is the callable itself, two *different* functions that both read
the same header are two separate dependencies; and a lambda created inline in each
endpoint (`Depends(lambda: ...)`) is a new callable every time, so it's never shared.

## Common mistakes

- **Calling the dependency yourself**: `Depends(get_db())` passes the *result* of calling
  it at import time. For a generator dependency, registering the route failed with
  `TypeError: <generator object get_db at 0x…> is not a callable object`. Pass the
  function: `Depends(get_db)`.
- **Forgetting `Depends`** and writing `user: dict = get_current_user`. No error at all:
  FastAPI treated `user` as an optional JSON body whose default is the function object.
  A request with no body made the endpoint report `{'type': 'function'}`; a request
  with a JSON body got that body as the "user". That's an authentication bypass waiting
  to happen.
- **Using `dependencies=[...]` and expecting a value.** That list discards return values;
  declare a parameter if you need it.
- **Heavy work in dependencies that run on every request**, like loading a large file.
  Load once at startup (Level 3 lesson 5) and inject a reference.
- **Leaking `dependency_overrides`** between tests.
- **Global singletons instead of dependencies.** A module-level client works until you
  need to replace it in a test or configure it per environment.

## Exercise

1. Write a `Pagination` dataclass dependency with `limit` and `offset`, plus a computed
   `slice()` method. Use it in two endpoints.
2. Build a chain `get_api_key → get_tenant → get_current_user` where the tenant comes
   from an `X-Tenant` header and users are looked up per tenant. Return 404 for an
   unknown tenant and 401 for an unknown user.
3. Put `require_admin` on a router, then add one endpoint to that router that should be
   public. What happens? How would you restructure?
4. In a test, override `get_current_user` with a reader (not an admin) and assert that
   `/admin/stats` returns 403.
