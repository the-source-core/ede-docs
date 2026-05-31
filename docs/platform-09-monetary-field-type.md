<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Monetary Field Type — Implementation Docs

**Module:** `ede.core.kernel` (`fields.Monetary`) + `ede.foundation.base` (`res.currency` formatting) + `ede.foundation.presentation` (widget resolution)
**Roadmap:** [roadmap/platform/09-monetary-field-type.md](../roadmap/platform/09-monetary-field-type.md)
**Status:** ✅ Delivered 2026-05-31 — kernel + currency + frontend shipped & verified; first adopters live in logistics (pricing / sales-crm / booking, 31 money fields) rendering per-currency across all 13 displaying views.
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A first-class kernel field, `fields.Monetary(currency_field="currency_id")`, for storing a money amount. Its **storage stays a plain decimal column** — it subclasses `Decimal` and keeps `field_type="decimal"`, so it adds no new persistence type and needs no migration on the amount column. What it adds is a declaration of intent: "this number is money, formatted by the currency it points at." It auto-applies the `monetary` UI widget and drives all formatting from the linked `res.currency` record.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Today every view that shows an amount must hand-wire `widget="monetary" option-currency-field="currency_id"` and also declare the currency as an invisible sibling field. The `monetary` widget exists but the formatting is hard-coded (symbol prefix, comma grouping, two decimals) and ignores the currency's own display rules. There is no field-level declaration of money and no boot-time guarantee that the named currency field is real. `fields.Monetary` collapses all of this into one declaration and makes output respect the currency master.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Model author:** declare `amount = fields.Monetary(currency_field="currency_id")` alongside a `currency_id = fields.Reference("res.currency")`. `currency_field` defaults to `currency_id`, so `fields.Monetary()` works with no arguments on the common case. No per-view widget wiring required.
- **Graceful degradation:** if the model has no `currency_id` (and no explicit `currency_field`), the value renders as a plain decimal — no boot error. An *explicit* `currency_field` that names a missing / non-currency field fails fast at boot.
- **View author:** a bare `<field name="amount"/>` renders as money automatically; an explicit `widget=` still overrides.
- **End user:** sees the amount formatted per the linked currency — symbol, symbol position, decimal places, decimal separator, and grouping separator all sourced from the `res.currency` row (e.g. `$1,234.50` vs `1.234,50 €`).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
Storage stays decimal; formatting is currency-driven and applied at the UI layer.

```
fields.Monetary(currency_field="currency_id")     # kernel — subclass of Decimal, field_type="decimal"
    └─ to_spec() → FieldSpec.default_widget="monetary" + constraints["currency_field"]
        └─ to_ui_dict() emits default_widget + currency_field
            └─ frontend resolveFieldComponent(): default-widget step → MonetaryField
                └─ MonetaryField reads res.currency row: symbol · symbol_position
                   · decimal_precision · decimal_separator · thousand_separator
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `res.currency` | Gains `symbol_position` / `decimal_separator` / `thousand_separator` formatting attributes (existing `symbol` + `decimal_precision` reused) | `src/ede/foundation/base/models/currency.py` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `fields.Monetary` | Decimal subclass carrying `currency_field` + `default_widget` | `src/ede/core/kernel/fields.py` |
| `FieldSpec` | New `default_widget`; `to_spec()` / `to_ui_dict()` emit widget + currency hints | `src/ede/core/kernel/fields.py` |
| Boot validation | `currency_field` must resolve to a `Reference(res.currency)` on the same model | `src/ede/core/kernel/model.py` |
| `resolveFieldComponent` | Default-widget resolution step | `src/frontend/src/workspace/views/fields/registry.ts` |
| `MonetaryField` | Currency-driven formatting | `src/frontend/src/workspace/views/fields/MonetaryField.tsx` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this change adds. No foundation settings or `ir.config` keys — formatting is data on `res.currency` rows, not configuration.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES`: no change — `fields.Monetary` is kernel; `res.currency` already ships in `foundation.base`.
- Manifest `depends`: no change.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `src/ede/foundation/base/data/` (currency seed) | Backfills `symbol_position` / `decimal_separator` / `thousand_separator` on seeded currencies (conventional defaults `before` / `.` / `,`) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| — | Monetary Field Type | ✅ Delivered 2026-05-31 | [platform/09](../roadmap/platform/09-monetary-field-type.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when the change reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| `fields.Monetary` kernel field (decimal-backed, auto `monetary` widget) | — | [fields.py](../src/ede/core/kernel/fields.py), [model.py](../src/ede/core/kernel/model.py) (`_validate_monetary_currency_fields`) | [platform/09](../roadmap/platform/09-monetary-field-type.md) |
| `res.currency` formatting attributes + `display_data_fields` chip plumbing | `res.currency` | [currency.py](../src/ede/foundation/base/models/currency.py), [decorators.py](../src/ede/core/kernel/decorators.py), migration `4d016112d761` | [platform/09](../roadmap/platform/09-monetary-field-type.md) |
| Frontend default-widget resolution + currency-driven formatting | — | [registry.ts](../src/frontend/src/workspace/views/fields/registry.ts), [MonetaryField.tsx](../src/frontend/src/workspace/views/fields/MonetaryField.tsx) | [platform/09](../roadmap/platform/09-monetary-field-type.md) |
| First adopters — 31 money fields across logistics pricing / sales-crm / booking | `pricing.rate`, `pricing.rate.charge.slab`, `pricing.spot.*`, `crm.quote.version`, `crm.quote.line`, `crm.handover`, `crm.opportunity`, `crm.lead`, `logistics.booking`, `logistics.booking.charge` | pricing Enh 01 · sales-crm Enh 12 · booking Enh 04 | [platform/09](../roadmap/platform/09-monetary-field-type.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| _none_ — first adopters live; per-view currency-in-readset audit clean across all 13 logistics views | — | — |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- `fields.Monetary` is **not** a new storage type — it persists as a decimal column. Switching an existing `Decimal` to `Monetary` does not migrate the amount column.
- Formatting is **currency-driven, not locale-driven** — per-user locale formatting is explicitly out of scope.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Amount columns: none — `field_type` stays `decimal`.
- `res.currency`: one additive migration adds three nullable / server-defaulted columns (`symbol_position`, `decimal_separator`, `thousand_separator`); applies on SQLite + PostgreSQL; no rename, no drop. Seeded currencies backfill conventional defaults.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Platform 02 — Computed Field Runtime](./platform-02-compute-field-runtime.md) (M2O dotted deps such as `currency_id.symbol`)
- `foundation.base` Enhancement 02 — Tenant Base Currency + Dual-Currency Storage (multi-currency, out of scope here)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-31. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
