# Foundation Model Naming Convention

This document defines the model key prefix conventions for **foundation-only** models —
models that live inside `ede.foundation.base` and are shared across all apps and domains.

Domain-specific models (inside `src/domains/`) use their own domain prefix and are not
covered here.

---

## Two prefixes: `res.*` and `ir.*`

Inspired by Odoo's battle-tested `base` module naming convention.

| Prefix | Scope | Python location | Examples |
|--------|-------|-----------------|---------|
| `res.*` | Shared business resources — entities, users, reference data | `ede.foundation.base/models/` | `res.organization`, `res.user`, `res.country`, `res.currency`, `res.lang` |
| `ir.*`  | Internal framework metadata — model registry, menus, actions | `ede.foundation.base/models/` | `ir.model`, `ir.field`, `ir.menu`, `ir.action` |
| `ede.*` | Kernel internals — **reserved, do not use in app code** | `ede.core.kernel.*` | `ede.crud` |

---

## `res.*` — Shared Business Resources

**What belongs here:** Entities that are universally shared across all business domains.
Every app that needs to reference an organisation, a user, or a country uses a `res.*` key.

**Key principle:**
- `res.user` is the **data model** (persisted user record).
- The `auth` app provides the **behaviour layer** (login, token issuance) that operates
  *on* `res.user`. This separation keeps the User aggregate clean and testable in isolation.

**Current models:**

| Model key | Table | Description |
|-----------|-------|-------------|
| `res.country` | `res_country` | ISO country reference (name, code, phone_code, currency_code) |
| `res.organization` | `res_organization` | Root organisation aggregate |
| `res.user` | `res_user` | Platform user (email, name, password_hash, is_active) |

**Planned:**

| Model key | Table | Description |
|-----------|-------|-------------|
| `res.currency` | `res_currency` | ISO 4217 currency (code, symbol, rounding) |
| `res.lang` | `res_lang` | Locale / language (code, name, direction) |
| `res.contact` | `res_contact` | Person or company contact |

---

## `ir.*` — Internal Framework Metadata

**What belongs here:** Entities that describe the framework itself — model registry,
menu tree, action definitions. These power the frontend's data-driven UI and admin tooling.

**Rules:**
- Populated at boot time or via migrations, not by user workflows.
- Read-only from the public API (except admin tooling).

**Planned:**

| Model key | Table | Description |
|-----------|-------|-------------|
| `ir.model` | `ir_model` | Registered model catalogue (key, label, app) |
| `ir.field` | `ir_field` | Field definitions per model |
| `ir.menu` | `ir_menu` | Navigation tree (title, icon, parent, action) |
| `ir.action` | `ir_action` | Declarative actions linked to menu items |

---

## `ede.*` — Kernel Internals (Reserved)

Used exclusively by `ede.core.kernel.*`. Do **not** use this prefix in app code.

Currently: `ede.crud` — the `CrudKernel` command dispatcher.

---

## Table Naming

Model key → SQL table name: dots replaced with underscores.

```
res.organization  →  res_organization
res.user          →  res_user
ir.model          →  ir_model
```

Handled automatically by `model_key_to_table_name()` in
`ede.core.adapters.persistence.sqlalchemy.schema`.

---

## URL Segment Convention

Dots become double-underscores (`__`) in HTTP URLs:

```
res.organization  →  res__organization
res.user          →  res__user
```

Converted back by `_url_to_model_key()` in `ede.foundation.base.api.crud_routes`.

---

## Command Strategy for `res.*` models

Generic CRUD (create / read / update / delete / search / count) is handled by
`CrudKernel` — callers use `ede.create`, `ede.update`, etc. with the appropriate
`model_key`.

Models only define **domain-specific commands** where business invariants or
side-effects are required:

| Model | Domain commands |
|-------|----------------|
| `res.country` | *(none — pure reference data)* |
| `res.organization` | `res.organization.change_country`, `res.organization.deactivate` |
| `res.user` | `res.user.register`, `res.user.set_password`, `res.user.deactivate`, `res.user.activate` |

---

## Quick-Reference Checklist

1. **Is it a shared business entity or user record?** → `res.*`
2. **Does it describe the framework structure** (menus, model registry)? → `ir.*`
3. **Is it domain-specific** (invoice, shipment, employee)? → use the domain prefix.

---

## Code-Bearing Masters — Always Declare `name_search_fields` + `display_name_format`

Any master that carries both a short *code* (`USD`, `FOB`, `BUDGET`, `[6109.10]`) and a long *name* (`US Dollar`, `Free on Board`, `Budget exceeded`, `T-shirts`) MUST declare both decorator kwargs. This makes every M2O picker pointing at the model search by code-or-name and render the dropdown / chip as `[CODE] Name` automatically — no view-XML edits needed anywhere.

```python
@api.model(
    "res.currency",
    record_name="name",
    name_search_fields=["code", "name"],          # picker matches typing "USD" or "Dollar"
    display_name_format="[{code}] {name}",        # picker renders "[USD] US Dollar"
)
class Currency(DomainModel):
    code = fields.Char(max_length=3, required=True, ...)
    name = fields.Char(max_length=100, required=True, ...)
```

### Why declare these on the model rather than per-view

- One declaration; every view that touches the model inherits the behaviour. `<field name="currency_id"/>` on a quote form, a list cell, a kanban card, a search bar — all render `[USD] US Dollar` and search both fields with no per-view XML changes.
- Renames are caught at boot. Both kwargs are validated against the model's declared fields at decoration time; renaming `code` → `iso_code` without updating the format raises `InvalidHandler` before the server starts. The format is part of the model's declared interface, not a free-form string buried in a view.
- The override path stays available — a single view that needs different formatting still overrides via the first-class `display_format` attribute on `<field>` (`<field name="currency_id" display_format="{code}"/>`).

### When to choose which fields

- **`name_search_fields`**: include every field the user might type. For currency, that's `["code", "name"]`. For commodity with an HS code, `["code", "name", "hs_reference"]`. Keep the list short (3 or fewer) — every field becomes an `ilike` clause in an OR-domain.
- **`display_name_format`**: pick the field combination users want to *see* — typically `[{code}] {name}`. The bracketed code anchors the eye while the name disambiguates. Use the field that uniquely identifies the row at a glance (so for incoterms, `term_code` like `FOB`, not the disambiguated `code` like `FOB_2020`).

### When NOT to declare them

- Models with no short code (party records keyed only by display name).
- Models where the only "code" is an internal UUID slug (defeats the readability point).
- Pure transactional records — `display_name_format` is for masters that get *referenced*, not for masters that get *operated on*.

See [`docs/foundation-presentation.md`](foundation-presentation.md#m2o-templated-display--multi-field-name-search) for the full wire-format / frontend-precedence documentation.

---

## Upcoming: `dbid` + `record_ref_id`

The current auto-field `id` (UUID v4) will be complemented by an auto-incremental
integer `dbid`. When this lands, the UUID `id` will be renamed to `record_ref_id`
and `dbid` becomes the primary integer surrogate key. All existing model definitions
remain unchanged; only the auto-field layer will be updated.
