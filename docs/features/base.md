# Base — Platform Substrate

The cross-domain masters and platform metadata every other module relies on. Reference them directly from your domain models — no app dependency dance needed.

```python
# Look up a country, a unit of measure, a partner — anywhere in your code.
india    = env.models["res.country"].search([("iso_code", "=", "IN")])[0]
kilogram = env.models["res.uom"].search([("code", "=", "KG")])[0]
acme     = env.models["res.partner"].create({
    "name": "Acme Logistics",
    "country_id": india.id,
})
```

---

## What you get

The `foundation.base` module ships three families of models. Every other module — foundation or domain — references these as-is.

### Cross-domain business masters (`res.*`)

| Model | Purpose |
|---|---|
| `res.country` | ISO country codes, names, default timezone. |
| `res.state` | Province / state under a country. |
| `res.city` | Cities under a country + state. |
| `res.timezone` | IANA timezones with UTC offsets. |
| `res.language` | BCP-47 language tags. Referenced by `res.user.language_id`. |
| `res.currency` | ISO 4217 currencies. |
| `res.exchange.rate` | FX rates by date and rate-type (daily / monthly / company / customs / locked). |
| `res.exchange.rate.type` | Rate-type classifications. |
| `res.uom.category` | UOM groupings (weight, volume, dimension, quantity). |
| `res.uom` | Units of measure with conversion factor inside a category. |
| `res.partner` | Universal party master — customer, vendor, carrier, employee contact. |
| `res.partner.role.master` | Role definitions assignable to partners. |
| `res.partner.role` | Role assignment join (`res.partner` ↔ role master). |
| `res.partner.address` | Multiple addresses per partner (billing / pickup / delivery / …). |
| `res.organization` | Legal entity / company within the tenant. |
| `res.user` | Application users. |

### Platform metadata (`ir.*`)

| Model | Purpose |
|---|---|
| `ir.menu` | App-switcher entries and side-menu trees. |
| `ir.action` | `ir.action` records bind menus to models / commands. |
| `ir.sequence` | Numbered sequence generators (`PO/2026/0001`, etc.). |
| `ir.config` | Runtime key-value store for tenant-tunable settings. |
| `ir.rbac.role` | RBAC role definitions. |
| `ir.rbac.permission` | Permission definitions (`<resource>:<action>`). |
| `ir.rbac.binding` | User-role assignments, optionally scoped to an organization. |
| `ir.rbac.record.rule` | Per-role row-level filters layered on top of permissions. |
| `ir.rbac.decision.log` | Per-request authorization decision audit trail. |
| `ir.rbac.binding.change.log` | Binding-change audit trail. |
| `ir.model` | Reflection of registered domain models. |
| `ir.model.field` | Reflection of every registered field. |
| `ir.model.field.selection` | Reflection of Enum-field options. |
| `ir.model.property.definition` | Tenant-defined property schema via [Customization (Properties)](customization.md). |
| `ir.model.property.selection` | Option rows for selection-typed properties. |
| `ir.model.extension` | Mirror of `@api.extend_model` registrations — see [Model & View Extension SDK](extension-sdk.md). |
| `ir.data.reference` | XML/CSV record references for the data loader. |
| `ir.notification.setting` | Per-user opt-in/opt-out for notification channels. |

### Health-check

`foundation.base` also registers a `ping` model used by smoke tests and uptime probes. Send command `ping.send` → expect `pong` event.

---

## How to use it

### Create an organization

```python
acme = env.models["res.organization"].create({
    "name": "Acme Logistics Pvt Ltd",
    "country_id": env.models["res.country"].search([("iso_code", "=", "IN")])[0].id,
    "default_currency_id": env.models["res.currency"].search([("iso_code", "=", "INR")])[0].id,
})
```

`res.organization` is the unit of company-scoping — every record that is `company_scope="strict"` or `"optional"` (see [Security & Authorization](security.md)) pins to one organization at create time.

### Look up a partner

```python
partner = env.models["res.partner"].search([
    ("name", "ilike", "Acme"),
])
```

Use the universal `res.partner` model for customers, vendors, carriers, agents — anything that transacts with the business. Attach role tags via `res.partner.role` so a partner can be both a customer and a vendor without duplication.

### Convert between UOMs

```python
kg = env.models["res.uom"].search([("code", "=", "KG")])[0]
gm = env.models["res.uom"].search([("code", "=", "G")])[0]

# 250 grams in kilograms
kg_value = (250 * gm.factor) / kg.factor   # → 0.25
```

`res.uom.factor` is the conversion factor *relative to the category's reference UOM* (defined on `res.uom.category`). All UOMs in the same category share that reference, so any-to-any conversion is one division.

### Convert between currencies

```python
inr   = env.models["res.currency"].search([("iso_code", "=", "INR")])[0]
usd   = env.models["res.currency"].search([("iso_code", "=", "USD")])[0]
rate  = env.models["res.exchange.rate"].search([
    ("from_currency_id", "=", inr.id),
    ("to_currency_id",   "=", usd.id),
    ("rate_date",        "<=", today),
], order="rate_date desc", limit=1)[0]

usd_amount = inr_amount * float(rate.rate)
```

Multiple rate types coexist (`daily` for live FX, `customs` for declaration purposes, `locked` for contract pricing). Pick the right `rate_type_id` for the operation you're booking.

### Generate a sequential number

```python
po_number = env.models["ir.sequence"].next_by_code("purchase.order")
# → "PO/2026/00042"
```

Define the sequence once (prefix, suffix, padding, reset rule) via an XML data file or the Settings UI; consume it from anywhere.

### Read runtime config

```python
default_pageview_size = env.config.get_int("LIST_VIEW_DEFAULT_PAGE_SIZE", 80)
```

`ir.config` holds runtime-tunable settings that aren't framework-level (those live in `ede.conf` / pydantic settings). Use it for business-tunable knobs administrators may change in the Settings UI.

---

## Manifest dependency

Every other app must list `foundation.base` (transitively) in its `depends`:

```python
# src/domains/blog/post/__manifest__.py
{
    "name": "Blog Post",
    "depends": ["base"],
    # …
}
```

`foundation.base` itself has no `depends` — it's the root of the dependency graph.

---

## Reference

-   Source: `src/ede/foundation/base/`
-   Architecture: [Foundation Apps](../11-foundation-apps.md) for the platform layer's design.
-   See also: [Security & Authorization](security.md) for `ir.rbac.*` deep-dive, [Customization (Properties)](customization.md) for `ir.model.property.definition`, [Model & View Extension SDK](extension-sdk.md) for `@api.extend_model` + `<extend ref="...">`, [Internationalization](i18n.md) for translations layered on `res.language`.
