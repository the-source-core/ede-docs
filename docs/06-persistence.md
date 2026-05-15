# EDE Framework — Persistence

## Design Philosophy

Persistence in EDE is:
- **Protocol-driven**: `PersistenceProvider` is a Python `Protocol`, not a base class.
  Any object implementing the protocol is valid.
- **Pluggable**: swap providers via `DEFAULT_PERSISTENCE_PROVIDER` setting.
- **Tenant-isolated**: each tenant gets its own SQLite file or PostgreSQL database.
- **Schema-separated**: schema metadata is ORM-agnostic (`ColumnSpec`, `ForeignKeySpec`, etc.)
  and translated into SQLAlchemy `Table` objects by an adapter.

### Why this approach future-proofs the system

**The domain never imports from persistence.** `DomainModel` fields declare `storage_type`
as a string (`"char"`, `"integer"`, `"uuid"`). The adapter converts those to the
correct SQLAlchemy column types. If you swap to a different persistence backend (Mongo,
DynamoDB, a time-series DB for specific models), you implement the `PersistenceProvider`
Protocol — no changes to field declarations, no changes to model methods.

**Schema-as-data, not schema-as-code.** The ORM-agnostic `ModelSchemaSpec` is a plain
dataclass that can be inspected, serialized, or used to drive migration generation,
documentation, or API schema export. It is not bound to any specific ORM or DB technology.

**`NullPersistenceProvider` for fast testing.** Unit tests that don't touch the DB
run with `DEFAULT_PERSISTENCE_PROVIDER = "none"`. Test suites stay fast even as the
model count grows. Integration tests opt in to SQLite in-memory when they need actual
persistence behaviour.

---

## Providers

Configure via `settings.py`:

```python
DEFAULT_PERSISTENCE_PROVIDER = "sqlalchemy"   # or "none"
```

| Key | Class | Notes |
|---|---|---|
| `"sqlalchemy"` | `SqlAlchemyPersistenceProvider` | Full SQL backend (SQLite/PostgreSQL) |
| `"none"` | `NullPersistenceProvider` | No-op; always returns health error; raises on save |

---

## PersistenceProvider Protocol

`src/ede/core/services/persistence/contracts.py`

```python
class PersistenceProvider(Protocol):
    def uow(self, tenant_id: str | None = None) -> UnitOfWork: ...
    def health_check(self, tenant_id: str | None = None) -> Dict[str, Any]: ...
    def save(self, model: Any) -> Any: ...
```

### UnitOfWork Protocol

```python
class UnitOfWork(Protocol):
    def commit(self) -> None: ...
    def rollback(self) -> None: ...
    def close(self) -> None: ...
```

The ORM layer never uses `PersistenceProvider.save()` directly. It calls `uow()` to get a
SQLAlchemy session, then executes Core SQL statements within that session.

---

## SQLAlchemy Provider

`src/ede/core/adapters/persistence/sqlalchemy/sqlalchemy_provider.py`

### Configuration

```python
# SQLite (dev)
DATABASE_ENGINE = "sqlite"
SQLITE_TENANT_DATABASE_DIR = ".ede/databases"

# PostgreSQL (prod)
DATABASE_ENGINE = "postgresql"
POSTGRES_HOST = "localhost"
POSTGRES_PORT = 5432
POSTGRES_USER = "ede"
POSTGRES_PASSWORD = "secret"
DATABASE_NAME_PREFIX = "db_ede"
```

### Database URL Builder

`src/ede/core/adapters/persistence/sqlalchemy/database_url.py`

Per-tenant database URLs:

```
SQLite:    sqlite:///{SQLITE_TENANT_DATABASE_DIR}/{DATABASE_NAME_PREFIX}_{tenant_id}.db
PostgreSQL: postgresql://{user}:{password}@{host}:{port}/{DATABASE_NAME_PREFIX}_{tenant_id}
```

Example:
- tenant `"acme"` + SQLite → `sqlite:///.ede/databases/db_ede_acme.db`
- tenant `"acme"` + PostgreSQL → `postgresql://ede:secret@localhost:5432/db_ede_acme`

### PostgreSQL DB Provisioning

`src/ede/core/adapters/persistence/sqlalchemy/admin.py`

`ensure_database_exists(tenant_id, settings)` — connects to `postgres` default DB and
creates the tenant DB if it doesn't exist. Called by `ede migrate upgrade` before running
Alembic.

---

## Tenant Database Identity

`src/ede/core/services/persistence/tenant_identity.py`

`TenantDatabaseIdentityResolver` resolves `tenant_id` → `TenantDatabaseIdentity`:

```python
@dataclass(frozen=True)
class TenantDatabaseIdentity:
    tenant_id: str
    database_name: str    # e.g. "db_ede_acme"
```

Used by the persistence provider to build the connection URL for each tenant.

---

## SQLAlchemy MetaData Builder

`src/ede/core/adapters/persistence/sqlalchemy/metadata_builder.py`

Converts EDE ORM-agnostic schemas into SQLAlchemy `Table` objects.

```python
result = SqlAlchemyMetadataBuilder().build(
    model_classes=registry.list_models(),
    database_engine="sqlite",  # or "postgresql"
)
# result.metadata   → sqlalchemy.MetaData (all tables)
# result.schemas    → Dict[str, ModelSchemaSpec]
```

### Column Type Mapping

| EDE FieldSpec type | SQLite | PostgreSQL |
|---|---|---|
| `"uuid"` | `String(36)` | `UUID` (native) |
| `"char"` | `String(max_length)` | `String(max_length)` |
| `"integer"` | `Integer` | `Integer` |
| `"decimal"` | `Numeric(precision, scale)` | `Numeric(precision, scale)` |
| `"boolean"` | `Boolean` | `Boolean` |
| `"date"` | `Date` | `Date` |
| `"datetime"` | `DateTime` | `DateTime(timezone=True)` |
| `"json"` | `Text` | `JSONB` |
| `"text"` | `Text` | `Text` |

### Index Naming Convention

All index names are deterministic:
- Non-unique: `idx_{table_name}_{col1}_{col2}`
- Unique: `uidx_{table_name}_{col1}_{col2}`

### Table Ownership Metadata

Each `Table` object carries `.info` dict with:
```python
table.info = {
    "ede_model_key": "logistics.shipment",
    "ede_app_key":   "logistics.shipment",
}
```
This is used by Alembic to determine which app "owns" a table for per-app migration
version locations.

---

## Schema Specs (ORM-Agnostic)

`src/ede/core/kernel/schema.py`

These specs are the source of truth for schema — adapter-independent:

### ColumnSpec

```python
@dataclass(frozen=True)
class ColumnSpec:
    name: str
    storage_type: str       # "uuid", "char", "integer", etc.
    required: bool
    primary_key: bool
    unique: bool
    readonly: bool
    default: Any
    default_factory: Any
    constraints: dict       # e.g. {"max_length": 100}
    compute: Any            # None or ComputeSpec
```

### ForeignKeySpec

```python
@dataclass(frozen=True)
class ForeignKeySpec:
    column_name: str        # FK column in current table
    target_table: str       # target model's table name
    target_column: str      # always "record_uuid"
    on_delete: str          # "set_null", "cascade", "restrict"
```

### IndexSpec

```python
@dataclass(frozen=True)
class IndexSpec:
    name: str               # deterministic: idx_ or uidx_ prefix
    columns: tuple          # column names
    unique: bool
```

### JoinTableSpec (Many2Many)

```python
@dataclass(frozen=True)
class JoinTableSpec:
    table_name: str         # "{model_table}_{field_name}_rel"
    left_table: str
    right_table: str
    left_column: str        # FK to left model's record_uuid
    right_column: str       # FK to right model's record_uuid
    foreign_keys: tuple
    indexes: tuple
```

### ModelSchemaSpec

```python
@dataclass(frozen=True)
class ModelSchemaSpec:
    model_key: str
    table_name: str
    columns: tuple          # of ColumnSpec
    foreign_keys: tuple     # of ForeignKeySpec
    indexes: tuple          # of IndexSpec
    join_tables: tuple      # of JoinTableSpec (from Many2Many fields)
```

---

## Domain Filter (WHERE Clause DSL)

`src/ede/core/services/persistence/domain_filter.py`

The domain filter is a list of `(field, operator, value)` triples. It is parsed into a
SQLAlchemy `WHERE` clause.

```python
domain = [
    ("status", "=", "pending"),
    ("weight_kg", ">=", 1.0),
    ("origin_country", "is not null", None),
]
```

Nested AND/OR is not supported at MVP — all conditions are AND-ed.

| Operator | SQL | Notes |
|---|---|---|
| `"="` | `col = value` | |
| `"!="` | `col != value` | |
| `">"` | `col > value` | |
| `">="` | `col >= value` | |
| `"<"` | `col < value` | |
| `"<="` | `col <= value` | |
| `"like"` | `col LIKE value` | Case-sensitive; use `%` wildcards |
| `"ilike"` | `col ILIKE value` | Case-insensitive; SQLite: uses LOWER() |
| `"in"` | `col IN (...)` | value must be a list |
| `"not in"` | `col NOT IN (...)` | value must be a list |
| `"is null"` | `col IS NULL` | |
| `"is not null"` | `col IS NOT NULL` | |

---

## MetaData Cache

SQLAlchemy `MetaData` is cached globally per `(frozenset(model_keys), engine_type)` pair.
This avoids rebuilding table metadata on every request.

**Invalidation**: The cache is rebuilt only at process restart or if model classes change
(i.e., after reloading in development). Never manually invalidate in production.

---

## Health Check

```python
health = env.persistence.health_check(tenant_id="acme")
# → {"provider": "sqlalchemy", "ok": True, "tenant_id": "acme"}
# or on failure:
# → {"provider": "sqlalchemy", "ok": False, "error": "..."}
```

`NullPersistenceProvider` always returns `{"ok": False, "reason": "null provider"}`.

---

## Null Provider

Use `DEFAULT_PERSISTENCE_PROVIDER = "none"` for:
- Unit tests that don't need DB
- Development spikes
- Apps that only process events without persisting state

```python
class NullPersistenceProvider:
    def uow(self, tenant_id=None) -> NullUnitOfWork: ...
    def health_check(...) -> {"provider": "none", "ok": False, ...}
    def save(...) -> raise RuntimeError("NullPersistenceProvider cannot save")
```

---

## Adding a New Persistence Provider

Implement the `PersistenceProvider` Protocol:

```python
class MyCustomProvider:
    provider_key = "my_custom"

    def uow(self, tenant_id=None) -> MyUnitOfWork:
        ...

    def health_check(self, tenant_id=None) -> dict:
        ...

    def save(self, model) -> Any:
        ...

    @classmethod
    def from_settings(cls, settings, resolver, tenant_id_getter) -> "MyCustomProvider":
        ...
```

Register in `src/ede/core/adapters/__init__.py`:

```python
registry.register_message_broker_provider("my_custom", MyCustomProvider.from_settings)
```

Then set `DEFAULT_PERSISTENCE_PROVIDER = "my_custom"` in settings.
