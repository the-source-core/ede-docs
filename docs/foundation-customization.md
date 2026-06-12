<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Customization (Properties + Persistent Model Registry) — Implementation Docs

**Module theme:** `foundation.customization` (theme — code lands in `foundation.base` for the new `ir.model*` and `ir.model.property.*` masters, the registry-sync service, the `<properties/>` DSL element, and the React `PropertiesEditor` widget)
**Roadmap:** [roadmap/foundation/customization/README.md](../roadmap/foundation/customization/README.md)
**Status:** ✅ Phase 1 Delivered (2026-05-10 — all 5 features shipped; 1424 pytest + 388 vitest green; browser walkthrough confirmed live by user on `pricing.rate`)
**First adopter:** `pricing.rate` opted in via `@api.model("pricing.rate", custom_properties=True)`; admin UI uses real Many2one pickers (model_id, comodel_id) against the synced `ir.model` mirror, not free-text model keys.
**Layer:** Foundation theme

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A two-mechanism customization layer for the EDE platform. (1) **`ir.model` / `ir.model.field` (read-only mirror)** — a persistent reflection of the runtime model registry, refreshed on every `ede migrate upgrade`. Code is the source of truth; the DB is the introspection mirror, addressable by stable named external IDs (`base.model_res_partner`, `base.field_res_partner__name`) registered through the existing `ir.data.reference` table. (2) **Properties (JSON column + per-tenant schema)** — an opt-in `properties` JSONB column auto-injected by `@api.model("...", custom_properties=True)`, with per-tenant schema rows in `ir.model.property.definition` declaring keys, types (Char / Integer / Decimal / Boolean / Date / Selection / Many2one / Many2many), and defaults. A new `<properties/>` DSL element renders a dynamic tab in the React FormView; an admin UI under Settings → Customizations → Custom Fields lets tenants manage definitions without code or DDL.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
EDE has no introspection surface and no customization story today. The model registry (`src/ede/core/registry.py`) is runtime-only — RBAC, audit tooling, action wiring, API discovery, and downstream integrations have nowhere to read "which models exist, with what fields, of what type." Tenants cannot extend a model with an ad-hoc field (e.g. `industry_segment` on `crm.lead`) without a Python change + Alembic migration + redeploy. ERPs in the wild need O(100) custom fields per tenant. This module is the cheapest unlock for both — JSON-only storage in Phase 1 (no runtime DDL), with optional JSONB-path search in Phase 2 and optional real-DDL custom columns deferred to Phase 3.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End user (admin)** — Settings → Customizations → Custom Fields opens the per-tenant property-definition list; admins pick a model, declare keys + types + selection options + defaults, save, and the field appears on every record's "Custom Properties" tab on the next form load.
- **End user (any signed-in user)** — opens any opt-in model's form view, sees a "Custom Properties" tab next to the standard sheet, edits values via the same widgets used for native fields (m2o picker, m2m tag input, date picker, etc.).
- **Programmatic consumer (downstream module)** — `env.ref("base.model_crm_lead")` resolves a stable handle to the `ir.model` record; the `properties` field on opt-in models is a plain dict and supports `record.get_property(key)` / `record.set_property(key, value)` for type-aware reads with reference resolution.
- **Integration boundary** — produces `ir.model*` rows on every `ede migrate upgrade` (after data-load, before orphan cleanup); produces a `properties` JSONB column on opt-in models; consumes nothing beyond the existing `ir.data.reference` external-ID infrastructure and the existing FormView DSL parser.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
┌──────────────────────────── Code (source of truth) ─────────────────────────────┐
│  @api.model("crm.lead")  →  Registry (in-memory)  →  SQLAlchemy MetaData         │
└──────────────┬─────────────────────────────────────────────┬─────────────────────┘
               │ A: sync on `ede migrate upgrade`            │ B: opt-in adds JSON col
               ▼                                             ▼
┌── ir.model ────────────┐  ┌── ir.model.field ──────┐    crm.lead.properties (JSONB)
│ model_key (uniq)       │  │ model_id  → ir.model    │
│ label, table_name      │  │ name, label, field_type │
│ app_key, abstract      │  │ comodel, required, …    │  ┌── ir.model.property.definition ──┐
│ default_order, …       │  │ state = "base" | "manual│  │ tenant_id (per-tenant DB)         │
│ custom_properties      │  └─────────────────────────┘  │ model_key, key, label             │
└────────────────────────┘                               │ property_type (char/int/.../m2m)  │
                                                         │ comodel, default_value, sequence  │
   Phase B: Action RenderPlan                            └───────────────────────────────────┘
   ▼
   FormView → <properties/> → PropertiesEditor.tsx → record.properties (read/write JSON)
```
External IDs link both layers: `RegistrySync` writes one `ir.data.reference` row per `ir.model` / `ir.model.field` / `ir.model.field.selection` (e.g. `base.model_res_partner`, `base.field_res_partner__name`). `env.ref("module.name")` resolves any of them at runtime.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.model` | Persistent mirror of every registered DomainModel | [src/ede/foundation/base/models/ir_model.py](../src/ede/foundation/base/models/ir_model.py) |
| `ir.model.field` | Persistent mirror of every registered FieldSpec | [src/ede/foundation/base/models/ir_model.py](../src/ede/foundation/base/models/ir_model.py) |
| `ir.model.field.selection` | Selection options per Enum field | [src/ede/foundation/base/models/ir_model.py](../src/ede/foundation/base/models/ir_model.py) |
| `ir.model.property.definition` | Tenant-scoped property schema (m2o `model_id` / `comodel_id` to `ir.model`) | [src/ede/foundation/base/models/ir_model_property.py](../src/ede/foundation/base/models/ir_model_property.py) |
| `ir.model.property.selection` | Selection options per Selection-type property | [src/ede/foundation/base/models/ir_model_property.py](../src/ede/foundation/base/models/ir_model_property.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `RegistrySync` | Mirrors the in-memory `Registry` into `ir.model*` rows on `ede migrate upgrade` (idempotent upsert via `ir.data.reference`) | `src/ede/foundation/base/services/registry_sync.py` (planned) |
| `PropertiesValidator` | Pre-create / pre-update hook auto-registered on opt-in models — coerces values, type-checks, verifies FK existence for reference / many2many | `src/ede/foundation/base/services/property_validator.py` (planned) |
| `Env.ref()` | Resolves `module.name` external IDs to RecordSets via `ir.data.reference` | `src/ede/core/env.py` (planned addition) |
| `RecordSet.get_property()` / `set_property()` | Type-aware property accessors (m2o → RecordSet, m2m → RecordSet, scalar → coerced) | `src/ede/core/orm/recordset.py` (planned addition) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.create` / `ede.update` (on opt-in models) | Standard CRUD writes — when payload includes `properties` | Pre-hook runs `PropertiesValidator`; values are coerced, type-checked, FKs verified |
| `ede.create` / `ede.update` / `ede.delete` (on `ir.model.property.definition`) | Admin UI under Settings → Customizations → Custom Fields | Standard CRUD; type-change forbidden after first save |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.record.created` / `ede.record.updated` / `ede.record.deleted` | Standard CRUD on `ir.model*` and `ir.model.property.*` rows | Audit, search-index reindex (future) |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| Standard CRUD | All `ir.model*` and `ir.model.property.*` access via the generic `CrudKernel` routes | (no module-specific controller) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.{model_key}.create` (auto-injected on `custom_properties=True` models) | Runs `PropertiesValidator` against the `properties` payload |
| `pre.{model_key}.update` (auto-injected on `custom_properties=True` models) | Runs `PropertiesValidator` against the `properties` payload |
| `pre.ir.model.property.definition.update` | Forbids `property_type` changes on already-saved definitions |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none — Phase 1 has no state-machine fields._
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Empty rows are fine; missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES`: no new entry — code lands inside existing `foundation.base`.
- `ACTIVE_DOMAINS`: n/a (foundation theme).
- Manifest `depends`: no change to `foundation.base`'s manifest.
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
| `src/ede/foundation/base/data/customization_rbac.csv` (planned) | 6 RBAC permissions: `ir.model.read`, `ir.model.field.read`, `ir.model.property.definition.{read,create,update,delete}` |
| `src/ede/foundation/base/data/customization_menus.xml` (planned) | Settings → Customizations → Custom Fields menu + ir.action linking to `ir.model.property.definition` list view |
<!-- /SYNC-BLOCK -->

### CLI flags
| Flag | Command | Purpose |
|---|---|---|
| `--no-registry-sync` (planned) | `ede migrate upgrade` | Skip the registry-sync phase (CI / debugging) |

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Persistent Registry + Properties + UI | ✅ Delivered (2026-05-10 — first adopter `pricing.rate`; browser walkthrough confirmed) | [roadmap](../roadmap/foundation/customization/phase-1/README.md) |
| Phase 2 | JSONB-Path Search Domains | 🔴 Not Started | [roadmap](../roadmap/foundation/customization/phase-2/README.md) |
| Phase 4 | Anchored Property Fields · DB-Backed Views · AI-Assisted Customization | 🟡 In Progress — A 🟡 substrate (2026-06-11) · B ✅ Delivered (2026-06-12, walkthrough confirmed) · C 🔴 | [roadmap](../roadmap/foundation/customization/phase-4/README.md) |
| Phase 3 | Manual Fields via Runtime DDL (deferred) | 🔴 Not Started | _no phase folder yet — deferred until customer demand_ |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| `ir.model*` Registry Sync | `ir.model`, `ir.model.field`, `ir.model.field.selection` | [ir_model.py](../src/ede/foundation/base/models/ir_model.py), [registry_sync.py](../src/ede/foundation/base/services/registry_sync.py), [env.py](../src/ede/core/env.py) (`env.ref`), [migrate.py](../src/ede/cli/commands/migrate.py) (`--no-registry-sync`) | [Phase 1 / 01](../roadmap/foundation/customization/phase-1/01-ir-model-registry-sync.md) |
| Properties storage + per-tenant schema | `ir.model.property.definition`, `ir.model.property.selection` | [ir_model_property.py](../src/ede/foundation/base/models/ir_model_property.py), [property_validator.py](../src/ede/foundation/base/services/property_validator.py), [decorators.py](../src/ede/core/kernel/decorators.py) (`custom_properties=True`), [recordset.py](../src/ede/core/orm/recordset.py) (`get_property`/`set_property`) | [Phase 1 / 02](../roadmap/foundation/customization/phase-1/02-properties-storage-and-schema.md) |
| `<DynamicProperties/>` Form-View Tab | _(no models — DSL element + frontend widget)_ | [parser.py](../src/ede/core/services/presentation/dsl/parser.py), [view.rng](../src/ede/core/services/presentation/dsl/schemas/view.rng), [presentations.py](../src/ede/foundation/presentation/models/presentations.py), [PropertiesEditor.tsx](../src/frontend/src/workspace/views/PropertiesEditor.tsx), [FormView.tsx](../src/frontend/src/workspace/views/FormView.tsx), [WorkspaceActionController.tsx](../src/frontend/src/workspace/components/action/WorkspaceActionController.tsx) (dirty filter) | [Phase 1 / 03](../roadmap/foundation/customization/phase-1/03-form-view-properties-tab.md) |
| Admin Custom Fields UI | _(uses ir.model.property.definition)_ | [ir_model_property_views.xml](../src/ede/foundation/base/views/ir_model_property_views.xml), [ir_model_views.xml](../src/ede/foundation/base/views/ir_model_views.xml), [customization_menus.xml](../src/ede/foundation/base/data/customization_menus.xml), [ir.rbac.permission.csv](../src/ede/foundation/base/data/ir.rbac.permission.csv) (11 new rows) | [Phase 1 / 04](../roadmap/foundation/customization/phase-1/04-admin-custom-fields-ui.md) |
| First adopter — `pricing.rate` | `pricing.rate` (gains `properties` JSONB column) | [rate.py](../src/domains/logistics/pricing/models/rate.py), [pricing_rate_form.xml](../src/domains/logistics/pricing/views/pricing_rate_form.xml), Alembic [c4f7a2e1d958](../src/domains/logistics/pricing/migrations/versions/c4f7a2e1d958_pricing_rate_custom_properties.py) | [Phase 1 / 05](../roadmap/foundation/customization/phase-1/05-verification.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| JSONB-path search domains for `properties.<key>` not implemented | 🔴 | [Phase 2](../roadmap/foundation/customization/phase-2/README.md) |
| Manual fields with runtime DDL (state="manual" rows) | 🔴 | Phase 3 — deferred until customer demand |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
_(populated as integration learnings emerge)_

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1 introduces five new tables (`ir_model`, `ir_model_field`, `ir_model_field_selection`, `ir_model_property_definition`, `ir_model_property_selection`) via Alembic revision `d3a9f2c4e1b8` under `src/ede/foundation/base/migrations/versions/`. `model_id` and `comodel_id` on `ir_model_property_definition` are UUID FK columns to `ir_model.record_uuid`; the unique constraint is `(model_id, key)`.
- The first `ede migrate upgrade` after deployment runs the registry-sync phase and seeds `ir.model*` from the live registry — no data backfill required.
- **First adopter:** `pricing.rate` — Alembic revision `c4f7a2e1d958` (`pricing_rate.properties` JSONB column).
- For any future model adopting `@api.model("...", custom_properties=True)`, a follow-up Alembic migration is needed to add the `properties` JSONB column on that model's table — see `c4f7a2e1d958` as the reference template.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `rbac.role_internal_user` | `ir.model.read`, `ir.model.field.read` (introspection only) |
| `rbac.role_system_admin` | `ir.model.property.definition.{read,create,update,delete}` (manage custom fields) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Approval Workflow Engine](./foundation-approval.md) — future consumer of `env.ref("base.model_*")` for approval-flow targeting
- [Foundation Workflow Engine](./foundation-workflow.md) — future consumer of `ir.model*` for stage-field discovery
- [Platform — ORM Active Filter](./platform-orm-active-filter.md) — `ir.model.property.definition.active=True` opts the model into the existing auto-filter
- [Foundation Model Naming](./foundation-model-naming.md) — `res.*` vs `ir.*` conventions
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-10 (Phase 1 flipped 🟡 → ✅ — all 5 features delivered, browser walkthrough confirmed live by user on `pricing.rate`; first adopter shipped). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
