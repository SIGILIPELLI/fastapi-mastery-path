---
description: "Upgrade FastAPI projects safely: Pydantic v1-to-v2 changes that break silently, bump-pydantic pitfalls, warnings as errors and an upgrade routine."
---

# 09 · Upgrading: Pydantic v1 to v2 and FastAPI Releases

FastAPI is still on `0.x` version numbers and releases often; Pydantic went through a
major rewrite (v1 → v2) that changed both APIs and behaviour. A FastAPI codebase that
isn't upgraded for a year or two accumulates exactly the kind of debt that turns a
routine security patch into a project. This lesson shows what actually happens when old
code meets new libraries — including changes that **don't** raise errors — and a routine
that keeps upgrades small.

## Old Pydantic code on Pydantic v2

A model written in Pydantic v1 style:

```python
from typing import Optional
from pydantic import BaseModel, validator, root_validator


class Book(BaseModel):
    title: str
    subtitle: Optional[str]
    price: float

    class Config:
        orm_mode = True
        allow_population_by_field_name = True
        anystr_strip_whitespace = True

    @validator("title")
    def title_not_blank(cls, v):
        if not v.strip():
            raise ValueError("blank title")
        return v

    @root_validator
    def cheap_books_need_subtitle(cls, values):
        return values
```

Importing it on Pydantic 2.14.0 **failed** outright:

```text
pydantic.errors.PydanticUserError: If you use `@root_validator` with pre=False (the default) you MUST specify `skip_on_failure=True`. Note that `@root_validator` is deprecated and should be replaced with `@model_validator`.
```

After adding `skip_on_failure=True` it imported, with these warnings (paths trimmed):

```text
legacy_models.py:15: PydanticDeprecatedSince20: Pydantic V1 style `@validator` validators are deprecated. You should migrate to Pydantic V2 style `@field_validator` validators, ...
legacy_models.py:21: PydanticDeprecatedSince20: Pydantic V1 style `@root_validator` validators are deprecated. ...
legacy_models.py:5: PydanticDeprecatedSince20: Support for class-based `config` is deprecated, use ConfigDict instead. ...
UserWarning: Valid config keys have changed in V2:
* 'allow_population_by_field_name' has been renamed to 'validate_by_name'
* 'anystr_strip_whitespace' has been renamed to 'str_strip_whitespace'
* 'orm_mode' has been renamed to 'from_attributes'
```

The warnings are the easy part. Here is what changed **silently**:

```text
Book(title='Dune', price=9.99)
  -> ValidationError [('missing', ('subtitle',))]
Book(title='  Dune ', subtitle=None, price='9.99')
  -> ok: Book(title='  Dune ', subtitle=None, price=9.99)
```

1. **`Optional[str]` no longer implies a default.** In v1, `subtitle: Optional[str]`
   meant "may be `None`, defaults to `None`". In v2 it means "may be `None`" — but it's
   *required*. Every API client that omitted `subtitle` now gets a 422.
2. **The renamed config keys were not applied.** The warning says the keys were
   *renamed*, but the old names were ignored: the title kept its spaces. The strip
   behaviour your clients relied on is simply gone, with only a `UserWarning` (which many
   setups never display) to say so.

The `.dict()` method still worked — with a `PydanticDeprecatedSince20` warning (Level 1
lesson 6 showed it).

## Worked example: what `bump-pydantic` produced

`bump-pydantic` (0.8.0 here) is the official codemod for v1 → v2 migrations. Its diff for
the file above:

```diff
-from pydantic import BaseModel, validator, root_validator
+from pydantic import field_validator, model_validator, ConfigDict, BaseModel

 class Book(BaseModel):
     title: str
-    subtitle: Optional[str]
+    subtitle: Optional = None
     price: float
+    model_config = ConfigDict(from_attributes=True, populate_by_name=True, str_strip_whitespace=True)

-    class Config:
-        orm_mode = True
-        allow_population_by_field_name = True
-        anystr_strip_whitespace = True
-
-    @validator("title")
+    @field_validator("title")
+    @classmethod
     def title_not_blank(cls, v):
 ...
-    @root_validator(skip_on_failure=True)
+    @model_validator(skip_on_failure=True)
+    @classmethod
     def cheap_books_need_subtitle(cls, values):
```

(Lightly condensed; the `-` line for `subtitle` is the original source.) Most of it is
right: validators renamed, `@classmethod` added, config moved to `ConfigDict`, and
`= None` added to keep the v1 default. But two lines are wrong:

- `subtitle: Optional = None` lost the `[str]` — the field now accepts any type.
- `@model_validator(skip_on_failure=True)` doesn't exist in v2. Importing the converted
  file failed: `TypeError: model_validator() got an unexpected keyword argument 'skip_on_failure'`.

It also chose `populate_by_name`, an older spelling of `validate_by_name`. Treat codemods
as a first draft: run them, then read every change and run the tests. The hand-finished
version:

```python
from pydantic import BaseModel, ConfigDict, field_validator, model_validator


class Book(BaseModel):
    model_config = ConfigDict(from_attributes=True, validate_by_name=True,
                              str_strip_whitespace=True)

    title: str
    subtitle: str | None = None          # v1's Optional[...] implied "= None"; v2 doesn't
    price: float

    @field_validator("title")
    @classmethod
    def title_not_blank(cls, v: str) -> str:
        if not v:
            raise ValueError("blank title")
        return v

    @model_validator(mode="after")
    def cheap_books_need_subtitle(self):
        return self
```

Run with `python -W error` (every warning is an exception), it imported cleanly and
behaved like the v1 original:

```text
Book(title='Dune', subtitle=None, price=9.99)
{'title': 'Dune', 'subtitle': None, 'price': 1.0}
```

The stripped title is back, `subtitle` is optional again, and there are no warnings.
Note `if not v:` replaced `if not v.strip():` — with `str_strip_whitespace`, the value is
already stripped when the validator runs.

Pydantic v2 also ships the old API as `pydantic.v1` (it reported version 1.10.26
here), which helps migrate non-FastAPI code module by module. But **FastAPI itself no
longer accepts them**: declaring an endpoint with a `pydantic.v1` model as its body
failed on 0.143.0 with
`PydanticV1NotSupportedError: pydantic.v1 models are no longer supported by FastAPI. Please update the response model <class '__main__.M'>.`
Models used in endpoints must be migrated before you can upgrade FastAPI that far.

## Make deprecations fail in CI

Most of the warnings above are invisible in normal runs. Make the test suite treat them
as errors:

```toml
# pyproject.toml
[tool.pytest.ini_options]
filterwarnings = [
    "error::DeprecationWarning",
    "error::PendingDeprecationWarning",
]
```

An app still using `@app.on_event("startup")` then failed at collection:

```text
E   DeprecationWarning:
E           on_event is deprecated, use lifespan event handlers instead.
E
E           Read more about it in the
E           [FastAPI docs for Lifespan Events](https://fastapi.tiangolo.com/advanced/events/).
!!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
```

Pydantic's `PydanticDeprecatedSince20` subclasses `DeprecationWarning`, so those are
caught too. Two that aren't:

- Pydantic's "Valid config keys have changed" is a plain **`UserWarning`** — exactly the
  one that hid a behaviour change above. Add `"error::UserWarning"` if you can, or a
  specific filter on its message.
- Starlette's `StarletteDeprecationWarning` (the test-client `httpx2` notice in Level 1
  lesson 1) subclasses **`UserWarning`**, not `DeprecationWarning`, in Starlette 1.7.0.

When a third-party library emits a warning you can't fix yet, add a targeted `ignore`
for that message and module, with a comment and a date — not a blanket ignore.

## A routine that keeps upgrades boring

1. **Pin exact versions** (`fastapi==0.143.0`, `pydantic==2.14.0`, …) in a lock file or
   `requirements.txt`, so builds are reproducible and upgrades are deliberate.
2. **Upgrade often, a little at a time** — monthly or with each release — rather than
   jumping a year of releases. Tools like Dependabot or Renovate open the pull requests.
3. **Read the release notes** for FastAPI, Starlette and Pydantic between your version
   and the target. FastAPI's 0.x minor releases have contained behaviour changes before
   (this course met several version-specific details: `include_router` internals,
   yield-dependency scopes, content-type handling for JSON bodies).
4. **Run the full suite with warnings as errors**, plus the OpenAPI breaking-change check
   from lesson 6 — a dependency upgrade can change your *schema*.
5. **Deploy to staging and run a load test** (lesson 1) for upgrades of the server stack.

## How It Actually Works

Pydantic v2's `BaseModel` metaclass still recognises v1 constructs to ease migration:
`@validator` and `@root_validator` are wrappers that emit `PydanticDeprecatedSince20` and
register v2-style validators; a nested `class Config` is read and converted, warning for
keys whose names changed — and those renamed keys are dropped rather than translated,
which is why stripping silently stopped. The `Optional` change isn't a compatibility
shim at all: v2 deliberately separated "nullable" from "has a default".

`filterwarnings = ["error::DeprecationWarning"]` installs a warnings filter that raises
instead of printing. Warnings raised at import time (decorators, class creation) surface
during collection, which is why the `on_event` example failed before any test ran.

## Common mistakes

- **Unpinned dependencies**, so a deploy picks up a new FastAPI release nobody tested.
- **Trusting a codemod's output** without reading the diff and running tests.
- **Missing the silent changes**: optional fields without defaults, renamed config keys.
- **Hiding warnings** in test output instead of failing on them.
- **Big-bang upgrades** across many releases at once.
- **Testing only that the app starts**, not that its OpenAPI schema and behaviour are
  unchanged.

## Exercise

1. Take a v1-style model from an old project (or the one above), run `bump-pydantic`,
   and list every line you had to fix by hand.
2. Write a test that asserts `Book(title="  Dune ", price=1).title == "Dune"` and
   `Book(title="Dune", price=1).subtitle is None`. Run it against the unmigrated model
   on Pydantic v2 and watch both catch the silent changes.
3. Add `filterwarnings` to the Level 3 project's `pyproject.toml` with
   `error::DeprecationWarning` and `error::UserWarning`, and fix or explicitly filter
   whatever fails.
4. Write an upgrade checklist for your team, including who reads which release notes and
   which checks must pass before merge.
