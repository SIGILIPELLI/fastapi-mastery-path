---
description: "JWT access tokens in FastAPI with PyJWT: claims, expiry, signature checks, the alg=none trap, and rotating refresh tokens with reuse detection."
---

# 02 · JWT Access Tokens & Refresh Tokens

Lesson 1's tokens were random strings the server looked up on every request. That's a
fine design. The other common one is a **JSON Web Token (JWT)**: a token that carries its
own claims (who the user is, when it expires) and a signature, so any server holding the
key can verify it without a database lookup. This lesson shows what's actually inside a
JWT, the checks that make it safe, and the refresh-token pattern that makes short-lived
access tokens practical.

```bash
pip install pyjwt
```

(PyJWT 2.15.1 here.)

## What a JWT is

```python
import secrets
from datetime import datetime, timedelta, timezone
import jwt

SECRET = "test-only-secret-change-me-0123456789abcdef"   # from settings in a real app
ALG = "HS256"
ACCESS_TTL = timedelta(minutes=15)

def make_token(sub: str, kind: str, ttl: timedelta, now: datetime | None = None):
    now = now or datetime.now(timezone.utc)
    jti = secrets.token_hex(8)
    payload = {"sub": sub, "type": kind, "iat": now, "exp": now + ttl,
               "jti": jti, "iss": "bookshop"}
    return jwt.encode(payload, SECRET, algorithm=ALG), jti
```

A token issued at a fixed time, split on its two dots and base64-decoded:

```text
token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJhZGEiLCJ0eXB...
header : {'alg': 'HS256', 'typ': 'JWT'}
payload: {'sub': 'ada', 'type': 'access', 'iat': 1791460800, 'exp': 1791461700, 'jti': 'ba2bfc290872d267', 'iss': 'bookshop'}
```

Three parts: **header** (algorithm), **payload** (claims), **signature** (an HMAC-SHA256
of the first two parts with the secret). PyJWT converted the `datetime`s to Unix
timestamps. The registered claims used:

| Claim | Meaning |
|-------|---------|
| `sub` | subject — who the token is about |
| `exp` | expiry time; PyJWT rejects the token after it |
| `iat` | issued-at time |
| `iss` | issuer — who created it |
| `jti` | a unique token ID, useful for revocation lists |

`type` is a custom claim distinguishing access tokens from refresh tokens.

!!! warning "JWTs are signed, not encrypted"
    The payload is base64, not ciphertext. Decoding without the key worked fine:
    `jwt.decode(token, options={"verify_signature": False})["sub"]` returned `'ada'`.
    Never put anything in a JWT you wouldn't show the user — no emails you need to keep
    private, no internal IDs you consider sensitive, certainly no passwords.

## The checks that matter

Every decode was done with `jwt.decode(token, SECRET, algorithms=["HS256"], issuer="bookshop")`.
The results:

```text
expired (issued 2026-10-08 12:00, 15 min) -> ExpiredSignatureError: Signature has expired
fresh                                    -> {'sub': 'ada', 'type': 'access', 'iat': 1791478715, 'exp': 1791479615, ...}
payload edited to sub=admin              -> InvalidSignatureError: Signature verification failed
alg=none                                 -> InvalidAlgorithmError: The specified alg value is not allowed
wrong secret                             -> InvalidSignatureError: Signature verification failed
```

- **Expiry** is checked automatically when `exp` is present.
- **Tampering** with the payload breaks the signature.
- **`alg: none`**: the JWT spec allows unsigned tokens. A token with header
  `{"alg":"none"}` and *no signature* was created; it was rejected because
  `algorithms=["HS256"]` lists the only algorithm you accept. Libraries that read the
  algorithm from the token's own header and trust it have had real vulnerabilities from
  this. **Always pass `algorithms=` explicitly.**
- **Key length**: signing with a 5-byte key was *accepted*, with only an
  `InsecureKeyLengthWarning: The HMAC key is 5 bytes long, which is below the minimum recommended length of 32 bytes for SHA256`.
  A warning is easy to miss in logs. Generate keys with `secrets.token_urlsafe(32)` or
  longer, and treat that warning as an error.

## Access tokens in FastAPI

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()
oauth2 = OAuth2PasswordBearer(tokenUrl="token")

def decode_or_401(token: str, expected_type: str) -> dict:
    try:
        claims = jwt.decode(token, SECRET, algorithms=[ALG], issuer="bookshop",
                            options={"require": ["exp", "sub", "type", "jti"]})
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token expired", headers={"WWW-Authenticate": "Bearer"})
    except jwt.PyJWTError:
        raise HTTPException(401, "Invalid token", headers={"WWW-Authenticate": "Bearer"})
    if claims["type"] != expected_type:
        raise HTTPException(401, "Wrong token type", headers={"WWW-Authenticate": "Bearer"})
    return claims

def current_user(token: Annotated[str, Depends(oauth2)]) -> str:
    return decode_or_401(token, "access")["sub"]

@app.get("/me")
def me(user: Annotated[str, Depends(current_user)]):
    return {"user": user}
```

`options={"require": [...]}` rejects tokens that lack those claims — without it, a token
with no `exp` would never expire. The `type` check stops a long-lived refresh token from
being used as an access token:

```text
me: {'user': 'ada'}
refresh used as access: {'detail': 'Wrong token type'}
```

`current_user` does no database lookup. That's the selling point of JWTs, and also their
main weakness: a token stays valid until it expires even if you disable the user, change
their role, or they log out. Short access-token lifetimes (5–15 minutes) limit the
damage; refresh tokens make short lifetimes bearable.

## Worked example: rotating refresh tokens with reuse detection

The client holds two tokens: a short-lived access token for API calls and a long-lived
refresh token used only at `/token/refresh`. Each refresh **rotates**: the old refresh
token is marked used and a new pair is issued. If a used refresh token ever comes back,
someone has a copy — so the whole chain ("family") is revoked.

```python
from fastapi import Body
from fastapi.security import OAuth2PasswordRequestForm

REFRESH_TTL = timedelta(days=7)
USERS = {"ada": "pw"}                  # toy credentials; lesson 1 shows real hashing
REFRESH_STORE: dict[str, dict] = {}    # jti -> {"sub", "used", "family"}; a DB table in practice

def issue_pair(sub: str, family: str | None = None) -> dict:
    access, _ = make_token(sub, "access", ACCESS_TTL)
    refresh, jti = make_token(sub, "refresh", REFRESH_TTL)
    REFRESH_STORE[jti] = {"sub": sub, "used": False, "family": family or jti}
    return {"access_token": access, "refresh_token": refresh, "token_type": "bearer"}

@app.post("/token")
def login(form: Annotated[OAuth2PasswordRequestForm, Depends()]):
    if USERS.get(form.username) != form.password:
        raise HTTPException(401, "Incorrect username or password",
                            headers={"WWW-Authenticate": "Bearer"})
    return issue_pair(form.username)

@app.post("/token/refresh")
def refresh(refresh_token: Annotated[str, Body(embed=True)]):
    claims = decode_or_401(refresh_token, "refresh")
    record = REFRESH_STORE.get(claims["jti"])
    if record is None:
        raise HTTPException(401, "Unknown refresh token")
    if record["used"]:
        # Reuse of a rotated token: assume theft, revoke the whole family.
        for r in REFRESH_STORE.values():
            if r["family"] == record["family"]:
                r["used"] = True
        raise HTTPException(401, "Refresh token reuse detected; please log in again")
    record["used"] = True
    return issue_pair(claims["sub"], family=record["family"])
```

The scenario: a client refreshes normally; then an attacker replays the original refresh
token they stole; then the legitimate client tries its new refresh token.

```text
refresh #1: 200 ['access_token', 'refresh_token', 'token_type']
replay old refresh: 401 {'detail': 'Refresh token reuse detected; please log in again'}
new refresh after theft detected: 401 {'detail': 'Refresh token reuse detected; please log in again'}
```

The replay was refused *and* it killed the family, so the attacker can't use any token
they obtained — at the price of making the real user log in again, which is the right
trade. Note that refresh tokens are **stateful** here: the store is what makes rotation,
logout and revocation possible. Pure stateless JWTs can't do any of those.

## Access tokens: JWT or opaque?

| | Opaque (lesson 1) | JWT |
|---|---|---|
| Verify | database/cache lookup | signature check, no lookup |
| Revoke instantly | yes | no (wait for expiry, or keep a deny-list — which is a lookup) |
| Size | ~43 characters | 221 characters for the small token here; grows with claims |
| Readable by client | no | yes |
| Verify across services | need shared store | need only the key (or public key with RS256/ES256) |

For a single application talking to its own database, opaque tokens are simpler and
fully revocable. JWTs earn their complexity when several services must verify tokens
independently — and then asymmetric algorithms (RS256, ES256) let them verify with a
public key without being able to mint tokens.

## How It Actually Works

`jwt.encode` JSON-serializes the header and payload, base64url-encodes each without
padding, joins them with a dot, and computes `HMAC-SHA256(secret, "header.payload")` —
the third part. `jwt.decode`:

1. splits the token and decodes the header;
2. checks the header's `alg` is in the `algorithms` list you passed (the `alg=none`
   rejection happens here);
3. recomputes the HMAC and compares it in constant time to the signature;
4. decodes the payload and validates registered claims: `exp` (with optional `leeway`
   for clock skew between servers), `nbf`, `iat`, and `iss`/`aud` when you ask;
5. checks the `require` list.

Only after all of that does it return the claims. Everything before step 3 is
attacker-controlled input, which is why the order matters.

## Common mistakes

- **Omitting `algorithms=`**, or reading it from the token.
- **Weak or committed secrets.** Load from settings (`SecretStr`), 32+ random bytes,
  rotate if leaked.
- **Long-lived access tokens** (days) "to avoid refresh complexity" — every stolen token
  is valid for days.
- **Not requiring `exp`.**
- **Sensitive data in claims.** It's readable.
- **Storing tokens in `localStorage`** in a browser app, where any XSS can read them.
  For browser clients, an `HttpOnly` cookie (Level 1 lesson 8) with CSRF protection is
  usually safer.
- **Checking only the signature** and not the token type, issuer or audience.

## Exercise

1. Move `SECRET`, the TTLs and the issuer into a `Settings` class (Level 2 lesson 7) with
   `SecretStr`, and inject them.
2. Store refresh tokens in a SQLAlchemy table (`jti`, `sub`, `family`, `used`,
   `expires_at`) instead of a dict. Add `POST /logout` that revokes the family.
3. Add an `aud` claim and make the API reject tokens issued for a different audience.
4. Switch to RS256: generate a key pair with the `cryptography` package, sign with the
   private key and verify with the public key. What does a service that only verifies
   tokens need to hold?
