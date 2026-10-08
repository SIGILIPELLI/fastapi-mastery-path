---
description: "Pydantic v2 for FastAPI work: Field constraints, reusable Annotated types, field and model validators, aliases, strict mode and custom serialization."
---

# 06 · Pydantic v2 in Depth: Fields, Validators & Serialization

FastAPI hands every byte of input and output to Pydantic. The better you know Pydantic,
the less validation code you write in your endpoints — and the fewer bugs reach your
database. Everything in this lesson runs without FastAPI, which is a good habit: test
your models directly and fast, and keep endpoints thin.

!!! note "Version"
    This lesson is about **Pydantic v2** (run on 2.14.0). v2 is a rewrite with a Rust
    core and a different API from v1: `model_validate` instead of `parse_obj`,
    `model_dump` instead of `.dict()`, `field_validator` instead of `validator`,
    `model_config` instead of `class Config`. Level 4 lesson 9 covers migrating.

## A realistic model

Here's a book model that uses most of the tools you'll need day to day:

```python
from datetime import date
from decimal import Decimal
from typing import Annotated
from pydantic import (BaseModel, ConfigDict, Field, field_validator,
                      computed_field, AfterValidator, BeforeValidator, field_serializer)

def strip_isbn(v: str) -> str:
    digits = v.replace("-", "").replace(" ", "")
    if len(digits) != 13 or not digits.isdigit():
        raise ValueError("ISBN-13 must have 13 digits")
    return digits

ISBN = Annotated[str, AfterValidator(strip_isbn)]
Tag = Annotated[
    str,
    BeforeValidator(lambda v: v.strip().lower() if isinstance(v, str) else v),
    Field(min_length=1, max_length=30),
]

class Book(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    title: str = Field(min_length=1)
    isbn: ISBN
    price: Decimal = Field(gt=0, max_digits=8, decimal_places=2)
    published: date
    tags: list[Tag] = Field(default_factory=list, max_length=5)
    page_count: int = Field(alias="pageCount", gt=0)

    @field_validator("published")
    @classmethod
    def not_in_future(cls, v: date) -> date:
        if v > date(2026, 10, 8):      # a fixed date so the example is reproducible
            raise ValueError("published date is in the future")
        return v

    @computed_field
    @property
    def isbn_prefix(self) -> str:
        return self.isbn[:3]

    @field_serializer("price")
    def price_as_str(self, v: Decimal) -> str:
        return f"{v:.2f}"
```

In a real app you'd compare against `date.today()`; the fixed date keeps the output below
reproducible.

Validate some messy input:

```python
data = {"title": "  Dune ", "isbn": "978-0-441-17271-9", "price": "9.99",
        "published": "1965-08-01", "tags": [" Classic", "SciFi "], "pageCount": 412}
b = Book.model_validate(data)
print(repr(b))
```

```text
Book(title='Dune', isbn='9780441172719', price=Decimal('9.99'), published=datetime.date(1965, 8, 1), tags=['classic', 'scifi'], page_count=412, isbn_prefix='978')
```

Every field was cleaned: whitespace stripped from the title (`str_strip_whitespace`),
hyphens removed from the ISBN, tags lower-cased, the price parsed into an exact
`Decimal`, the date string into a `date`.

### Reading the pieces

| Tool | What it's for |
|------|---------------|
| `Field(...)` | constraints (`gt`, `max_length`, `decimal_places`), defaults, `default_factory`, aliases, docs (`description`, `examples`) |
| `Annotated[T, ...]` | attach validators and constraints to a type, then reuse that type anywhere (`ISBN`, `Tag`) |
| `BeforeValidator` | runs on the raw input, *before* type conversion — good for normalising |
| `AfterValidator` | runs on the already-converted value — good for rules |
| `@field_validator` | the same, written as a method; handy when the rule belongs to one model |
| `@model_validator` | rules that involve several fields |
| `@computed_field` | derived values included in output and schema |
| `@field_serializer` | control how a field is written out |
| `model_config` | model-wide behaviour (`extra`, `strict`, `str_strip_whitespace`, …) |

Reusable `Annotated` types are the biggest win. `ISBN` can now be used in a request
model, a response model, a query parameter and a settings class, and the rule lives in
one place.

## Every error, with a precise location

```python
Book.model_validate({**data, "isbn": "12345", "price": "9.999",
                     "published": "2030-01-01", "tags": ["a"] * 6, "subtitle": "x"})
```

```text
err: value_error ('isbn',) Value error, ISBN-13 must have 13 digits
err: decimal_max_places ('price',) Decimal input should have no more than 2 decimal places
err: value_error ('published',) Value error, published date is in the future
err: too_long ('tags',) List should have at most 5 items after validation, not 6
err: extra_forbidden ('subtitle',) Extra inputs are not permitted
```

A `ValueError` raised in your validator becomes a `value_error` with your message
(prefixed with `Value error, `). Inside FastAPI these become the 422 details you've
already seen, with `body` prepended to `loc`.

## Cross-field rules

```python
from pydantic import model_validator

class Promo(BaseModel):
    starts: date
    ends: date

    @model_validator(mode="after")
    def check_order(self):
        if self.ends < self.starts:
            raise ValueError("ends must be on or after starts")
        return self
```

```text
err: value_error () Value error, ends must be on or after starts
```

The empty `loc` `()` means "the whole model". `mode="after"` runs once all fields have
been validated, so `self.starts` is guaranteed to be a `date`. A `mode="before"` model
validator receives the raw input dict instead — useful for reshaping legacy payloads.

## Aliases: JSON names vs Python names

APIs often use `camelCase`; Python uses `snake_case`. An alias maps between them:

```python
print(b.model_dump(by_alias=True, include={"page_count"}))
```

```text
{'pageCount': 412}
```

With just `alias=`, input **must** use the alias. Sending `page_count` failed twice over:

```text
err: missing ('pageCount',) Field required
err: extra_forbidden ('page_count',) Extra inputs are not permitted
```

To accept either name, set `validate_by_name=True` (alongside the default
`validate_by_alias=True`) in `model_config`; with that, both `A(page_count=1)` and
`A(pageCount=2)` worked. FastAPI dumps response models **by alias** by default, so a
response model with `alias="pageCount"` sends `pageCount` to clients.

## Serialization: python mode vs JSON mode

```text
python dump: {'title': 'Dune', 'isbn': '9780441172719', 'price': '9.99', 'published': datetime.date(1965, 8, 1), 'tags': ['classic', 'scifi'], 'page_count': 412, 'isbn_prefix': '978'}
json dump:   {"title":"Dune","isbn":"9780441172719","price":"9.99","published":"1965-08-01","tags":["classic","scifi"],"page_count":412,"isbn_prefix":"978"}
```

- `model_dump()` keeps Python objects (`date` stays a `date`), except where a custom
  serializer says otherwise — `price` is a string in both because of `price_as_str`.
- `model_dump_json()` produces a JSON string directly, converting dates to ISO strings.
  It's faster than `json.dumps(model.model_dump())` because the conversion happens in
  Rust.
- `model_dump(mode="json")` gives a dict of JSON-safe values — handy when you need a dict
  but will store it as JSON.

Why serialize `Decimal` money as a string? Many JSON clients parse numbers as binary
floating point, and `0.1 + 0.2` isn't `0.3` there. A string survives the trip exactly.

## Strict vs lax

By default Pydantic is **lax**: it converts compatible input.

```text
Q(qty=3.0)  -> ok: qty=3
Q(qty=3.5)  -> err: int_from_float ('qty',) Input should be a valid integer, got a number with a fractional part
```

It will convert, but never lose information. Strict mode refuses conversions:

```python
class StrictQty(BaseModel):
    model_config = ConfigDict(strict=True)
    qty: int
```

```text
StrictQty.model_validate({"qty": "3"})    -> err: int_type ('qty',) Input should be a valid integer
StrictQty.model_validate_json('{"qty": 3}') -> ok: qty=3
```

Use strict mode for fields where silent conversion would hide a client bug — quantities
in an order, for example. You can make a single field strict with `Field(strict=True)`
or the `StrictInt` type rather than a whole model.

## Validating things that aren't models

`TypeAdapter` gives you Pydantic validation for any type:

```python
from pydantic import TypeAdapter
ta = TypeAdapter(list[ISBN])
ta.validate_python(["9780441172719", "bad"])
```

```text
err: value_error (1,) Value error, ISBN-13 must have 13 digits
```

The `loc` `(1,)` is the list index. FastAPI uses the same mechanism internally for
parameters that aren't models (`list[str]` query parameters, a bare `int` body).

## How It Actually Works

When Python executes a `class Book(BaseModel)` statement, Pydantic's metaclass reads the
annotations and `Field()` calls and builds a **core schema**: a nested description of
every field, its type, constraints and validators. That schema is handed to
`pydantic-core`, a Rust library, which compiles it into a validator object and a
serializer object. This happens once, at import time.

Calling `Book.model_validate(data)` runs the compiled Rust validator over your input. It
walks the schema, calls back into Python only for your custom validator functions, and
collects errors rather than raising on the first one. On success it constructs the
instance, setting `__dict__` directly and recording `model_fields_set` (lesson 4's
`exclude_unset` relies on that).

Validator order for one field is: `BeforeValidator`s → the core type
check and conversion → constraints like `max_length` → `AfterValidator`s. That ordering
is why `Tag`'s lowercasing runs before the length check, and why `strip_isbn` receives a
guaranteed `str`.

Assignment after construction is **not** validated by default. In the run for this
lesson, `b.page_count = "lots"` was accepted and the attribute held the string `'lots'`.
Set `validate_assignment=True` in `model_config` to make assignments go through the
validator too; with it, the same assignment raised `int_parsing`.

`@computed_field` values are included in dumps and appear in the **serialization**
schema marked read-only:

```text
{'readOnly': True, 'title': 'Isbn Prefix', 'type': 'string'}
```

FastAPI generates separate input and output schemas when they differ, so `isbn_prefix`
shows up in response docs but isn't expected in request bodies.

## Common mistakes

- **Using v1 APIs.** `.dict()`, `.json()`, `parse_obj`, `@validator` and `class Config`
  still partly work in v2 with deprecation warnings, but new code should use the v2
  names.
- **Forgetting `@classmethod` under `@field_validator`.** Field validators run before
  the instance exists.
- **Returning nothing from a validator.** A validator must return the (possibly
  modified) value; forgetting `return v` turns the field into `None`.
- **Raising `HTTPException` inside a model validator.** Raise `ValueError` (or
  `AssertionError`); Pydantic turns it into a structured error. Models shouldn't know
  about HTTP.
- **`float` for money.** Use `Decimal` with `decimal_places`, or integer cents.
- **Mutable defaults via `= []`.** Pydantic copies defaults so it's safe here, unlike a
  plain class — but `default_factory=list` states the intent clearly and is required
  for non-copyable defaults.
- **Expecting assignment to be validated** without `validate_assignment=True`.

## Exercise

1. Write a reusable `Slug` type: lower-case, only `a-z`, `0-9` and `-`, 3–60 characters,
   with leading/trailing whitespace stripped. Test it with `TypeAdapter`.
2. Build an `Order` model with `items: list[OrderLine]` (each with `sku: Slug` and a
   strict `qty: int` between 1 and 99), `coupon: str | None`, and a model validator
   that rejects more than 20 lines in total quantity.
3. Give `Order` a `computed_field` `total_qty`, and a camelCase alias generator
   (`ConfigDict(alias_generator=...)` — look up `pydantic.alias_generators.to_camel`).
   Dump it by alias.
4. Try `.dict()` on your model and read the warning. Replace it with the v2 call.
