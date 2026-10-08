---
description: "Authorization in FastAPI: OAuth2 scopes with Security and SecurityScopes, role-checking dependency factories, and ownership checks that stop BOLA."
---

# 03 · Authorization: Scopes, Roles & Ownership

Authentication answers "who are you?". **Authorization** answers "what may you do?" —
and it's where most real API breaches happen. The OWASP API Security Top 10 has put
**Broken Object Level Authorization** (BOLA) at number one in both its 2019 and 2023
editions: the API checks that you're logged in, then hands you any record whose ID you
ask for.

This lesson builds three layers, each a dependency:

1. **Scopes** — what this *token* is allowed to do (a read-only token can't write, even
   if its user could).
2. **Roles** — what this *user* is allowed to do (admins manage users).
3. **Ownership** — whether this user may touch *this particular object*.

## The setup

Users carry a role and the scopes they're entitled to; tokens are JWTs as in lesson 2,
with a space-separated `scope` claim (the OAuth2 convention).

```python
from datetime import datetime, timedelta, timezone
from typing import Annotated
import jwt
from fastapi import Depends, FastAPI, HTTPException, Security
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm, SecurityScopes
from pydantic import BaseModel

SECRET, ALG = "test-only-secret-change-me-0123456789abcdef", "HS256"
SCOPES = {"books:read": "Read books", "books:write": "Create and edit your books",
          "admin": "Manage all users"}
USERS = {   # toy credentials; lesson 1 shows real password hashing
    "ada":   {"password": "pw", "role": "member", "scopes": ["books:read", "books:write"]},
    "bob":   {"password": "pw", "role": "member", "scopes": ["books:read", "books:write"]},
    "root":  {"password": "pw", "role": "admin",  "scopes": ["books:read", "books:write", "admin"]},
    "guest": {"password": "pw", "role": "viewer", "scopes": ["books:read"]},
}
NOTES = {1: {"id": 1, "owner": "ada", "text": "ada's private note"},
         2: {"id": 2, "owner": "bob", "text": "bob's private note"}}

app = FastAPI()
oauth2 = OAuth2PasswordBearer(tokenUrl="token", scopes=SCOPES)

class User(BaseModel):
    username: str
    role: str
    scopes: list[str]

@app.post("/token")
def login(form: Annotated[OAuth2PasswordRequestForm, Depends()]):
    u = USERS.get(form.username)
    if not u or u["password"] != form.password:
        raise HTTPException(401, "Incorrect username or password")
    granted = [s for s in form.scopes if s in u["scopes"]] if form.scopes else u["scopes"]
    token = jwt.encode({"sub": form.username, "scope": " ".join(granted),
                        "exp": datetime.now(timezone.utc) + timedelta(minutes=15)},
                       SECRET, algorithm=ALG)
    return {"access_token": token, "token_type": "bearer", "scope": " ".join(granted)}
```

The client may ask for fewer scopes than the user has (`scope=books:read` in the form),
and never gets more than the user has:

```text
ada token scope: books:read books:write | ada read-only token scope: books:read
guest asks for admin scope -> books:read
```

The guest asked for `admin` and was silently granted only what they're entitled to.

## Layer 1: scopes with `Security` and `SecurityScopes`

```python
def current_user(security_scopes: SecurityScopes,
                 token: Annotated[str, Depends(oauth2)]) -> User:
    authenticate = (f'Bearer scope="{security_scopes.scope_str}"'
                    if security_scopes.scopes else "Bearer")
    try:
        claims = jwt.decode(token, SECRET, algorithms=[ALG],
                            options={"require": ["exp", "sub"]})
    except jwt.PyJWTError:
        raise HTTPException(401, "Invalid token", headers={"WWW-Authenticate": authenticate})
    token_scopes = claims.get("scope", "").split()
    for needed in security_scopes.scopes:
        if needed not in token_scopes:
            raise HTTPException(403, f"Missing scope: {needed}",
                                headers={"WWW-Authenticate": authenticate})
    u = USERS[claims["sub"]]
    return User(username=claims["sub"], role=u["role"], scopes=token_scopes)

Reader = Annotated[User, Security(current_user, scopes=["books:read"])]
Writer = Annotated[User, Security(current_user, scopes=["books:write"])]
```

`Security(...)` is `Depends(...)` plus a list of required scopes. FastAPI collects the
scopes required by the whole dependency chain into the `SecurityScopes` parameter, so one
`current_user` function serves every permission level. Note the status codes: a bad or
missing token is **401** (authenticate again); a valid token lacking a scope is **403**
(re-authenticating as the same token won't help).

```text
ada read-only writes  (403, {'detail': 'Missing scope: books:write'}, 'Bearer scope="books:write"')
guest writes          (403, {'detail': 'Missing scope: books:write'}, 'Bearer scope="books:write"')
ada writes            (201, {'id': 3, 'owner': 'ada', 'text': 'hi'})
```

Scopes appear in the OpenAPI document per operation, so `/docs` shows which scopes each
endpoint needs and the Authorize dialog offers checkboxes:

```text
openapi security on POST /notes: [{'OAuth2PasswordBearer': ['books:write']}]
```

## Layer 2: roles with a dependency factory

A function that *returns* a dependency lets you parameterise checks:

```python
def require_role(*roles: str):
    def checker(user: Reader) -> User:
        if user.role not in roles:
            raise HTTPException(403, f"Requires role: {' or '.join(roles)}")
        return user
    return checker

@app.get("/admin/users", dependencies=[Security(current_user, scopes=["admin"])])
def list_users(user: Annotated[User, Depends(require_role("admin"))]):
    return sorted(USERS)
```

```text
ada lists users   (403, {'detail': 'Missing scope: admin'}, 'Bearer scope="admin"')
root lists users  (200, ['ada', 'bob', 'guest', 'root'])
```

This endpoint requires *both* the `admin` scope on the token and the `admin` role on the
user. Why both? The role says the person is an administrator; the scope says this
particular token was issued for administrative work. An admin's everyday read-only token
— the one stored in a reporting script, say — then can't delete users if it leaks.

## Layer 3: ownership — the one people forget

```python
def owned_note(note_id: int, user: Reader) -> dict:
    note = NOTES.get(note_id)
    # Same answer for "doesn't exist" and "not yours": don't reveal which IDs exist.
    if note is None or (note["owner"] != user.username and user.role != "admin"):
        raise HTTPException(404, "Note not found")
    return note

@app.get("/notes/{note_id}")
def read_note(note: Annotated[dict, Depends(owned_note)]):
    return note

@app.get("/notes-insecure/{note_id}")
def read_note_insecure(note_id: int, user: Reader):
    return NOTES[note_id]          # authenticated, but no ownership check (BOLA)
```

```text
ada reads own note      (200, {'id': 1, 'owner': 'ada', 'text': "ada's private note"})
ada reads bob's note    (404, {'detail': 'Note not found'})
ada reads missing note  (404, {'detail': 'Note not found'})
root reads bob's note   (200, {'id': 2, 'owner': 'bob', 'text': "bob's private note"})
INSECURE ada->bob       (200, {'id': 2, 'owner': 'bob', 'text': "bob's private note"})
```

The insecure endpoint is BOLA in four lines: Ada is properly authenticated with a valid
scope, and still reads Bob's note by changing one number in the URL. Sequential integer
IDs make it trivial to enumerate every record.

Making the *dependency* load the object means every endpoint that takes `note_id` can
reuse `owned_note`, and the endpoint body never sees an object the user isn't allowed to
have. Returning **404** for "not yours" — identical to "doesn't exist" — avoids
confirming that note 2 exists. (Some APIs prefer 403 for clarity; choose deliberately and
be consistent.)

## Worked example: ownership in the query, not after it

With a database, the strongest form puts the owner into the `WHERE` clause, so a row the
user can't see is never loaded:

```python
def owned_book(book_id: int, user: Reader, db: DB) -> Book:
    stmt = select(Book).where(Book.id == book_id)
    if user.role != "admin":
        stmt = stmt.where(Book.owner_id == user.id)
    book = db.scalar(stmt)
    if book is None:
        raise HTTPException(404, "Book not found")
    return book
```

The same idea applies to list endpoints: `select(Book).where(Book.owner_id == user.id)`
as the base query, with filters added on top (Level 2 lesson 9). Forgetting it on a list
endpoint leaks everyone's data at once. (This variant is the pattern used in the Level 3
project, where it is tested against a database; it isn't run on its own here.)

## How It Actually Works

`Security(dep, scopes=[...])` creates the same dependency node as `Depends`, with extra
scope information. While building the dependant tree at startup, FastAPI accumulates the
scopes from each `Security` on the path from the endpoint down to any dependency that
declares a `SecurityScopes` parameter, and passes that union in. For `/admin/users`, the
`current_user` reached through `dependencies=[Security(..., scopes=["admin"])]` receives
`["admin"]`, while the one reached through `require_role` → `Reader` receives
`["books:read"]`.

The per-request dependency cache (Level 2 lesson 1) includes the security scopes in its
key, so those two are separate cache entries — `current_user` runs twice for that
request, once per scope set. Logging `security_scopes.scopes` on each call during one
`GET /admin/users` recorded `[['admin'], ['books:read']]`. That's correct (each check must happen) and cheap here,
since decoding a JWT is fast; with a database lookup inside `current_user`, you might
cache the user lookup in a separate, scope-free dependency.

The scope strings themselves are just strings you choose. FastAPI doesn't interpret
`books:write`; it only checks list membership in your code and documents the scopes.

## Common mistakes

- **Authentication without object-level authorization** — the BOLA endpoint above.
  Every endpoint that takes an ID needs an ownership (or tenancy) check.
- **Checking ownership after loading and returning data in error messages**, e.g.
  `"Note 2 belongs to bob"`.
- **401 vs 403 confusion.** 401: no valid credentials. 403: valid credentials,
  insufficient permission.
- **Trusting role or scope claims sent by the client** in a body or header. They must
  come from a token you signed or from your database.
- **Granting requested scopes without intersecting** them with the user's entitlements.
- **Role checks scattered in endpoint bodies** (`if user.role == ...`). Put them in
  dependencies or on routers so a new endpoint can't forget them.
- **Forgetting list endpoints** when adding ownership checks.

## Exercise

1. Add `PUT /notes/{note_id}` and `DELETE /notes/{note_id}` that reuse `owned_note` and
   also require the `books:write` scope. Write tests for owner, non-owner, admin and a
   read-only token.
2. Add `GET /notes` that returns only the caller's notes (all notes for admins).
3. Introduce teams: a note belongs to a team, and any member may read it but only the
   author may delete it. Express both rules as dependencies.
4. Write a test that loops over note IDs 1–100 as a non-admin and asserts that every
   response is either the caller's own note or a 404.
