# Archivable Models — `active` Field & Soft-Archive Auto-Filter

## What this is

EDE provides a **soft-archive** convention: any DomainModel that declares a Boolean `active` field is treated as *archivable*. The framework then automatically hides `active=False` rows from every read path so archived records do not bleed into list views, dropdowns, relational reads, or reports.

This is opt-in per model, not a property of every DomainModel. You decide whether a given entity needs soft-archive by adding the field.

## How to make a model archivable

Declare an `active` Boolean field with `default=True`:

```python
from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel

@api.model("res.country")
class Country(DomainModel):
    code = fields.Char(max_length=3, required=True)
    name = fields.Char(max_length=100, required=True)
    active = fields.Boolean(default=True, string="Active")
```

That's it. The framework detects the field at class creation time, sets the class attribute `__ede_has_active__ = True`, and the auto-filter activates.

## What auto-filtering covers

When a model is archivable, archived rows (`active=False`) are hidden from:

| Read path | Filtered? |
|---|---|
| `env.models["res.country"].search(domain)` | ✅ Yes |
| `env.models["res.country"].search_count(domain)` | ✅ Yes |
| `env.models["res.country"].read_group(...)` | ✅ Yes |
| `parent.country_id` (Reference / Many2One) | ❌ No — direct UUID lookup |
| `parent.children` (One2Many) | ✅ Yes |
| `parent.tags` (Many2Many) | ✅ Yes |
| `record.read()` / `record.exists()` / `Model.browse(uuid)` | ❌ No — direct UUID lookup |
| `env.dispatch(Command("ede.search", …))` (CRUD bus) | ✅ Yes |
| `env.dispatch(Command("ede.count", …))` | ✅ Yes |
| `env.dispatch(Command("ede.read_group", …))` | ✅ Yes |

By-UUID lookups stay unfiltered because the caller already has the UUID and is asking for that specific record — surprising them with an empty result for an archived row would be wrong.

## How to read archived records (opt-out)

There are two ways to bypass the auto-filter when you need to:

### 1. Explicit `active` leaf in the domain (per-call opt-out)

If your domain mentions the `active` field at any operator, the framework treats it as the caller's intent and does not inject the `active=True` clause:

```python
# Return everything — both active and archived.
all_records = env.models["res.country"].search([("active", "in", [True, False])])

# Return only archived.
only_archived = env.models["res.country"].search([("active", "=", False)])
```

### 2. `env.with_active_test(False)` (per-scope opt-out)

For admin tools, data exports, and bulk maintenance, clone an env with the auto-filter disabled. All ORM reads inside that env clone — including relational reads — return both active and archived rows:

```python
admin_env = env.with_active_test(False)
all_addresses = admin_env.models["res.partner.address"].search([])
```

Use this for cross-cutting bypasses; prefer the domain-leaf form for one-off queries.

## Frontend behaviour

When a model has `__ede_has_active__ = True`, the React webclient automatically:

- Includes `has_active: true` in the SearchViewDef returned by `presentation.load_action`.
- Renders an **"Archived"** toggle chip in the search panel's filters list.
- When the user activates the toggle, an explicit `[["active","in",[true,false]]]` leaf is added to the search domain — this trips the per-call opt-out, and the list shows both active and archived rows.

No frontend code change is required when you add `active` to a new model. The toggle appears for any archivable model.

## When to declare `active` (and when NOT to)

**Do** declare `active` on:

- Reference / master data that users may want to deprecate without deleting (countries, currencies, UoMs, partners, tags, roles, sequences).
- Configuration entities that ship with seed data and may be turned off per tenant.

**Do not** declare `active` on:

- Transactional records (orders, shipments, invoices, postings, journal entries). These have a `state` / `status` field that drives their lifecycle. Deletes for transactional records are typically forbidden by lifecycle hooks.
- Pure log / audit / event records — they shouldn't be hidden after the fact.
- Models where archival semantics are not meaningful.

## Naming convention

The auto-filter only triggers on a field named exactly **`active`** with `field_type == "boolean"`. Some legacy foundation models use `is_active` instead — those keep their current behaviour and do **not** get auto-filtered. A separate naming-unification effort tracks renaming `is_active` → `active` across the platform.

## Reference

| Symbol | Location |
|---|---|
| `__ede_has_active__` registry flag | [src/ede/core/kernel/model.py](../src/ede/core/kernel/model.py) — set in `DomainModel.__init_subclass__` |
| Domain helpers | [src/ede/core/services/persistence/domain_filter.py](../src/ede/core/services/persistence/domain_filter.py) — `domain_mentions_active`, `augment_with_active_filter`, `apply_active_filter` |
| Env opt-out | [src/ede/core/env.py](../src/ede/core/env.py) — `Env.with_active_test()` |
| Tests | [src/tests/orm/test_active_filter.py](../src/tests/orm/test_active_filter.py) |
| Frontend toggle | [src/frontend/src/workspace/components/search/ControlPanelSearchView.tsx](../src/frontend/src/workspace/components/search/ControlPanelSearchView.tsx) |
