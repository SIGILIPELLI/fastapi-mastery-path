---
description: "Run FastAPI behind nginx or a load balancer: proxy headers, which IPs to trust, X-Forwarded-For spoofing, root_path prefixes and body limits."
---

# 03 · Behind a Reverse Proxy: root_path, Forwarded Headers & HTTPS

In production your app rarely talks to clients directly. A reverse proxy or load balancer
— nginx, Caddy, a cloud load balancer, a Kubernetes ingress — terminates TLS, and
forwards plain HTTP to Uvicorn. That changes three things your app sees:

1. **The client address** is the proxy's, unless the proxy passes the original in
   `X-Forwarded-For`.
2. **The scheme** is `http` (the proxy-to-app hop), even though the client used
   `https`, unless the proxy passes `X-Forwarded-Proto`.
3. **The path** may have a prefix the app doesn't know about, if the proxy serves the
   API under `/api`.

Get these wrong and you get `http://` links in an `https` site, redirect loops, rate
limits that put every user in one bucket, and audit logs full of the load balancer's IP.

No nginx was available on the machine used for this lesson, so the proxy was simulated
by sending the headers a proxy would add with `curl`, directly to Uvicorn 0.54.0. The
nginx configuration at the end is illustrative and was not run.

## The test endpoint

```python
from fastapi import FastAPI, Request

app = FastAPI()

@app.get("/books/{book_id}")
def book(book_id: int, request: Request):
    return {"client": request.client.host, "scheme": request.url.scheme,
            "url": str(request.url),
            "self": str(request.url_for("book", book_id=book_id)),
            "root_path": request.scope.get("root_path")}
```

Every request below carried the headers a TLS-terminating proxy would add:

```bash
curl -H "X-Forwarded-For: 203.0.113.7" -H "X-Forwarded-Proto: https" \
     -H "Host: shop.example.com" localhost:8717/books/1
```

## Proxy headers and who to trust

**Default settings** (`uvicorn app:app`):

```text
{"client":"203.0.113.7","scheme":"https","url":"https://shop.example.com/books/1","self":"https://shop.example.com/books/1","root_path":""}
```

The forwarded values were applied. Uvicorn's proxy-header support is on by default, but
it trusts the headers **only from `127.0.0.1`** — and `curl` was connecting from
localhost, playing the part of a proxy on the same machine.

**With `--no-proxy-headers`:**

```text
{"client":"127.0.0.1","scheme":"http","url":"http://shop.example.com/books/1",...}
```

The real connection details: client `127.0.0.1`, scheme `http`. A `url_for` link now says
`http://` — on an HTTPS site, that's mixed content or a redirect loop.

**With `--forwarded-allow-ips 10.0.0.5`** (trust only a proxy at that address):

```text
{"client":"127.0.0.1","scheme":"http",...}
```

The headers came from `127.0.0.1`, which is no longer trusted, so they were ignored.
This is the setting you need in practice: your proxy usually isn't on localhost (it's
another container, or a load balancer in the same network), so the default trusts
nothing useful. Set `--forwarded-allow-ips` (or the `FORWARDED_ALLOW_IPS` environment
variable) to your proxy's address or subnet.

## Worked example: spoofing `X-Forwarded-For`

`X-Forwarded-For` is a list: each proxy **appends** the address it received the request
from. A client can send its own fake header, and an honest proxy just appends to it:

```text
X-Forwarded-For: 1.2.3.4, 203.0.113.7
                 ^ client-supplied lie   ^ added by your proxy (the real client)
```

With the default (trust only the local proxy), that request produced:

```text
{"client":"203.0.113.7", ...}
```

Uvicorn walked the list from the right, skipping trusted proxies, and took the first
untrusted address — the real client. Now the tempting shortcut,
`--forwarded-allow-ips '*'` ("trust everyone"):

```text
X-Forwarded-For: 1.2.3.4, 203.0.113.7   -> {"client":"1.2.3.4", ...}
X-Forwarded-For: 1.2.3.4                -> {"client":"1.2.3.4", ...}
```

The client chose its own IP address. Every per-IP rate limit (Level 3 lesson 9), IP
allow-list and audit log is now under the attacker's control. `'*'` is only safe when
the app is unreachable except through the proxy — enforce that with network rules, not
hope.

## `root_path`: serving under a prefix

Suppose the proxy serves the API at `https://shop.example.com/api/...` and **strips**
`/api` before forwarding, so the app still receives `/books/1`. The app works, but every
URL it *generates* — `url_for`, `Location` headers, the docs page's link to
`openapi.json` — lacks the prefix. Tell the app about it:

```bash
uvicorn app:app --root-path /api
```

```text
GET /books/1      -> {"client":"203.0.113.7","scheme":"https","url":"https://shop.example.com/api/books/1","self":"https://shop.example.com/api/books/1","root_path":"/api"}
GET /api/books/1  -> {"detail":"Not Found"}
```

The app still matched the stripped path `/books/1`, and generated URLs now include
`/api`. A request that arrived *with* the prefix got a 404 — `--root-path` assumes the
proxy strips it. (If your proxy doesn't strip the prefix, mount the routers under `/api`
in the app instead, and don't set `root_path`.)

The docs adapt too. With `--root-path /api`, the Swagger page referenced
`url: '/api/openapi.json'` and the schema gained `servers: [{'url': '/api'}]`, so
"Try it out" sends requests to the prefixed URL the browser can actually reach.

## An nginx configuration to match

Not run here — adapt and test it in your environment:

```nginx
upstream bookshop { server app:8000; }

server {
    listen 443 ssl;
    server_name shop.example.com;
    # ssl_certificate / ssl_certificate_key ...

    client_max_body_size 5m;          # reject big bodies before they reach Python

    location /api/ {
        proxy_pass http://bookshop/;  # trailing slash: strips the /api/ prefix
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }

    location /api/jobs/ {             # long-lived streams (Level 3 lesson 6)
        proxy_pass http://bookshop/jobs/;
        proxy_buffering off;
        proxy_read_timeout 1h;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

with the app started as
`fastapi run app/main.py --root-path /api --forwarded-allow-ips <nginx address>`.
`$proxy_add_x_forwarded_for` appends the connecting address to any existing header —
exactly the behaviour the spoofing example relied on. `client_max_body_size` is the real
upload limit that Level 1 lesson 8 said the app itself doesn't have.

## How It Actually Works

Uvicorn's proxy-header support is an ASGI middleware it wraps around your app. For a
connection from a trusted address it rewrites the ASGI scope *before* your app sees it:
`scope["client"]` becomes the selected `X-Forwarded-For` entry, and `scope["scheme"]`
becomes the `X-Forwarded-Proto` value. Everything downstream — `request.client`,
`request.url`, `url_for`, Starlette's HTTPS redirect middleware — reads the scope, so it
all follows. To pick the client address, it walks the `X-Forwarded-For` list from the
right, skipping addresses in the trusted set; with `'*'` everything is trusted, so it
ends at the leftmost — client-controlled — entry.

`root_path` is also an ASGI scope key (the spec defines it for exactly this case). The
`--root-path` option sets it on every request. Starlette uses it when building URLs, and
FastAPI uses it for the docs' schema URL and the `servers` entry.

`Host` isn't rewritten by Uvicorn; it's taken from the request, which is why the proxy
must pass the original with `proxy_set_header Host $host`, and why
`TrustedHostMiddleware` (Level 3 lesson 4) is worth having.

## Common mistakes

- **`--forwarded-allow-ips '*'`** on an app reachable without the proxy.
- **Not setting `--forwarded-allow-ips` at all** when the proxy isn't on localhost, so
  every request looks like it came from the proxy over `http`.
- **Using the leftmost `X-Forwarded-For` entry** in your own code. Use
  `request.client.host` after configuring trust correctly.
- **Setting `root_path` when the proxy doesn't strip the prefix** (404s), or forgetting it
  when it does (broken links and docs).
- **No body-size limit at the proxy.**
- **Proxy timeouts shorter than your slowest legitimate request**, producing 502/504s
  that never appear in the app's logs.

## Exercise

1. Reproduce the four runs above. Then add `TrustedHostMiddleware` and send a request with
   a `Host` that isn't allowed.
2. Write a test that fails if the client address can be spoofed with
   `X-Forwarded-For`. The test client bypasses Uvicorn's proxy-header handling, so start
   Uvicorn in a subprocess and send real requests to it.
3. If you have nginx or Caddy available, run the configuration above in front of the
   bookmarks project and check `url_for` output, `/api/docs`, and the client IP in your
   logs.
4. Make the Level 3 login rate limiter key on `request.client.host` behind the proxy, and
   confirm that two different `X-Forwarded-For` clients get separate buckets only when
   trust is configured correctly.
