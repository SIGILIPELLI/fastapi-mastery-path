---
description: "Manage schema changes for a FastAPI + SQLAlchemy app with Alembic: autogenerate, naming conventions, backfilling NOT NULL columns, SQLite batch mode."
---

# 06 · Database Migrations with Alembic

`Base.metadata.create_all()` creates missing tables and does nothing else. The day you add
a column to a model, `create_all` silently ignores it, and your app crashes on the first
query that selects the new column. **Migrations** are versioned scripts that move a
database from one schema to the next, and **Alembic** (written by SQLAlchemy's author) is
the standard tool for SQLAlchemy projects.

Everything below was run with **Alembic 1.20.0** and SQLite, including the failures,
which are the most instructive part.

## Setting up

```bash
pip install alembic
alembic init migrations
```

That creates `alembic.ini`, a `migrations/` directory with `env.py`, and an empty
`migrations/versions/`. Other templates exist; `alembic list_templates` showed `generic`,
`async`, `multidb`, `pyproject` and `pyproject_async`. Use `async` if your app uses an
async engine — its `env.py` runs migrations through `run_sync`.

Two edits connect Alembic to your models. In `alembic.ini`, set the URL (in a real
project, read it from settings in `env.py` instead — lesson 7):

```ini
sqlalchemy.url = sqlite:///bookshop.db
```

In `migrations/env.py`, replace `target_metadata = None`:

```python
from app.models import Base

target_metadata = Base.metadata
```

The generated `alembic.ini` already contains `prepend_sys_path = .`, which is what lets
`env.py` import `app.models` when you run `alembic` from the project root.

For SQLite only, also pass `render_as_batch=True` to `context.configure(...)` in the
online-migration function. The reason is below.

## The first migration

The models:

```python
# app/models.py
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class Author(Base):
    __tablename__ = "authors"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

class Book(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
```

```bash
alembic revision --autogenerate -m "create authors and books"
```

```text
INFO  [alembic.autogenerate.compare.tables] Detected added table 'authors'
INFO  [alembic.autogenerate.compare.tables] Detected added table 'books'
Generating .../migrations/versions/4bc842e0ad93_create_authors_and_books.py ...  done
```

The generated file (header trimmed):

```python
revision: str = '4bc842e0ad93'
down_revision: Union[str, Sequence[str], None] = None

def upgrade() -> None:
    """Upgrade schema."""
    op.create_table('authors',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('name', sa.String(length=100), nullable=False),
    sa.PrimaryKeyConstraint('id')
    )
    op.create_table('books',
    sa.Column('id', sa.Integer(), nullable=False),
    sa.Column('title', sa.String(length=200), nullable=False),
    sa.Column('author_id', sa.Integer(), nullable=False),
    sa.ForeignKeyConstraint(['author_id'], ['authors.id'], ),
    sa.PrimaryKeyConstraint('id')
    )

def downgrade() -> None:
    """Downgrade schema."""
    op.drop_table('books')
    op.drop_table('authors')
```

Autogenerate compared `Base.metadata` with the (empty) database and wrote the difference.
`down_revision = None` marks it as the first in the chain. Apply it:

```bash
alembic upgrade head
```

```text
INFO  [alembic.runtime.migration] Running upgrade  -> 4bc842e0ad93, create authors and books
```

The database now has an `alembic_version` table whose single row, `4bc842e0ad93`, records
where this database is in the chain.

## Worked example: adding a required, unique column

The next model change adds an ISBN (required, unique) and an optional publication year:

```python
    isbn: Mapped[str] = mapped_column(String(13), unique=True)
    published_year: Mapped[int | None]
```

There's already one row in `books` (`Dune`). Autogenerate again:

```text
INFO  [alembic.autogenerate.compare.tables] Detected added column 'books.isbn'
INFO  [alembic.autogenerate.compare.tables] Detected added column 'books.published_year'
INFO  [alembic.autogenerate.compare.constraints] Detected added unique constraint None on '('isbn',)'
UserWarning: Autogenerate rendered a drop_constraint() directive for an unnamed constraint on table 'books'; the migration will fail unless a constraint name is added to the directive.  Consider using a naming convention so that constraint names are known ahead of time.
```

### Failure 1: unnamed constraints

`alembic upgrade head` on that script failed with:

```text
ValueError: Constraint must have a name
```

The model's `unique=True` doesn't give the constraint a name, and you can't later drop or
alter a constraint you can't name. The fix is a **naming convention** on the metadata, so
every constraint gets a predictable name:

```python
from sqlalchemy import MetaData

NAMING = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING)
```

Set this up **before your first migration** in a new project. After deleting the bad
script and regenerating, Alembic reported `Detected added unique constraint 'uq_books_isbn'`.

### Failure 2: NOT NULL on a table with rows

The regenerated script added `isbn` as `nullable=False` in one step. Upgrading failed:

```text
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) NOT NULL constraint failed: _alembic_tmp_books.isbn
```

The existing `Dune` row has no ISBN, so a `NOT NULL` column can't be added. Worse, the
failed attempt **left a half-finished temporary table** (`_alembic_tmp_books`) behind —
the log line `Will assume non-transactional DDL` meant there was no transaction to roll
back — and the next attempt failed with
`table _alembic_tmp_books already exists` until it was dropped by hand.

Autogenerate can't know how to fill in old rows. That's your job, in three steps:

```python
def upgrade() -> None:
    """Add isbn (required, unique) and published_year to books."""
    # 1. Add the column as nullable so existing rows are valid.
    with op.batch_alter_table("books") as batch_op:
        batch_op.add_column(sa.Column("isbn", sa.String(length=13), nullable=True))
        batch_op.add_column(sa.Column("published_year", sa.Integer(), nullable=True))

    # 2. Backfill. A real project would load real ISBNs; here we derive a unique
    #    placeholder value from the id so the NOT NULL + UNIQUE constraints can apply.
    op.execute("UPDATE books SET isbn = printf('000000%07d', id) WHERE isbn IS NULL")

    # 3. Now enforce the constraints.
    with op.batch_alter_table("books") as batch_op:
        batch_op.alter_column("isbn", existing_type=sa.String(length=13), nullable=False)
        batch_op.create_unique_constraint(batch_op.f("uq_books_isbn"), ["isbn"])
```

(`printf` is SQLite's string-format function; on PostgreSQL you'd use `lpad` or
`format`.)

```text
INFO  [alembic.runtime.migration] Running upgrade 4bc842e0ad93 -> 6c7eb8fe5d64, add isbn and published_year
```

The row and the resulting schema:

```text
1|Dune|0000000000001|
CREATE TABLE "books" (
	id INTEGER NOT NULL,
	title VARCHAR(200) NOT NULL,
	author_id INTEGER NOT NULL,
	isbn VARCHAR(13) NOT NULL,
	published_year INTEGER,
	PRIMARY KEY (id),
	CONSTRAINT uq_books_isbn UNIQUE (isbn),
	FOREIGN KEY(author_id) REFERENCES authors (id)
);
```

## Day-to-day commands

```text
$ alembic history
4bc842e0ad93 -> 6c7eb8fe5d64 (head), add isbn and published_year
<base> -> 4bc842e0ad93, create authors and books

$ alembic current
6c7eb8fe5d64 (head)

$ alembic check
No new upgrade operations detected.

$ alembic downgrade -1
INFO  [alembic.runtime.migration] Running downgrade 6c7eb8fe5d64 -> 4bc842e0ad93, add isbn and published_year
```

`alembic check` is worth running in CI: it fails if the models have changes that no
migration covers. After adding an unmigrated `nickname` column to the model, it printed
`FAILED: New upgrade operations detected: [('add_column', None, 'books', Column('nickname', ...))]`
and exited with status 255. After the downgrade, the `books` table was back to its three original
columns — and the backfilled ISBNs were gone. Downgrades that drop columns destroy data;
treat them as a development tool, not a production undo button.

`alembic upgrade head --sql` prints the SQL instead of running it, for DBAs who review
scripts. On SQLite it stopped partway with:

```text
FAILED: This operation cannot proceed in --sql mode; batch mode with dialect sqlite requires a live database connection with which to reflect the table "books". ...
```

because batch mode needs to read the live table (next section). Against PostgreSQL,
offline SQL generation works for these operations.

## Running migrations for a FastAPI app

- Remove `Base.metadata.create_all()` from app startup once you adopt Alembic. Two
  sources of truth for the schema will drift.
- Run `alembic upgrade head` as a **separate deployment step**, before the new app
  version starts. Don't run it inside the app at startup: with several workers or
  replicas, they'd all try at once.
- Tests can create tables with `create_all` on a throwaway database (lesson 8), or run
  the migrations — slower but it tests the migrations too.

## How It Actually Works

**Autogenerate** reflects the live database (reads its tables, columns, constraints and
indexes) and compares the result with `target_metadata`. Each difference becomes an
operation in the script. It's a diff, not a mind reader: it detects added and removed
tables and columns, nullability and many type changes, but can't detect a **rename** (it
sees a drop plus an add, which would delete the data) and can't invent backfill values.
Always read the generated script.

**The revision chain** is a linked list: each script has a `revision` ID and a
`down_revision`. `upgrade head` reads `alembic_version`, finds the path from there to the
head, and runs each `upgrade()` in order, updating `alembic_version` after each.

**Batch mode** exists because SQLite's `ALTER TABLE` can't change a column's nullability
or add a constraint to an existing table. `batch_alter_table` works around it with
"move and copy": create `_alembic_tmp_books` with the new schema, `INSERT ... SELECT`
the old rows into it, drop the old table, rename the new one. You saw the `INSERT ...
SELECT` step fail on the `NOT NULL` column. On PostgreSQL, the same operations become
plain `ALTER TABLE` statements, and because PostgreSQL supports **transactional DDL**, a
failed migration rolls back completely instead of leaving debris.

## Common mistakes

- **No naming convention** — unnamed constraints that later migrations can't drop.
- **Adding a `NOT NULL` column in one step** to a table with data. Add nullable,
  backfill, then enforce.
- **Trusting autogenerate for renames.** Edit the script to use
  `op.alter_column(..., new_column_name=...)`.
- **Editing a migration that has already run** somewhere else. Write a new one.
- **Running migrations at app startup** from every worker.
- **Using ORM models inside migrations.** Models change; old migrations must keep
  working. Use `op.execute` or a small `sa.table(...)` definition inside the script.

## Exercise

1. Set up Alembic for the lesson 4 bookshop app, with a naming convention from the
   start. Generate and apply the initial migration.
2. Add `Book.price` as a required `Numeric(8, 2)` column. Write the three-step migration
   with a backfill of `0.00`, and apply it to a database that has rows.
3. Rename `published_year` to `year_published`. Look at what autogenerate produces,
   then fix the script so no data is lost. Verify by checking a row before and after.
4. Add `alembic check` to a script that fails (non-zero exit) when models and migrations
   disagree. Change a model without a migration and run it.
