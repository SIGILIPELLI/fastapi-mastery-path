---
description: "Shape FastAPI's generated OpenAPI: metadata, tags, docstrings, examples, documented error responses, readable operation IDs and typed TypeScript clients."
---

# 07 · Shaping the OpenAPI Schema

FastAPI writes an OpenAPI document for you from your code. Out of the box it's correct
but bland: summaries derived from function names, operation IDs like `root__get`, no
examples, and error responses that only list the automatic 422. Since that document
drives Swagger UI, ReDoc, client generators, contract tests and API gateways, a little
care here pays off for every consumer of your API.

## App-level metadata

```python
from fastapi import FastAPI
from fastapi.routing import APIRoute

def operation_id(route: APIRoute) -> str:
    return f"{route.tags[0]}-{route.name}" if route.tags else route.name

tags_metadata = [
    {"name": "books", "description": "Browse and manage the catalogue."},
    {"name": "admin", "description": "Operations for staff. Requires the `admin` scope."},
]

app = FastAPI(
    title="Bookshop API",
    version="2.3.0",
    summary="Catalogue and orders for the bookshop.",
    description="Prices are in **cents**. All timestamps are UTC.",
    openapi_tags=tags_metadata,
    generate_unique_id_function=operation_id,
    servers=[{"url": "https://api.example.com", "description": "Production"}],
)
```

The resulting document:

```text
info: {'title': 'Bookshop API', 'summary': 'Catalogue and orders for the bookshop.', 'description': 'Prices are in **cents**. All timestamps are UTC.', 'version': '2.3.0'}
servers: [{'url': 'https://api.example.com', 'description': 'Production'}]
tags: ['books', 'admin']
```

- `description` is Markdown; Swagger UI renders it at the top of `/docs`. Put the things
  every client must know there — units, time zones, auth, rate limits.
- `version` is **your API's** version, not FastAPI's.
- `openapi_tags` sets the order and descriptions of tag groups.
- `servers` tells client generators and Swagger UI where the API lives. With it set,
  "Try it out" sends requests to that URL — leave it out (or list your dev server
  first) if you want `/docs` to hit the server it's served from.

## Per-endpoint documentation

```python
from typing import Annotated
from fastapi import APIRouter, HTTPException, Path
from pydantic import BaseModel, ConfigDict, Field

class BookIn(BaseModel):
    model_config = ConfigDict(json_schema_extra={
        "examples": [{"title": "Dune", "price_cents": 999}]})
    title: str = Field(min_length=1, description="Title as printed on the cover.")
    price_cents: int = Field(gt=0, description="Price in cents.", examples=[999])

class Book(BookIn):
    id: int

class Problem(BaseModel):
    detail: str

router = APIRouter(prefix="/books", tags=["books"])

@router.get("/{book_id}", response_model=Book,
            responses={404: {"model": Problem, "description": "No book with this ID"}})
def get_book(book_id: Annotated[int, Path(ge=1, examples=[42])]):
    """Fetch one book.

    Returns the current price. Archived books are not returned.
    """
    ...

@router.post("", response_model=Book, status_code=201, summary="Add a book")
def create_book(book: BookIn):
    ...

@router.get("/legacy/search", deprecated=True)
def legacy_search(q: str):
    return []

app.include_router(router)

@app.get("/internal/metrics", include_in_schema=False)
def metrics():
    return {}
```

What that produced, one line per operation:

```text
GET   /books/{book_id}       operationId=books-get_book    summary='Get Book'      deprecated=False responses=['200', '404', '422']
POST  /books                 operationId=books-create_book summary='Add a book'    deprecated=False responses=['201', '422']
GET   /books/legacy/search   operationId=books-legacy_search summary='Legacy Search' deprecated=True responses=['200', '422']
```

- **Docstrings become descriptions.** `get_book`'s description was
  `'Fetch one book.\n\nReturns the current price. Archived books are not returned.'`
  Without `summary=`, the summary is derived from the function name ("Get Book").
- **`responses=`** documents status codes the code raises. FastAPI can't infer that
  `HTTPException(404)` happens inside your function; without this, clients don't know a
  404 is possible or what its body looks like. `Problem` appeared in the schema list.
- **`status_code=201`** replaced the default `200` in the docs.
- **`deprecated=True`** is shown struck through in Swagger UI and marked in generated
  clients — the polite first step before removal (Level 4 lesson 6).
- **`include_in_schema=False`** hides `/internal/metrics` entirely.

### Examples and field docs

Field descriptions and examples flow straight into the schema:

```text
BookIn example: [{'price_cents': 999, 'title': 'Dune'}]
price field: {'type': 'integer', 'exclusiveMinimum': 0.0, 'title': 'Price Cents', 'description': 'Price in cents.', 'examples': [999]}
```

Swagger UI pre-fills "Try it out" with the model example. `gt=0` became
`exclusiveMinimum` — the validation rules *are* the documentation.

## Operation IDs that make good client method names

The default operation ID for `get_book` would be `get_book_books__book_id__get` —
unique, but an ugly method name in a generated SDK. The `generate_unique_id_function`
above produced `books-get_book`, and generators turn that into something like
`books.getBook()` or `booksGetBook()`. Operation IDs must be unique across the document;
including the tag guards against two routers with the same function name. Changing them
later breaks generated clients, so settle on a scheme early.

## Worked example: generating a typed TypeScript client

The document was saved with `json.dump(app.openapi(), ...)` and fed to the
`openapi-typescript` tool (7.13.0, run with Node 26 via `npx`):

```bash
npx -y openapi-typescript openapi.json -o schema.d.ts
```

```text
✨ openapi-typescript 7.13.0
🚀 openapi.json → schema.d.ts [21.5ms]
```

Part of the 248-line output:

```typescript
export interface operations {
    "books-get_book": {
        parameters: {
            query?: never;
            header?: never;
            path: {
                book_id: number;
            };
            cookie?: never;
        };
        requestBody?: never;
        responses: {
            /** @description Successful Response */
            200: { headers: { [name: string]: unknown; };
                   content: { "application/json": components["schemas"]["Book"]; }; };
            /** @description No book with this ID */
            404: { headers: { [name: string]: unknown; };
                   content: { "application/json": components["schemas"]["Problem"]; }; };
            /** @description Validation Error */
            422: { ...HTTPValidationError... };
        };
    };
```

(Condensed for width.) And the model:

```typescript
Book: {
    /**
     * Title
     * @description Title as printed on the cover.
     */
    title: string;
    /**
     * Price Cents
     * @description Price in cents.
     * @example 999
     */
    price_cents: number;
    /** Id */
    id: number;
};
```

Your Pydantic descriptions and examples became doc comments in the frontend's editor,
and the documented 404 became a typed response the frontend must handle. Regenerating in
CI whenever the API changes turns breaking changes into compile errors on the client side
instead of production bugs.

## Hiding or customising the docs

To disable the docs in production, turn off the schema URL:

```python
app = FastAPI(openapi_url=None)
```

```text
[('/openapi.json', 404), ('/docs', 404), ('/redoc', 404)]
```

All three disappeared, since the UIs depend on the schema. (Hiding docs is not security —
the endpoints still exist — but it reduces casual discovery of internal APIs. Driving this
from settings, per environment, is common.)

To add things FastAPI doesn't generate — vendor extensions, extra security schemes —
override `app.openapi` with a function that builds the schema once and caches it:

```python
from fastapi.openapi.utils import get_openapi

def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    schema = get_openapi(title="Bookshop API", version="1", routes=app.routes)
    schema["info"]["x-logo"] = {"url": "https://example.com/logo.png"}
    app.openapi_schema = schema
    return schema

app.openapi = custom_openapi
```

```text
{'title': 'Bookshop API', 'version': '1', 'x-logo': {'url': 'https://example.com/logo.png'}}
```

## How It Actually Works

`app.openapi()` walks every route that has `include_in_schema=True`. For each
`APIRoute` it combines: the HTTP method and path; summary and description (explicit, or
from the name and docstring); the parameters from the dependant tree (path, query,
header, cookie — including those contributed by dependencies); the request body schema;
the response model for the success status; anything in `responses=`; and the automatic
`422` entry whenever the route has parameters to validate.

Model schemas come from Pydantic's `model_json_schema()`, and are placed under
`components/schemas` and referenced with `$ref`. When a model's input and output shapes
differ (computed fields, defaults that are optional on input but always present on
output), FastAPI may emit separate `-Input` and `-Output` schemas.

The result is cached in `app.openapi_schema`; `/openapi.json` serves the cached dict.
That's why the custom function checks the cache first, and why routes added after the
first call won't appear unless you clear it.

## Common mistakes

- **Undocumented error responses.** If the code raises 404 or 409, document it with
  `responses=`.
- **Changing operation IDs casually** and breaking generated clients.
- **`servers` pointing only at production**, so Swagger UI on staging sends "Try it out"
  requests to production.
- **Examples that fail your own validation** — they confuse readers and break contract
  tests. Test them: `BookIn.model_validate(example)` for each.
- **Treating hidden docs as security.**
- **Huge `description` strings in code.** Keep long prose in a Markdown file and load it.

## Exercise

1. Add a shared `responses` dict for 401 and 403 to an admin router so every admin
   endpoint documents them.
2. Write a test that validates every `examples` entry in your models' JSON schemas with
   the model itself.
3. Generate TypeScript types with `openapi-typescript`, then change a field from `int`
   to `str` in a response model and compare the generated output.
4. Make docs available only when `settings.environment != "prod"`, and test both cases.
