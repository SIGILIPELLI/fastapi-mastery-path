---
description: "Typed configuration for FastAPI with pydantic-settings: env vars, .env files, prefixes, nested settings, SecretStr, JSON lists, caching and test overrides."
---

# 07 · Configuration with pydantic-settings

Every app has values that differ between your laptop, CI and production: the database
URL, a signing key, allowed CORS origins, feature flags. Hard-coding them means editing
code to deploy; reading `os.environ["..."]` everywhere means string-typed values, no
validation, and a `KeyError` at 3 a.m. when one is missing.

`pydantic-settings` gives you a typed `Settings` class that reads environment variables
(and optionally a `.env` file), validates them with the same Pydantic rules you already
know, and fails **at startup** if something is wrong. It's a separate package:

```bash
pip install pydantic-settings
```

(Examples ran on pydantic-settings 2.15.0.)

## A settings class

```python
# settings.py
from functools import lru_cache
from typing import Literal

from pydantic import AnyHttpUrl, BaseModel, Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class SmtpSettings(BaseModel):
    host: str = "localhost"
    port: int = 25


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_prefix="BOOKSHOP_",
        env_file=".env",
        env_nested_delimiter="__",
        extra="ignore",
    )

    environment: Literal["dev", "test", "prod"] = "dev"
    database_url: str = "sqlite:///bookshop.db"
    secret_key: SecretStr
    debug: bool = False
    cors_origins: list[AnyHttpUrl] = []
    page_size_max: int = Field(100, ge=1, le=1000)
    smtp: SmtpSettings = SmtpSettings()


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

- `env_prefix="BOOKSHOP_"`: `secret_key` is read from `BOOKSHOP_SECRET_KEY`. A prefix
  avoids collisions with unrelated variables like `DEBUG` set by some other tool.
- `env_file=".env"`: also read a `.env` file if present. Real environment variables win
  over the file.
- `env_nested_delimiter="__"`: `BOOKSHOP_SMTP__HOST` sets `smtp.host`.
- `secret_key` has no default, so it's **required**.
- `extra="ignore"`: unrelated keys in `.env` don't cause errors.

## What happens with different environments

These are the real results of constructing `Settings()` under four setups.

**Nothing set at all:**

```text
no env, no .env -> ValidationError
      missing ('secret_key',) Field required | input: {}
```

That's the behaviour you want: the app refuses to start without a secret, instead of
starting with an insecure default.

**A `.env` file** containing `BOOKSHOP_SECRET_KEY=from-dotenv`, `BOOKSHOP_DEBUG=true` and
`UNRELATED=1`:

```text
with .env -> {'environment': 'dev', 'database_url': 'sqlite:///bookshop.db', 'secret_key': '**********', 'debug': True, 'cors_origins': [], 'page_size_max': 100} | smtp: host='localhost' port=25 | repr secret: SecretStr('**********')
```

`"true"` became `True`; `UNRELATED` was ignored; the secret is masked even in a JSON dump.

**Real environment variables on top of the same `.env`:**

```text
BOOKSHOP_SECRET_KEY=from-env  BOOKSHOP_ENVIRONMENT=prod
BOOKSHOP_CORS_ORIGINS='["https://shop.example.com"]'
BOOKSHOP_SMTP__HOST=mail.internal  BOOKSHOP_SMTP__PORT=587

-> {'environment': 'prod', ..., 'debug': True, 'cors_origins': ['https://shop.example.com/'], ...} | smtp: host='mail.internal' port=587
```

`environment` came from the real variable; `debug` still came from `.env` because nothing
overrode it. The origin gained a trailing `/` — `AnyHttpUrl` normalises URLs, which
matters later when you compare against a browser's `Origin` header (which has no
trailing slash). For CORS, a `list[str]` with your own validation is often less
surprising.

**Bad values:**

```text
bad values -> ValidationError
      literal_error ('environment',) Input should be 'dev', 'test' or 'prod' | input: staging
      url_parsing ('cors_origins', 0) Input should be a valid URL, relative URL without a base | input: not a url
      greater_than_equal ('page_size_max',) Input should be greater than or equal to 1 | input: 0
```

All problems reported at once, with the field names. Compare that with discovering a typo
in `ENVIRONMENT` because some code path behaved oddly in production.

### Lists must be JSON

Setting `BOOKSHOP_CORS_ORIGINS=https://a.com` (not a JSON array) didn't produce a
validation error at all:

```text
list not JSON -> pydantic_settings.exceptions.SettingsError: error parsing value for field "cors_origins" from source "EnvSettingsSource"
```

Complex types — lists, dicts, nested models without a delimiter — are parsed from
environment variables as **JSON**. Write `BOOKSHOP_CORS_ORIGINS='["https://a.com"]'`.

## Secrets stay secret

```text
str of secret: ********** | get_secret_value: x
settings=Settings(environment='dev', database_url='sqlite:///bookshop.db', secret_key=SecretStr('**********'), ...
```

`SecretStr` hides its value in `str()`, `repr()` and dumps, so logging the settings object
— which people do when debugging config — doesn't leak the key. You must call
`.get_secret_value()` deliberately where the real value is needed. Note that
`database_url` was **not** hidden: a URL with an embedded password should be a
`SecretStr` too, or be split into separate fields.

## Using settings in FastAPI

```python
from typing import Annotated
from fastapi import Depends, FastAPI
from settings import Settings, get_settings

app = FastAPI()
SettingsDep = Annotated[Settings, Depends(get_settings)]

@app.get("/info")
def info(settings: SettingsDep):
    return {"env": settings.environment, "debug": settings.debug}
```

`@lru_cache` on `get_settings` means the environment is read and validated **once**;
every request gets the same object. Two requests reported the same object ID:

```text
same object across requests: True {'env': 'dev', 'debug': False, ...}
```

The flip side: changing an environment variable after the first call has no effect until
the cache is cleared.

```text
after changing env, cached: False
after cache_clear: True
```

That's correct for production (config doesn't change under a running process) and a
trap in tests.

## Worked example: overriding settings in tests

Because settings arrive through a dependency, a test can replace them without touching
the environment:

```python
app.dependency_overrides[get_settings] = lambda: Settings(
    _env_file=None, secret_key="test", environment="test")
```

```text
override: {'env': 'test', 'debug': True, ...}
```

Look closely: `debug` is `True`, though the test didn't ask for it. `BOOKSHOP_DEBUG=true`
was still set in the process environment from the previous step, and `_env_file=None`
only disables the `.env` file, not environment variables. Pass every value the test
depends on explicitly, or clear the relevant variables with pytest's `monkeypatch.delenv`.

Code that needs settings *outside* a request — creating the database engine at import
time, for instance — can call `get_settings()` directly. The dependency override won't
apply there, which is one reason to create such resources in the lifespan handler
(Level 3 lesson 5) where tests can control them.

## How It Actually Works

`BaseSettings` is a Pydantic model whose `__init__` first gathers values from a list of
**sources**, then validates the merged result like any model. The default priority,
highest first:

1. arguments passed to `Settings(...)` (init kwargs);
2. environment variables;
3. the `.env` file (dotenv);
4. secrets directory files (if `secrets_dir` is configured — used by Docker/Kubernetes
   secrets);
5. field defaults.

For each field, the env source computes the variable name (prefix + field name,
case-insensitive by default), reads the raw string, and — if the field's type is
"complex" (list, dict, model) — runs `json.loads` on it. That's where `SettingsError`
came from: the JSON decode failed before Pydantic validation ever ran. With a nested
delimiter, `BOOKSHOP_SMTP__PORT` is split on `__` and assembled into
`{"smtp": {"port": "587"}}`, which Pydantic then validates into `SmtpSettings`.

You can reorder or replace sources by overriding the `settings_customise_sources`
classmethod — for example, to load from a cloud secret manager.

## Common mistakes

- **Defaults for secrets** (`secret_key: str = "changeme"`). Production starts happily
  with a known key. Make secrets required.
- **Committing `.env`.** Add it to `.gitignore`; commit a `.env.example` with dummy
  values instead.
- **Comma-separated lists in env vars.** They must be JSON arrays by default.
- **Constructing `Settings()` in many places**, re-reading the environment each time and
  possibly getting different results. Use one cached function.
- **Forgetting the cache in tests**: env changes don't take effect until
  `get_settings.cache_clear()`.
- **Logging the database URL** with its password. Use `SecretStr` or separate fields.

## Exercise

1. Add `allowed_hosts: list[str]` and `rate_limit_per_minute: int` (1–10,000) to the
   settings. Set them via environment variables and check the parsed values.
2. Make `database_url` a `SecretStr` and update the code that creates the engine to call
   `get_secret_value()`.
3. Write a pytest fixture that clears every `BOOKSHOP_` variable with `monkeypatch`, sets
   only what the test needs, and clears `get_settings`'s cache before and after.
4. Add a model validator that refuses `debug=True` when `environment == "prod"`, and test
   that the app fails to start with that combination.
