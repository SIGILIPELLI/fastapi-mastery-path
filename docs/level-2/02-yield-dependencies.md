---
description: "Dependencies with yield for setup and teardown: commit and rollback patterns, request vs function scope, and the commit-after-response trap."
---

# 02 · Yield Dependencies, Caching & Overrides

Some resources need cleaning up: a database session must be closed, a transaction
committed or rolled back, a lock released, a temporary file deleted. A dependency
written as a **generator** — with `yield` instead of `return` — gives you a place to do
setup before the endpoint and teardown after it, guaranteed, even when the endpoint
fails.

## Setup, yield, teardown

To see the order of events without a real database, here's a fake session that logs
what happens to it:

```python
from typing import Annotated
from fastapi import Depends, FastAPI, HTTPException

log: list[str] = []
app = FastAPI()

class FakeSession:
    def __init__(self): self.pending = []
    def add(self, x): self.pending.append(x)
    def commit(self): log.append(f"commit {self.pending}")
    def rollback(self): log.append("rollback"); self.pending.clear()
    def close(self): log.append("close")

def get_session():
    s = FakeSession()
    log.append("open")
    try:
        yield s
        s.commit()
    except Exception as exc:
        log.append(f"saw {type(exc).__name__}")
        s.rollback()
        raise
    finally:
        s.close()

Session = Annotated[FakeSession, Depends(get_session)]

@app.post("/ok")
def ok(s: Session):
    s.add("book"); log.append("endpoint")
    return {"ok": True}

@app.post("/conflict")
def conflict(s: Session):
    s.add("dup"); log.append("endpoint")
    raise HTTPException(409, "duplicate")

@app.post("/bug")
def bug(s: Session):
    s.add("x"); log.append("endpoint")
    return 1 / 0
```

The event logs:

```text
/ok       200 ['open', 'endpoint', "commit ['book']", 'close']
/conflict 409 ['open', 'endpoint', 'saw HTTPException', 'rollback', 'close']
/bug      500 ['open', 'endpoint', 'saw ZeroDivisionError', 'rollback', 'close']
```

The structure is the one to memorise:

- code before `yield` → setup;
- `yield value` → the value injected into the endpoint;
- code after `yield` inside `try` → runs only on success;
- `except` → runs if the endpoint (or a later dependency) raised; the exception is
  thrown *into* the generator at the `yield`;
- `finally` → always runs.

Every path closed the session. That guarantee is the whole point.

## Always re-raise

What if a dependency catches an exception and doesn't re-raise it?

```python
def swallow():
    try:
        yield "s"
    except HTTPException:
        log.append("swallowed")

@app.post("/swallow")
def sw(s: Annotated[str, Depends(swallow)]):
    raise HTTPException(409, "dup")
```

The client got `500 Internal Server Error` — not the 409 — and the server raised:

```text
FastAPIError Response not awaited. There's a high chance that the application code is raising an exception and a dependency with yield has a block with a bare except, or a block with except Exception, and is not raising the exception again. ...
```

By swallowing the exception, the dependency erased the error that would have produced the
response, leaving FastAPI with nothing to send. **An `except` block in a yield
dependency must end with `raise`**, unless it raises a different exception on purpose.

## When does teardown run? `scope`

This is the subtle part, and it changed across FastAPI releases. In the version used here
(0.143.0), `Depends` takes a `scope` argument:

- **`scope="request"`** (the default for yield dependencies): teardown runs **after the
  response has been sent**.
- **`scope="function"`**: teardown runs right after the endpoint function returns,
  **before** the response is sent.

The difference is visible with a streaming response, whose body is produced *after* the
endpoint returns:

```python
from fastapi.responses import StreamingResponse

@app.get("/stream-fn")
def stream_fn(s: Annotated[str, Depends(get_session_fn, scope="function")]):
    def gen():
        log.append("stream chunk 1"); yield b"a"
        log.append("stream chunk 2"); yield b"b"
    return StreamingResponse(gen())
```

```text
/stream-fn  200 b'ab' ['open-fn', 'close-fn', 'stream chunk 1', 'stream chunk 2']
/stream-req 200 b'ab' ['open-req', 'stream chunk 1', 'stream chunk 2', 'close-req']
```

With function scope, the session was closed before the stream produced anything — if
the generator had used it, it would have hit a closed session. With request scope, the
session stayed open until the stream finished.

## Worked example: the commit-after-response trap

Request scope has a cost. If the work *after* `yield` can fail — and a database commit
can fail (a constraint violation, a lost connection) — the client may already have its
answer:

```python
def failing_commit_req():
    yield "s"
    log.append("committing (request scope)")
    raise RuntimeError("commit failed")

def failing_commit_fn():
    yield "s"
    log.append("committing (function scope)")
    raise RuntimeError("commit failed")

@app.post("/req")
def req(s: Annotated[str, Depends(failing_commit_req)]):
    return {"saved": True}

@app.post("/fn")
def fn(s: Annotated[str, Depends(failing_commit_fn, scope="function")]):
    return {"saved": True}
```

```text
/req 200 {"saved":true} ['committing (request scope)']
/fn  500 Internal Server Error ['committing (function scope)']
```

With the default scope, the client was told `{"saved": true}` and then the commit
failed. The exception still surfaced on the server (the test client re-raised
`RuntimeError: commit failed` when exceptions weren't suppressed), but the client never
knew. That's a lie in your API.

Two safe patterns:

1. **Commit inside the endpoint (or service function)** and use the dependency only to
   provide the session and roll back / close. A commit failure is then an ordinary
   exception that produces a 500 (or a 409 you map), before any response exists. Lesson 4 of this level
   uses this.
2. **Use `scope="function"`** for dependencies whose teardown can fail in a way the
   client must hear about — and don't combine them with streaming responses that need
   the resource.

## Caching and overrides with yield dependencies

The per-request cache from lesson 1 applies: if three dependencies in one request all
need `get_session`, they share one session, opened once and closed once. That's what
makes "one transaction per request" work without passing the session around by hand.

Overrides work the same way, and an override can itself be a generator. A common test
setup replaces the real session with one bound to a test database:

```python
def override_session():
    s = make_test_session()
    try:
        yield s
    finally:
        s.close()

app.dependency_overrides[get_session] = override_session
```

## How It Actually Works

FastAPI wraps each generator dependency in a context manager — `asynccontextmanager`
for `async def` generators, and `contextmanager` run through
`contextmanager_in_threadpool` for plain `def` generators — and enters it on an
`AsyncExitStack`. The route handler in `fastapi/routing.py` (a modified copy of
Starlette's `request_response`) creates **two** nested stacks for every request, and the
nesting explains everything above:

```python
async with AsyncExitStack() as request_stack:
    scope["fastapi_inner_astack"] = request_stack
    async with AsyncExitStack() as function_stack:
        scope["fastapi_function_astack"] = function_stack
        response = await f(request)        # solve dependencies, call endpoint
    await response(scope, receive, send)   # send the response
    response_awaited = True
if not response_awaited:
    raise FastAPIError("Response not awaited. ...")
```

(Condensed from the 0.143.0 source.) When solving dependencies, a generator with
`scope="function"` is entered on `function_stack`; otherwise on `request_stack`.
`function_stack` closes as soon as the endpoint returns — before
`await response(...)` sends anything. `request_stack` closes only after the response has
been sent, which is why a failing commit there can't change the status code any more.

When the endpoint raises, the stacks unwind and each context manager's `__exit__`
`throw()`s the exception into your generator at the `yield` — that's how your `except`
block sees it. If your generator handles it and returns normally, the exception is
treated as handled, `response_awaited` is never set, and you get exactly the
`Response not awaited` error shown earlier.

FastAPI also refuses one combination at startup: a request-scoped dependency can't depend
on a function-scoped one, since the inner resource would be closed before the outer one
had finished with it.

Teardown order is the reverse of setup, like nested `with` blocks: a dependency that
depends on the session is torn down before the session itself.

!!! note "Version history"
    The timing of yield-dependency teardown has changed more than once in FastAPI's
    history, which is why older tutorials disagree. If you're on an older version
    without `scope`, test the behaviour of your version with a logging dependency like
    the one above before relying on it.

## Common mistakes

- **Swallowing exceptions** in an `except` block, producing `Response not awaited` 500s.
- **Committing after `yield` with the default scope**, so a failed commit is reported to
  the client as success.
- **Using a function-scoped resource in a `StreamingResponse`** — it's already closed.
- **Doing slow cleanup in request scope and expecting it to be free.** It runs after the
  response, but the connection, the session and (for `def` generators) a thread-pool
  slot stay in use until it finishes.
- **`yield` more than once.** A dependency generator must yield exactly once.
- **Returning instead of yielding** and wondering why `finally` runs before the endpoint.

## Exercise

1. Write a `get_lock` yield dependency that acquires a `threading.Lock` before `yield`
   and releases it in `finally`. Prove with a log that the lock is released even when
   the endpoint raises `HTTPException`.
2. Reproduce the commit-after-response trap, then fix it both ways: move the commit into
   the endpoint, and switch to `scope="function"`. Check the status codes.
3. Make two dependencies `get_session` and `get_repo(session = Depends(get_session))`,
   both with logging teardown. Confirm that teardown runs in reverse order and that the
   session is opened only once when an endpoint uses both.
4. Try to use a function-scoped dependency inside a `StreamingResponse` generator and
   describe what goes wrong.
