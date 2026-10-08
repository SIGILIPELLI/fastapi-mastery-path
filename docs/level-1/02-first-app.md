---
description: "Install FastAPI in a virtual environment, run an app with fastapi dev and fastapi run, and read the generated /docs and /openapi.json."
---

# 02 · Installing FastAPI & Your First App

This lesson gets a real app running, shows what each command does, and walks through the
interactive documentation FastAPI generates for you.

## Install into a virtual environment

Always give each project its own environment so its package versions don't collide with
anything else on your machine:

```bash
mkdir bookshop && cd bookshop
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install "fastapi[standard]"
```

The `[standard]` extra pulls in the pieces most projects want:

- **fastapi-cli**, which provides the `fastapi` command;
- **uvicorn** with its faster optional dependencies (`uvloop`, `httptools`, `watchfiles`
  for auto-reload);
- **httpx** for the test client, **jinja2** for templates, **python-multipart** for form
  and file uploads, **email-validator** for the `EmailStr` type.

If you install plain `pip install fastapi`, you get the framework but no server and no
`fastapi` command. That's sometimes what you want for a library, but not for an
application.

Check what you have:

```bash
pip list | grep -iE "^(fastapi|fastapi-cli|starlette|pydantic|uvicorn) "
```

When this course was written, that printed `fastapi 0.143.0`, `fastapi-cli 0.0.32`,
`pydantic 2.14.0`, `starlette 1.7.0` and `uvicorn 0.54.0` (the order may vary). Your
numbers will be newer; record them in a `requirements.txt` or `pyproject.toml`.

## The smallest useful app

```python
# main.py
from fastapi import FastAPI

app = FastAPI(title="Bookshop API", version="0.1.0")


@app.get("/")
def root():
    return {"message": "Hello from the bookshop"}


@app.get("/health")
async def health():
    return {"status": "ok"}
```

- `app` is the ASGI application. The variable name matters only because commands find it
  by name.
- `title` and `version` go into the OpenAPI document; they have no effect on behaviour.
- One endpoint is `def`, one is `async def`. Both work; Level 2 lesson 3 explains exactly
  how they differ. For now: use `async def` only when the function awaits something.

## `fastapi dev`: development mode

```bash
fastapi dev main.py
```

The output from a real run (port changed with `--port 8702`):

```text
 ⚡️ Starting FastAPI in development mode

 🐍 Using import string: main:app

 🌐 Server started at http://127.0.0.1:8702
    Documentation at http://127.0.0.1:8702/docs

  Logs:

INFO:     Will watch for changes in these directories: ['/…/first']
INFO:     Uvicorn running on http://127.0.0.1:8702 (Press CTRL+C to quit)
INFO:     Started reloader process [70276] using WatchFiles
INFO:     Started server process [70279]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

Two things to notice:

- It listens on **127.0.0.1** — only your own machine can connect. That's a safe default
  for development.
- There are **two processes**: a reloader that watches files, and the server it starts.
  When you save a file, the reloader kills the server process and starts a new one. That's
  why module-level state (a global dict, a counter) resets every time you save.

In another terminal:

```bash
curl -s localhost:8702/
curl -s localhost:8702/health
```

```text
{"message":"Hello from the bookshop"}
{"status":"ok"}
```

and the server logs one line per request:

```text
INFO:     127.0.0.1:51469 - "GET / HTTP/1.1" 200 OK
INFO:     127.0.0.1:51471 - "GET /health HTTP/1.1" 200 OK
```

## `fastapi run`: production mode

```bash
fastapi run
```

```text
 ⚡️ Starting FastAPI in production mode

 🐍 Using import string: main:app (auto-discovered, use --verbose to learn more)

 💡 You can configure an entrypoint in pyproject.toml for this app with:

    [tool.fastapi]
    entrypoint = "main:app"

 🌐 Server started at http://0.0.0.0:8703
    Documentation at http://0.0.0.0:8703/docs
```

Differences from `dev`:

| | `fastapi dev` | `fastapi run` |
|---|---|---|
| Auto-reload | on | off |
| Default host | `127.0.0.1` | `0.0.0.0` (all interfaces) |
| Intended for | your laptop | containers and servers |
| Workers | 1 | 1 unless you pass `--workers N` |

Without a path argument, the CLI looked for a default file (it found `main.py`) and a
variable named `app` inside it. When your app lives inside a package, tell the CLI where
it is in `pyproject.toml`. This layout was tested:

```text
ep/
├── pyproject.toml
└── shop/
    ├── __init__.py
    └── api.py          # defines app = FastAPI(...)
```

```toml
[project]
name = "shop"
version = "0.1.0"

[tool.fastapi]
entrypoint = "shop.api:app"
```

Running `fastapi run` from `ep/` printed `Using import string: shop.api:app`. Running it
from a subdirectory that couldn't see `pyproject.toml` failed with
`Could not find a default file to run, please provide an explicit path` — the CLI reads
the config from the current directory, so run it from the project root.

### Uvicorn directly

`fastapi run` is a convenience wrapper around Uvicorn. The equivalent direct command is:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

`main:app` is an **import string**: module `main`, attribute `app`. You'll see this form
in Dockerfiles and process managers, and Level 4 lesson 2 uses it for multi-worker
deployments.

## The generated documentation

Open `http://127.0.0.1:8000/docs` in a browser. Swagger UI lists every endpoint; clicking
one and pressing **Try it out → Execute** sends a real request from your browser. `/redoc`
shows the same information in a read-only layout.

Both pages are drawn from `/openapi.json`. Here is the start of the real document for the
app above:

```json
{
    "openapi": "3.1.0",
    "info": {
        "title": "Bookshop API",
        "version": "0.1.0"
    },
    "paths": {
        "/": {
            "get": {
                "summary": "Root",
                "operationId": "root__get",
                "responses": {
                    "200": {
                        "description": "Successful Response",
                        "content": {
                            "application/json": {
                                "schema": {}
                            }
                        }
                    }
                }
            }
        },
```

- `summary` was derived from the function name `root`.
- `operationId` combines the function name, the path and the method; client generators
  use it to name methods (Level 3 lesson 7 shows how to make these readable).
- The response `schema` is `{}` — "anything" — because the function returns a plain dict
  with no declared type. Lesson 5 fixes that with response models.

## Worked example: a tiny bookshop catalogue

```python
from fastapi import FastAPI

app = FastAPI(title="Bookshop API", version="0.1.0")

BOOKS = [
    {"id": 1, "title": "Dune", "year": 1965},
    {"id": 2, "title": "Neuromancer", "year": 1984},
]


@app.get("/books")
def list_books():
    return BOOKS


@app.get("/books/count")
def count_books():
    return {"count": len(BOOKS)}
```

`GET /books` returns the list as a JSON array and `GET /books/count` returns
`{"count": 2}`. Add a third book while `fastapi dev` is running and save: the reloader
restarts the server and the count becomes `3`. Now add a book *at runtime* (you'll learn
how in lesson 4) and then save a file — the runtime addition is gone, because the module
was re-imported. This is the first sign that real data belongs in a database.

## How It Actually Works

`fastapi dev main.py` does roughly this:

1. Resolves `main.py` to an import string. If the file is inside a package (a directory
   with `__init__.py`), it walks upward to find the package root and adds that directory
   to `sys.path`, so `from .models import Book` style imports keep working.
2. Imports the module and looks for a `FastAPI` instance, unless you pass `--app NAME`.
   It isn't limited to the name `app`: with a file containing `server = FastAPI()`,
   `fastapi run --verbose` reported `from main import server` and
   `Using import string: main:server`. The `--verbose` flag prints each step of this
   discovery, which is the quickest way to debug "could not find app" errors.
3. Calls `uvicorn.run("main:app", reload=True, host="127.0.0.1", ...)`.

With `reload=True`, Uvicorn doesn't serve requests in the process you started. It starts a
**supervisor** that uses `watchfiles` to watch the directory, and spawns a child process
that imports your app and serves. On a file change it terminates the child and spawns a
new one. That's why reload needs an import string rather than an app object: the child
process must be able to import the app freshly.

The `0.0.0.0` default in production mode exists because inside a container, `127.0.0.1`
refers to the container itself, and traffic forwarded from outside would never reach a
server bound only to loopback.

## Common mistakes

- **`ModuleNotFoundError: No module named 'main'`** — you ran the command from a different
  directory than the file. Run it where `main.py` lives, or pass the path.
- **Using `fastapi dev` in production.** Auto-reload watches the filesystem and restarts
  the process; it adds overhead and its restarts drop in-flight requests.
- **Binding `0.0.0.0` on a laptop on public Wi-Fi.** Anyone on the network can reach your
  dev server. The `dev` default of `127.0.0.1` is deliberate.
- **Expecting module-level variables to persist.** Every reload, and every worker in a
  multi-worker setup, has its own copy.
- **Installing `fastapi` without `[standard]`** and then wondering why the `fastapi`
  command doesn't exist.

## Exercise

1. Create the bookshop project above in a fresh virtual environment. Record the exact
   versions `pip list` reports in a `requirements.txt` with `==` pins.
2. Run it with `fastapi dev`, then with `fastapi run --port 8001`. From a second terminal,
   find both processes with `ps aux | grep -i uvicorn` (or Task Manager) and explain the
   difference in process count.
3. Move `main.py` into a package `bookshop/api.py`, add the `[tool.fastapi]` entrypoint,
   and confirm `fastapi run` finds it without arguments.
4. Open `/docs`, use **Try it out** on `/books`, and copy the `curl` command Swagger UI
   shows you. Run it in a terminal.
