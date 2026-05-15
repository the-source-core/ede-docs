# 7. Migrations

You generated and applied migrations in earlier chapters without paying much attention. This chapter opens the hood: where the files live, how the framework runs them per-tenant, how to read what Alembic emitted, and when to drop into hand-authored SQL for the cases the autogen can't handle.

```bash
# The two commands you'll use every day.
ede migrate generate -m "blog.post add author_id" --app blog.post
ede migrate upgrade -t system
```

By the end you'll understand: what `ede migrate generate` actually does, where it writes the file, why every app has its own version chain, how the upgrade runs against every tenant, and what to do when autogen can't infer your change (FKs that need a name, columns that need a backfill, SQLite vs PostgreSQL gotchas).

## What you're going to build

Alongside the existing `blog.post` migration, you'll:

1.  Inspect the autogen output and identify what it captured automatically.
2.  Add a hand-authored migration that performs a data backfill — adding a column with a non-null default that requires per-row population.
3.  Apply it across two tenants and confirm both arrive at the same schema state.

## 1. Where migrations live

Every app keeps its own version chain under `migrations/versions/` next to the models. The structure for the blog app:

```
src/domains/blog/post/
├── __init__.py
├── __manifest__.py
├── models/
│   └── post.py
└── migrations/
    └── versions/
        ├── 20260514_120000_initial.py
        ├── 20260514_150000_fields_expansion.py
        └── 20260515_090000_add_view_count.py
```

This is **per-app version locations** — every app has its own Alembic revision tree. The framework joins them at runtime into one upgrade plan, but each tree is editable in isolation. Two apps can independently add migrations without merging revision graphs.

Top-level configuration:

-   `alembic.ini` at the repo root — Alembic's bootstrap file.
-   `src/migrations/env.py` — the Alembic env script. Loads every active app, walks each app's `migrations/versions/`, registers them as version locations.

You rarely touch either. Adding an app to `ACTIVE_DOMAINS` / `ACTIVE_MODULES` is what registers a new version location.

## 2. `ede migrate generate` — what it actually does

```bash
ede migrate generate -m "blog.post add author_id" --app blog.post
```

What happens:

1.  Boots a minimal env against a **reference database** (a clean SQLite file the framework keeps for autogen). This DB carries the current schema after every previously-applied migration.
2.  Compares the reference DB schema against the **declared field specs** on every model in `--app`'s registry.
3.  Emits an Alembic revision file with `op.create_table(...)`, `op.add_column(...)`, `op.create_index(...)`, `op.add_constraint(...)` calls that bring the reference DB to the new declared state.
4.  Writes the file to `src/domains/<domain>/<app>/migrations/versions/<timestamp>_<slug>.py`.
5.  Updates the reference DB by applying the new revision so the next `generate` starts from the post-this-change state.

The `--app` flag is important: it scopes the diff to **one app's models**. Without it, autogen would look at every active model and emit one giant revision touching tables from multiple apps — which then can't be cleanly rolled back per-app.

The `-m` description becomes the docstring of the generated file. Keep it short and specific.

## 3. Reading a generated migration

Open one of your existing migrations. Its shape:

```python
"""blog.post fields expansion

Revision ID: 1f3a7b9c4d12
Revises: a1b2c3d4e5f6
Create Date: 2026-05-14 15:00:00.123456
"""
from alembic import op
import sqlalchemy as sa


revision = "1f3a7b9c4d12"
down_revision = "a1b2c3d4e5f6"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.add_column(
        "blog_post",
        sa.Column("slug", sa.String(length=200), nullable=False),
    )
    op.add_column(
        "blog_post",
        sa.Column("status", sa.String(length=64), nullable=False),
    )
    op.create_index(
        "ix_blog_post_slug",
        "blog_post",
        ["slug"],
        unique=True,
    )


def downgrade() -> None:
    op.drop_index("ix_blog_post_slug", table_name="blog_post")
    op.drop_column("blog_post", "status")
    op.drop_column("blog_post", "slug")
```

Five things to notice:

1.  **`revision` and `down_revision`** wire the file into the linked-list chain. Don't edit these — Alembic generates them.
2.  **Every `upgrade`** has a corresponding `downgrade` autogen attempt. Test both before pushing to anything multi-developer.
3.  **Index names** are explicit (`ix_blog_post_slug`). Always preferable to letting the DB auto-name — it makes future drops portable across SQLite and PostgreSQL.
4.  **`nullable=False` without a default** on `add_column` will fail on any existing table that has rows. The autogen doesn't know your data; you have to add a `server_default` or a backfill (see section 5).
5.  **Table names** are derived from model keys: `blog.post` → `blog_post`. The dot becomes an underscore. Override with `@api.model("...", table="custom_name")` if you must.

## 4. `ede migrate upgrade` — running migrations per tenant

EDE is multi-tenant. Each tenant has its own database. The upgrade command runs Alembic against one tenant at a time:

```bash
# Upgrade just the system tenant (your local dev tenant by default).
ede migrate upgrade -t system

# Upgrade a different tenant.
ede migrate upgrade -t acme

# Upgrade every active tenant (production sweep).
ede migrate upgrade -t all
```

What `upgrade` does for one tenant:

1.  Connects to the tenant's database (URL derived from `DATABASE_URL_<tenant>` env / config).
2.  Reads the `alembic_version_<app>` table that tracks the current head per app.
3.  For each app, walks the version chain forward, applying every pending revision in order.
4.  Runs every app's seed-data loader (CSVs + XMLs from each manifest's `data:` list) after schema migrations complete.

Re-running is idempotent — Alembic skips revisions whose IDs are already in `alembic_version_<app>`.

## 5. When autogen can't help — hand-authored migrations

Autogen handles the easy cases — new columns, dropped columns, new tables, new indexes. It cannot:

-   Add a `nullable=False` column to a table that already has rows (no default to use).
-   Backfill values in existing rows before tightening a constraint.
-   Rename a column without losing data (autogen emits add-then-drop).
-   Migrate enum values, restructure JSON payloads, split one table into two.

For these you write the migration by hand. Use `ede migrate generate -m "<description>" --app <app>` to scaffold an empty revision file with the correct `revision` / `down_revision` already set, then fill in the body.

Example: add a `view_count` column with `NOT NULL` and a per-row backfill.

```python
"""blog.post add view_count with backfill"""
from alembic import op
import sqlalchemy as sa


revision = "7c8d9e0f1234"
down_revision = "1f3a7b9c4d12"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Step 1: add the column as nullable.
    op.add_column(
        "blog_post",
        sa.Column("view_count", sa.Integer(), nullable=True),
    )

    # Step 2: backfill every existing row with 0.
    op.execute("UPDATE blog_post SET view_count = 0 WHERE view_count IS NULL")

    # Step 3: tighten the constraint.
    with op.batch_alter_table("blog_post") as batch_op:
        batch_op.alter_column(
            "view_count",
            existing_type=sa.Integer(),
            nullable=False,
            server_default="0",
        )


def downgrade() -> None:
    op.drop_column("blog_post", "view_count")
```

Two things this example demonstrates:

-   **The three-step pattern** for tightening a column on a populated table: add nullable → backfill → tighten. Run on PostgreSQL as a single transaction; on SQLite the framework's `op.batch_alter_table` rewrites the table.
-   **`op.batch_alter_table(...)`** is the cross-database way to alter a column. Native `ALTER` is not supported in SQLite, so Alembic emulates it by creating a new table, copying data, and renaming. Always wrap `alter_column` and constraint changes in `batch_alter_table` when the migration needs to run on both engines.

## 6. Foreign keys and indexes — name them yourself

PostgreSQL has a 63-character identifier limit. Auto-generated names from SQLAlchemy can exceed it for complex tables — the migration fails to apply in production after working fine on SQLite. Always pass explicit names:

```python
# Good — explicit, short, deterministic.
op.create_foreign_key(
    "fk_blog_post_author",
    source_table="blog_post",
    referent_table="res_partner",
    local_cols=["author_id"],
    remote_cols=["record_uuid"],
    ondelete="RESTRICT",
)

op.create_index(
    "ix_blog_post_author",
    "blog_post",
    ["author_id"],
)
```

Naming convention used throughout the framework:

| Object | Pattern | Example |
|---|---|---|
| Index | `ix_<table>_<column>` | `ix_blog_post_slug` |
| Unique index | `uq_<table>_<column>` | `uq_blog_post_slug` |
| Foreign key | `fk_<source_table>_<column>` (drop the `_id` suffix) | `fk_blog_post_author` |
| Check constraint | `ck_<table>_<descriptive_name>` | `ck_blog_post_rating_range` |

Use these and your migrations will apply cleanly on both SQLite and PostgreSQL.

## 7. Multi-head merges

Two developers working on the same app in parallel will each generate a revision pointing at the same `down_revision`. When the second one's branch lands, Alembic refuses to upgrade — there are now two "heads" in the version chain.

Resolve by generating a merge revision:

```bash
ede migrate merge -m "merge feature branches" --app blog.post
```

This emits a revision whose `down_revision` is a tuple of both head IDs and whose `upgrade` / `downgrade` are no-ops (just a graph join). Commit and continue.

Avoid this by rebasing your branch onto the new `down_revision` before adding fresh migrations, when possible.

## 8. Where the actual SQL goes

Curious what hits the DB? Run with the SQL preview flag:

```bash
ede migrate upgrade -t system --sql
```

Alembic prints every statement it *would* run, in order, without executing. Use this in CI to check migrations against a snapshot of production schema before they merge.

## 9. Resetting the dev DB

When iterating heavily on schema, you sometimes want to start over rather than chase a migration chain. Delete the SQLite file and re-run the full chain:

```bash
rm var/system.db                  # or wherever your tenant DB lives
ede migrate upgrade -t system
```

This is **never** acceptable in production. In dev it's faster than authoring four corrective migrations.

## 10. The pre-migration checklist

Before pushing any migration:

-   [ ] `ede migrate generate` ran cleanly (no surprising columns / drops).
-   [ ] Every `op.add_column(..., nullable=False)` either has a `server_default` or a backfill block before the tighten.
-   [ ] Every `op.create_foreign_key` / `op.create_index` has an explicit short name.
-   [ ] Cross-database alterations are wrapped in `op.batch_alter_table`.
-   [ ] `ede migrate upgrade -t system` succeeds against a fresh SQLite tenant.
-   [ ] `ede migrate upgrade -t system` succeeds against a fresh PostgreSQL tenant (if you target both).
-   [ ] `downgrade()` runs without errors.
-   [ ] You're committing **both** the new migration file and the corresponding model changes in the same commit.

---

## What you just did

-   Saw where migrations live (per-app under `migrations/versions/`) and how the framework joins them at upgrade time.
-   Read what `ede migrate generate --app <app>` emits and which kinds of change it can — and can't — infer.
-   Authored a **hand-authored migration** with the add-nullable → backfill → tighten pattern.
-   Learned the FK / index naming conventions that keep migrations portable across SQLite and PostgreSQL.
-   Picked up the pre-merge checklist that catches the common breakages.

[:octicons-arrow-right-24: **Next — Deploying**](08-deploying.md)
