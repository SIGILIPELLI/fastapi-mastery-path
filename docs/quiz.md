---
title: "Final Quiz"
description: "Test what you have learned across all four levels of FastAPI Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this print?\n```python\nfrom fastapi import FastAPI, HTTPException, Depends, Query, Header, Request\nfrom fastapi.responses import JSONResponse\nfrom fastapi.testclient import TestClient\nfrom pydantic import BaseModel, Field\napp = FastAPI()\n@app.get('/items/{i}')\ndef get(i: int, q: str | None = None):\n    return {'i': i, 'q': q}\nc=TestClient(app)\nprint(c.get('/items/3?q=x').json(), c.get('/items/abc').status_code)\n```\n",
    "options": [
      "{'i': '3', 'q': 'x'} 400",
      "{'i': 3} 404",
      "Error",
      "{'i': 3, 'q': 'x'} 422"
    ],
    "correctIndex": 3,
    "explanation": "Path params are converted; invalid ones give 422. Verified by running it with FastAPI's TestClient."
  },
  {
    "question": "What does this print?\n```python\nfrom fastapi import FastAPI, HTTPException, Depends, Query, Header, Request\nfrom fastapi.responses import JSONResponse\nfrom fastapi.testclient import TestClient\nfrom pydantic import BaseModel, Field\napp = FastAPI()\nclass U(BaseModel):\n    name: str\n    pw: str\nclass Out(BaseModel):\n    name: str\n@app.post('/u', response_model=Out)\ndef mk(u: U): return u\nc=TestClient(app)\nprint(c.post('/u', json={'name':'a','pw':'x'}).json())\n```\n",
    "options": [
      "{'name': 'a', 'pw': 'x'}",
      "{'name': 'a'}",
      "Error",
      "{}"
    ],
    "correctIndex": 1,
    "explanation": "response_model filters the output. Verified by running it with FastAPI's TestClient."
  },
  {
    "question": "What does this print?\n```python\nfrom fastapi import FastAPI, HTTPException, Depends, Query, Header, Request\nfrom fastapi.responses import JSONResponse\nfrom fastapi.testclient import TestClient\nfrom pydantic import BaseModel, Field\napp = FastAPI()\n@app.get('/n')\ndef n():\n    raise HTTPException(status_code=404, detail='nope')\nc=TestClient(app)\nr=c.get('/n')\nprint(r.status_code, r.json())\n```\n",
    "options": [
      "200 {}",
      "404 {'detail': 'nope'}",
      "500 {}",
      "404 nope"
    ],
    "correctIndex": 1,
    "explanation": "HTTPException → JSON error. Verified by running it with FastAPI's TestClient."
  },
  {
    "question": "What does this print?\n```python\nfrom fastapi import FastAPI, HTTPException, Depends, Query, Header, Request\nfrom fastapi.responses import JSONResponse\nfrom fastapi.testclient import TestClient\nfrom pydantic import BaseModel, Field\napp = FastAPI()\nev=[]\ndef dep():\n    ev.append('in')\n    try:\n        yield 1\n    finally:\n        ev.append('out')\n@app.get('/d')\ndef d(x=Depends(dep)):\n    ev.append('handler'); return x\nc=TestClient(app); c.get('/d'); print(ev)\n```\n",
    "options": [
      "['in', 'handler', 'out']",
      "['in', 'out', 'handler']",
      "['handler', 'in', 'out']",
      "['in']"
    ],
    "correctIndex": 0,
    "explanation": "Cleanup runs after the handler. Verified by running it with FastAPI's TestClient."
  },
  {
    "question": "What does this print?\n```python\nfrom fastapi import FastAPI, HTTPException, Depends, Query, Header, Request\nfrom fastapi.responses import JSONResponse\nfrom fastapi.testclient import TestClient\nfrom pydantic import BaseModel, Field\napp = FastAPI()\ndef real(): return 'real'\n@app.get('/w')\ndef w(x=Depends(real)): return x\nc=TestClient(app)\napp.dependency_overrides[real]=lambda: 'fake'\nprint(c.get('/w').json())\n```\n",
    "options": [
      "fake",
      "Error",
      "None",
      "real"
    ],
    "correctIndex": 0,
    "explanation": "Overrides are how tests replace dependencies. Verified by running it with FastAPI's TestClient."
  },
  {
    "question": "A client sends price=-5 to an endpoint whose model has Field(gt=0). What happens?",
    "options": [
      "200",
      "FastAPI returns 422 before your function runs",
      "Your function runs",
      "500"
    ],
    "correctIndex": 1,
    "explanation": "Validation errors list location and type."
  },
  {
    "question": "A def endpoint calls a blocking library and an async def one calls time.sleep(2). Which is the problem?",
    "options": [
      "The async def one blocks the event loop; def endpoints run in a thread pool",
      "def one",
      "Both fine",
      "Neither"
    ],
    "correctIndex": 0,
    "explanation": "Use await or run_in_threadpool."
  },
  {
    "question": "The model gains a column but the table isn't updated and queries fail. What did you skip?",
    "options": [
      "A dependency",
      "A migration (Alembic); create_all won't alter existing tables",
      "A restart",
      "A cache clear"
    ],
    "correctIndex": 1,
    "explanation": "Version your schema."
  },
  {
    "question": "Where should the database URL and JWT secret come from?",
    "options": [
      "Code",
      "README",
      "Environment variables/settings, never committed",
      "Docs"
    ],
    "correctIndex": 2,
    "explanation": "pydantic-settings."
  },
  {
    "question": "How do you give each request its own DB session and guarantee cleanup?",
    "options": [
      "A module variable",
      "Middleware only",
      "A global",
      "A yield dependency with try/finally"
    ],
    "correctIndex": 3,
    "explanation": "Dependency lifecycle."
  },
  {
    "question": "A list endpoint returns 2 million rows. What do you add?",
    "options": [
      "A cache only",
      "More RAM",
      "Compression",
      "Pagination with validated limit/offset (or cursors) and an index on the sort key"
    ],
    "correctIndex": 3,
    "explanation": "Bound response size."
  },
  {
    "question": "How should passwords be stored for a login endpoint?",
    "options": [
      "As salted, slow hashes (bcrypt/argon2), compared in constant time",
      "Plain",
      "MD5",
      "Base64"
    ],
    "correctIndex": 0,
    "explanation": "Never reversible."
  },
  {
    "question": "A JWT's signature is valid but exp is in the past. What do you do?",
    "options": [
      "Log only",
      "Accept",
      "Reject it (401); always verify expiry as well as the signature",
      "Refresh silently"
    ],
    "correctIndex": 2,
    "explanation": "Short lifetimes."
  },
  {
    "question": "A logged-in user requests an admin endpoint. Which status do you return?",
    "options": [
      "401",
      "403 Forbidden",
      "200",
      "404"
    ],
    "correctIndex": 1,
    "explanation": "401 means unauthenticated."
  },
  {
    "question": "A browser app on another origin gets blocked calling your API. What do you configure?",
    "options": [
      "CORSMiddleware with explicit allowed origins (not '*' with credentials)",
      "DB",
      "Proxy",
      "Auth"
    ],
    "correctIndex": 0,
    "explanation": "CORS is browser-enforced."
  },
  {
    "question": "An email must be sent after the response, but it takes 20 s and must survive a crash. What do you use?",
    "options": [
      "BackgroundTasks",
      "sleep",
      "Threads",
      "A job queue (Celery/ARQ) rather than BackgroundTasks"
    ],
    "correctIndex": 3,
    "explanation": "Durable, retryable."
  },
  {
    "question": "Behind Nginx, client IPs and redirects look wrong. What do you set?",
    "options": [
      "Auth",
      "Proxy headers handling (--proxy-headers / forwarded-allow-ips) and root_path if mounted",
      "CORS",
      "DB"
    ],
    "correctIndex": 1,
    "explanation": "X-Forwarded-* headers."
  },
  {
    "question": "A release renames a response field. How do you avoid breaking clients?",
    "options": [
      "Version the API (/v1 → /v2), keep v1 working and deprecate with notice",
      "Rename silently",
      "Break",
      "Remove v1"
    ],
    "correctIndex": 0,
    "explanation": "Additive changes first."
  },
  {
    "question": "A Pydantic v1 codebase must move to v2. What's the safest approach?",
    "options": [
      "Rewrite",
      "Pin versions, read the migration guide, update APIs (model_dump, field_validator) and rely on tests",
      "Upgrade blindly",
      "Skip"
    ],
    "correctIndex": 1,
    "explanation": "CI protects upgrades."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A validated notes API

Complete create, read and list_notes: create stores a note and returns only id and title (the response model hides the body), read raises the NotFound domain exception handled as a JSON 404, and list_notes paginates with validated limit/offset. The script exercises the API with TestClient.

**Starter code**

```python
from fastapi import FastAPI, Depends, Query, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field

app = FastAPI()

class NoteIn(BaseModel):
    title: str = Field(min_length=1, max_length=50)
    body: str = ""

class NoteOut(BaseModel):
    id: int
    title: str

class NotFound(Exception):
    def __init__(self, note_id):
        self.note_id = note_id

@app.exception_handler(NotFound)
def not_found(request: Request, exc: NotFound):
    return JSONResponse(status_code=404, content={"error": "note_not_found", "id": exc.note_id})

class Store:
    def __init__(self):
        self.notes = {}
        self.next_id = 1

def get_store():
    return STORE

STORE = Store()

@app.post("/notes", response_model=NoteOut, status_code=201)
def create(note: NoteIn, store: Store = Depends(get_store)):
    # assign the next id, store {id, title, body}; the response model must hide the body
    return {"id": 0, "title": ""}

@app.get("/notes/{note_id}", response_model=NoteOut)
def read(note_id: int, store: Store = Depends(get_store)):
    # raise NotFound(note_id) when the note doesn't exist
    return {"id": 0, "title": ""}

@app.get("/notes", response_model=list[NoteOut])
def list_notes(limit: int = Query(2, ge=1, le=10), offset: int = Query(0, ge=0), store: Store = Depends(get_store)):
    # notes ordered by id, sliced with offset and limit
    return []

c = TestClient(app)
r = c.post("/notes", json={"title": "a", "body": "secret body"})
print(r.status_code, r.json())
c.post("/notes", json={"title": "b"})
c.post("/notes", json={"title": "c"})
print([n["title"] for n in c.get("/notes").json()], [n["title"] for n in c.get("/notes?limit=2&offset=2").json()])
r = c.get("/notes/99")
print(r.status_code, r.json())
r = c.post("/notes", json={"title": ""})
print(r.status_code, r.json()["detail"][0]["loc"])
print(c.get("/notes?limit=0").status_code)
```

**Expected output**

```text
201 {'id': 1, 'title': 'a'}
['a', 'b'] ['c']
404 {'error': 'note_not_found', 'id': 99}
422 ['body', 'title']
422
```

<details><summary>Show solution and explanation</summary>

<pre><code>from fastapi import FastAPI, Depends, Query, Request
from fastapi.responses import JSONResponse
from fastapi.testclient import TestClient
from pydantic import BaseModel, Field

app = FastAPI()

class NoteIn(BaseModel):
    title: str = Field(min_length=1, max_length=50)
    body: str = &quot;&quot;

class NoteOut(BaseModel):
    id: int
    title: str

class NotFound(Exception):
    def __init__(self, note_id):
        self.note_id = note_id

@app.exception_handler(NotFound)
def not_found(request: Request, exc: NotFound):
    return JSONResponse(status_code=404, content={&quot;error&quot;: &quot;note_not_found&quot;, &quot;id&quot;: exc.note_id})

class Store:
    def __init__(self):
        self.notes = {}
        self.next_id = 1

def get_store():
    return STORE

STORE = Store()

@app.post(&quot;/notes&quot;, response_model=NoteOut, status_code=201)
def create(note: NoteIn, store: Store = Depends(get_store)):
    nid = store.next_id
    store.next_id += 1
    store.notes[nid] = {&quot;id&quot;: nid, &quot;title&quot;: note.title, &quot;body&quot;: note.body}
    return store.notes[nid]

@app.get(&quot;/notes/{note_id}&quot;, response_model=NoteOut)
def read(note_id: int, store: Store = Depends(get_store)):
    if note_id not in store.notes:
        raise NotFound(note_id)
    return store.notes[note_id]

@app.get(&quot;/notes&quot;, response_model=list[NoteOut])
def list_notes(limit: int = Query(2, ge=1, le=10), offset: int = Query(0, ge=0), store: Store = Depends(get_store)):
    items = [store.notes[k] for k in sorted(store.notes)]
    return items[offset: offset + limit]

c = TestClient(app)
r = c.post(&quot;/notes&quot;, json={&quot;title&quot;: &quot;a&quot;, &quot;body&quot;: &quot;secret body&quot;})
print(r.status_code, r.json())
c.post(&quot;/notes&quot;, json={&quot;title&quot;: &quot;b&quot;})
c.post(&quot;/notes&quot;, json={&quot;title&quot;: &quot;c&quot;})
print([n[&quot;title&quot;] for n in c.get(&quot;/notes&quot;).json()], [n[&quot;title&quot;] for n in c.get(&quot;/notes?limit=2&amp;offset=2&quot;).json()])
r = c.get(&quot;/notes/99&quot;)
print(r.status_code, r.json())
r = c.post(&quot;/notes&quot;, json={&quot;title&quot;: &quot;&quot;})
print(r.status_code, r.json()[&quot;detail&quot;][0][&quot;loc&quot;])
print(c.get(&quot;/notes?limit=0&quot;).status_code)
</code></pre>

<p>The response model keeps `body` out of responses, a domain exception handler gives a consistent 404 shape, and Query(ge=, le=) validates pagination before your code runs. Verified by running the solution with FastAPI&#x27;s TestClient.</p>
</details>

### Authentication, roles and ownership

Implement current_user (401 for missing or unknown Bearer tokens), require_role (a dependency factory that raises 403) and the document endpoint's ownership rule (return 404, not 403, for documents the caller may not see).

**Starter code**

```python
from fastapi import FastAPI, Depends, Header, HTTPException
from fastapi.testclient import TestClient

app = FastAPI()

USERS = {"tok-ada": {"name": "ada", "role": "admin"}, "tok-bo": {"name": "bo", "role": "user"}}
DOCS = {1: {"owner": "ada", "text": "ada's doc"}, 2: {"owner": "bo", "text": "bo's doc"}}

def current_user(authorization: str | None = Header(default=None)):
    # 401 when the header is missing/not Bearer or the token is unknown; otherwise return the user dict
    return {"name": "", "role": ""}

def require_role(role):
    # return a dependency that depends on current_user and raises 403 unless user["role"] == role
    def checker(user=Depends(current_user)):
        return user
    return checker

@app.get("/docs/{doc_id}")
def read_doc(doc_id: int, user=Depends(current_user)):
    # 404 if missing; 404 (not 403) if the doc belongs to someone else and the user isn't an admin
    return {}

@app.get("/admin/stats")
def stats(user=Depends(require_role("admin"))):
    return {"docs": len(DOCS)}

c = TestClient(app)
def show(r): print(r.status_code, r.json()["detail"] if r.status_code >= 400 else r.json())
show(c.get("/docs/1"))
show(c.get("/docs/1", headers={"Authorization": "Bearer nope"}))
show(c.get("/docs/1", headers={"Authorization": "Bearer tok-ada"}))
show(c.get("/docs/1", headers={"Authorization": "Bearer tok-bo"}))
show(c.get("/docs/2", headers={"Authorization": "Bearer tok-ada"}))
show(c.get("/admin/stats", headers={"Authorization": "Bearer tok-bo"}))
show(c.get("/admin/stats", headers={"Authorization": "Bearer tok-ada"}))
```

**Expected output**

```text
401 missing token
401 invalid token
200 {'owner': 'ada', 'text': "ada's doc"}
404 no such doc
200 {'owner': 'bo', 'text': "bo's doc"}
403 forbidden
200 {'docs': 2}
```

<details><summary>Show solution and explanation</summary>

<pre><code>from fastapi import FastAPI, Depends, Header, HTTPException
from fastapi.testclient import TestClient

app = FastAPI()

USERS = {&quot;tok-ada&quot;: {&quot;name&quot;: &quot;ada&quot;, &quot;role&quot;: &quot;admin&quot;}, &quot;tok-bo&quot;: {&quot;name&quot;: &quot;bo&quot;, &quot;role&quot;: &quot;user&quot;}}
DOCS = {1: {&quot;owner&quot;: &quot;ada&quot;, &quot;text&quot;: &quot;ada&#x27;s doc&quot;}, 2: {&quot;owner&quot;: &quot;bo&quot;, &quot;text&quot;: &quot;bo&#x27;s doc&quot;}}

def current_user(authorization: str | None = Header(default=None)):
    if not authorization or not authorization.startswith(&quot;Bearer &quot;):
        raise HTTPException(status_code=401, detail=&quot;missing token&quot;, headers={&quot;WWW-Authenticate&quot;: &quot;Bearer&quot;})
    user = USERS.get(authorization.removeprefix(&quot;Bearer &quot;))
    if user is None:
        raise HTTPException(status_code=401, detail=&quot;invalid token&quot;)
    return user

def require_role(role):
    def checker(user=Depends(current_user)):
        if user[&quot;role&quot;] != role:
            raise HTTPException(status_code=403, detail=&quot;forbidden&quot;)
        return user
    return checker

@app.get(&quot;/docs/{doc_id}&quot;)
def read_doc(doc_id: int, user=Depends(current_user)):
    doc = DOCS.get(doc_id)
    if doc is None:
        raise HTTPException(status_code=404, detail=&quot;no such doc&quot;)
    if doc[&quot;owner&quot;] != user[&quot;name&quot;] and user[&quot;role&quot;] != &quot;admin&quot;:
        raise HTTPException(status_code=404, detail=&quot;no such doc&quot;)   # don&#x27;t leak existence
    return doc

@app.get(&quot;/admin/stats&quot;)
def stats(user=Depends(require_role(&quot;admin&quot;))):
    return {&quot;docs&quot;: len(DOCS)}

c = TestClient(app)
def show(r): print(r.status_code, r.json()[&quot;detail&quot;] if r.status_code &gt;= 400 else r.json())
show(c.get(&quot;/docs/1&quot;))
show(c.get(&quot;/docs/1&quot;, headers={&quot;Authorization&quot;: &quot;Bearer nope&quot;}))
show(c.get(&quot;/docs/1&quot;, headers={&quot;Authorization&quot;: &quot;Bearer tok-ada&quot;}))
show(c.get(&quot;/docs/1&quot;, headers={&quot;Authorization&quot;: &quot;Bearer tok-bo&quot;}))
show(c.get(&quot;/docs/2&quot;, headers={&quot;Authorization&quot;: &quot;Bearer tok-ada&quot;}))
show(c.get(&quot;/admin/stats&quot;, headers={&quot;Authorization&quot;: &quot;Bearer tok-bo&quot;}))
show(c.get(&quot;/admin/stats&quot;, headers={&quot;Authorization&quot;: &quot;Bearer tok-ada&quot;}))
</code></pre>

<p>Authentication (who are you) is separate from authorization (what may you do): roles gate endpoints, ownership gates objects, and returning 404 for others&#x27; documents avoids revealing they exist. Verified by running the solution with FastAPI&#x27;s TestClient.</p>
</details>
