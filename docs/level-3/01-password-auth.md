---
description: "Store passwords safely with Argon2 via pwdlib and build FastAPI's OAuth2 password flow: token endpoint, bearer dependency, logout and timing-safe login."
---

# 01 · Password Hashing & the OAuth2 Password Flow

Authentication is where small mistakes become headlines. This lesson does the two
foundational pieces properly: storing passwords so a database leak doesn't hand out
everyone's credentials, and the login flow FastAPI's security utilities are built
around. Lesson 2 replaces the server-side tokens used here with JWTs; lesson 3 adds
permissions.

For the wider security picture — threat models, TLS, sessions vs tokens — see the
[Cybersecurity Mastery Path](https://sigilipelli.github.io/cybersecurity-mastery-path/).

## Never store passwords — store slow hashes

A password hash for storage must be:

- **one-way** — you can check a guess, not recover the password;
- **salted** — a random value per hash, so identical passwords produce different hashes
  and precomputed tables don't work;
- **deliberately slow and memory-hungry** — so an attacker with a stolen database can
  try only a limited number of guesses per second.

Fast hashes like SHA-256 fail the third test badly: GPUs compute them by the billion.
Use a password-hashing algorithm: **Argon2id** (the current OWASP first choice), scrypt,
or bcrypt. Recent versions of FastAPI's security tutorial use the `pwdlib` library, and so
does this lesson:

```bash
pip install "pwdlib[argon2]"
```

(pwdlib 0.3.1 with argon2-cffi 25.1.0 here.)

```python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()
h1 = password_hash.hash("correct horse battery staple")
h2 = password_hash.hash("correct horse battery staple")
print(h1)
print(h1 != h2)
print(password_hash.verify("correct horse battery staple", h1))
print(password_hash.verify("Correct horse battery staple", h1))
```

```text
$argon2id$v=19$m=65536,t=3,p=4$GA5Q3ryHgsHQ2nBab/+zVA$3igI7E4GA91j+gDhrXK129lXg3h/epxGlMKO53KIgsA
True
True
False
```

The stored string is self-describing: algorithm `argon2id`, version 19, **64 MiB of
memory** (`m=65536` KiB), 3 iterations, 4 lanes, then the salt and the hash. Because the
parameters are inside the string, you can raise them later and old hashes still verify.
One verification took **about 40 ms** on the machine used — slow enough to hurt an
attacker, fast enough for a login.

!!! note "passlib"
    Older FastAPI tutorials use `passlib` with bcrypt. It still appears in many projects,
    but it's no longer actively maintained and has had compatibility problems with newer
    `bcrypt` releases. For new code, use `pwdlib` (or `argon2-cffi` directly).

## The OAuth2 password flow

OAuth2 defines several ways to get a token. The "password" flow is the simplest: the
client sends a username and password **as form data** to a token endpoint and receives a
bearer token; later requests send `Authorization: Bearer <token>`. FastAPI provides two
helpers:

- `OAuth2PasswordRequestForm` — a dependency that parses the form fields the spec
  requires;
- `OAuth2PasswordBearer` — a dependency that extracts the bearer token from the header,
  and documents the scheme in OpenAPI so `/docs` gets an **Authorize** button.

```python
import secrets
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel

USERS = {"ada": {"username": "ada", "hashed_password": h1, "disabled": False}}
TOKENS: dict[str, str] = {}          # token -> username (server-side session store)
DUMMY_HASH = password_hash.hash("dummy-password-for-timing")

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class Token(BaseModel):
    access_token: str
    token_type: str

def authenticate(username: str, password: str) -> dict | None:
    user = USERS.get(username)
    if user is None:
        password_hash.verify(password, DUMMY_HASH)   # same cost as a real check
        return None
    if not password_hash.verify(password, user["hashed_password"]):
        return None
    return user

@app.post("/token", response_model=Token)
def login(form: Annotated[OAuth2PasswordRequestForm, Depends()]):
    user = authenticate(form.username, form.password)
    if user is None:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Incorrect username or password",
                            headers={"WWW-Authenticate": "Bearer"})
    token = secrets.token_urlsafe(32)
    TOKENS[token] = user["username"]
    return Token(access_token=token, token_type="bearer")

def current_user(token: Annotated[str, Depends(oauth2_scheme)]) -> dict:
    username = TOKENS.get(token)
    if username is None:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid or expired token",
                            headers={"WWW-Authenticate": "Bearer"})
    user = USERS[username]
    if user["disabled"]:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Account disabled")
    return user

@app.get("/me")
def me(user: Annotated[dict, Depends(current_user)]):
    return {"username": user["username"]}

@app.post("/logout", status_code=204)
def logout(token: Annotated[str, Depends(oauth2_scheme)]):
    TOKENS.pop(token, None)
```

The token here is a random 256-bit value (`secrets.token_urlsafe(32)`) stored on the
server — effectively a session ID. That design is simple and easy to revoke; lesson 2
compares it with self-contained JWTs.

## The flow, request by request

```text
me, no token        -> 401 {'detail': 'Not authenticated'} {'www-authenticate': 'Bearer'}
login wrong pw      -> 401 {'detail': 'Incorrect username or password'} {'www-authenticate': 'Bearer'}
login as JSON       -> 422 {'detail': [{'type': 'missing', 'loc': ['body', 'username'], ...}, {'type': 'missing', 'loc': ['body', 'password'], ...}]}
login ok            -> 200 {'access_token': '8HfQR6eSVmWO1p2mFJHZhY_5BoIetM1XXGY0RXplkmM', 'token_type': 'bearer'}
me with token       -> 200 {'username': 'ada'}
me with Basic scheme-> 401 {'detail': 'Not authenticated'} {'www-authenticate': 'Bearer'}
logout              -> 204
me after logout     -> 401 {'detail': 'Invalid or expired token'} {'www-authenticate': 'Bearer'}
```

- With no `Authorization` header, `OAuth2PasswordBearer` itself raised the 401 — your
  dependency never ran.
- The token endpoint takes **form data**, as the OAuth2 spec requires; JSON gets a 422.
  The form model has the spec's fields: `grant_type`, `username`, `password`, `scope`,
  `client_id`, `client_secret` (from the generated OpenAPI schema).
- A header with a scheme other than `Bearer` is treated as no token at all. (The check
  in FastAPI's source is case-insensitive: `scheme.lower() != "bearer"`.)
- The wrong-password message doesn't say *which* of username or password was wrong.
  Saying "no such user" tells an attacker which usernames exist.
- Logout works instantly because the server holds the token list.

## Worked example: the timing leak

The error *message* doesn't reveal whether a username exists — but the *time taken* can.
Without the `DUMMY_HASH` line, `authenticate` returns immediately for unknown users and
spends ~40 ms hashing for known ones. Median of five login attempts each:

```text
without dummy hash:
median login time, existing user wrong pw: 39 ms
median login time, unknown user:           1 ms

with dummy hash:
median login time, existing user wrong pw: 38 ms
median login time, unknown user:           38 ms
```

A 38 ms difference is trivially measurable over a network, letting anyone test whether
`ceo@company.com` has an account. Verifying against a dummy hash makes both paths cost
the same. Rate limiting (lesson 9) is the other half of the defence: Argon2 slows each
guess, rate limits cap how many guesses there are.

## How It Actually Works

`OAuth2PasswordBearer(tokenUrl="token")` is a class with an `async __call__(request)`,
which makes an *instance* usable as a dependency. When called, it reads the
`Authorization` header, splits it into scheme and parameter, and — unless you passed
`auto_error=False` — raises a 401 with `WWW-Authenticate: Bearer` if the scheme isn't
`bearer`. It returns only the token string; deciding whether the token is valid is your
dependency's job.

It also subclasses FastAPI's `SecurityBase`, which is how OpenAPI learns about it. The
generated schema contained:

```text
{'OAuth2PasswordBearer': {'type': 'oauth2', 'flows': {'password': {'scopes': {}, 'tokenUrl': 'token'}}}}
```

Swagger UI reads that, shows the **Authorize** button, posts the form to `tokenUrl`, and
attaches the bearer token to subsequent "Try it out" requests. `tokenUrl` is relative
here; if your API is mounted under a prefix, make it match the real path.

`OAuth2PasswordRequestForm` is a plain class whose `__init__` parameters are all
`Form(...)` fields, used with `Depends()`. Nothing about it is magic — you could write
it yourself.

On the hashing side, Argon2id's cost comes from filling and repeatedly mixing a large
block of memory (64 MiB here). An attacker's GPU has many cores but limited memory per
core, so memory-hard hashing removes most of its advantage — which is the reason Argon2
is preferred over older CPU-only-cost algorithms.

## Common mistakes

- **Storing passwords encrypted** (reversible) or with a fast hash (`sha256(password)`).
- **Revealing which part was wrong** ("no such user" vs "wrong password"), or leaking it
  through timing.
- **Comparing tokens or hashes with `==` yourself.** Use the library's `verify`; for
  your own secret comparisons use `secrets.compare_digest`.
- **Logging request bodies on the token endpoint**, which writes passwords to log files.
- **Hashing in `async def` endpoints.** 40 ms of CPU blocks the event loop (Level 2
  lesson 3); keep login endpoints `def`, or offload with `run_in_threadpool`.
- **No rate limiting on `/token`.** Slow hashes don't help if there are a million
  attempts per hour.
- **Forgetting `WWW-Authenticate: Bearer`** on 401s.

## Exercise

1. Add `POST /users` (sign-up) with a password policy: at least 12 characters and not in
   a small blocklist of common passwords. Store only the hash.
2. Add `needs_rehash` handling: on successful login, check
   `password_hash.verify_and_update(...)` (read pwdlib's docs for its return value) and
   store the new hash if the parameters have changed.
3. Make tokens expire: store `(username, expires_at)` and reject expired tokens with 401.
4. Reproduce the timing measurement on your machine with and without the dummy hash.
