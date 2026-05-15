# EDE Framework — Domain Model & Fields

## DomainModel

`DomainModel` is the base class for all business entities in EDE. Subclass it, decorate with
`@api.model(key)`, declare fields, and attach command handlers.

```python
from ede.core import api
from ede.core.kernel.model import DomainModel
from ede.core.kernel import fields
from ede.core.types import Command

@api.model("logistics.shipment")
class Shipment(DomainModel):
    tracking_number = fields.Char(max_length=50, required=True, unique=True)
    status          = fields.Char(max_length=20, required=True, default="pending")
    weight_kg       = fields.Decimal(required=False)
    origin_country  = fields.Reference("res.country")

    @api.on_command("shipment.create")
    def create(self, cmd: Command) -> dict:
        values = cmd.payload
        record = self.env.models["logistics.shipment"].create(values)
        return record.read()[0]

    @api.on_command("shipment.confirm")
    def confirm(self, cmd: Command) -> dict:
        record = cmd.record    # RecordSet resolved from cmd.model_id
        record.write({"status": "confirmed"})
        self.emit("shipment.confirmed", {"id": record.id})
        return {"ok": True}
```

---

## @api.model Decorator

```python
@api.model(
    model_key,                    # Required: "domain_type.model_name"
    table=None,                   # Optional: override SQL table name (default: key with . → _)
    description=None,             # Optional: human-readable description
    default_order=None,           # Optional: SQL ORDER BY default (e.g. "created_at_utc desc")
    name_search_fields=None,      # Optional: fields for name_search fuzzy matching
    auto_fields=True,             # Set False to disable auto-field injection
    abstract=False,               # Set True → no table, no migration
)
```

The decorator:
1. Sets `__ede_model_key__`, `__ede_table_name__` (derived or explicit), and other metadata.
2. Auto-registers the class into `runtime.registry` when `runtime.auto_register_models=True`
   (always True during module loading).
3. Returns the class unchanged (transparent decorator).

---

## Auto Fields

Every `DomainModel` subclass gets these fields injected automatically unless
`__ede_disable_auto_fields__ = True` (or `auto_fields=False` in `@api.model`):

| Field | Type | Notes |
|---|---|---|
| `dbid` | Integer | Auto-increment primary key. DB-managed. Never set in INSERT payload. |
| `record_uuid` | UUID | Unique external identity. Used in API URLs, foreign keys, `RecordSet._ids`. |
| `created_at_utc` | DateTime | Set by repo on INSERT. Indexed. |
| `updated_at_utc` | DateTime | Updated by repo on every UPDATE. Indexed. |
| `revision` | Integer | Defaults to 0. Incremented on each write (optimistic concurrency hint). |

**`id` property** is kept as a backward-compat alias for `record_uuid`.

**All FK columns** (Reference fields, M2M join tables) point to `record_uuid` of the target
table — never `dbid`.

---

## Abstract Models

Use `AbstractDomainModel` (or `abstract=True` in `@api.model`) to define shared field mixins.
Abstract models:
- Contribute fields to concrete subclasses
- Do **not** create a DB table
- Do **not** generate migrations
- Are not registered as models in the registry

```python
from ede.core.kernel.model import AbstractDomainModel

class TimestampedResource(AbstractDomainModel):
    published_at = fields.DateTime(required=False)
    is_active    = fields.Boolean(default=True)

@api.model("content.article")
class Article(TimestampedResource):
    title = fields.Char(max_length=200, required=True)
    # inherits: published_at, is_active, + auto fields
```

---

## Field Descriptors

All field types live in `ede.core.kernel.fields`. They are Python descriptors — declared as
class attributes.

### Common Parameters (all field types)

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `required` | bool | False | If True, INSERT without this field raises validation error |
| `unique` | bool | False | Adds unique index on column |
| `index` | bool | False | Adds non-unique index on column |
| `readonly` | bool | False | Prevents writes via `write()` |
| `default` | Any | None | Literal default value |
| `default_factory` | callable | None | Factory called per record (e.g. `list`) |
| `label` | str | None | Human-readable label for UI |
| `help` | str | None | Tooltip / help text for UI |

---

### Scalar Field Types

#### `fields.UUID`
```python
model_ref = fields.UUID(required=True)
```
Stores a UUID. In SQLite: `String(36)`. In PostgreSQL: native `UUID`.

#### `fields.Char`
```python
name = fields.Char(max_length=100, required=True, unique=True)
```
Variable-length string (`VARCHAR`). `max_length` becomes a `maxlength` constraint.

#### `fields.Integer`
```python
quantity = fields.Integer(required=True, default=0)
```
SQL `INTEGER`.

#### `fields.Decimal`
```python
price = fields.Decimal(precision=10, scale=2)
```
SQL `NUMERIC(precision, scale)`. Use for currency, measurements. Not Float.

#### `fields.Boolean`
```python
is_active = fields.Boolean(default=True)
```
SQL `BOOLEAN`.

#### `fields.Date`
```python
due_date = fields.Date(required=False)
```
SQL `DATE`. Values as Python `date` objects.

#### `fields.DateTime`
```python
dispatched_at = fields.DateTime(required=False)
```
SQL `DATETIME` (SQLite) / `TIMESTAMP` (PostgreSQL). Always UTC. Values as Python `datetime`.

#### `fields.Enum`
```python
status = fields.Enum(["pending", "confirmed", "shipped", "delivered"], default="pending")
```
Stored as `VARCHAR`. The selection list is serialized for UI field rendering.

#### `fields.JSON`
```python
metadata = fields.JSON(default_factory=dict)
```
Stored as `TEXT` (SQLite) or `JSONB` (PostgreSQL). Use for flexible unstructured data.

---

### Relational Field Types

#### `fields.Reference` (Many-to-One / FK)
```python
country = fields.Reference("res.country")
warehouse = fields.Reference("logistics.warehouse", on_delete="restrict")
```
Adds a FK column pointing to `record_uuid` of the target model's table.
Always indexed. `on_delete` defaults to `"set_null"`.

Accessing via RecordSet returns a single-record RecordSet of the related model:
```python
shipment = env.models["logistics.shipment"].browse(some_id)
country = shipment.country     # → RecordSet of res.country (1 record)
print(country.name)
```

#### `fields.One2Many` (reverse FK)
```python
order_lines = fields.One2Many("logistics.order_line", backref="order")
```
Not stored as a column. Declared on the parent model. `backref` is the FK field on the child
model that points back.

Read-only by convention. Write via relational command tuples in `write()`.

```python
order.write({
    "order_lines": [
        RelationalCommand.create({"product": "Widget", "qty": 3}),
        RelationalCommand.delete(existing_line_id),
    ]
})
```

#### `fields.Many2Many` (join table)
```python
tags = fields.Many2Many("res.tag")
```
Not stored as a column. EDE auto-derives a join table name:
`{model_table}_{field_name}_rel`.

Write via relational command tuples:
```python
record.write({
    "tags": [
        RelationalCommand.link(tag_id),    # add existing tag
        RelationalCommand.unlink(tag_id),  # remove link (does not delete tag)
        RelationalCommand.set([id1, id2]), # replace all links
        RelationalCommand.clear(),         # remove all links
    ]
})
```

---

### Computed Fields

A field can have a `method` (compute) to derive its value from other fields:

```python
@api.model("logistics.shipment")
class Shipment(DomainModel):
    weight_kg = fields.Decimal()
    weight_lb = fields.Decimal(
        method="compute_weight_lb",
        store=False,          # not persisted (default)
        depends_on=["weight_kg"],
    )

    def compute_weight_lb(self):
        return self.weight_kg * 2.20462
```

- `store=False` (default): computed on read, never persisted.
- `store=True`: persisted on write when `depends_on` fields change.
- Computed fields are always `readonly=True`.

---

## FieldSpec (Metadata Object)

Every field descriptor produces a `FieldSpec` — an immutable, serializable metadata object
used for schema generation and UI rendering.

```python
# Access field specs on a model class:
specs = Shipment.__ede_field_specs__      # Dict[str, FieldSpec]
spec  = specs["tracking_number"]
print(spec.name)         # "tracking_number"
print(spec.field_type)   # "char"
print(spec.required)     # True
print(spec.max_length)   # 50
print(spec.to_ui_dict()) # serialized for frontend
```

`to_ui_dict()` output includes selection options for `Enum`, target model key for relational
fields, and all constraints.

---

## Schema Metadata (ORM-Agnostic)

`ede.core.kernel.schema` converts DomainModel field specs into database-agnostic schema specs:

```python
from ede.core.kernel.schema import build_model_schema

schema = build_model_schema(Shipment)
# schema.table_name     → "logistics_shipment"
# schema.columns        → tuple of ColumnSpec
# schema.foreign_keys   → tuple of ForeignKeySpec  (from Reference fields)
# schema.indexes        → tuple of IndexSpec
# schema.join_tables    → tuple of JoinTableSpec    (from Many2Many fields)
```

These specs are consumed by `SqlAlchemyMetadataBuilder` to produce SQLAlchemy `Table`
objects for migration and query execution.

---

## Model Method Utilities

### `self.env`
The request-scoped `Env` object. Access other models, dispatch commands, emit events.

### `self.emit(event_name, payload)`
Enqueues an event to the EventQueue:
```python
self.emit("shipment.confirmed", {"shipment_id": record.id, "tenant_id": self.env.tenant_id})
```

### `cmd.record`
Inside a command handler, `cmd.record` returns the `RecordSet` targeted by `cmd.model_id`:
```python
@api.on_command("shipment.confirm")
def confirm(self, cmd: Command) -> dict:
    record = cmd.record     # → RecordSet
    record.write({"status": "confirmed"})
```
`cmd.record` is only valid when `cmd.model_id` is set (targeted commands on existing records).

---

## Model Key Rules

The model key in `@api.model` must follow:
```
"{domain_type}.{descriptive_name}"
```

Foundation models use the `res.*` / `ir.*` convention:
- `res.*` — shared business resources (`res.country`, `res.organization`, `res.user`)
- `ir.*` — framework internal metadata (`ir.session`, `ir.menu`, `ir.action`)

User domain models use `{domain_type}.{model_name}`:
- `logistics.shipment`
- `crm.lead`

Never use `ede.*` — that namespace is reserved for framework kernel commands (`ede.create`,
`ede.update`, `ede.delete`, etc.).

---

## Why DomainModel Is the Right Boundary

### Business logic lives in one place

`DomainModel` is the only place where business rules, state transitions, field
definitions, and domain-specific commands co-exist. There is no service layer with
scattered logic, no "fat controller" pattern, and no SQL leaking into methods.
When you need to understand how `res.organization.deactivate` works, you go to one
class.

### Fields drive the entire system

Field descriptors on a `DomainModel` are not just documentation — they are the
single source of truth that feeds:
- **Migrations**: `SqlAlchemyMetadataBuilder` converts `FieldSpec` → SQLAlchemy `Table`
- **UI**: `FieldSpec.to_ui_dict()` serializes field metadata for the frontend renderer
- **Validation**: `required`, `unique`, `max_length` are enforced by the repository
- **Field tracking**: `track_fields` hooks auto-register against declared field names

Declaring a field once propagates correctly to all layers automatically.

### Abstract models prevent duplication without shared tables

`AbstractDomainModel` lets you share field definitions (e.g. `is_active`, `published_at`)
across models without creating a shared DB table. This is the correct tool for reuse
within the domain layer — no mixin inheritance gymnastics, no ORM base class tricks.
