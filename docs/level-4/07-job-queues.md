---
description: "Handle work that outlives a request: a durable job table, 202 Accepted with polling, atomic claiming, leases, retries with backoff and idempotency keys."
---

# 07 · Long-Running Work: Job Queues and Polling

Some work doesn't fit in a request: generating a big report, transcoding a video, sending
a thousand emails, calling a slow partner API. Level 3 lesson 5 showed that
`BackgroundTasks` runs *after* the response but *in the web process*, with no
persistence, no retries and no visibility. When the work must happen, it needs a
**job queue**: the API records the job durably and answers immediately; separate
**worker** processes do the work; the client polls (or gets notified) for the result.

Production systems usually use a queue library with a broker — Celery, RQ or Dramatiq
(Redis or RabbitMQ), ARQ (Redis, asyncio), or a managed cloud queue. No broker was
available for this course, so this lesson builds the essential mechanics with a SQLite
table. That's genuinely useful for small systems (a PostgreSQL version of the same table
is a well-known pattern), and it makes every moving part visible — the same parts you
configure in Celery or ARQ.

## The protocol

1. `POST /reports` → **202 Accepted**, a job ID, and `Location: /jobs/{id}`.
2. `GET /jobs/{id}` → `queued` / `running` (with `Retry-After`) / `done` with a result /
   `failed` with an error.
3. Workers loop: claim a job atomically, run it, record the outcome.

## The job store

```python
# jobstore.py
"""A tiny durable job queue in SQLite. Shared by the API and the worker."""
import json
import sqlite3
import time
import uuid

DB = "jobs.db"


def connect() -> sqlite3.Connection:
    conn = sqlite3.connect(DB, timeout=10, isolation_level=None)   # autocommit; explicit BEGIN
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    return conn


def init() -> None:
    with connect() as c:
        c.execute("""CREATE TABLE IF NOT EXISTS jobs (
            id TEXT PRIMARY KEY,
            kind TEXT NOT NULL,
            payload TEXT NOT NULL,
            idempotency_key TEXT UNIQUE,
            status TEXT NOT NULL DEFAULT 'queued',      -- queued|running|done|failed
            attempts INTEGER NOT NULL DEFAULT 0,
            max_attempts INTEGER NOT NULL DEFAULT 3,
            run_after REAL NOT NULL DEFAULT 0,
            locked_until REAL,
            result TEXT,
            error TEXT,
            created_at REAL NOT NULL)""")


def enqueue(kind: str, payload: dict, idempotency_key: str | None = None) -> tuple[str, bool]:
    c = connect()
    job_id = uuid.uuid4().hex
    try:
        c.execute("INSERT INTO jobs (id, kind, payload, idempotency_key, created_at) VALUES (?,?,?,?,?)",
                  (job_id, kind, json.dumps(payload), idempotency_key, time.time()))
        return job_id, True
    except sqlite3.IntegrityError:
        row = c.execute("SELECT id FROM jobs WHERE idempotency_key = ?", (idempotency_key,)).fetchone()
        return row["id"], False


def get(job_id: str) -> dict | None:
    row = connect().execute("SELECT * FROM jobs WHERE id = ?", (job_id,)).fetchone()
    return dict(row) if row else None


def claim(worker: str, lease_seconds: float = 30) -> dict | None:
    """Atomically take one runnable job. A crashed worker's lease expires and the job is retried."""
    now = time.time()
    row = connect().execute(
        """UPDATE jobs SET status = 'running', attempts = attempts + 1, locked_until = ?
           WHERE id = (SELECT id FROM jobs
                       WHERE (status = 'queued' AND run_after <= ?)
                          OR (status = 'running' AND locked_until < ?)
                       ORDER BY created_at LIMIT 1)
           RETURNING *""", (now + lease_seconds, now, now)).fetchone()
    return dict(row) if row else None


def finish(job_id: str, result: dict) -> None:
    connect().execute("UPDATE jobs SET status='done', result=?, locked_until=NULL WHERE id=?",
                      (json.dumps(result), job_id))


def fail(job: dict, error: str) -> None:
    c = connect()
    if job["attempts"] >= job["max_attempts"]:
        c.execute("UPDATE jobs SET status='failed', error=?, locked_until=NULL WHERE id=?", (error, job["id"]))
    else:
        backoff = 2 ** job["attempts"] * 0.25
        c.execute("UPDATE jobs SET status='queued', error=?, run_after=?, locked_until=NULL WHERE id=?",
                  (error, time.time() + backoff, job["id"]))
```

The important parts:

- **`claim`** is a single `UPDATE ... WHERE id = (SELECT ... LIMIT 1) RETURNING *`. Two
  workers can't claim the same job, because the database applies the update atomically —
  no "select, then update" race. (`RETURNING` needs SQLite 3.35 or later; this ran on
  SQLite 3.53.4. On PostgreSQL the equivalent uses `SELECT ... FOR UPDATE SKIP LOCKED`.)
- **Leases** (`locked_until`): a claimed job belongs to its worker only for a limited
  time. If the worker dies, the lease expires and another worker picks the job up.
- **Retries with exponential backoff**: a failure re-queues the job with `run_after` in
  the future, until `max_attempts`.
- **Idempotency keys**: a `UNIQUE` column, so a client retrying a `POST` gets the
  original job back instead of a duplicate.
- **WAL mode** lets the API read while a worker writes.

## The API

```python
# api.py
import json
from typing import Annotated

from fastapi import FastAPI, Header, HTTPException, Request, Response, status
from pydantic import BaseModel, Field

import jobstore

jobstore.init()
app = FastAPI()


class ReportRequest(BaseModel):
    year: int = Field(ge=2000, le=2100)
    fail_times: int = Field(0, ge=0, le=5)      # for the demo: make the job fail N times first


@app.post("/reports", status_code=status.HTTP_202_ACCEPTED)
def request_report(body: ReportRequest, request: Request, response: Response,
                   idempotency_key: Annotated[str | None, Header()] = None):
    job_id, created = jobstore.enqueue("sales_report", body.model_dump(), idempotency_key)
    url = str(request.url_for("job_status", job_id=job_id))
    response.headers["Location"] = url
    if not created:
        response.status_code = status.HTTP_200_OK
    return {"job_id": job_id, "status_url": url, "created": created}


@app.get("/jobs/{job_id}")
def job_status(job_id: str, response: Response):
    job = jobstore.get(job_id)
    if job is None:
        raise HTTPException(404, "Job not found")
    if job["status"] in ("queued", "running"):
        response.headers["Retry-After"] = "1"
    return {"id": job["id"], "status": job["status"], "attempts": job["attempts"],
            "result": json.loads(job["result"]) if job["result"] else None,
            "error": job["error"]}
```

## The worker

```python
# worker.py
import json, os, signal, sys, time

import jobstore

running = True
def stop(*_):
    global running
    running = False
signal.signal(signal.SIGTERM, stop)

NAME = f"worker-{os.getpid()}"


def sales_report(payload: dict, attempt: int) -> dict:
    if attempt <= payload.get("fail_times", 0):
        raise RuntimeError(f"upstream timeout (simulated, attempt {attempt})")
    time.sleep(1.0)                                  # the slow part
    return {"year": payload["year"], "total_cents": 123_456_00, "by": NAME}


HANDLERS = {"sales_report": sales_report}

print(f"{NAME} started", flush=True)
while running:
    job = jobstore.claim(NAME, lease_seconds=float(os.environ.get("LEASE_SECONDS", "30")))
    if job is None:
        time.sleep(0.2)
        continue
    print(f"{NAME} took {job['id'][:8]} attempt {job['attempts']}", flush=True)
    try:
        result = HANDLERS[job["kind"]](json.loads(job["payload"]), job["attempts"])
        jobstore.finish(job["id"], result)
        print(f"{NAME} finished {job['id'][:8]}", flush=True)
    except Exception as exc:
        jobstore.fail(job, str(exc))
        print(f"{NAME} failed {job['id'][:8]}: {exc}", flush=True)
print(f"{NAME} stopped cleanly", flush=True)
```

It runs as its own process — `python worker.py` — as many copies as you want. It handles
`SIGTERM` by finishing the current job and exiting, so deploys don't abandon work.

## Running it

The API under Uvicorn and two workers, all on one machine:

```bash
uvicorn api:app --port 8718
python worker.py &
python worker.py &
```

Request a report, then send the *same* request again with the same idempotency key:

```text
HTTP/1.1 202 Accepted
location: http://localhost:8718/jobs/97e0a08d604b476d9554247bf2871ce9
{"job_id":"97e0a08d604b476d9554247bf2871ce9","status_url":"http://localhost:8718/jobs/97e0a08d604b476d9554247bf2871ce9","created":true}

{"job_id":"97e0a08d604b476d9554247bf2871ce9",...,"created":false} [200]
```

Same job ID, `created: false`, status 200 — no duplicate report. A second job was
submitted that fails twice before succeeding (`fail_times: 2`). Polling both once a
second:

```text
t+0s
retry-after: 1
{"id":"97e0a08d...","status":"running","attempts":1,"result":null,"error":null}
{"id":"8c25bc18...","status":"queued","attempts":1,"result":null,"error":"upstream timeout (simulated, attempt 1)"}
t+1s
{"id":"97e0a08d...","status":"done","attempts":1,"result":{"year":2026,"total_cents":12345600,"by":"worker-74857"},"error":null}
{"id":"8c25bc18...","status":"queued","attempts":2,"result":null,"error":"upstream timeout (simulated, attempt 2)"}
t+2s
{"id":"8c25bc18...","status":"running","attempts":3,...}
t+3s
{"id":"8c25bc18...","status":"done","attempts":3,"result":{"year":2025,"total_cents":12345600,"by":"worker-74859"},"error":"upstream timeout (simulated, attempt 2)"}
```

(IDs shortened; the first job's later identical lines omitted.) The workers' logs:

```text
worker-74857 started
worker-74857 took 97e0a08d attempt 1
worker-74857 finished 97e0a08d
worker-74859 started
worker-74859 took 8c25bc18 attempt 1
worker-74859 failed 8c25bc18: upstream timeout (simulated, attempt 1)
worker-74859 took 8c25bc18 attempt 2
worker-74859 failed 8c25bc18: upstream timeout (simulated, attempt 2)
worker-74859 took 8c25bc18 attempt 3
worker-74859 finished 8c25bc18
```

The two jobs ran in parallel on different workers; the failing one was retried after
growing delays and succeeded on its third attempt. The finished job keeps its last error
in the `error` column — useful for diagnosis; clear it in `finish` if you'd rather not
show it.

## Worked example: a worker dies mid-job

A worker was started with a 2-second lease (`LEASE_SECONDS=2`), given a job, and killed
with `kill -9` half a second into the 1-second task:

```text
killed 74903 mid-job
{"id":"4e06b5f8...","status":"running","attempts":1,"result":null,"error":null}
```

The job was stuck in `running` — nobody was working on it. A new worker was started;
once the lease expired, it claimed the job:

```text
{"id":"4e06b5f8...","status":"done","attempts":2,"result":{"year":2024,"total_cents":12345600,"by":"worker-74914"},"error":null}

worker-74903 started
worker-74903 took 4e06b5f8 attempt 1
worker-74914 started
worker-74914 took 4e06b5f8 attempt 2
worker-74914 finished 4e06b5f8
worker-74914 stopped cleanly
```

The last line is the graceful `SIGTERM` exit. The crash recovery has a consequence you
must design for: the killed worker might have done *part* of the job — sent some emails,
charged a card — before dying, and the retry does it again. Queues give you
**at-least-once** execution. Make jobs **idempotent** (check "already sent?" before
sending; pass an idempotency key to payment providers) so running twice is harmless. Set
the lease longer than the longest normal run, or extend it periodically from the worker
(a "heartbeat"); a lease that expires while a slow job is still running causes a
duplicate run.

## How It Actually Works

The queue is just rows with a state machine:

```text
queued ──claim──▶ running ──finish──▶ done
   ▲                 │
   └──fail (retry)───┤
                     └──fail (no attempts left)──▶ failed
running with expired lease ──claim──▶ running (attempt n+1)
```

Correctness rests on the claim being one atomic statement. SQLite serialises writers, so
two workers' `UPDATE`s run one after the other; the second sees the job already
`running` with a fresh lease, the subquery picks a different job or none. PostgreSQL
achieves the same with row locks: `FOR UPDATE SKIP LOCKED` lets many workers claim
different rows concurrently without blocking each other.

Brokered systems implement the same ideas differently: Celery's "acks late" plus a
visibility timeout is a lease; retries with countdowns are `run_after`; result backends
hold the `result` column. Knowing the mechanics makes their configuration options make
sense.

## Common mistakes

- **Long work in the request**, so proxies time out and clients retry — starting the
  work twice.
- **`BackgroundTasks` for work that must not be lost.**
- **Select-then-update claiming**, letting two workers take the same job.
- **No leases**: a crashed worker's job stays `running` forever.
- **Non-idempotent jobs** under at-least-once delivery.
- **Polling without `Retry-After`** or backoff, hammering the status endpoint.
- **Unbounded retries** — a poison job that always fails, retried forever. Cap attempts
  and alert on `failed`.

## Exercise

1. Add `max_attempts` and the backoff base to settings, and expose `GET /jobs?status=failed`
   for operators, with a `POST /jobs/{id}/retry` that re-queues a failed job.
2. Add a heartbeat: the worker extends `locked_until` every few seconds while a job runs.
   Prove with a 10-second job and a 3-second lease that it runs only once.
3. Push job completion to clients with Server-Sent Events (Level 3 lesson 6) instead of
   polling.
4. Port the store to PostgreSQL with `FOR UPDATE SKIP LOCKED`, or replace it with ARQ or
   Celery, and map each concept above (lease, retry, idempotency, result) to its option
   in that tool.
