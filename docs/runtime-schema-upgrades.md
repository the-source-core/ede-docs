# Runtime schema upgrades

How `ede migrate upgrade` brings a tenant's database up to date, and where to put
the rare change a schema comparison cannot work out on its own.

---

## The short version

The models are the source of truth for the **shape** of the database. On every
upgrade the platform compares what the models declare against what the tenant
actually has, and applies the difference. You change a field; you do not write a
migration.

What a comparison can never work out is what a value should **become** — a
renamed column's data, a new column filled from an old one. Those go in small,
version-keyed scripts under the app that owns them.

---

## The four phases of an upgrade

| Phase | What runs | What it is for |
|---|---|---|
| **1 — Before** | `upgrades/<version>/before_*.py` | Anything that must happen while the **old** shape still exists. Renames live here. |
| **2 — Shape** | The schema synchroniser | Create missing tables, add missing columns, then apply every foreign key, index and NOT NULL. |
| **3 — Content** | Registry sync, view sync, data loader | The existing code-is-truth passes, unchanged. |
| **4 — After** | `upgrades/<version>/after_*.py`, then `ir.app` is written | Backfills, now that the shape and the seed data are both in place. |

The recorded version is written **last**. A failure anywhere before that leaves
the app at its old version, so the transition is retried on the next run rather
than being silently skipped.

---

## There is no configuration

There is one mechanism, so there is nothing to choose. `ede migrate upgrade -t <tenant>`
runs the four phases above. `ede migrate check -t <tenant>` reports the same comparison
and writes nothing — use it in CI, and against a tenant you are about to upgrade.

### The one refusal you may hit

Every database built by the old migration files still carries an `alembic_version`
table. The upgrade refuses only when that tenant never applied a migration whose
**data** a comparison cannot reproduce:

```
tenant 'acme' never applied migration(s) that carried data a schema comparison
cannot reproduce, so converging it would give it the right shape and the wrong
contents.
  not reached: ['d1c4e8a72f93']
```

A comparison reproduces the **shape** of every past change and none of the **data**
those changes carried — the backfills that filled a new column from an old one. A
tenant that stopped short would be converged into the right shape with the wrong
contents, and nothing afterwards would detect it. Upgrade it once with a build from
before the migration files were removed, then run this again.

### What is deliberately *not* required

**Revisions belonging to domains this deployment does not activate.** Domain subsets
are a shipped capability: an installation running `["logistics_base",
"logistics_freight_forwarding"]` never activates `logistics_nvocc_agency`, so Alembic
never applied that domain's migrations — correctly. Requiring them would refuse a
tenant that is perfectly up to date with its *own* history. The check reads the live
`ACTIVE_DOMAINS` / `ACTIVE_MODULES` set.

**Revisions that only created tables and indexes.** Those are exactly what the
comparison rebuilds; refusing over one would be refusing on the grounds that this
mechanism works. When a tenant is behind on schema-only revisions the upgrade says so
and proceeds:

```
[EDE schema] tenant ms_ff_only had not reached 2 schema-only revision(s)
(11f1d74e49cc, 2ef6fddc0497); the comparison rebuilds those, so the upgrade proceeds
```

A database with no `alembic_version` table at all was built by this mechanism and has
no history to have missed. It proceeds without comment.

**Known limitation, stated rather than hidden.** `alembic_version` records *heads*,
not every applied revision, so the check is head-granular. A tenant holding an app's
final head necessarily applied that app's whole chain. A tenant holding an earlier
head is checked only against the data-carrying heads; an intermediate backfill it
skipped cannot be seen from here, because the graph that would reveal it was deleted
with the files.

The frozen head list lives in `src/ede/core/services/migration/final_migration_state.py`.
It is **history, not configuration** — editing it to make a stubborn tenant pass does
not upgrade that tenant, it only stops anyone from noticing that it was never upgraded.
Once no pre-cutover database remains anywhere, that module and its callers can go.

## What the synchroniser will and will not do

It adds and it adjusts. It never removes.

| What it finds | What it does |
|---|---|
| A table the models declare and the database lacks | Creates it — **without** its foreign keys, which are applied in the closing pass |
| A column the models declare and the database lacks | Adds it, nullable |
| The models require a value, the database allows empty | Fills from the declared default, then applies NOT NULL. **No default and rows still empty → left nullable with a warning** |
| The models allow empty, the database requires a value | Lifts the requirement |
| A required column no model declares at all | Lifts the requirement. The column and its data stay |
| A column or table the models no longer declare | Nothing. It stays |
| A type that differs | Nothing. A type change can lose data, so it is a script's job |

**Why nothing is ever dropped.** Deactivating a module would otherwise be
indistinguishable from deleting it, and the difference is a customer's data. It is
also what makes deploying an older build ordinary rather than a recovery procedure:
old code sends no value for columns it does not know about, so those lose their
NOT NULL instead of their contents, and the old build simply writes alongside them.

**The price**, stated plainly: a tenant migrated from the old mechanism keeps
retired columns and indexes forever, and keeps any column type the models have
since changed. Expect a handful of permanent `type_mismatch` and `extra_index`
findings from `ede migrate check` on such a tenant. They are the cost of the row
above, not a defect.

---

## Writing an upgrade script

```
src/ede/foundation/notifications/
    __manifest__.py             ← "version": "0.3.0"
    upgrades/
        0.2.0/
            before_rename_channel.py
            after_backfill_channel.py
        0.3.0/
            after_split_recipients.py
```

Each file defines exactly one function:

```python
from sqlalchemy import text


def migrate(context):
    context.connection.execute(text(
        "UPDATE ir_notification SET recipient_partner_id = ("
        "    SELECT partner_id FROM res_user WHERE res_user.record_uuid = recipient_user_id"
        ")"
    ))
```

`context` carries:

| Field | Notes |
|---|---|
| `connection` | A live SQLAlchemy connection, inside a transaction |
| `env` | The ORM environment — **`after` only**. `None` in a `before` script |
| `app_key`, `from_version`, `to_version`, `script_version`, `phase` | |
| `logger` | |

**Why a `before` script gets no `env`.** It runs while the old shape is still in
place, but the ORM is built from the *new* models. It would describe a database
that does not exist yet. Use `connection` and write the SQL you mean.

### When a script runs

When the tenant's recorded version crosses the script's: `recorded < version <= manifest`.
So arriving at `0.2.0` runs `0.2.0`'s scripts, and already being there does not.
Versions are compared numerically, so `1.10.0` correctly follows `1.2.0`.

**A fresh install runs nothing.** An app with no recorded version has no old shape
to convert — its tables are created in their final form.

### Two rules that are easy to get wrong

**Make `before` scripts idempotent.** They commit in their own transaction. If a
later phase fails, the version is not recorded — correctly, so the transition
retries — but the `before` scripts have already committed and will run again.
Write `ADD COLUMN IF NOT EXISTS`, `WHERE new_column IS NULL`, and so on.

**A failure aborts the upgrade, by design.** These run once per version transition.
Swallowing a failure would leave the version recorded as done and the data half
converted, with nothing left to detect it.

### Bump the manifest version

A script is only found if the app's `version` moves past it. Changing
`__manifest__.py`'s `"version"` is what triggers the run — and what `ir.app`
records afterwards.

---

## Where the state lives

`ir.app` — one row per activated app, holding `app_key`, `installed_version`,
`state` and `installed_at_utc`. It answers the one question nothing else could:
*has this app moved since this tenant was last upgraded?* The registry knows what
the code activates; `alembic_version` recorded revisions and said nothing about
modules; data seeding is idempotent and keeps no state at all.

---

## Related

- `docs/reports/Runtime Schema Upgrade — Approach.pdf` — the design rationale, the measured risk, and the staged plan this delivered.
