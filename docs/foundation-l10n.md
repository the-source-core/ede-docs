<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Localization SDK (l10n) — Implementation Docs

**Module:** `foundation.l10n` (`src/ede/foundation/l10n/`)
**Roadmap:** [roadmap/foundation/l10n/README.md](../roadmap/foundation/l10n/README.md)
**Status:** 🔴 Not Started — blocked on [`foundation.base` Phase 2 — Model & View Extension SDK](../roadmap/foundation/base/phase-2/README.md). Restructured 2026-05-17: the generic decorator + registry merge + view inheritance machinery were lifted out of l10n Phase 1 into base Phase 2 so every consumer (assistant, customization, marketplace, future) shares one SDK. l10n now layers country-scope + per-org activation on top.
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A country-scope extension SDK that turns one shared model into many country-specific variants without duplicating the model, the form, or the workflow. Domains declare the universal fields. Country packs (`src/localizations/<cc>/`) declare country-specific additions. The runtime evaluator decides which additions apply, per record, based on the record's organization.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every regulated business object — a freight charge, a sale order, a partner, an invoice — needs country-specific fields, validators, workflows, and master data on top of a country-agnostic base. A logistics charge in India needs HSN code; the same charge in the EU needs CN code + EORI; in the US it needs Schedule B. Hardcoding country fields on the base model is wrong; creating one model per country is also wrong (cross-domain pollution, no shared form). The right answer is an SDK that lets a country pack extend an existing model with additional fields, validators, and workflows, gated by the organization's country.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX entry points:** Settings → Localization → Country Packs (per-org activation toggle matrix — Phase 3). Forms across the platform auto-show/hide country-scoped fields based on the active org's pack list (no per-screen UX surface).
- **Programmatic entry points for other modules:**
  - `@api.extend_model("model.key", country="CC")` — decorator a country pack uses to inject fields into an existing model.
  - `@api.country_validator(model="key", country="CC")` — decorator for country-scoped validators (Phase 2).
  - `env.l10n.is_in_country_scope(record, "CC")` — predicate any consumer can call to check pack scope.
- **Integration boundary:** l10n produces a *country-scope predicate*. Workflow, approval, validation, view rendering, and required enforcement all read from the same predicate. The SDK never reaches into other modules — it exposes one decorator + one predicate, and consumers opt in.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
Two records in the same physical table, same Python class, different country-pack-gated behaviour. One predicate (`is_in_country_scope(record, "CC")`) drives form rendering, validation, workflow eligibility, and required enforcement.

```
[Domain layer]                              [Localization layer]
─────────────                               ────────────────────
@api.model("logistics.charge")              @api.extend_model("logistics.charge",
class Charge(DomainModel):                                       country="IN")
    amount = fields.Decimal(...)            class ChargeIN:
    currency_id = fields.Reference(...)         hsn_code = fields.Char(required=True)
    # universal fields only                     gst_treatment = fields.Enum(...)
              │                                            │
              │ field registration                        │ extension registration
              ▼                                            ▼
        ┌────────────────────────────────────────────────────┐
        │              foundation.l10n registry              │
        │  charge fields = base ∪ extensions[country=record's│
        │                                       org country] │
        └─────────────────────┬──────────────────────────────┘
                              │
        ┌─────────────────────┴───────────────────────┐
        ▼                                             ▼
    Indian org's record                       US org's record
    sees hsn_code (required)                  hsn_code is NULL
    sees gst_treatment                        no GST field on form
    Indian workflows fire                     US workflows fire
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `res.country.localization.pack` | Catalog of available country packs. Read-only — rows seeded by each pack at install. | (Phase 1) `src/ede/foundation/l10n/models/country_localization_pack.py` |
| `res.organization.localization_pack_rel` | M2M join driving per-org activation. | (Phase 1) auto-injected by Many2Many on `res.organization` |
| `ir.l10n.extension` | Registry mirror of every `@api.extend_model` registration. Read-only. | (Phase 1) `src/ede/foundation/l10n/models/extension.py` |
| `ir.l10n.validator` | Registry mirror of every `@api.country_validator` registration. Read-only. | (Phase 2) `src/ede/foundation/l10n/models/validator.py` |
| `res.country.tax_code` | Country-scoped master for HSN / Schedule B / CN code etc. One model, many countries. | (Phase 2) `src/ede/foundation/l10n/models/country_tax_code.py` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `ScopeEvaluator` | The single predicate `is_in_country_scope(record, "CC")`. Generalized in `foundation.marketplace` Phase 1 to also evaluate extension scope. | (Phase 1) `src/ede/foundation/l10n/services/scope_evaluator.py` |
| `ExtensionRegistry` | Tracks every `@api.extend_model` registration; merges extension fields into base model at boot. | (Phase 1) `src/ede/foundation/l10n/services/extension_registry.py` |
| `ViewComposer` | Composes base view + country pack `<extend ref="...">` fragments at render time. | (Phase 2) `src/ede/foundation/l10n/services/view_composer.py` |
| `CountryValidatorRegistry` | Resolves `@api.country_validator` hooks to fire only when scope predicate matches. | (Phase 2) `src/ede/foundation/l10n/services/validator_registry.py` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `res.organization.activate_localization_pack` | Admin toggles pack on for an org | Adds pack to `localization_pack_ids`; fires `l10n.pack.activated` event |
| `res.organization.deactivate_localization_pack` | Admin toggles pack off | Removes pack from `localization_pack_ids`; hides country fields on subsequent reads |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `l10n.pack.activated` | After a pack is added to `res.organization.localization_pack_ids` | Pack-specific bootstrap routines (Phase 2 packs may seed per-org master rows on first activation) |
| `l10n.pack.deactivated` | After a pack is removed from an org's M2M | Audit log |
| `l10n.extension.registered` | Boot-time, once per `@api.extend_model` registration | Registry sync (writes `ir.l10n.extension` row) |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `GET /api/l10n/packs` | List installed country packs | `L10nController.list_packs` (Phase 1) |
| `POST /api/l10n/packs/{pack_code}/activate?org_id=…` | Activate a pack for an org | `L10nController.activate` (Phase 3) |
| `POST /api/l10n/packs/{pack_code}/deactivate?org_id=…` | Deactivate a pack for an org | `L10nController.deactivate` (Phase 3) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.{model}.create` (country-scoped validators) | Country-scoped validators registered via `@api.country_validator` register as regular hooks but check `ScopeEvaluator` before raising — non-matching orgs see no validation. |
| `pre.{model}.update` (country-scoped validators) | Same pattern as create. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_ — l10n has no state machine of its own. Country packs may register country-scoped workflows that participate in `foundation.workflow`'s state machine; the engine reads `country_scope` from `ir.workflow.definition` and asks the `ScopeEvaluator` whether a given record is in scope for a given workflow before allowing transitions.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` (in `src/ede/foundation/settings.py`): add `"l10n"` after `"presentation"`. (Phase 1)
- `ACTIVE_LOCALIZATIONS` (new key, in `src/localizations/settings.py`): list of country pack codes to load, e.g. `["in"]`. (Phase 2 — introduced alongside the first pack.)
- Manifest `depends`: `foundation.l10n` depends on `["base", "presentation"]`. Each country pack depends on `["l10n", "base"]` + every domain module it extends.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `DEFAULT_LOCALIZATION_PACKS` | `list[str]` | `[]` | `DEFAULT_LOCALIZATION_PACKS` | Country pack codes activated on every newly-created `res.organization` by default. Empty list = no default activation. (Phase 1) |
| `L10N_STRICT_MISSING_PACK` | `bool` | `True` | `L10N_STRICT_MISSING_PACK` | If `True`, boot fails when a country pack listed in `ACTIVE_LOCALIZATIONS` is not importable. If `False`, missing packs are warned and skipped. (Phase 2) |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `l10n.org_country_overrides_pack` | system | bool | `False` | When `True`, an org's `country_id` change auto-syncs `localization_pack_ids` to the matching country's pack. When `False`, pack activation is fully explicit. (Phase 3) |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| Settings → Localization → Country Packs | `src/ede/foundation/l10n/views/l10n_settings.xml` | Read-only matrix of (org × installed pack) with toggle per cell. (Phase 3) |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `src/ede/foundation/l10n/data/res.country.localization.pack.csv` | Empty in Phase 1 (catalog rows are written by each pack at install). |
| `src/localizations/in/data/res.country.localization.pack.csv` | India pack catalog row. (Phase 2) |
| `src/localizations/in/data/res.country.tax_code.csv` | ~50 sample HSN rows for common freight commodities. (Phase 2) |
| `src/ede/foundation/l10n/data/ir.rbac.permission.csv` | RBAC rows: `l10n.read`, `l10n.activate_pack`, `l10n.manage_packs`. (Phase 1) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Country-Scope Predicate + Per-Org Activation + Test-Fixture Pack | 🔴 Not Started — blocked on [`foundation.base` Phase 2](../roadmap/foundation/base/phase-2/README.md) | [phase-1](../roadmap/foundation/l10n/phase-1-implementation.md) |
| Phase 2 | Workflow Gating + First Real Pack (`l10n.in`) | 🔴 Not Started | [phase-2](../roadmap/foundation/l10n/phase-2-implementation.md) |
| Phase 3 | Multi-Country Expansion + Reporting/Document Integration | 🔴 Not Started | [phase-3](../roadmap/foundation/l10n/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Hard prereq: model & view extension SDK not yet shipped in foundation.base | 🔴 | [`foundation.base` Phase 2](../roadmap/foundation/base/phase-2/README.md) |
| Country-scope predicate factory, per-org activation M2M, and test-fixture pack not built | 🔴 | [Phase 1](../roadmap/foundation/l10n/phase-1-implementation.md) |
| No country pack exists — `localizations.in` is the first pack target | 🔴 | [Phase 2](../roadmap/foundation/l10n/phase-2-implementation.md) |
| View `<extend>` element — moved to `foundation.base` Phase 2 (consumer side here is just usage) | 🔴 | [`foundation.base` Phase 2](../roadmap/foundation/base/phase-2/README.md) |
| Workflow engine has no `country_scope` field on `ir.workflow.definition` | 🔴 | [Phase 2 — workflow gating workstream](../roadmap/foundation/l10n/phase-2-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- (none yet — populate as integration learnings emerge)

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1 adds `localization_pack_ids` Many2Many on `res.organization`. Additive — no existing data touched.
- Phase 2 adds `country_scope` `Char(2)` nullable on `ir.workflow.definition`. Backfill: NULL means "all countries" (default).
- Phase 2 adds the `res.country.tax_code` table. Initial rows are seeded by each pack — no system-level seed.
- Pack uninstall after activation with data: out of v1 scope. Once an org has saved data in country-scoped fields, deactivating the pack hides the fields but does not drop the columns or the data.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Any user | `l10n.read` — list installed packs |
| Org admin | `l10n.activate_pack` — toggle a pack for their own org |
| System admin | `l10n.manage_packs` — install/uninstall packs system-wide |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [foundation.marketplace](./foundation-marketplace.md) — peer module that generalizes the l10n SDK into a third-party extension lifecycle (vendor portal, vetting, per-tenant on-demand activation). Hard-depends on l10n Phase 1.
- [foundation.workflow](./foundation-workflow.md) — workflow engine; l10n Phase 2 adds a single `country_scope` column.
- [foundation.presentation](./foundation-presentation.md) — view DSL; l10n Phase 2 adds the `<extend>` element.
- [foundation.i18n](./foundation-i18n.md) — peer concern. l10n owns country-scoped *data*; i18n owns locale-scoped *strings* and *formatting*.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-17 (restructure: SDK pieces lifted to `foundation.base` Phase 2; l10n Phase 1 narrowed to country-scope + per-org activation). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
