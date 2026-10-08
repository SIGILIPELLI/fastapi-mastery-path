---
description: "FastAPI capstone: an event-booking API that never overbooks under concurrent load, with JWT scopes, problem details, request IDs, migrations and tests."
---

# 10 · Capstone — A Production-Ready Event Booking API

The capstone is a ticket-booking service: organisers create events with a fixed number of
seats; attendees book and cancel seats. It's a small domain with one hard requirement
that most of this course has been building toward:

> **Never sell more seats than exist** — not under concurrent requests, not with several
> worker processes, not when a client retries.

Around that core it applies the production practices from Level 4: validated settings
that refuse unsafe production configs, one error contract, request IDs and JSON logs,
body limits, health checks, token verification with scopes, migrations, and tests that
include a real concurrency test — then a load test against four worker processes.

## Requirements

| Area | Rule |
|---|---|
| Events | organisers (scope `events:write`) create events with a timezone-aware start time and a capacity; anyone can read them |
| Bookings | attendees (scope `bookings`) book 1–10 seats; sold out → 409; event started → 409; cancel returns seats; cancel is idempotent |
| Retries | an `Idempotency-Key` header makes a booking request safe to retry: same key, same booking |
| Auth | tokens come from an identity provider; the API verifies signature, expiry, issuer and audience — it never handles passwords |
| Errors | every error is `application/problem+json` with a `request_id` |
| Ops | JSON access logs; `X-Request-ID` on every response; 16 KB body limit; `/health/live` and `/health/ready`; docs off in prod |
| Config | the app refuses to start in prod with a short secret, SQLite, or wildcard CORS |

## Layout

```text
booking/
├── alembic.ini, migrations/        (alembic init -t async)
├── mint_token.py                   development helper: issue test tokens
├── app/
│   ├── config.py        settings + production guards
│   ├── models.py        Event, Booking (with CHECK and UNIQUE constraints)
│   ├── schemas.py       I/O models, timezone handling
│   ├── errors.py        domain errors + problem-details handlers
│   ├── observability.py JSON logs, request IDs, body limit
│   ├── db.py            session dependency (from lifespan state)
│   ├── auth.py          token verification + scopes
│   ├── routers/         events.py, bookings.py, health.py
│   └── main.py          app factory
└── tests/               conftest.py, test_events.py, test_bookings.py
```

## Configuration

```python
# app/config.py
from functools import lru_cache
from typing import Literal

from pydantic import SecretStr, field_validator, model_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="BOOKING_", env_file=".env", extra="ignore",
                                      hide_input_in_errors=True)

    environment: Literal["dev", "test", "prod"] = "dev"
    database_url: str = "sqlite+aiosqlite:///booking.db"
    jwt_secret: SecretStr
    jwt_issuer: str = "https://id.example.com"
    jwt_audience: str = "booking-api"
    max_body_bytes: int = 16_384
    max_seats_per_booking: int = 10
    cors_origins: list[str] = []
    docs_enabled: bool = True

    @field_validator("cors_origins")
    @classmethod
    def no_wildcards_or_slashes(cls, v: list[str]) -> list[str]:
        for origin in v:
            if origin == "*" or origin.endswith("/"):
                raise ValueError(f"invalid CORS origin {origin!r}")
        return v

    @model_validator(mode="after")
    def production_rules(self):
        if self.environment == "prod":
            if len(self.jwt_secret.get_secret_value()) < 32:
                raise ValueError("jwt_secret must be at least 32 characters in prod")
            if self.database_url.startswith("sqlite"):
                raise ValueError("use a server database in prod")
        return self


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

The model validator turns operational rules into startup failures. With
`BOOKING_ENVIRONMENT=prod` and a short secret:

```text
Value error, jwt_secret must be at least 32 characters in prod [type=value_error]
```

With a long secret but the default SQLite URL: `Value error, use a server database in prod`.
With `BOOKING_CORS_ORIGINS='["*"]'`: `Value error, invalid CORS origin '*'`.

`hide_input_in_errors=True` was added after the first run of that check printed
`input_value={'environment': 'prod', 'jwt_secret': 'short'}` — the **secret itself** in
the error message, which would have landed in the deployment logs. Pydantic includes the
offending input in validation errors by default; for settings, turn that off.

## Models and schemas

```python
# app/models.py
from datetime import datetime, timezone

from sqlalchemy import CheckConstraint, ForeignKey, MetaData, String, UniqueConstraint
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


class Event(Base):
    __tablename__ = "events"
    __table_args__ = (
        CheckConstraint("seats_left >= 0", name="seats_left_nonnegative"),
        CheckConstraint("seats_left <= capacity", name="seats_left_le_capacity"),
    )
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    starts_at: Mapped[datetime]
    capacity: Mapped[int]
    seats_left: Mapped[int]
    organizer: Mapped[str] = mapped_column(String(100))
    version: Mapped[int] = mapped_column(default=1)


class Booking(Base):
    __tablename__ = "bookings"
    __table_args__ = (
        UniqueConstraint("user_id", "idempotency_key", name="uq_bookings_user_key"),
        CheckConstraint("seats > 0", name="seats_positive"),
    )
    id: Mapped[int] = mapped_column(primary_key=True)
    event_id: Mapped[int] = mapped_column(ForeignKey("events.id"), index=True)
    user_id: Mapped[str] = mapped_column(String(100), index=True)
    seats: Mapped[int]
    status: Mapped[str] = mapped_column(String(10), default="confirmed")   # confirmed|cancelled
    idempotency_key: Mapped[str | None] = mapped_column(String(64))
    created_at: Mapped[datetime] = mapped_column(default=utcnow)
```

The `CHECK` constraints are the database's own guarantee: even a bug that bypassed the
booking logic couldn't drive `seats_left` below zero. `UNIQUE(user_id, idempotency_key)`
makes idempotency safe under races.

```python
# app/schemas.py
from datetime import datetime, timezone
from typing import Annotated, Literal

from pydantic import AfterValidator, BaseModel, ConfigDict, Field


def _utc(v: datetime) -> datetime:
    return v.replace(tzinfo=timezone.utc) if v.tzinfo is None else v.astimezone(timezone.utc)


def _aware(v: datetime) -> datetime:
    if v.tzinfo is None:
        raise ValueError("datetime must include a timezone offset")
    return v.astimezone(timezone.utc)


UTCDateTime = Annotated[datetime, AfterValidator(_utc)]
AwareDateTime = Annotated[datetime, AfterValidator(_aware)]


class Strict(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)


class EventIn(Strict):
    title: str = Field(min_length=1, max_length=200)
    starts_at: AwareDateTime
    capacity: int = Field(ge=1, le=100_000)


class EventOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    starts_at: UTCDateTime
    capacity: int
    seats_left: int


class BookingIn(Strict):
    seats: int = Field(ge=1)


class BookingOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    event_id: int
    seats: int
    status: Literal["confirmed", "cancelled"]
    created_at: UTCDateTime
```

`starts_at` must include an offset on input (a naive `2030-01-01T10:00:00` is ambiguous —
whose 10 o'clock?) and is normalised to UTC; output gets the UTC marker back after the
SQLite round trip (Level 2 project).

## Errors and observability

```python
# app/errors.py
"""One error shape for the whole API: RFC 9457 problem details."""
import logging
from http import HTTPStatus

from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

log = logging.getLogger("booking")
PROBLEM = "application/problem+json"


class DomainError(Exception):
    status = 409
    type = "about:blank"

    def __init__(self, detail: str):
        self.detail = detail


class SoldOut(DomainError):
    type = "https://errors.example.com/sold-out"


class EventStarted(DomainError):
    type = "https://errors.example.com/event-started"


class NotFound(DomainError):
    status = 404


def problem(request: Request, status: int, detail: str | None = None, type_: str = "about:blank",
            headers: dict | None = None, **extra) -> JSONResponse:
    body = {"type": type_, "title": HTTPStatus(status).phrase, "status": status,
            "instance": request.url.path, "request_id": getattr(request.state, "request_id", None)}
    if detail:
        body["detail"] = detail
    body.update(extra)
    return JSONResponse(body, status_code=status, media_type=PROBLEM, headers=headers)


def install(app: FastAPI) -> None:
    @app.exception_handler(DomainError)
    async def domain(request: Request, exc: DomainError):
        return problem(request, exc.status, exc.detail, exc.type)

    @app.exception_handler(StarletteHTTPException)
    async def http(request: Request, exc: StarletteHTTPException):
        return problem(request, exc.status_code, str(exc.detail), headers=exc.headers)

    @app.exception_handler(RequestValidationError)
    async def validation(request: Request, exc: RequestValidationError):
        errors = [{"field": ".".join(str(p) for p in e["loc"][1:]), "message": e["msg"], "type": e["type"]}
                  for e in exc.errors()]
        return problem(request, 422, "Request validation failed",
                       "https://errors.example.com/validation", errors=errors)

    @app.exception_handler(Exception)
    async def unhandled(request: Request, exc: Exception):
        log.error("unhandled error", exc_info=exc,
                  extra={"request_id": getattr(request.state, "request_id", None)})
        return problem(request, 500, "Internal error")
```

```python
# app/observability.py
import json
import logging
import re
import sys
import time
import uuid
from contextvars import ContextVar

request_id_var: ContextVar[str] = ContextVar("request_id", default="-")
SAFE_ID = re.compile(r"^[A-Za-z0-9-]{1,64}$")


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        entry = {"ts": round(record.created, 3), "level": record.levelname, "logger": record.name,
                 "msg": record.getMessage(),
                 "request_id": getattr(record, "request_id", None) or request_id_var.get()}
        for key in ("method", "route", "status", "duration_ms", "user"):
            if hasattr(record, key):
                entry[key] = getattr(record, key)
        if record.exc_info:
            entry["exc"] = self.formatException(record.exc_info).splitlines()[-1]
        return json.dumps(entry)


def configure_logging() -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    logging.basicConfig(level=logging.INFO, handlers=[handler], force=True)


class RequestContext:
    """Request ID + access log, as a pure ASGI middleware."""

    def __init__(self, app):
        self.app = app
        self.log = logging.getLogger("booking.access")

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        incoming = dict(scope["headers"]).get(b"x-request-id", b"").decode("latin-1")
        rid = incoming if SAFE_ID.match(incoming) else uuid.uuid4().hex
        scope.setdefault("state", {})["request_id"] = rid
        token = request_id_var.set(rid)
        start, status = time.perf_counter(), {"code": 500}

        async def send_wrapper(message):
            if message["type"] == "http.response.start":
                status["code"] = message["status"]
                message.setdefault("headers", []).append((b"x-request-id", rid.encode()))
            await send(message)

        try:
            await self.app(scope, receive, send_wrapper)
        finally:
            route = getattr(scope.get("route"), "path", "unmatched")
            self.log.info("request", extra={"method": scope["method"], "route": route,
                                            "status": status["code"],
                                            "duration_ms": round((time.perf_counter() - start) * 1000, 2)})
            request_id_var.reset(token)


class BodyLimit:
    def __init__(self, app, max_bytes: int):
        self.app, self.max = app, max_bytes

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        declared = dict(scope["headers"]).get(b"content-length")
        if declared is not None and declared.isdigit() and int(declared) > self.max:
            body = json.dumps({"type": "about:blank", "title": "Content Too Large", "status": 413}).encode()
            await send({"type": "http.response.start", "status": 413,
                        "headers": [(b"content-type", b"application/problem+json"),
                                    (b"content-length", str(len(body)).encode())]})
            await send({"type": "http.response.body", "body": body})
            return
        await self.app(scope, receive, send)
```

`RequestContext` is registered **last**, so it's the outermost of the app's own
middleware: the 413 from `BodyLimit` gets a request ID and an access-log line too. The
request ID goes into `scope["state"]` so the problem handlers — including the 500
handler, which runs outside all user middleware (lesson 4) — can include it.

## Authentication

```python
# app/db.py
from collections.abc import AsyncIterator
from typing import Annotated

from fastapi import Depends, Request
from sqlalchemy.ext.asyncio import AsyncSession


async def get_db(request: Request) -> AsyncIterator[AsyncSession]:
    async with request.state.sessionmaker() as session:
        yield session


DB = Annotated[AsyncSession, Depends(get_db)]
```

```python
# app/auth.py
"""Verifies access tokens issued by the identity provider. This service never sees passwords."""
from dataclasses import dataclass
from typing import Annotated

import jwt
from fastapi import Depends, HTTPException, Security, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer, SecurityScopes

from app.config import Settings, get_settings

bearer = HTTPBearer(auto_error=False)


@dataclass(frozen=True)
class Principal:
    user_id: str
    scopes: frozenset[str]


def _401(detail: str) -> HTTPException:
    return HTTPException(status.HTTP_401_UNAUTHORIZED, detail, headers={"WWW-Authenticate": "Bearer"})


def principal(security_scopes: SecurityScopes,
              creds: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer)],
              settings: Annotated[Settings, Depends(get_settings)]) -> Principal:
    if creds is None:
        raise _401("Missing bearer token")
    try:
        claims = jwt.decode(creds.credentials, settings.jwt_secret.get_secret_value(),
                            algorithms=["HS256"], audience=settings.jwt_audience,
                            issuer=settings.jwt_issuer, options={"require": ["exp", "sub", "aud", "iss"]})
    except jwt.ExpiredSignatureError:
        raise _401("Token expired") from None
    except jwt.PyJWTError:
        raise _401("Invalid token") from None
    scopes = frozenset(claims.get("scope", "").split())
    missing = [s for s in security_scopes.scopes if s not in scopes]
    if missing:
        raise HTTPException(status.HTTP_403_FORBIDDEN, f"Missing scope: {' '.join(missing)}")
    return Principal(user_id=claims["sub"], scopes=scopes)


Attendee = Annotated[Principal, Security(principal, scopes=["bookings"])]
Organizer = Annotated[Principal, Security(principal, scopes=["events:write"])]
```

This service *verifies* tokens; issuing them is the identity provider's job (this is the
common shape for an API behind single sign-on). Verification checks signature, expiry,
**audience** (the token was issued for this API, not another one) and **issuer**.
`HTTPBearer` is used instead of `OAuth2PasswordBearer` because there's no password flow
here. For development and tests, `mint_token.py` plays the identity provider:

```python
# mint_token.py
"""Development helper: mint a token like the identity provider would. Never deploy this."""
import sys
from datetime import datetime, timedelta, timezone

import jwt

from app.config import get_settings


def mint(sub: str, scopes: str, minutes: int = 15) -> str:
    s = get_settings()
    now = datetime.now(timezone.utc)
    return jwt.encode({"sub": sub, "scope": scopes, "iat": now, "exp": now + timedelta(minutes=minutes),
                       "iss": s.jwt_issuer, "aud": s.jwt_audience},
                      s.jwt_secret.get_secret_value(), algorithm="HS256")


if __name__ == "__main__":
    print(mint(sys.argv[1], sys.argv[2] if len(sys.argv) > 2 else "bookings"))
```

## Routers

```python
# app/routers/events.py
from typing import Annotated

from fastapi import APIRouter, Query, Request, Response, status
from sqlalchemy import select

from app.auth import Organizer
from app.db import DB
from app.errors import NotFound
from app.models import Event
from app.schemas import EventIn, EventOut

router = APIRouter(prefix="/events", tags=["events"])


@router.post("", response_model=EventOut, status_code=status.HTTP_201_CREATED)
async def create_event(data: EventIn, who: Organizer, db: DB, request: Request, response: Response):
    event = Event(title=data.title, starts_at=data.starts_at, capacity=data.capacity,
                  seats_left=data.capacity, organizer=who.user_id)
    db.add(event)
    await db.commit()
    response.headers["Location"] = str(request.url_for("get_event", event_id=event.id))
    return event


@router.get("", response_model=list[EventOut])
async def list_events(db: DB, limit: Annotated[int, Query(ge=1, le=100)] = 20,
                      after_id: Annotated[int, Query(ge=0)] = 0):
    stmt = select(Event).where(Event.id > after_id).order_by(Event.id).limit(limit)
    return (await db.scalars(stmt)).all()


@router.get("/{event_id}", response_model=EventOut)
async def get_event(event_id: int, db: DB, response: Response):
    event = await db.get(Event, event_id)
    if event is None:
        raise NotFound(f"Event {event_id} not found")
    response.headers["Cache-Control"] = "no-cache"
    return event
```

```python
# app/routers/bookings.py
from datetime import datetime, timezone
from typing import Annotated

from fastapi import APIRouter, Depends, Header, Response, status
from sqlalchemy import select, update
from sqlalchemy.exc import IntegrityError

from app.auth import Attendee
from app.config import Settings, get_settings
from app.db import DB
from app.errors import DomainError, EventStarted, NotFound, SoldOut
from app.models import Booking, Event
from app.schemas import BookingIn, BookingOut

router = APIRouter(tags=["bookings"])
IdemKey = Annotated[str | None, Header(max_length=64, pattern=r"^[A-Za-z0-9_-]+$")]


class TooManySeats(DomainError):
    status = 422


@router.post("/events/{event_id}/bookings", response_model=BookingOut,
             status_code=status.HTTP_201_CREATED)
async def book(event_id: int, data: BookingIn, who: Attendee, db: DB, response: Response,
               settings: Annotated[Settings, Depends(get_settings)],
               idempotency_key: IdemKey = None):
    if data.seats > settings.max_seats_per_booking:
        raise TooManySeats(f"At most {settings.max_seats_per_booking} seats per booking")
    if idempotency_key:
        existing = await db.scalar(select(Booking).where(Booking.user_id == who.user_id,
                                                         Booking.idempotency_key == idempotency_key))
        if existing is not None:
            response.status_code = status.HTTP_200_OK
            return existing

    event = await db.get(Event, event_id)
    if event is None:
        raise NotFound(f"Event {event_id} not found")
    starts_at = event.starts_at.replace(tzinfo=timezone.utc) if event.starts_at.tzinfo is None else event.starts_at
    if starts_at <= datetime.now(timezone.utc):
        raise EventStarted("Bookings are closed: the event has started")

    # The whole correctness argument is this one statement: decrement only if enough
    # seats remain. Concurrent requests can't both pass the check.
    result = await db.execute(
        update(Event)
        .where(Event.id == event_id, Event.seats_left >= data.seats)
        .values(seats_left=Event.seats_left - data.seats, version=Event.version + 1))
    if result.rowcount != 1:
        await db.rollback()
        raise SoldOut(f"Not enough seats left for {data.seats}")
    booking = Booking(event_id=event_id, user_id=who.user_id, seats=data.seats,
                      idempotency_key=idempotency_key)
    db.add(booking)
    try:
        await db.commit()
    except IntegrityError:       # same idempotency key raced with itself
        await db.rollback()
        existing = await db.scalar(select(Booking).where(Booking.user_id == who.user_id,
                                                         Booking.idempotency_key == idempotency_key))
        response.status_code = status.HTTP_200_OK
        return existing
    return booking


@router.get("/me/bookings", response_model=list[BookingOut])
async def my_bookings(who: Attendee, db: DB):
    stmt = select(Booking).where(Booking.user_id == who.user_id).order_by(Booking.id)
    return (await db.scalars(stmt)).all()


@router.post("/bookings/{booking_id}/cancel", response_model=BookingOut)
async def cancel(booking_id: int, who: Attendee, db: DB):
    booking = await db.scalar(select(Booking).where(Booking.id == booking_id,
                                                    Booking.user_id == who.user_id))
    if booking is None:
        raise NotFound(f"Booking {booking_id} not found")
    if booking.status == "cancelled":
        return booking                                   # idempotent
    booking.status = "cancelled"
    await db.execute(update(Event).where(Event.id == booking.event_id)
                     .values(seats_left=Event.seats_left + booking.seats, version=Event.version + 1))
    await db.commit()
    return booking
```

```python
# app/routers/health.py
from fastapi import APIRouter, Response
from sqlalchemy import text

from app.db import DB

router = APIRouter(tags=["health"], include_in_schema=False)


@router.get("/health/live")
async def live():
    return {"status": "ok"}


@router.get("/health/ready")
async def ready(db: DB, response: Response):
    try:
        await db.execute(text("SELECT 1"))
    except Exception:
        response.status_code = 503
        return {"status": "unavailable", "database": "down"}
    return {"status": "ok", "database": "up"}
```

```python
# app/main.py
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from app import errors
from app.config import get_settings
from app.observability import BodyLimit, RequestContext, configure_logging
from app.routers import bookings, events, health


@asynccontextmanager
async def lifespan(app: FastAPI):
    engine = create_async_engine(get_settings().database_url)
    yield {"sessionmaker": async_sessionmaker(engine, expire_on_commit=False)}
    await engine.dispose()


def create_app() -> FastAPI:
    settings = get_settings()
    configure_logging()
    docs = settings.docs_enabled and settings.environment != "prod"
    app = FastAPI(title="Event Booking API", version="1.0.0", lifespan=lifespan,
                  openapi_url="/openapi.json" if docs else None,
                  telemetry={"exclude": lambda scope: scope.get("path", "").startswith("/health")})
    errors.install(app)
    app.include_router(events.router)
    app.include_router(bookings.router)
    app.include_router(health.router)
    app.add_middleware(BodyLimit, max_bytes=settings.max_body_bytes)
    if settings.cors_origins:
        app.add_middleware(CORSMiddleware, allow_origins=settings.cors_origins,
                           allow_methods=["GET", "POST"],
                           allow_headers=["Authorization", "Content-Type", "Idempotency-Key"])
    app.add_middleware(RequestContext)          # outermost of ours: every response gets an ID
    return app


app = create_app()
```

The `telemetry` option excludes health checks from FastAPI's built-in tracing (lesson 4),
so probes every few seconds don't drown real traffic.

## How It Actually Works: why it can't overbook

The booking path does **not** read `seats_left`, check it in Python, and write it back.
Two concurrent requests doing that could both read `1`, both decide there's room, and
both write — two bookings for one seat. Instead:

```sql
UPDATE events
SET seats_left = seats_left - :n, version = version + 1
WHERE id = :id AND seats_left >= :n
```

The check and the decrement are one statement. The database executes conflicting updates
one at a time; the second one re-evaluates `seats_left >= :n` against the already
decremented value, matches zero rows, and the code turns `rowcount != 1` into `SoldOut`.
The booking row is inserted in the same transaction, so a seat is never decremented
without a booking or vice versa. And the `CHECK (seats_left >= 0)` constraint backs it
up.

Idempotency has the same shape: the fast path looks up an existing booking by
`(user_id, key)`; if two retries race past that lookup, the `UNIQUE` constraint lets only
one insert win, and the loser's `IntegrityError` handler returns the winner's booking.

Where the earlier read happens (checking that the event exists and hasn't started), it
can't cause overbooking — at worst a booking succeeds a moment before the start time.

## Tests

```python
# tests/conftest.py
import os

os.environ.setdefault("BOOKING_JWT_SECRET", "test-secret-" + "x" * 40)   # before importing the app
os.environ.setdefault("BOOKING_ENVIRONMENT", "test")

from datetime import datetime, timedelta, timezone

import pytest
from httpx import ASGITransport, AsyncClient
from sqlalchemy import event
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.pool import StaticPool

from app.db import get_db
from app.main import app
from app.models import Base
from mint_token import mint


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
    async def override():
        yield db
    app.dependency_overrides[get_db] = override
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()


@pytest.fixture
async def real_client(tmp_path):
    """Per-request sessions on a temporary file database: for concurrency tests."""
    engine = create_async_engine(f"sqlite+aiosqlite:///{tmp_path / 'test.db'}")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    maker = async_sessionmaker(engine, expire_on_commit=False)

    async def override():
        async with maker() as session:
            yield session
    app.dependency_overrides[get_db] = override
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
    await engine.dispose()


def auth(sub: str, scopes: str = "bookings") -> dict:
    return {"Authorization": f"Bearer {mint(sub, scopes)}"}


ORGANIZER = auth("org-1", "events:write bookings")


def future(days: int = 30) -> str:
    return (datetime.now(timezone.utc) + timedelta(days=days)).isoformat()


async def make_event(client, capacity: int = 10, starts_at: str | None = None) -> dict:
    r = await client.post("/events", headers=ORGANIZER,
                          json={"title": "PyCon talk", "starts_at": starts_at or future(),
                                "capacity": capacity})
    assert r.status_code == 201, r.text
    return r.json()
```

```python
# tests/test_events.py
import pytest

from tests.conftest import ORGANIZER, auth, future, make_event

pytestmark = pytest.mark.anyio


async def test_create_and_get_event(client):
    ev = await make_event(client, capacity=50)
    assert ev["seats_left"] == 50 and ev["starts_at"].endswith("Z")
    r = await client.get(f"/events/{ev['id']}")
    assert r.json() == ev and r.headers["cache-control"] == "no-cache"


async def test_only_organizers_create_events(client):
    r = await client.post("/events", headers=auth("u1"), json={"title": "x", "starts_at": future(), "capacity": 5})
    assert r.status_code == 403 and r.headers["content-type"] == "application/problem+json"
    r = await client.post("/events", json={"title": "x", "starts_at": future(), "capacity": 5})
    assert r.status_code == 401 and r.headers["www-authenticate"] == "Bearer"


async def test_naive_datetimes_rejected(client):
    r = await client.post("/events", headers=ORGANIZER,
                          json={"title": "x", "starts_at": "2030-01-01T10:00:00", "capacity": 5})
    assert r.status_code == 422
    body = r.json()
    assert body["type"].endswith("/validation") and body["errors"][0]["field"] == "starts_at"


async def test_unknown_event_is_problem_404(client):
    r = await client.get("/events/999", headers={"X-Request-ID": "trace-me"})
    assert r.status_code == 404
    assert r.json()["request_id"] == "trace-me" and r.headers["x-request-id"] == "trace-me"


async def test_bad_tokens(client):
    from datetime import datetime, timedelta, timezone
    import jwt
    for claims in ({"sub": "u", "aud": "other-api", "iss": "https://id.example.com"},
                   {"sub": "u", "aud": "booking-api", "iss": "https://evil.example"}):
        claims["exp"] = datetime.now(timezone.utc) + timedelta(minutes=5)
        tok = jwt.encode(claims, "test-secret-" + "x" * 40, algorithm="HS256")
        r = await client.get("/me/bookings", headers={"Authorization": f"Bearer {tok}"})
        assert r.status_code == 401


async def test_body_limit(client):
    r = await client.post("/events", headers={**ORGANIZER, "content-type": "application/json"},
                          content=b'{"title": "' + b"x" * 20_000 + b'"}')
    assert r.status_code == 413


async def test_health(client):
    assert (await client.get("/health/live")).json() == {"status": "ok"}
    assert (await client.get("/health/ready")).json() == {"status": "ok", "database": "up"}
```

```python
# tests/test_bookings.py
import asyncio

import pytest

from tests.conftest import auth, future, make_event

pytestmark = pytest.mark.anyio
ADA, BOB = auth("ada"), auth("bob")


async def test_book_and_cancel_restores_seats(client):
    ev = await make_event(client, capacity=5)
    r = await client.post(f"/events/{ev['id']}/bookings", headers=ADA, json={"seats": 2})
    assert r.status_code == 201 and r.json()["status"] == "confirmed"
    assert (await client.get(f"/events/{ev['id']}")).json()["seats_left"] == 3
    bid = r.json()["id"]
    assert (await client.post(f"/bookings/{bid}/cancel", headers=ADA)).json()["status"] == "cancelled"
    assert (await client.post(f"/bookings/{bid}/cancel", headers=ADA)).status_code == 200   # idempotent
    assert (await client.get(f"/events/{ev['id']}")).json()["seats_left"] == 5


async def test_sold_out_is_409_problem(client):
    ev = await make_event(client, capacity=3)
    await client.post(f"/events/{ev['id']}/bookings", headers=ADA, json={"seats": 3})
    r = await client.post(f"/events/{ev['id']}/bookings", headers=BOB, json={"seats": 1})
    assert r.status_code == 409 and r.json()["type"].endswith("/sold-out")


async def test_idempotency_key_returns_same_booking(client):
    ev = await make_event(client, capacity=5)
    h = {**ADA, "Idempotency-Key": "order-123"}
    a = await client.post(f"/events/{ev['id']}/bookings", headers=h, json={"seats": 2})
    b = await client.post(f"/events/{ev['id']}/bookings", headers=h, json={"seats": 2})
    assert (a.status_code, b.status_code) == (201, 200) and a.json()["id"] == b.json()["id"]
    assert (await client.get(f"/events/{ev['id']}")).json()["seats_left"] == 3


async def test_cannot_cancel_someone_elses_booking(client):
    ev = await make_event(client)
    bid = (await client.post(f"/events/{ev['id']}/bookings", headers=ADA, json={"seats": 1})).json()["id"]
    assert (await client.post(f"/bookings/{bid}/cancel", headers=BOB)).status_code == 404


async def test_started_events_and_seat_limits(client):
    past = await make_event(client, starts_at="2020-01-01T10:00:00+00:00")
    r = await client.post(f"/events/{past['id']}/bookings", headers=ADA, json={"seats": 1})
    assert r.status_code == 409 and r.json()["type"].endswith("/event-started")
    ev = await make_event(client, capacity=100)
    assert (await client.post(f"/events/{ev['id']}/bookings", headers=ADA, json={"seats": 11})).status_code == 422
    assert (await client.post(f"/events/{ev['id']}/bookings", headers=ADA, json={"seats": 0})).status_code == 422


async def test_no_overbooking_under_concurrency(real_client):
    ev = await make_event(real_client, capacity=10)
    users = [auth(f"user-{i}") for i in range(40)]
    results = await asyncio.gather(*[
        real_client.post(f"/events/{ev['id']}/bookings", headers=h, json={"seats": 1}) for h in users])
    codes = [r.status_code for r in results]
    print("\n201s:", codes.count(201), "409s:", codes.count(409), "other:", len(codes) - codes.count(201) - codes.count(409))
    assert codes.count(201) == 10 and codes.count(409) == 30
    assert (await real_client.get(f"/events/{ev['id']}")).json()["seats_left"] == 0
```

```text
$ python -m pytest -q -s
...
201s: 10 409s: 30 other: 0
...
13 passed in 0.33s
```

(JSON access-log lines from `-s` omitted.) The concurrency test fired **40 simultaneous
single-seat bookings** at a 10-seat event through `real_client` (separate sessions per
request on a file database): exactly 10 succeeded, 30 got the sold-out problem, and
`seats_left` ended at 0.

## Worked example: four processes, 200 requests, 50 seats

A test inside one process can't prove much about several processes. So: migrate a real
database file, start four workers, create a 50-seat event, and hit it with 200 booking
requests, 40 at a time, using `ab`:

```bash
export BOOKING_JWT_SECRET="dev-secret-$(python -c 'import secrets; print(secrets.token_hex(24))')"
alembic upgrade head
fastapi run app/main.py --port 8719 --workers 4
```

```text
$ alembic revision --autogenerate -m "events and bookings"
INFO  [alembic.autogenerate.compare.tables] Detected added table 'events'
INFO  [alembic.autogenerate.compare.tables] Detected added table 'bookings'
$ alembic upgrade head
INFO  [alembic.runtime.migration] Running upgrade  -> 8af391d0b4dc, events and bookings

POST /events  {"title":"Launch party","starts_at":"2030-06-01T18:00:00+02:00","capacity":50}
{"id":1,"title":"Launch party","starts_at":"2030-06-01T16:00:00Z","capacity":50,"seats_left":50}
```

(The `+02:00` start time was stored and returned as UTC.) Then:

```bash
ab -q -n 200 -c 40 -p seat.json -T application/json \
   -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8719/events/1/bookings
```

```text
Complete requests:      200
Failed requests:        191
Non-2xx responses:      150
Requests per second:    319.93 [#/sec] (mean)
```

`ab`'s "Failed requests" counts responses whose **length** differs from the first one —
201 bodies and 409 bodies differ in length, and the booking IDs vary — so it isn't an
error count. The checks that matter:

```text
GET /events/1
{"id":1,"title":"Launch party","starts_at":"2030-06-01T16:00:00Z","capacity":50,"seats_left":0}

SELECT count(*), sum(seats) FROM bookings WHERE event_id=1 AND status='confirmed'
50|50

access log: 51 responses with status 201 (50 bookings + 1 event), 150 with 409, 0 with 500
```

Exactly 50 seats sold across four independent processes, 150 clean sold-out responses,
no server errors, and no `database is locked` failures at this load. An unknown event and
the readiness probe, against the same server:

```text
HTTP/1.1 404 Not Found
content-type: application/problem+json
x-request-id: a895fd72c4984cc8bfe38a609c374280
{"type":"about:blank","title":"Not Found","status":404,"instance":"/events/9999","request_id":"a895fd72c4984cc8bfe38a609c374280","detail":"Event 9999 not found"}

{"status":"ok","database":"up"}
```

!!! note "What this run does and doesn't prove"
    It proves the booking logic is correct under real multi-process concurrency on
    SQLite, which serialises writers. It doesn't measure production throughput: SQLite
    on a laptop, with `ab` on the same machine, says little about a PostgreSQL server
    under real traffic. The settings guard refuses SQLite in prod for that reason. The
    same `UPDATE ... WHERE seats_left >= :n` is correct on PostgreSQL, where concurrent
    updates to one row wait on its row lock and then re-check the condition. That wasn't
    run here.

## Going further

What a real deployment would add, each covered earlier: a container image and graceful
shutdown settings (lesson 2); the proxy configuration with trusted forwarded headers
(lesson 3); Prometheus metrics or an OTLP exporter for the built-in telemetry (lesson 4);
rate limits on booking per user (Level 3 lesson 9); confirmation emails through a job
queue, with the booking ID as the idempotency key so retries don't double-send
(lesson 7); and the OpenAPI breaking-change check in CI (lesson 6).

## Common mistakes

- **Read-check-write for inventory**: the classic overbooking race.
- **Idempotency without a unique constraint**: two racing retries both insert.
- **Naive datetimes** for events: ambiguous input, wrong comparisons.
- **Secrets in startup error messages**: set `hide_input_in_errors=True` on settings.
- **Middleware order** that leaves some responses (413s, 500s) without request IDs.
- **Trusting single-process tests** for multi-process correctness.

## Exercise

1. Add waitlists: when an event is sold out, `POST /events/{id}/waitlist` records the
   user; a cancellation offers the seat to the first person on the list via a job.
   Write a concurrency test for "two cancellations, one waitlisted user".
2. Add `PATCH /events/{id}` for organisers to change capacity, with `If-Match` on the
   `version` column (Level 3 lesson 9). Reducing capacity below seats already sold must
   fail — enforce it in the `UPDATE`'s `WHERE` clause.
3. Add per-user booking rate limits and a `GET /me/bookings` cursor pagination.
4. Run the multi-process load test against PostgreSQL if you can, with 8 workers, and
   compare: does anything about correctness change? Does throughput?
