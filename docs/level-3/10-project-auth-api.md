---
description: "Level 3 project: a multi-user bookmarks API with Argon2 sign-up, JWT access and rotating refresh tokens, scopes, ownership, rate limits and async tests."
---

# 10 · Project — A Multi-User Authenticated API

The Level 3 project is a bookmarks service where many users keep their own links and can
share some publicly. It brings together every lesson in this level: Argon2 password
storage, the OAuth2 password flow, short-lived JWT access tokens with rotating refresh
tokens stored in the database, scopes and roles, object ownership, a login rate limit,
lifespan-managed resources, async SQLAlchemy, Alembic, and an async test suite.

## Requirements

| Area | Rule |
|---|---|
| Sign-up | lower-case username 3–30 chars (`a-z0-9_`); password 12–128 chars, not on a common-password list; stored as Argon2id |
| Login | OAuth2 password form; identical errors for "no user" and "wrong password"; constant-cost check; 5 attempts burst, 5/minute per username |
| Tokens | 15-minute access JWT with scopes; 7-day refresh JWT, **rotated** on use; reuse of an old refresh token revokes the whole family; logout revokes the family |
| Bookmarks | users see and change only their own; an owner may mark a bookmark public; anyone can list a user's public bookmarks |
| Admin | needs the `admin` scope *and* an admin account; can list and disable users; disabled users lose access immediately |
| Data | async SQLAlchemy on SQLite via aiosqlite; schema via Alembic's async template; UTC timestamps |

## Layout

```text
marks/
├── alembic.ini, migrations/          (alembic init -t async)
├── app/
│   ├── config.py      settings (JWT secret required)
│   ├── models.py      User, Bookmark, RefreshToken
│   ├── schemas.py     request/response models
│   ├── db.py          get_db from lifespan state
│   ├── security.py    hashing, tokens, current_user, scopes, login limiter
│   ├── routers/       auth.py, bookmarks.py, admin.py
│   └── main.py        lifespan + app factory
└── tests/
    ├── conftest.py
    ├── test_auth.py
    └── test_bookmarks.py
```

## Configuration and models

```python
# app/config.py
from functools import lru_cache

from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="MARKS_", env_file=".env", extra="ignore")

    database_url: str = "sqlite+aiosqlite:///marks.db"
    jwt_secret: SecretStr
    access_minutes: int = 15
    refresh_days: int = 7
    login_burst: int = 5
    login_per_minute: int = 5
    cors_origins: list[str] = []


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`jwt_secret` has no default: the app refuses to start without one (Level 2 lesson 7).

```python
# app/models.py
from datetime import datetime, timezone

from sqlalchemy import Boolean, ForeignKey, MetaData, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

NAMING = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


def utcnow() -> datetime:
    return datetime.now(timezone.utc)


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING)


class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(30), unique=True)
    password_hash: Mapped[str] = mapped_column(String(200))
    is_admin: Mapped[bool] = mapped_column(Boolean, default=False)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)


class Bookmark(Base):
    __tablename__ = "bookmarks"
    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    url: Mapped[str] = mapped_column(String(2000))
    title: Mapped[str] = mapped_column(String(200))
    is_public: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(default=utcnow)


class RefreshToken(Base):
    __tablename__ = "refresh_tokens"
    jti: Mapped[str] = mapped_column(String(32), primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    family: Mapped[str] = mapped_column(String(32), index=True)
    used: Mapped[bool] = mapped_column(Boolean, default=False)
```

The refresh-token table stores only the token's `jti`, its owner, its **family** (the
`jti` of the first token in the chain) and whether it has been used — never the token
itself.

## Schemas

```python
# app/schemas.py
from datetime import datetime, timezone
from typing import Annotated

from pydantic import AfterValidator, BaseModel, ConfigDict, Field, HttpUrl, field_validator


def _assume_utc(v: datetime) -> datetime:
    return v.replace(tzinfo=timezone.utc) if v.tzinfo is None else v


UTCDateTime = Annotated[datetime, AfterValidator(_assume_utc)]
COMMON_PASSWORDS = {"password1234", "qwertyuiop12", "123456789012", "iloveyou1234"}


class Strict(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)


class Register(Strict):
    username: str = Field(min_length=3, max_length=30, pattern=r"^[a-z0-9_]+$")
    password: str = Field(min_length=12, max_length=128)

    @field_validator("password")
    @classmethod
    def not_common(cls, v: str) -> str:
        if v.lower() in COMMON_PASSWORDS:
            raise ValueError("password is too common")
        return v


class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    username: str
    is_admin: bool


class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class BookmarkIn(Strict):
    url: HttpUrl
    title: str = Field(min_length=1, max_length=200)
    is_public: bool = False


class BookmarkPatch(Strict):
    title: str | None = Field(None, min_length=1, max_length=200)
    is_public: bool | None = None


class BookmarkOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    url: str
    title: str
    is_public: bool
    created_at: UTCDateTime
```

`HttpUrl` rejects non-HTTP schemes such as `javascript:` (tested below) — important for
links that will be rendered in someone's browser. `extra="forbid"` stops a client from
smuggling in `owner_id`.

## Security

```python
# app/db.py
from collections.abc import AsyncIterator

from fastapi import Request
from sqlalchemy.ext.asyncio import AsyncSession


async def get_db(request: Request) -> AsyncIterator[AsyncSession]:
    async with request.state.sessionmaker() as session:
        yield session
```

```python
# app/security.py
import math
import secrets
import time
from datetime import datetime, timedelta, timezone
from typing import Annotated

import jwt
from fastapi import Depends, HTTPException, Security, status
from fastapi.security import OAuth2PasswordBearer, SecurityScopes
from pwdlib import PasswordHash
from sqlalchemy.ext.asyncio import AsyncSession

from app.config import Settings, get_settings
from app.db import get_db
from app.models import User

ALG = "HS256"
password_hash = PasswordHash.recommended()
DUMMY_HASH = password_hash.hash("dummy-password-for-timing-only")
SCOPES = {"bookmarks": "Manage your bookmarks", "admin": "Administer users"}
oauth2 = OAuth2PasswordBearer(tokenUrl="auth/token", scopes=SCOPES)

DB = Annotated[AsyncSession, Depends(get_db)]
SettingsDep = Annotated[Settings, Depends(get_settings)]


def _unauthorized(detail: str, scopes: SecurityScopes | None = None) -> HTTPException:
    value = f'Bearer scope="{scopes.scope_str}"' if scopes and scopes.scopes else "Bearer"
    return HTTPException(status.HTTP_401_UNAUTHORIZED, detail, headers={"WWW-Authenticate": value})


def make_token(settings: Settings, user: User, kind: str, scopes: list[str] | None = None,
               family: str | None = None) -> tuple[str, str]:
    now = datetime.now(timezone.utc)
    ttl = timedelta(minutes=settings.access_minutes) if kind == "access" else timedelta(days=settings.refresh_days)
    jti = secrets.token_hex(16)
    claims = {"sub": str(user.id), "type": kind, "iat": now, "exp": now + ttl, "jti": jti}
    if scopes is not None:
        claims["scope"] = " ".join(scopes)
    if family is not None:
        claims["fam"] = family
    return jwt.encode(claims, settings.jwt_secret.get_secret_value(), algorithm=ALG), jti


def decode(settings: Settings, token: str, kind: str) -> dict:
    try:
        claims = jwt.decode(token, settings.jwt_secret.get_secret_value(), algorithms=[ALG],
                            options={"require": ["exp", "sub", "type", "jti"]})
    except jwt.ExpiredSignatureError:
        raise _unauthorized("Token expired") from None
    except jwt.PyJWTError:
        raise _unauthorized("Invalid token") from None
    if claims["type"] != kind:
        raise _unauthorized("Wrong token type")
    return claims


async def current_user(security_scopes: SecurityScopes, token: Annotated[str, Depends(oauth2)],
                       db: DB, settings: SettingsDep) -> User:
    claims = decode(settings, token, "access")
    granted = claims.get("scope", "").split()
    for needed in security_scopes.scopes:
        if needed not in granted:
            raise HTTPException(status.HTTP_403_FORBIDDEN, f"Missing scope: {needed}")
    user = await db.get(User, int(claims["sub"]))
    if user is None or not user.is_active:
        raise _unauthorized("Account not found or disabled", security_scopes)
    return user


Member = Annotated[User, Security(current_user, scopes=["bookmarks"])]
Admin = Annotated[User, Security(current_user, scopes=["admin"])]


class LoginLimiter:
    """Token bucket per username. In-process: use a shared store with several workers."""

    def __init__(self) -> None:
        self.state: dict[str, tuple[float, float]] = {}

    def check(self, key: str, burst: int, per_minute: int, now: float | None = None) -> None:
        now = time.monotonic() if now is None else now
        rate = per_minute / 60
        tokens, last = self.state.get(key, (float(burst), now))
        tokens = min(burst, tokens + (now - last) * rate)
        if tokens < 1:
            self.state[key] = (tokens, now)
            raise HTTPException(status.HTTP_429_TOO_MANY_REQUESTS, "Too many login attempts",
                                headers={"Retry-After": str(math.ceil((1 - tokens) / rate))})
        self.state[key] = (tokens - 1, now)


login_limiter = LoginLimiter()
```

Design notes:

- `current_user` loads the user on every request, so **disabling an account takes
  effect immediately**, even though JWTs are self-contained. That one primary-key lookup
  buys the revocability that pure JWT designs lack.
- `Member` and `Admin` are `Annotated` aliases with the scope requirement baked in.
- Hashing and verifying are run with `run_in_threadpool` from the `async def` endpoints:
  ~40 ms of Argon2 must not block the event loop (Level 2 lesson 3).
- The login limiter is in-process, as the docstring warns.

## Routers

```python
# app/routers/auth.py
from typing import Annotated

from fastapi import APIRouter, Body, Depends, HTTPException, status
from fastapi.concurrency import run_in_threadpool
from fastapi.security import OAuth2PasswordRequestForm
from sqlalchemy import select, update
from sqlalchemy.exc import IntegrityError

from app.models import RefreshToken, User
from app.schemas import Register, TokenPair, UserOut
from app.security import (DB, DUMMY_HASH, Member, SettingsDep, _unauthorized, decode,
                          login_limiter, make_token, password_hash)

router = APIRouter(prefix="/auth", tags=["auth"])


async def _issue(db, settings, user: User, scopes: list[str], family: str | None = None) -> TokenPair:
    access, _ = make_token(settings, user, "access", scopes=scopes)
    refresh, jti = make_token(settings, user, "refresh", scopes=scopes, family=family)
    db.add(RefreshToken(jti=jti, user_id=user.id, family=family or jti))
    await db.commit()
    return TokenPair(access_token=access, refresh_token=refresh)


@router.post("/register", response_model=UserOut, status_code=status.HTTP_201_CREATED)
async def register(data: Register, db: DB):
    hashed = await run_in_threadpool(password_hash.hash, data.password)
    user = User(username=data.username, password_hash=hashed)
    db.add(user)
    try:
        await db.commit()
    except IntegrityError:
        await db.rollback()
        raise HTTPException(status.HTTP_409_CONFLICT, "Username taken") from None
    return user


@router.post("/token", response_model=TokenPair)
async def token(form: Annotated[OAuth2PasswordRequestForm, Depends()], db: DB, settings: SettingsDep):
    login_limiter.check(form.username.lower(), settings.login_burst, settings.login_per_minute)
    user = await db.scalar(select(User).where(User.username == form.username.lower()))
    stored = user.password_hash if user else DUMMY_HASH
    ok = await run_in_threadpool(password_hash.verify, form.password, stored)
    if user is None or not ok or not user.is_active:
        raise _unauthorized("Incorrect username or password")
    allowed = ["bookmarks"] + (["admin"] if user.is_admin else [])
    scopes = [s for s in form.scopes if s in allowed] if form.scopes else allowed
    return await _issue(db, settings, user, scopes)


@router.post("/refresh", response_model=TokenPair)
async def refresh(refresh_token: Annotated[str, Body(embed=True)], db: DB, settings: SettingsDep):
    claims = decode(settings, refresh_token, "refresh")
    record = await db.get(RefreshToken, claims["jti"])
    if record is None:
        raise _unauthorized("Unknown refresh token")
    if record.used:
        await db.execute(update(RefreshToken).where(RefreshToken.family == record.family)
                         .values(used=True))
        await db.commit()
        raise _unauthorized("Refresh token reuse detected; log in again")
    record.used = True
    user = await db.get(User, record.user_id)
    if user is None or not user.is_active:
        await db.commit()
        raise _unauthorized("Account not found or disabled")
    return await _issue(db, settings, user, claims.get("scope", "").split(), family=record.family)


@router.post("/logout", status_code=status.HTTP_204_NO_CONTENT)
async def logout(refresh_token: Annotated[str, Body(embed=True)], user: Member, db: DB,
                 settings: SettingsDep):
    claims = decode(settings, refresh_token, "refresh")
    record = await db.get(RefreshToken, claims["jti"])
    if record is not None and record.user_id == user.id:
        await db.execute(update(RefreshToken).where(RefreshToken.family == record.family)
                         .values(used=True))
        await db.commit()


@router.get("/me", response_model=UserOut)
async def me(user: Member):
    return user
```

```python
# app/routers/bookmarks.py
from typing import Annotated

from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy import select

from app.models import Bookmark, User
from app.schemas import BookmarkIn, BookmarkOut, BookmarkPatch
from app.security import DB, Member

router = APIRouter(tags=["bookmarks"])


async def owned_bookmark(bookmark_id: int, user: Member, db: DB) -> Bookmark:
    bm = await db.scalar(select(Bookmark).where(Bookmark.id == bookmark_id,
                                                Bookmark.owner_id == user.id))
    if bm is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Bookmark not found")
    return bm


Owned = Annotated[Bookmark, Depends(owned_bookmark)]


@router.get("/bookmarks", response_model=list[BookmarkOut])
async def my_bookmarks(user: Member, db: DB,
                       limit: Annotated[int, Query(ge=1, le=100)] = 20,
                       offset: Annotated[int, Query(ge=0)] = 0):
    stmt = (select(Bookmark).where(Bookmark.owner_id == user.id)
            .order_by(Bookmark.created_at.desc(), Bookmark.id.desc()).limit(limit).offset(offset))
    return (await db.scalars(stmt)).all()


@router.post("/bookmarks", response_model=BookmarkOut, status_code=status.HTTP_201_CREATED)
async def create_bookmark(data: BookmarkIn, user: Member, db: DB):
    bm = Bookmark(owner_id=user.id, url=str(data.url), title=data.title, is_public=data.is_public)
    db.add(bm)
    await db.commit()
    return bm


@router.get("/bookmarks/{bookmark_id}", response_model=BookmarkOut)
async def get_bookmark(bm: Owned):
    return bm


@router.patch("/bookmarks/{bookmark_id}", response_model=BookmarkOut)
async def update_bookmark(changes: BookmarkPatch, bm: Owned, db: DB):
    for field, value in changes.model_dump(exclude_unset=True, exclude_none=True).items():
        setattr(bm, field, value)
    await db.commit()
    return bm


@router.delete("/bookmarks/{bookmark_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_bookmark(bm: Owned, db: DB):
    await db.delete(bm)
    await db.commit()


@router.get("/users/{username}/bookmarks", response_model=list[BookmarkOut])
async def public_bookmarks(username: str, db: DB):
    stmt = (select(Bookmark).join(User, User.id == Bookmark.owner_id)
            .where(User.username == username, Bookmark.is_public.is_(True))
            .order_by(Bookmark.created_at.desc(), Bookmark.id.desc()).limit(100))
    return (await db.scalars(stmt)).all()
```

`owned_bookmark` puts the owner into the `WHERE` clause (lesson 3's strongest form), and
every single-bookmark endpoint reuses it. The public listing filters on `is_public` in
SQL and caps the result at 100.

```python
# app/routers/admin.py
from fastapi import APIRouter, HTTPException, status
from sqlalchemy import select

from app.models import User
from app.schemas import UserOut
from app.security import DB, Admin

router = APIRouter(prefix="/admin", tags=["admin"])


@router.get("/users", response_model=list[UserOut])
async def list_users(admin: Admin, db: DB):
    if not admin.is_admin:            # scope says "admin token"; this says "admin person"
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Admins only")
    return (await db.scalars(select(User).order_by(User.id))).all()


@router.post("/users/{user_id}/disable", status_code=status.HTTP_204_NO_CONTENT)
async def disable_user(user_id: int, admin: Admin, db: DB):
    if not admin.is_admin:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "Admins only")
    user = await db.get(User, user_id)
    if user is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "User not found")
    user.is_active = False
    await db.commit()
```

```python
# app/main.py
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from app.config import get_settings
from app.routers import admin, auth, bookmarks


@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = create_async_engine(get_settings().database_url)
    yield {"sessionmaker": async_sessionmaker(engine, expire_on_commit=False)}
    await engine.dispose()


def create_app() -> FastAPI:
    settings = get_settings()
    app = FastAPI(title="Bookmarks API", version="1.0.0", lifespan=lifespan)
    app.include_router(auth.router)
    app.include_router(bookmarks.router)
    app.include_router(admin.router)
    if settings.cors_origins:
        app.add_middleware(CORSMiddleware, allow_origins=settings.cors_origins,
                           allow_credentials=False, allow_methods=["*"],
                           allow_headers=["Authorization", "Content-Type"])
    return app


app = create_app()
```

## Migrations

Created with the async template (`alembic init -t async migrations`). In
`migrations/env.py`, `target_metadata = None` was replaced with:

```python
from app.config import get_settings
from app.models import Base

config.set_main_option("sqlalchemy.url", get_settings().database_url)
target_metadata = Base.metadata
```

and `render_as_batch=True` was added to `context.configure(...)` in
`do_run_migrations`. With `MARKS_JWT_SECRET` set (settings validation runs even for
migrations):

```text
$ alembic revision --autogenerate -m "users bookmarks refresh tokens"
INFO  [alembic.autogenerate.compare.tables] Detected added table 'users'
INFO  [alembic.autogenerate.compare.tables] Detected added table 'bookmarks'
INFO  [alembic.autogenerate.compare.tables] Detected added table 'refresh_tokens'
Generating .../versions/47047566546b_users_bookmarks_refresh_tokens.py ...  done
$ alembic upgrade head
INFO  [alembic.runtime.migration] Running upgrade  -> 47047566546b, users bookmarks refresh tokens
```

## Running it

```bash
export MARKS_JWT_SECRET=$(python -c "import secrets; print(secrets.token_urlsafe(48))")
alembic upgrade head
fastapi run app/main.py --port 8712
```

A session with `curl` (tokens truncated):

```text
POST /auth/register  {"username":"ada","password":"a long passphrase"}
{"id":1,"username":"ada","is_admin":false}

POST /auth/token  username=ada&password=a long passphrase
{'access_token': 'eyJhbGciOiJIUzI1NiIs...', 'refresh_token': 'eyJhbGciOiJIUzI1NiIs...', 'token_type': 'bearer'}

POST /bookmarks  (Bearer)  {"url":"https://www.python.org","title":"Python","is_public":true}
{"id":1,"url":"https://www.python.org/","title":"Python","is_public":true,"created_at":"2026-10-08T17:18:04.302793Z"}

GET /users/ada/bookmarks
[{"id":1,"url":"https://www.python.org/","title":"Python","is_public":true,"created_at":"2026-10-08T17:18:04.302793Z"}]

GET /bookmarks  (no token)
401
```

The server log had no tracebacks.

## Tests

```python
# tests/conftest.py
import os

os.environ["MARKS_JWT_SECRET"] = "test-secret-" + "x" * 40   # before importing the app

import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy import event, update
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.pool import StaticPool

from app import security
from app.db import get_db
from app.main import app
from app.models import Base, User


@pytest.fixture(scope="session")
def anyio_backend():
    return "asyncio"


@pytest.fixture(scope="session")
async def engine():
    engine = create_async_engine("sqlite+aiosqlite://", poolclass=StaticPool)

    @event.listens_for(engine.sync_engine, "connect")
    def _no_pysqlite_tx(dbapi_connection, record):
        dbapi_connection.isolation_level = None

    @event.listens_for(engine.sync_engine, "begin")
    def _emit_begin(conn):
        conn.exec_driver_sql("BEGIN")

    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest.fixture
async def db(engine):
    async with engine.connect() as conn:
        outer = await conn.begin()
        session = AsyncSession(bind=conn, join_transaction_mode="create_savepoint",
                               expire_on_commit=False)
        try:
            yield session
        finally:
            await session.close()
            await outer.rollback()


@pytest.fixture
async def client(db):
    async def override_get_db():
        yield db
    app.dependency_overrides[get_db] = override_get_db
    security.login_limiter.state.clear()
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()


async def register_and_login(client, username, password="a long passphrase", scope=None):
    r = await client.post("/auth/register", json={"username": username, "password": password})
    assert r.status_code == 201, r.text
    data = {"username": username, "password": password}
    if scope:
        data["scope"] = scope
    r = await client.post("/auth/token", data=data)
    assert r.status_code == 200, r.text
    return r.json()


def bearer(tokens):
    return {"Authorization": f"Bearer {tokens['access_token']}"}


@pytest.fixture
async def ada(client):
    return await register_and_login(client, "ada")


@pytest.fixture
async def bob(client):
    return await register_and_login(client, "bob")


@pytest.fixture
async def admin(client, db):
    await client.post("/auth/register", json={"username": "root", "password": "a long passphrase"})
    await db.execute(update(User).where(User.username == "root").values(is_admin=True))
    await db.flush()
    r = await client.post("/auth/token", data={"username": "root", "password": "a long passphrase"})
    return r.json()
```

The secret is set in the environment **before** importing the app, because
`create_app()` reads settings at import time. The limiter's state is cleared per test so
tests don't throttle each other.

```python
# tests/test_auth.py
import pytest

from tests.conftest import bearer, register_and_login

pytestmark = pytest.mark.anyio


async def test_register_validates_and_hashes(client, db):
    from app.models import User
    r = await client.post("/auth/register", json={"username": "Ada!", "password": "short"})
    assert r.status_code == 422
    fields = {e["loc"][-1] for e in r.json()["detail"]}
    assert fields == {"username", "password"}
    r = await client.post("/auth/register", json={"username": "ada", "password": "password1234"})
    assert r.status_code == 422 and "too common" in r.json()["detail"][0]["msg"]
    r = await client.post("/auth/register", json={"username": "ada", "password": "a long passphrase"})
    assert r.status_code == 201 and r.json() == {"id": 1, "username": "ada", "is_admin": False}
    stored = (await db.get(User, 1)).password_hash
    assert stored.startswith("$argon2id$") and "passphrase" not in stored
    r = await client.post("/auth/register", json={"username": "ada", "password": "another passphrase"})
    assert r.status_code == 409


async def test_login_errors_are_identical(client, ada):
    wrong_pw = await client.post("/auth/token", data={"username": "ada", "password": "nope"})
    no_user = await client.post("/auth/token", data={"username": "zed", "password": "nope"})
    assert wrong_pw.status_code == no_user.status_code == 401
    assert wrong_pw.json() == no_user.json()


async def test_me_requires_token(client, ada):
    assert (await client.get("/auth/me")).status_code == 401
    r = await client.get("/auth/me", headers=bearer(ada))
    assert r.json()["username"] == "ada"


async def test_refresh_rotation_and_reuse_detection(client, ada):
    r1 = await client.post("/auth/refresh", json={"refresh_token": ada["refresh_token"]})
    assert r1.status_code == 200
    replay = await client.post("/auth/refresh", json={"refresh_token": ada["refresh_token"]})
    assert replay.status_code == 401 and "reuse" in replay.json()["detail"]
    after = await client.post("/auth/refresh", json={"refresh_token": r1.json()["refresh_token"]})
    assert after.status_code == 401


async def test_logout_revokes_refresh(client, ada):
    r = await client.post("/auth/logout", json={"refresh_token": ada["refresh_token"]}, headers=bearer(ada))
    assert r.status_code == 204
    r = await client.post("/auth/refresh", json={"refresh_token": ada["refresh_token"]})
    assert r.status_code == 401


async def test_login_rate_limited(client, ada):
    codes = [(await client.post("/auth/token", data={"username": "ada", "password": "bad"})).status_code
             for _ in range(6)]
    # ada's fixture used 1 of 5 tokens; 4 more 401s, then 429s.
    assert codes == [401, 401, 401, 401, 429, 429]


async def test_disabled_user_loses_access(client, ada, admin):
    me = (await client.get("/auth/me", headers=bearer(ada))).json()
    assert (await client.post(f"/admin/users/{me['id']}/disable", headers=bearer(admin))).status_code == 204
    assert (await client.get("/auth/me", headers=bearer(ada))).status_code == 401
    r = await client.post("/auth/refresh", json={"refresh_token": ada["refresh_token"]})
    assert r.status_code == 401
```

```python
# tests/test_bookmarks.py
import pytest

from tests.conftest import bearer, register_and_login

pytestmark = pytest.mark.anyio


async def make(client, who, **kw):
    payload = {"url": "https://example.com/a", "title": "A", **kw}
    r = await client.post("/bookmarks", json=payload, headers=bearer(who))
    assert r.status_code == 201, r.text
    return r.json()


async def test_crud_own_bookmark(client, ada):
    bm = await make(client, ada, title="FastAPI docs", url="https://fastapi.tiangolo.com")
    assert bm["url"] == "https://fastapi.tiangolo.com/" and bm["created_at"].endswith("Z")
    r = await client.patch(f"/bookmarks/{bm['id']}", json={"title": "Docs"}, headers=bearer(ada))
    assert r.json()["title"] == "Docs" and r.json()["url"] == bm["url"]
    assert (await client.delete(f"/bookmarks/{bm['id']}", headers=bearer(ada))).status_code == 204
    assert (await client.get(f"/bookmarks/{bm['id']}", headers=bearer(ada))).status_code == 404


async def test_cannot_touch_other_users_bookmarks(client, ada, bob):
    bm = await make(client, ada)
    for method in ("GET", "PATCH", "DELETE"):
        kw = {"json": {"title": "pwned"}} if method == "PATCH" else {}
        r = await client.request(method, f"/bookmarks/{bm['id']}", headers=bearer(bob), **kw)
        assert r.status_code == 404, method
    assert (await client.get("/bookmarks", headers=bearer(bob))).json() == []


async def test_public_listing_shows_only_public(client, ada):
    await make(client, ada, title="private")
    await make(client, ada, title="shared", is_public=True)
    r = await client.get("/users/ada/bookmarks")
    assert [b["title"] for b in r.json()] == ["shared"]


async def test_rejects_bad_urls_and_extra_fields(client, ada):
    for payload in ({"url": "javascript:alert(1)", "title": "x"},
                    {"url": "https://ok.example", "title": "x", "owner_id": 2}):
        r = await client.post("/bookmarks", json=payload, headers=bearer(ada))
        assert r.status_code == 422


async def test_scopes(client, ada, admin):
    assert (await client.get("/admin/users", headers=bearer(ada))).status_code == 403
    r = await client.get("/admin/users", headers=bearer(admin))
    assert [u["username"] for u in r.json()] == ["ada", "root"]
    readonly_admin = await client.post("/auth/token", data={"username": "root", "password": "a long passphrase",
                                                            "scope": "bookmarks"})
    assert (await client.get("/admin/users", headers=bearer(readonly_admin.json()))).status_code == 403
```

```text
$ python -m pytest -q
............                                                             [100%]
12 passed in 1.53s
```

Most of the 1.5 s is Argon2 doing its job — every registration and login hashes or
verifies a password. That's an argument for keeping the number of logins in tests small
(fixtures that log in once) rather than for weakening the hash in tests; if you do lower
the cost for tests, do it through settings so production can't inherit it.

## Worked example: what the tests prove

Three tests encode security properties that are easy to break during refactoring:

- **`test_cannot_touch_other_users_bookmarks`** sends `GET`, `PATCH` and `DELETE` for
  Ada's bookmark as Bob and requires a 404 for each. If someone later writes a new
  endpoint that loads bookmarks by ID without `owned_bookmark`, the equivalent test for
  that endpoint is the one to copy.
- **`test_refresh_rotation_and_reuse_detection`** replays an old refresh token and checks
  that the *new* one stops working too.
- **`test_disabled_user_loses_access`** proves an admin action takes effect on an
  already-issued access token *and* on the refresh token.

`test_login_rate_limited` documents the budget exactly: the `ada` fixture's login used
one of five tokens, so four more failures get 401 and the next ones get 429.

## How It Actually Works

Follow `GET /bookmarks/7` with Bob's token:

1. `oauth2` extracts the bearer token (401 if missing).
2. `current_user` gets `SecurityScopes(["bookmarks"])` from the `Member` alias, decodes
   and verifies the JWT, checks the scope, and loads Bob by ID from the request's session.
3. `owned_bookmark` runs `SELECT ... WHERE id = 7 AND owner_id = <bob>`. No row: 404.
   Ada's bookmark never leaves the database.
4. Because `current_user` is a dependency of both `owned_bookmark` and (via `Member`)
   anything else in the request, the per-request cache means the user is loaded once.

And a refresh: `decode` verifies the refresh JWT; the `RefreshToken` row is fetched by
`jti`; if it's already used, one `UPDATE refresh_tokens SET used = 1 WHERE family = ?`
revokes the chain; otherwise the row is marked used and a new pair is issued with the
same `family`, all committed in one transaction.

## What's still missing for production

Honest gaps, covered in Level 4: the rate limiter and any caching are per process; there
are no structured logs or request IDs; no `/health` endpoint; no deployment
configuration; no protection against very large request bodies; refresh tokens are never
cleaned up (add a periodic delete of expired rows); and SQLite is fine for a demo but
not for concurrent writers at scale.

## Exercise

1. Add `PUT /auth/password` (requires the current password) that changes the hash and
   revokes every refresh-token family for the user. Test that old refresh tokens fail.
2. Add tags to bookmarks (many-to-many) with `selectinload`, and a `?tag=` filter on
   `GET /bookmarks`. Count queries in a test.
3. Add an `ETag` and `If-Match` to `PATCH /bookmarks/{id}` (lesson 9) using an integer
   `version` column, with an Alembic migration.
4. Write a test that loops bookmark IDs 1–50 as a user who owns none and asserts every
   response is 404.
