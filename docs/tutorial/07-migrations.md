# 7. Schema Upgrades

In earlier chapters you added fields to `blog.post` and they simply appeared in the database. This chapter opens the hood: why there was nothing to generate, what actually happens when you run `ede migrate upgrade`, and how to handle the two changes the platform genuinely cannot work out on its own.

```bash
# The two commands you'll use every day.
ede migrate check   -t system     # what would change; writes nothing
ede migrate upgrade -t system     # apply it, then load data
```

By the end you'll understand: why there are no migration files, what the comparison will and will not do to your database, why nothing is ever dropped, and where a rename or a data backfill goes.

## What you're going to build

Working from the existing `blog.post` model, you'll:

1.  Add a field and watch `ede migrate check` predict exactly what will change.
2.  Apply it, and confirm two tenants arrive at the same schema.
3.  Ship a **data backfill** as a version-keyed upgrade script — the one case a comparison cannot infer.

## 1. There are no migration files

Most frameworks record a *history*: a numbered chain of files, each describing one change, replayed in order against every database.

EDE does not. The models **are** the schema. On every upgrade the platform reads what your models declare, reads what the database actually has, and applies the difference.

That is the whole mental model, and it has one consequence worth stating plainly:

!!! tip "Adding a field is a one-step change"

    Change the model. That's it. Nothing to generate, nothing to commit alongside it, nothing to keep in sync.

Add a field to `blog.post`:

```python
subtitle = fields.Char(max_length=200, label="Subtitle")
```

Then ask what would happen:

```bash
ede migrate check -t system
```

```
  Schema check — tenant 'system' (database db_ede_system)
  355 tables declared by the models, 355 in the database

    BLOCKING      1    WARNING      0    INFO      0

  BLOCKING (1)
    [blocking] missing_column   blog_post.subtitle   declared as VARCHAR(200), absent from the database
```

`check` writes nothing. It is safe to run against production, and it is the habit worth forming: look before you upgrade.

## 2. Applying it

```bash
ede migrate upgrade -t system
```

```
schema_upgrade: 0 table(s), 1 column(s), 0 foreign key(s), 0 index(es), 0 relaxation(s), 0 NOT NULL application(s)
```

Run `ede migrate check` again and it reports nothing. Run the upgrade a second time and it also does nothing — the comparison is idempotent by construction, because it compares state rather than replaying steps.

Now do the same for a second tenant:

```bash
ede migrate upgrade -t acme
```

Both tenants arrive at the same schema, and neither needed to know what the other had already done. There is no per-tenant bookkeeping to drift.

## 3. Two passes, and why ordering stopped mattering

Internally the upgrade runs in two passes:

| Pass | What it does |
|---|---|
| **1 — Structure** | Create missing tables, add missing columns |
| **2 — Connections** | Apply every foreign key, index and NOT NULL |

A foreign key cannot be created until the table it points at exists. If each one were applied the moment its column was created, the *order* in which modules were processed would decide whether your upgrade worked — and that order depends on which modules a customer has activated.

Deferring every connection until all structure exists removes the question. By the time any foreign key is applied, every table it could reference is already there. A module pointing at another module's table is no longer an ordering problem; it cannot fail.

## 4. What the comparison will and will not do

This is fixed in advance, so it is predictable rather than a judgement call:

| What it finds | What it does |
|---|---|
| A table your models declare, absent from the DB | **Creates** it |
| A column your models declare, absent from the DB | **Adds** it, nullable |
| Your models require a value, the DB allows empty | Fills from the declared default, then applies **NOT NULL** |
| Your models allow empty, the DB requires a value | **Relaxes** the requirement |
| A required column no model declares at all | **Relaxes** it. The column and its data stay |
| A column or table your models no longer declare | **Nothing.** It stays |
| A column whose type differs | **Nothing.** A type change can lose data |

### Why nothing is ever dropped

Two reasons, and neither is timidity.

**Deactivating a module would otherwise look exactly like deleting it** — and the difference is a customer's data.

**It is what makes going backwards safe.** Deploy an older build against a newer database and it knows nothing about the newer columns, so it sends no value for them. If those columns were still mandatory, every save would fail. Lifting the requirement instead turns a rollback from a recovery procedure into ordinary behaviour: the old build runs the same comparison against its own older models and carries on. The newer data sits where it is, waiting.

The price, stated honestly: retired columns stay in the table forever. That is the trade.

## 5. The two things a comparison cannot infer

A comparison can tell you a column is **missing**. It can never tell you what a value should **become**.

- **A rename.** Comparing sees "one column vanished, another appeared". Applied literally that is silent data loss — the new column is created empty and the old one holds the only copy.
- **A backfill.** A new column derived from an old one, a row split across two tables. Nothing in the declared shape says where the values come from.

These, and only these, ship as files:

```
src/domains/blog/
    __manifest__.py             ← "version": "1.1.0"
    upgrades/
        1.1.0/
            after_backfill_slug.py
```

```python
from sqlalchemy import text


def migrate(context):
    context.connection.execute(text(
        "UPDATE blog_post SET slug = lower(replace(title, ' ', '-')) "
        "WHERE slug IS NULL"
    ))
```

`context` carries `connection`, `app_key`, `from_version`, `to_version`, `logger`, and — for `after` scripts only — `env`, the full ORM environment.

### before vs after

| | Runs | Gets |
|---|---|---|
| `before_*.py` | While the **old** shape is still in place | `connection` only |
| `after_*.py` | Once the shape and seed data are both in place | `connection` **and** `env` |

A rename belongs in `before` — that is the only moment the old column still holds the data. A `before` script deliberately gets no `env`: the ORM at that point is built from your **new** models and would describe a database that does not exist yet.

### When it runs

When the tenant's recorded version crosses the script's: `recorded < version <= manifest`. So arriving at `1.1.0` runs `1.1.0`'s scripts, and already being there does not.

!!! warning "Bump the manifest version, or nothing runs"

    The script directory is named after a version. If `__manifest__.py` still says `"version": "1.0.0"`, your `1.1.0` script is never found. This is the mistake everyone makes once.

Two more rules worth internalising:

- **A fresh install runs nothing.** An app with no recorded version has no old shape to convert — its tables are created in their final form.
- **`before` scripts must be idempotent.** They commit in their own transaction. If a later phase fails, the version is not recorded — correctly, so the transition retries — but your `before` script has already committed and will run again. Write `WHERE new_column IS NULL`.

## 6. Where the state lives

One table, `ir.app`, holds a row per activated app: its `app_key`, the `installed_version` this tenant has, and when it arrived. It answers the only question a version-keyed script needs — *has this app moved since this tenant was last upgraded?*

```sql
SELECT app_key, installed_version FROM ir_app ORDER BY app_key;
```

The version is written **last**, after the schema converged, the data loaded and the `after` scripts ran. A failure anywhere before that leaves the app at its old version, so the transition retries rather than being silently skipped.

## 7. Resetting the dev DB

```bash
dropdb db_ede_system && ede migrate upgrade -t system
```

A fresh database is built directly, in its final shape — no history to replay, so this is fast.

## 8. The pre-merge checklist

-   [ ] `ede migrate check -t <tenant>` reports no blocking drift.
-   [ ] If you renamed a column: a `before_*.py` script copies the data, and the manifest version is bumped.
-   [ ] If you added a column that must be populated from an existing one: an `after_*.py` script does it, and the manifest version is bumped.
-   [ ] If you retired a column: you have accepted that it stays in the table, keeping its data and losing only its NOT NULL.
-   [ ] `./run_tests.sh` green.

## What you just did

-   Learned why there are no migration files — the models are the schema, and the upgrade applies the difference.
-   Used `ede migrate check` to see a change before applying it, and confirmed two tenants converge independently.
-   Understood the two-pass structure that makes cross-module foreign keys an ordering non-problem.
-   Learned the add-and-adjust-never-delete rule, and why it is what makes deploying an older build safe.
-   Shipped a **data backfill** as a version-keyed `after` script — one of only two cases that still need a file.

Full reference: [Runtime Schema Upgrades](../runtime-schema-upgrades.md).

[:octicons-arrow-right-24: **Next — Deploying**](08-deploying.md)
