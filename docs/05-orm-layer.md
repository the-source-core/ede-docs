# EDE Framework — ORM Layer (RecordSet / ModelProxy)

## Overview

EDE does not use SQLAlchemy's ORM mapper. Instead it uses **SQLAlchemy Core** (Table, select,
insert, update, delete) wrapped in a clean record-oriented abstraction:

```
env.models["model.key"]   →  ModelProxy
ModelProxy.search(...)    →  RecordSet
RecordSet.write(...)      →  dispatch ede.update → repo → DB
```

The ORM layer lives in `src/ede/core/orm/`.

---

## Why a Custom ORM Layer?

### All writes route through the CommandBus

`RecordSet.write()` dispatches `ede.update`. `ModelProxy.create()` dispatches `ede.create`.
This means every mutation — whether triggered by an HTTP endpoint or by code inside an
event handler — passes through the same pipeline: lifecycle hooks fire, events are emitted,
audit trails are captured. There is no "back door" to the database that skips these.

If you used SQLAlchemy's ORM session directly in your handlers, some writes would emit
events and others wouldn't, some would trigger hooks and others wouldn't. Consistency
would break down as the codebase grows.

### `RecordSet` as the universal handle

`RecordSet` replaces the ORM object pattern. It is the same type whether you called
`browse()`, `search()`, or received it from a command handler. This uniformity means
every operation (read, write, delete, iterate) has the same API everywhere — no
"detached instance" errors, no "lazy load outside session" failures.

### Per-tenant isolation is invisible to application code

The ORM layer routes every query through `env.tenant_id` → per-tenant database URL.
Application code calls `env.models["logistics.shipment"].search(...)` and gets results
from the correct tenant's database. No filter clauses, no tenant ID columns, no risk of
data leakage between tenants.

---

## ModelProxy

`ModelProxy` is the entry point for a specific model in the current `Env`. Access it via:

```python
shipments = env.models["logistics.shipment"]   # → ModelProxy
```

`env.models` returns a `ModelAccessor` (dict-like proxy). `env.models["key"]` is the
equivalent of the Odoo `self.env["model.name"]` pattern.

### `ModelProxy.create(values)` → `RecordSet`

Creates a new record. Routes through `CommandBus` (dispatches `ede.create`). Emits
`ede.record.created` event. Returns a single-record `RecordSet`.

```python
record = env.models["logistics.shipment"].create({
    "tracking_number": "TRK-001",
    "status": "pending",
    "weight_kg": 2.5,
})
print(record.id)   # → "uuid-..."
```

Relational fields (M2M links, O2M creates) are processed **after** the scalar INSERT,
in the same transaction.

### `ModelProxy.browse(*ids)` → `RecordSet`

Creates a lazy `RecordSet` without hitting the DB. Fields are loaded on first access.

```python
record = env.models["logistics.shipment"].browse("uuid-1234")
# No DB hit yet.
print(record.tracking_number)   # DB hit here (field access triggers load)
```

Passing multiple IDs returns a multi-record `RecordSet`:
```python
records = env.models["logistics.shipment"].browse("uuid-1", "uuid-2", "uuid-3")
```

### `ModelProxy.search(domain, order, limit, offset)` → `RecordSet`

Runs a filtered query. Returns a pre-populated `RecordSet` (cache filled from single DB
round-trip).

```python
pending = env.models["logistics.shipment"].search(
    domain=[("status", "=", "pending")],
    order="created_at_utc desc",
    limit=50,
    offset=0,
)
for record in pending:
    print(record.tracking_number)
```

Domain format: list of `(field, operator, value)` triples.

| Operator | Meaning |
|---|---|
| `"="` | Equal |
| `"!="` | Not equal |
| `">"`, `">="`, `"<"`, `"<="` | Comparisons |
| `"like"` | SQL LIKE (case-sensitive) |
| `"ilike"` | SQL LIKE (case-insensitive) |
| `"in"` | SQL IN list |
| `"not in"` | SQL NOT IN list |
| `"is null"` | NULL check |
| `"is not null"` | NOT NULL check |

Multiple conditions are AND-ed by default.

### `ModelProxy.search_count(domain)` → `int`

```python
count = env.models["logistics.shipment"].search_count([("status", "=", "pending")])
```

### `ModelProxy.read_group(domain, group_by, aggregate, order, limit, offset)` → `List[Dict]`

Aggregation query:

```python
results = env.models["logistics.shipment"].read_group(
    domain=[("status", "!=", "cancelled")],
    group_by=["status"],
    aggregate={"total_weight": ("weight_kg", "sum"), "count": ("dbid", "count")},
)
# → [{"status": "pending", "total_weight": 12.5, "count": 5}, ...]
```

---

## RecordSet

`RecordSet` is the live, env-bound handle to 0-N records of the same model. It is the
primary object you interact with after creation, browse, or search. Never instantiate it
directly — always get it from `ModelProxy`.

```
str(rs) → "res.organization(uuid-1, uuid-2)"
```

### Identity Properties

```python
record.id           # → str  — record_uuid of the single record (raises on multi-record set)
record.record_uuid  # → str  — same as .id
record.dbid         # → int  — auto-increment PK (read from cache, triggers DB load if needed)
record.ids          # → List[str] — record_uuids of ALL records in this set (safe on any size)
```

### Boolean and Length

```python
bool(record)    # True if the set has at least one record (len > 0)
len(record)     # number of records in the set

# Common pattern: check before reading a nullable relation
country = shipment.country_id   # → RecordSet (may be empty if FK is null)
if country:                     # equivalent to country.exists()
    print(country.name)
```

### Field Access (Lazy Load)

Fields are loaded on first access. After `browse()` the cache is empty; first field read
triggers a single DB round-trip that fills all scalar fields.

```python
record = env.models["logistics.shipment"].browse("uuid-1234")
# cache empty — no DB hit yet
print(record.status)            # triggers DB read → fills all scalar fields in cache
print(record.tracking_number)   # cache hit — no DB hit
```

Field access on a **multi-record `RecordSet`** raises `ValueError`. Iterate first:

```python
# WRONG — raises ValueError if results contains > 1 record
results = env.models["logistics.shipment"].search([("status", "=", "pending")])
print(results.tracking_number)   # ValueError: cannot read field on multi-record set

# CORRECT
for shipment in results:
    print(shipment.tracking_number)   # each shipment is a 1-record RecordSet
```

### Iteration

Each iteration step yields a **single-record `RecordSet`** sharing the parent's cache:

```python
for shipment in env.models["logistics.shipment"].search([("status", "=", "pending")]):
    print(shipment.id, shipment.tracking_number)
```

### Indexing and Slicing

```python
results = env.models["logistics.shipment"].search([])

first = results[0]        # → single-record RecordSet
page  = results[0:10]     # → RecordSet with first 10 records
last  = results[-1]       # → single-record RecordSet (last)
```

### `record.read(fields=None)` → `List[Dict]`

Returns raw field values as a list of dicts, one per record. Optionally restrict columns:

```python
data = record.read()
# → [{"id": "uuid-...", "tracking_number": "TRK-001", "status": "pending", ...}]

# Only specific columns
data = record.read(fields=["tracking_number", "status"])
# → [{"tracking_number": "TRK-001", "status": "pending"}]
```

### `record.write(values)` → `RecordSet`

Updates fields on every record in the set. Routes through `CommandBus` (`ede.update` per
record). Emits `ede.record.updated`. Returns `self` (cache invalidated).

```python
record.write({
    "status": "confirmed",
    "dispatched_at": datetime.utcnow(),
})
```

Relational fields use command tuples (see Relational Commands below).

### `record.delete()` → `bool`

Deletes all records in the set. Routes through `CommandBus` (`ede.delete` per record).
Emits `ede.record.deleted`. Returns `True` if all records were deleted.

```python
record.delete()
```

On an empty RecordSet `delete()` is a no-op and returns `True`.

### `record.exists()` → `RecordSet`

Returns a **new `RecordSet`** containing only IDs that still exist in the DB (not a bool).
Use it after optional-FK traversal or when IDs may have been deleted externally.

```python
# Safe traversal of optional FK
country = shipment.country_id   # may be empty RecordSet if FK is null
country = country.exists()      # re-confirm the FK target still exists in DB
if country:
    print(country.name)

# Filter a multi-record set to only live records
live = env.models["logistics.shipment"].browse("uuid-1", "uuid-2", "uuid-3").exists()
```

### `record.invalidate_cache()` → `None`

Clears the local field-value cache. Forces a DB reload on the next field access.

```python
record.write({"status": "confirmed"})
record.invalidate_cache()
print(record.status)   # "confirmed" — re-loaded from DB
```

---

## `RecordSet.new()` — Uncommitted RecordSet

`RecordSet.new()` creates an **in-memory RecordSet** that is not backed by a DB row.
It is used internally by the `CommandBus` to provide `cmd.record` to **pre-create
lifecycle hooks** before the record is written to the database.

```python
rs = RecordSet.new("logistics.shipment", {"tracking_number": "TRK-001", "status": "draft"}, env)

rs._is_new        # True
rs._ids           # []  — no record_uuid yet
bool(rs)          # False — no IDs
rs.tracking_number  # "TRK-001"  — reads from in-memory values dict
rs.status           # "draft"
rs.missing_field    # None — unknown keys return None (not AttributeError)
```

**You will encounter this inside pre-create hooks:**

```python
@api.model("logistics.shipment")
class Shipment(DomainModel):

    @api.on_hook("pre.logistics.shipment.create")
    def validate_route(self, cmd: Command) -> None:
        rs = cmd.record         # RecordSet.__new__ — in-memory, not in DB
        if rs.origin == rs.destination:
            raise ValueError("Origin and destination cannot be the same.")
```

`RecordSet.new()` is not intended for direct use in application code — use
`ModelProxy.create()` instead.

---

## Relational Field Access

### Reference (Many-to-One)

Accessing a `Reference` field returns a **single-record `RecordSet`** of the related model:

```python
shipment = env.models["logistics.shipment"].browse("uuid-1234")
country = shipment.origin_country    # → RecordSet of res.country (1 record)
print(country.name)                  # "Germany"
```

If the FK is null, returns an **empty `RecordSet`** (0 records). Always check `exists()`:

```python
if shipment.origin_country.exists():
    print(shipment.origin_country.name)
```

### One2Many

```python
order = env.models["logistics.order"].browse("uuid-5678")
lines = order.order_lines     # → RecordSet of logistics.order_line (N records)
for line in lines:
    print(line.product, line.qty)
```

### Many2Many

```python
article = env.models["content.article"].browse("uuid-9999")
tags = article.tags    # → RecordSet of res.tag (N records)
```

---

## Relational Commands (Write Tuples)

When writing relational fields (One2Many, Many2Many), use `RelationalCommand` tuples.

```python
from ede.core.orm.commands import RelationalCommand as RC

record.write({
    "tags": [
        RC.link("existing-tag-uuid"),      # add existing record to M2M
        RC.unlink("old-tag-uuid"),         # remove link (does not delete)
        RC.create({"name": "new-tag"}),    # create new + link (M2M) / create with FK (O2M)
        RC.update("line-uuid", {"qty": 5}),# update linked record
        RC.delete("line-uuid"),            # unlink + delete
        RC.clear(),                        # remove all links
        RC.set(["id1", "id2"]),            # replace entire set
    ]
})
```

Each tuple format: `(command_id, record_id_or_0, values_or_ids_or_0)`

| Method | Tuple | Effect |
|---|---|---|
| `RC.link(id)` | `(4, id, 0)` | Add existing record to M2M or set FK (O2M) |
| `RC.unlink(id)` | `(3, id, 0)` | Remove link; do not delete record |
| `RC.create(values)` | `(0, 0, values)` | Create new record + link |
| `RC.update(id, values)` | `(1, id, values)` | Update linked record |
| `RC.delete(id)` | `(2, id, 0)` | Unlink + delete record |
| `RC.clear()` | `(5, 0, 0)` | Remove all links |
| `RC.set(ids)` | `(6, 0, ids)` | Replace all links with given IDs |

---

## Transactions

EDE uses an explicit transaction context manager. Without it, each ORM operation
auto-commits.

### Without transaction (auto-commit per operation)

```python
# Each call is its own transaction
record = env.models["logistics.shipment"].create({...})
record.write({"status": "confirmed"})  # separate transaction
```

### With `env.transaction()` (atomic)

```python
with env.transaction():
    record = env.models["logistics.shipment"].create({...})
    record.write({"status": "confirmed"})
    env.models["logistics.order"].create({"shipment_id": record.id, ...})
# commit on context manager exit; rollback on exception
```

The context manager:
1. Sets `env._active_uow` to the current `UnitOfWork`
2. All ORM ops within see `_active_uow` → use its session → no auto-commit
3. On exit without exception: calls `uow.commit()`
4. On exception: calls `uow.rollback()` then re-raises

### With explicit tenant

```python
with env.transaction(tenant_id="acme"):
    # operates on "acme" tenant DB
    ...
```

---

## Generic Repository (internal)

The `SqlAlchemyGenericRepository` is an internal implementation detail. App code should
never call it directly — use `RecordSet` and `ModelProxy` instead.

The repository handles:
- `create()` — INSERT + auto-fill dbid (auto-increment), record_uuid (UUID4), timestamps, revision=0
- `update()` — UPDATE by `record_uuid` + increment revision + update `updated_at_utc`
- `delete()` — DELETE by `record_uuid`
- `read_one()` — SELECT by `record_uuid`
- `search()` — SELECT with WHERE (domain filter) + ORDER BY + LIMIT + OFFSET
- `count()` — SELECT COUNT with WHERE
- `read_group()` — GROUP BY + aggregate + date truncation (PostgreSQL: `date_trunc`, SQLite: `strftime`)
- `m2m_add()` / `m2m_remove()` — INSERT / DELETE on join tables

MetaData is cached globally per `(model_keys, engine_type)` pair for performance.

---

## Important Invariants

1. **Never write `dbid` in payload** — it's DB-managed (auto-increment). Only `record_uuid`
   is used as external identity.

2. **Foreign keys point to `record_uuid`** — all Reference fields and M2M join table columns
   use `record_uuid`, not `dbid`.

3. **`RecordSet._ids` stores `record_uuid` strings** — used for all WHERE clauses and ORM ops.

4. **Relational commands run after scalar write** — in the same transaction if one is active.

5. **`ModelProxy.browse()` is lazy** — no DB hit until field access. Safe to call with
   IDs you haven't verified exist. Use `record.exists()` to filter to only live records.

7. **`record.exists()` returns a `RecordSet`, not a `bool`** — it filters the set to
   records that are confirmed in the DB. Use `bool(record.exists())` or just `if record.exists():` to check existence.

6. **Event emission from CRUD** — `ede.record.created`, `ede.record.updated`,
   `ede.record.deleted` are emitted by the CRUD handlers. Emission is fire-and-forget;
   exceptions during emit are swallowed (to avoid breaking mutations when event queue is
   unavailable).
