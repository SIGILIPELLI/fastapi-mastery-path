---
description: "Run FastAPI in production: worker processes, crash recovery by the supervisor, graceful shutdown, Gunicorn with Uvicorn workers and a container."
---

# 02 · Running in Production: Workers, Gunicorn & Containers

`fastapi dev` is for your laptop. In production the questions are different: how many
processes, what happens when one crashes, what happens to in-flight requests during a
deploy, and how the app is packaged. This lesson answers each one with a run you can
repeat.

## Worker processes

One Uvicorn process uses one CPU core for Python code (lesson 1 measured what that means
for CPU-bound endpoints). To use more cores, run several **worker processes**:

```bash
fastapi run app.py --port 8715 --workers 3
```

The startup log:

```text
INFO:     Uvicorn running on http://0.0.0.0:8715 (Press CTRL+C to quit)
INFO:     Started parent process [74361]
INFO:     Started server process [74366]
INFO:     Started server process [74365]
INFO:     Started server process [74364]
INFO:     Waiting for application startup.
...
```

and the process tree (`ps -o pid,ppid,command`, trimmed):

```text
74361     1  ... Python .venv/bin/fastapi run ...            ← parent (supervisor)
74363 74361  ... -c from multiprocessing.resource_tracker import main;main(6)
74364 74361  ... -c from multiprocessing...                   ← worker
74365 74361  ... -c from multiprocessing...                   ← worker
74366 74361  ... -c from multiprocessing...                   ← worker
```

The parent doesn't serve requests; it starts and watches the workers (plus Python's
multiprocessing resource tracker). Each worker is a complete, separate copy of your app
with its **own lifespan, own connection pool, own in-memory caches and rate limiters**.
Six sequential `curl` requests to a `/pid` endpoint:

```text
   1 {"pid":74364}
   5 {"pid":74366}
```

The operating system decides which worker accepts each new connection, so the split is
uneven for a handful of requests and evens out under load. Never assume consecutive
requests from one client hit the same worker.

### How many workers?

A common starting point is one worker per CPU core for mixed workloads, then measure.
Considerations:

- **Memory**: each worker loads everything. A 500 MB ML model × 8 workers is 4 GB.
- **Database connections**: pool size × workers × replicas must fit under the
  database's connection limit.
- **Containers**: in Kubernetes and similar platforms, a common pattern is **one worker
  per container** and scaling by adding containers, letting the orchestrator handle
  restarts and load balancing. Multiple workers per container is fine on a single VM.

## What happens when a worker dies

A worker was killed with `kill -9` (simulating a crash or an out-of-memory kill):

```text
INFO:     Waiting for child process [74366]
INFO:     Child process [74366] died
INFO:     Started server process [74399]
```

The parent noticed and started a replacement; afterwards requests were served by
`74364` and the new `74399`. Requests that were in progress on the killed worker were
lost — a client would see a dropped connection — which is why clients should retry
idempotent requests and why one-at-a-time crashes shouldn't take the service down.

## Graceful shutdown

Deploys stop old processes. A request to an endpoint that sleeps for 3 seconds was
started, and half a second later the parent received `SIGTERM` (what Docker, systemd and
Kubernetes send):

```text
{"finished":true,"pid":74399}curl exit=0
...
INFO:     Waiting for connections to close. (CTRL+C to force quit)
INFO:     127.0.0.1:56539 - "GET /slow HTTP/1.1" 200 OK
INFO:     Waiting for application shutdown.
INFO:     Application shutdown complete.
INFO:     Finished server process [74399]
INFO:     Stopping parent process [74361]
```

The worker stopped accepting new connections, **let the in-flight request finish** (the
client got its 200), ran the lifespan shutdown code, and exited. Two settings decide
how long that's allowed to take:

- Uvicorn's `--timeout-graceful-shutdown N` caps the wait for in-flight requests.
- The platform's grace period (Docker's `stop` timeout, Kubernetes'
  `terminationGracePeriodSeconds`) — after which it sends `SIGKILL`. Make the app's
  timeout shorter than the platform's.

Long-lived connections (WebSockets, SSE streams) keep a worker in "waiting for
connections to close" until the timeout; plan for clients to reconnect.

## Gunicorn as the process manager

Before Uvicorn had its own `--workers` supervisor, the standard production setup was
**Gunicorn** managing Uvicorn workers. It's still common, and adds features such as
`max_requests` (recycle workers periodically, which caps slow memory leaks) and a mature
configuration system:

```bash
gunicorn app:app -k uvicorn.workers.UvicornWorker -w 2 -b 127.0.0.1:8716 --graceful-timeout 10
```

```text
[2026-10-08 22:53:04 +0530] [74432] [INFO] Starting gunicorn 26.2.0
[2026-10-08 22:53:04 +0530] [74432] [INFO] Listening at: http://127.0.0.1:8716 (74432)
[2026-10-08 22:53:04 +0530] [74432] [INFO] Using worker: uvicorn.workers.UvicornWorker
[2026-10-08 22:53:04 +0530] [74434] [INFO] Booting worker with pid: 74434
[2026-10-08 22:53:04 +0530] [74435] [INFO] Booting worker with pid: 74435
...
[2026-10-08 22:53:05 +0530] [74434] [INFO] Application startup complete.
```

It worked, but the `uvicorn.workers` module in Uvicorn 0.54.0 contains:

```python
warnings.warn(
    "The `uvicorn.workers` module is deprecated. Please use `uvicorn-worker` package instead.\n"
    ...
    DeprecationWarning,
```

`DeprecationWarning`s are hidden by default, so the log didn't show it. For new setups,
install the separate `uvicorn-worker` package and use its worker class (check its README
for the exact class path), or simply use `fastapi run --workers` / `uvicorn --workers`.
Gunicorn 26.2.0 also opened a control socket under `~/.gunicorn/` in this run — worth
knowing if your container's home directory is read-only.

## Worked example: a container image

A Dockerfile for the Level 3 bookmarks project. **Docker wasn't available on the machine
used for this course, so this file was not built or run**; it follows the patterns in
the [Docker Mastery Path](https://sigilipelli.github.io/docker-mastery-path/) and the
behaviour tested above.

```dockerfile
FROM python:3.14-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

WORKDIR /app

# Dependencies first, so code changes don't invalidate this layer.
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app ./app
COPY migrations ./migrations
COPY alembic.ini .

# Don't run as root.
RUN useradd --create-home appuser
USER appuser

EXPOSE 8000
# Exec form so the server is PID 1 and receives SIGTERM directly.
CMD ["fastapi", "run", "app/main.py", "--port", "8000", "--workers", "1", "--proxy-headers"]
```

Notes on the choices:

- **Pinned dependencies** in `requirements.txt` (`fastapi==0.143.0`, …) — FastAPI's 0.x
  versioning means minor releases can change behaviour.
- **Exec-form `CMD`** (`["fastapi", ...]`, not a shell string). With the shell form,
  `/bin/sh` is PID 1 and may not forward `SIGTERM`, so the graceful shutdown above never
  happens and the container is killed after the timeout.
- **Migrations run separately**, e.g. a one-off `docker run ... alembic upgrade head`
  or a Kubernetes Job before the rollout (Level 2 lesson 6), not in `CMD`.
- **One worker**, scaling with more containers. Change `--workers` if you run on a VM.
- `--proxy-headers` is for running behind a load balancer — lesson 3 explains what it
  does and the setting that must go with it.
- Secrets (`MARKS_JWT_SECRET`) come from the platform's secret store as environment
  variables, never baked into the image.

## How It Actually Works

With `--workers N`, Uvicorn's parent process creates the listening socket **once**, then
starts N child processes with `multiprocessing` (spawn), passing the socket to each.
Every child runs its own event loop and calls `accept()` on the same socket; the kernel
hands each new connection to one of the waiting processes. That's why distribution is
up to the OS. The parent then monitors the children and replaces any that exit
unexpectedly — the `Child process [...] died` / `Started server process` pair.

On `SIGTERM`, the parent signals the children. Each child stops accepting, sets its
connections to close after their current response (HTTP keep-alive is turned off),
waits for in-flight requests (bounded by `--timeout-graceful-shutdown`), runs the
lifespan shutdown, and exits; the parent exits when all children are gone.

Gunicorn's model is the same "pre-fork" design: a master process owns the socket and
manages workers; the `UvicornWorker` class runs an Uvicorn server inside each Gunicorn
worker.

## Common mistakes

- **`fastapi dev` or `--reload` in production.**
- **Shell-form `CMD`**, so `SIGTERM` doesn't reach the server.
- **More workers than the database can handle** connections for.
- **State that assumes one process**: in-memory sessions, caches, rate limiters, WebSocket
  rooms (Level 3 lessons 5, 6 and 9).
- **Running migrations from every worker at startup.**
- **Grace periods that don't line up**, so requests are cut off mid-response on every
  deploy.
- **Running as root** in the container.

## Exercise

1. Start your Level 3 project with `--workers 3`, call a `/pid` endpoint 50 times, and
   count requests per worker.
2. Log in once and then call `/auth/token` repeatedly with a wrong password. With three
   workers, how many attempts get through before a 429? Explain using lesson 9 of
   Level 3.
3. Measure graceful shutdown: start a 10-second request, send `SIGTERM`, and try it with
   `--timeout-graceful-shutdown 2`. What does the client see?
4. If you have Docker, build the image above, run it with `docker run` and
   `docker stop`, and confirm from the logs that shutdown was graceful.
