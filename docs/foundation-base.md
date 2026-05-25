<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Base — Implementation Docs

**Module:** `foundation.base` (`src/ede/foundation/base/`)
**Roadmap:** [roadmap/foundation/base/README.md](../roadmap/foundation/base/README.md)
**Status:** ✅ Delivered — Phase 1 ✅ Delivered (baseline, pre-roadmap); [Phase 2 — Model & View Extension SDK](../roadmap/foundation/base/phase-2/README.md) ✅ Delivered 2026-05-18
**Layer:** Foundation engine — platform substrate

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
`foundation.base` is the platform substrate of the EDE framework. It ships three layers of capability that every other module relies on:

- **`res.*` cross-domain business masters** — geography, time, units of measure, parties, currencies, organizations, users, languages. Any model in any domain may reference these directly.
- **`ir.*` platform metadata** — menus, actions, sequences, runtime config store, model registry, RBAC permissions and roles, audit trail, data references, notification opt-ins.
- **Framework health-check** — `ping` model + `ping.listener` used by smoke tests and uptime probes.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Cross-domain primitives — countries, currencies, parties, units of measure, RBAC, sequences, menus, audit — are needed by every foundation engine and every business domain in the platform. Building them once here means no consuming module re-implements them, and no domain ends up depending on another domain just to share a master.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End users** — masters surface under the Settings app (Geography, Currencies, Units, Partners, Organizations, Users, Languages, Menus, Sequences, RBAC roles, Audit). Health-check `ping` is internal-only.
- **Other modules** — reference `res.*` / `ir.*` model keys directly in `Reference` fields, `One2Many`, `Many2Many`, and via `env.models["res.partner"]` lookups. Manifest must list `foundation.base` in `depends` (in practice it is the root of the dependency graph so every app inherits the dependency).
- **Integration boundary** — produces cross-domain masters and metadata; consumes nothing — `foundation.base` is the root of the manifest dependency graph and has no `depends` of its own.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
┌──────────────────────────────────────────────────────────────┐
│  Every other foundation engine and every domain module       │
│  (foundation.auth, foundation.communication, logistics.*,    │
│   crm.*, hr.*, …)                                            │
└────────────────────────────┬─────────────────────────────────┘
                             │ depends on
                             ▼
┌──────────────────────────────────────────────────────────────┐
│  foundation.base                                             │
│  ┌──────────────────────┐  ┌──────────────────────────────┐  │
│  │ res.* business       │  │ ir.* platform metadata       │  │
│  │  masters             │  │                              │  │
│  │  country / state /   │  │  menu / action / sequence    │  │
│  │  city / timezone     │  │  config + config.log         │  │
│  │  uom / uom.category  │  │  model + model.property      │  │
│  │  partner / role /    │  │  rbac.permission / role /    │  │
│  │  address             │  │  role.binding / access.grant │  │
│  │  currency /          │  │  audit.*                     │  │
│  │  exchange.rate       │  │  data.reference /            │  │
│  │  organization / user │  │  data.cleanup.log            │  │
│  │  org.unit / language │  │  notification.setting        │  │
│  └──────────────────────┘  └──────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ Framework health-check — ping / ping.listener            ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `res.country` | ISO countries with code, numeric code, timezone link | [models/country.py](../src/ede/foundation/base/models/country.py) |
| `res.state` | State / province under country | [models/geography.py](../src/ede/foundation/base/models/geography.py) |
| `res.city` | City under country + state | [models/geography.py](../src/ede/foundation/base/models/geography.py) |
| `res.timezone` | IANA timezone with UTC offset | [models/geography.py](../src/ede/foundation/base/models/geography.py) |
| `res.uom.category` | UOM category (weight, volume, dimension, quantity, time) | [models/uom.py](../src/ede/foundation/base/models/uom.py) |
| `res.uom` | Unit of measure with conversion factor | [models/uom.py](../src/ede/foundation/base/models/uom.py) |
| `res.partner` | Universal party master (customer, vendor, carrier, contact) | [models/partner.py](../src/ede/foundation/base/models/partner.py) |
| `res.partner.role.master` | Role definitions assigned to partners | [models/partner.py](../src/ede/foundation/base/models/partner.py) |
| `res.partner.role` | Partner ↔ role join | [models/partner.py](../src/ede/foundation/base/models/partner.py) |
| `res.partner.address` | Address records per partner (billing, pickup, delivery, …) | [models/partner.py](../src/ede/foundation/base/models/partner.py) |
| `res.currency` | ISO 4217 currencies | [models/currency.py](../src/ede/foundation/base/models/currency.py) |
| `res.exchange.rate` | FX rates with rate-type vocabulary | [models/exchange_rate.py](../src/ede/foundation/base/models/exchange_rate.py) |
| `res.organization` | Legal entity / company | [models/organization.py](../src/ede/foundation/base/models/organization.py) |
| `res.user` | System users | [models/user.py](../src/ede/foundation/base/models/user.py) |
| `org.unit` | Organizational unit (branch, department) | [models/org_unit.py](../src/ede/foundation/base/models/org_unit.py) |
| `res.language` | Supported languages | [models/language.py](../src/ede/foundation/base/models/language.py) |
| `ir.menu` | Web-client menu nodes | [models/menu.py](../src/ede/foundation/base/models/menu.py) |
| `ir.action` | Bound actions (window, server, URL) | [models/action.py](../src/ede/foundation/base/models/action.py) |
| `ir.sequence` | Code/number generators (`HO-YYYY-NNNNNN` style) | [models/sequence.py](../src/ede/foundation/base/models/sequence.py) |
| `ir.config` | Runtime config store | [models/ir_config.py](../src/ede/foundation/base/models/ir_config.py) |
| `ir.config.log` | Audit log of config changes | [models/ir_config_log.py](../src/ede/foundation/base/models/ir_config_log.py) |
| `ir.model` | Reflective model registry | [models/ir_model.py](../src/ede/foundation/base/models/ir_model.py) |
| `ir.model.property` | Per-model property metadata | [models/ir_model_property.py](../src/ede/foundation/base/models/ir_model_property.py) |
| `ir.rbac.permission` | Declared permission rows per module | [models/permission.py](../src/ede/foundation/base/models/permission.py) |
| `ir.role` | RBAC role definitions | [models/role.py](../src/ede/foundation/base/models/role.py) |
| `ir.role.binding` | User ↔ role binding | [models/role_binding.py](../src/ede/foundation/base/models/role_binding.py) |
| `ir.access.grant` | Resolved access grants | [models/access_grant.py](../src/ede/foundation/base/models/access_grant.py) |
| `ir.audit.*` | Audit trail | [models/audit.py](../src/ede/foundation/base/models/audit.py) |
| `ir.data.reference` | Tracking for declaratively loaded data | [models/data_reference.py](../src/ede/foundation/base/models/data_reference.py) |
| `ir.data.cleanup.log` | Audit of orphan-data cleanup runs | [models/data_cleanup_log.py](../src/ede/foundation/base/models/data_cleanup_log.py) |
| `ir.notification.setting` | Per-user notification opt-ins | [models/notification_setting.py](../src/ede/foundation/base/models/notification_setting.py) |
| `ping` | Framework health-check model | [models/ping.py](../src/ede/foundation/base/models/ping.py) |
| `ping.listener` | Health-check listener | [models/ping_listener.py](../src/ede/foundation/base/models/ping_listener.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none — `foundation.base` ships models + metadata; services that operate on these masters live in sibling foundation engines (auth, security, communication, …)_ | | |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.create` / `ede.read_one` / `ede.update` / `ede.delete` / `ede.search` / `ede.count` / `ede.read_group` | Generic CRUD from `CrudKernel` for every `res.*` / `ir.*` master | Standard CRUD |
| `res.organization.deactivate` | Programmatic deactivation of a company | Sets `active=False` (soft-archive) |
| `res.organization.change_country` | Programmatic country reassignment | Updates `country_id` reference |
| `res.user.register` / `res.user.set_password` / `res.user.activate` / `res.user.deactivate` | User lifecycle | Creates / updates `res.user` with password hash |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.record.created` / `ede.record.updated` / `ede.record.deleted` | CRUD on any `res.*` / `ir.*` master | Audit (`ir.audit.*`), communication (chatter follow-up), domain modules tracking master changes |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| Generic CRUD under `/api/{model_key}/...` | Standard CRUD for every registered master | `RouteController` subclasses generated by the platform |
| `GET /ping` | Framework health-check | `ping` controller |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.res.organization.delete` | `Organization.restrict_delete()` blocks hard deletes — orgs must be deactivated, not removed |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none — `foundation.base` masters are reference data; no per-record state machine. Lifecycle is `active=True ↔ active=False` via the soft-archive auto-filter._
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` (in `src/ede/foundation/settings.py`): `base` — first entry; every other foundation app and every domain depends on it.
- `ACTIVE_DOMAINS`: _not applicable — foundation app_
- Manifest `depends`: _none — root of the dependency graph_
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `DEFAULT_TENANT_ID` | `str` | `"system"` | `EDE_DEFAULT_TENANT_ID` | Tenant id used when no explicit tenant is bound on the request. |
| `DEBUG` | `bool` | `False` | `EDE_DEBUG` | Exposes full tracebacks on error responses; never enable in production. |
| `ENABLE_API_DOCS` | `bool` | `False` | `EDE_ENABLE_API_DOCS` | Exposes `/docs` and `/redoc` OpenAPI surfaces. |
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
| _none yet — future enhancement_ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| [data/res.country.csv](../src/ede/foundation/base/data/res.country.csv) | ISO countries. |
| [data/res.country.timezones.xml](../src/ede/foundation/base/data/res.country.timezones.xml) | Country ↔ timezone associations. |
| [data/res.timezone.csv](../src/ede/foundation/base/data/res.timezone.csv) | IANA timezones with UTC offset. |
| [data/res.currency.csv](../src/ede/foundation/base/data/res.currency.csv) | ISO 4217 currencies. |
| [data/res.exchange.rate.type.csv](../src/ede/foundation/base/data/res.exchange.rate.type.csv) | Exchange-rate type vocabulary. |
| [data/res.uom.category.csv](../src/ede/foundation/base/data/res.uom.category.csv) | UOM categories (weight, volume, dimension, quantity, time). |
| [data/res.uom.xml](../src/ede/foundation/base/data/res.uom.xml) | Units of measure with conversion factors. |
| [data/res.language.csv](../src/ede/foundation/base/data/res.language.csv) | Supported languages. |
| [data/base_menus.xml](../src/ede/foundation/base/data/base_menus.xml) | Settings menu skeleton. |
| [data/res_partner_menus.xml](../src/ede/foundation/base/data/res_partner_menus.xml) | Partner-related menu nodes. |
| [data/customization_menus.xml](../src/ede/foundation/base/data/customization_menus.xml) | Customization sub-menu skeleton. |
| [data/base_permissions.xml](../src/ede/foundation/base/data/base_permissions.xml) | Initial RBAC permission rows for masters. |
| [data/ir.rbac.permission.csv](../src/ede/foundation/base/data/ir.rbac.permission.csv) | Bulk permission seeds. |
| [data/rbac_roles.xml](../src/ede/foundation/base/data/rbac_roles.xml) | Standard role definitions. |
| [data/rbac_seed.xml](../src/ede/foundation/base/data/rbac_seed.xml) | Default role bindings + access grants. |
| [data/base_setup.xml](../src/ede/foundation/base/data/base_setup.xml) | Bootstrap rows for system tenant. |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 — Baseline | Platform-master + `ir.*` + health-check substrate | ✅ Delivered (baseline) | [roadmap/foundation/base/README.md](../roadmap/foundation/base/README.md) |
| [Phase 2](../roadmap/foundation/base/phase-2/README.md) — Model & View Extension SDK | `@api.extend_model` decorator + view inheritance + registry merge + boot validator + `ir.model.extension` mirror + admin UI + 40 pytest cases + developer guide | ✅ Delivered 2026-05-18 | [phase-2/README.md](../roadmap/foundation/base/phase-2/README.md) |

_Additional `res.*` / `ir.*` business-master models continue to be added under consumer roadmaps when they request them. Phase 2 is the exception — it's a kernel-level capability that every consumer needs, so it belongs here._
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Geography & Time | `res.country`, `res.state`, `res.city`, `res.timezone` | [models/country.py](../src/ede/foundation/base/models/country.py), [models/geography.py](../src/ede/foundation/base/models/geography.py) | [README.md](../roadmap/foundation/base/README.md) |
| Units of Measure | `res.uom.category`, `res.uom` | [models/uom.py](../src/ede/foundation/base/models/uom.py) | [README.md](../roadmap/foundation/base/README.md) |
| Parties | `res.partner`, `res.partner.role.master`, `res.partner.role`, `res.partner.address` | [models/partner.py](../src/ede/foundation/base/models/partner.py) | [README.md](../roadmap/foundation/base/README.md) |
| Money | `res.currency`, `res.exchange.rate` | [models/currency.py](../src/ede/foundation/base/models/currency.py), [models/exchange_rate.py](../src/ede/foundation/base/models/exchange_rate.py) | [README.md](../roadmap/foundation/base/README.md) |
| Organizations & Users | `res.organization`, `res.user`, `org.unit`, `res.language` | [models/organization.py](../src/ede/foundation/base/models/organization.py), [models/user.py](../src/ede/foundation/base/models/user.py), [models/org_unit.py](../src/ede/foundation/base/models/org_unit.py), [models/language.py](../src/ede/foundation/base/models/language.py) | [README.md](../roadmap/foundation/base/README.md) |
| Menus & Actions | `ir.menu`, `ir.action` | [models/menu.py](../src/ede/foundation/base/models/menu.py), [models/action.py](../src/ede/foundation/base/models/action.py) | [README.md](../roadmap/foundation/base/README.md) |
| Sequences | `ir.sequence` | [models/sequence.py](../src/ede/foundation/base/models/sequence.py) | [README.md](../roadmap/foundation/base/README.md) |
| Config store | `ir.config`, `ir.config.log` | [models/ir_config.py](../src/ede/foundation/base/models/ir_config.py), [models/ir_config_log.py](../src/ede/foundation/base/models/ir_config_log.py) | [README.md](../roadmap/foundation/base/README.md) |
| Model registry | `ir.model`, `ir.model.property` | [models/ir_model.py](../src/ede/foundation/base/models/ir_model.py), [models/ir_model_property.py](../src/ede/foundation/base/models/ir_model_property.py) | [README.md](../roadmap/foundation/base/README.md) |
| RBAC & Roles | `ir.rbac.permission`, `ir.role`, `ir.role.binding`, `ir.access.grant` | [models/permission.py](../src/ede/foundation/base/models/permission.py), [models/role.py](../src/ede/foundation/base/models/role.py), [models/role_binding.py](../src/ede/foundation/base/models/role_binding.py), [models/access_grant.py](../src/ede/foundation/base/models/access_grant.py) | [README.md](../roadmap/foundation/base/README.md) |
| Audit | `ir.audit.*` | [models/audit.py](../src/ede/foundation/base/models/audit.py) | [README.md](../roadmap/foundation/base/README.md) |
| Data references | `ir.data.reference`, `ir.data.cleanup.log` | [models/data_reference.py](../src/ede/foundation/base/models/data_reference.py), [models/data_cleanup_log.py](../src/ede/foundation/base/models/data_cleanup_log.py) | [README.md](../roadmap/foundation/base/README.md) |
| Notifications opt-in | `ir.notification.setting` | [models/notification_setting.py](../src/ede/foundation/base/models/notification_setting.py) | [README.md](../roadmap/foundation/base/README.md) |
| Health-check | `ping`, `ping.listener` | [models/ping.py](../src/ede/foundation/base/models/ping.py), [models/ping_listener.py](../src/ede/foundation/base/models/ping_listener.py) | [README.md](../roadmap/foundation/base/README.md) |
| Phase 2 — Model & View Extension SDK | `ir.model.extension` | [kernel/extensions.py](../src/ede/core/kernel/extensions.py), [registry.py](../src/ede/core/registry.py) (`register_extension` / `_merge_extension_into_base` / `validate_extensions` / `list_extensions`), [metadata_builder.py](../src/ede/core/adapters/persistence/sqlalchemy/metadata_builder.py) (soft-FK degradation), [dsl/parser.py](../src/ede/core/services/presentation/dsl/parser.py) (`<extend>` element), [view_registry.py](../src/ede/core/services/presentation/view_registry.py) (`compose_view_xml`), [ir_model_extension.py](../src/ede/foundation/base/models/ir_model_extension.py), [docs/foundation-base-extensions.md](foundation-base-extensions.md) | [Phase 2 README](../roadmap/foundation/base/phase-2/README.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| **Field-Change Audit Log (`ir.field.change.log`)** — generic append-only audit primitive; `__ede_audit_fields__` opt-in; `<audit-trail/>` DSL element; retention sweep through `foundation.jobs`. **Hard prereq for `logistics.sales-crm` Phase 2 · 03 slice 1.** First consumer is sales-crm; every domain that needs commercial / regulatory / compliance audit gets one primitive instead of N hand-rolled tables. | 🔴 Not Started (drafted 2026-05-25) | [enhancements/01-field-change-audit.md](../roadmap/foundation/base/enhancements/01-field-change-audit.md) |
| **Tenant Base Currency + Dual-Currency Storage** — one base currency per tenant; auto-stamped `_base_currency` sibling Decimal on every opt-in monetary field. Surfaced by sales-crm Phase 2 · 03 but cross-cutting (every domain with a money field). | 🔴 Not Started — design stub (drafted 2026-05-25) | [enhancements/02-res-currency-base-currency.md](../roadmap/foundation/base/enhancements/02-res-currency-base-currency.md) |
| **`res.partner` Duplicate Merge Governance** — `res.partner.merge` command + append-only audit + `data_steward` role + preview endpoint + cross-model FK discovery via `ir.model.field` registry. Surfaced by sales-crm Phase 2 · 03 but cross-cutting (every domain with `partner_id`). | 🔴 Not Started — design stub (drafted 2026-05-25) | [enhancements/03-res-partner-merge.md](../roadmap/foundation/base/enhancements/03-res-partner-merge.md) |
| **`res.organization.unit` Model + Per-Org Multi-Branch Toggle** — replaces polymorphic `ir.org.unit` with a typed `res.organization.unit` model FK'd to `res.organization`; auto-stamps one unit on org creation; introduces `base.allow_multiple_units` `ir.config` (scope=organization, default False) surfaced in General Settings; pre-create cap + pre-set toggle-down hooks; menu restructure to `Settings → Organization & Units → Organizations / Units`; Alembic backfill from `ir.org.unit` rows. **Foundation slice for branch switching** — enables enhancements [05](../roadmap/foundation/base/enhancements/05-user-org-unit-assignment.md) (user assignment) and `foundation.security/enhancements/01` (active-unit propagation + BranchSwitcher). | 🔴 Not Started (drafted 2026-05-25) | [enhancements/04-organization-units-model.md](../roadmap/foundation/base/enhancements/04-organization-units-model.md) |
| **`res.user` Org-Unit Assignment Fields + Cascade Validators** — 4 new fields on `res.user` (`allowed_organization_ids` M2M, `default_organization_id` Reference, `allowed_unit_ids` M2M → `res.organization.unit`, `default_unit_id` Reference). 4 cascade validators on `pre.res.user.create/update` (default-org-in-allowed · unit-orgs-subset-of-allowed-orgs · default-unit-in-allowed · default-unit-org-matches-default-org). Auto-resolution: single-unit orgs auto-fill unit pickers and hide them in the admin form. Reactive admin form: changing `default_organization_id` toggles unit-picker visibility against the org's `base.allow_multiple_units` setting. `PrincipalEnricher` reads `allowed_organization_ids` directly (was: derived from role bindings). Legacy `res.user.organization_ids` kept for one release as deprecation shim. **Depends on Enhancement 04** (units model must exist first). | 🔴 Not Started (drafted 2026-05-25) | [enhancements/05-user-org-unit-assignment.md](../roadmap/foundation/base/enhancements/05-user-org-unit-assignment.md) |
| **Team Substrate (`res.team` + `res.team.type` + `ir.team.role` + `res.team.role.assignment` + `TeamRoleService`)** — ✅ Delivered 2026-05-25. Generic team primitive distinct from legal org-unit hierarchy; team-functional roles decoupled from RBAC `ir.role`; sequence-aware role assignment for escalation; shared `TeamRoleService` consumed by both `foundation.approval` and `foundation.workflow`. Adds `res.user.team_ids` M2M + `primary_team_id` Reference. New `Settings → Teams & Roles` menu group with 4 child leaves + 16 RBAC permission rows. Bundled platform fix: M2M soft-FK degradation in `metadata_builder._build_join_tables`. 20 new pytest cases · 2617 pytest + 564 vitest green · `bun run build` clean. Developer guide: [docs/foundation-base-team-substrate.md](foundation-base-team-substrate.md). **Unblocks foundation.approval Enh 01, foundation.workflow Enh 01, logistics.sales-crm Enh 09, and eventual sales-crm Phase 2 · 02 Pricing Approval.** | ✅ Delivered 2026-05-25 | [enhancements/06-team-substrate.md](../roadmap/foundation/base/enhancements/06-team-substrate.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- Putting a domain-specific master in `foundation.base` to "share later". If only one domain uses it, it belongs in that domain. The bar for `res.*` is **two or more ERP domains would reference it**.
- Duplicating an existing platform master. Before creating a model, search `foundation.base/models/` — the master almost certainly exists.
- Referencing a `res.*` master by `dbid` instead of `record_uuid`. All FKs across the platform point at `record_uuid`.
- Hard-deleting an organization. `pre.res.organization.delete` vetoes the operation — soft-archive via `res.organization.deactivate` instead.
- Declaring a permission row in `foundation.base` for a domain feature. Each module ships its own permission rows in its own `data/`; `foundation.base` only ships the engine and the platform-wide permissions.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- This module is the migration root — its initial Alembic revision is the parent of every other foundation/domain migration tree.
- `res.partner.address` was added when `logistics.masters` Phase 1 needed it — additions to `foundation.base` are driven by the first consumer that needs them, not by speculative design.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Seeded standard roles | Defined in [data/rbac_roles.xml](../src/ede/foundation/base/data/rbac_roles.xml); bound via [data/rbac_seed.xml](../src/ede/foundation/base/data/rbac_seed.xml). Permission rows for masters live in [data/ir.rbac.permission.csv](../src/ede/foundation/base/data/ir.rbac.permission.csv) and [data/base_permissions.xml](../src/ede/foundation/base/data/base_permissions.xml). |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Platform Execution Rules](../roadmap/platform/00-execution-rules.md)
- [Roadmap Tracker](../roadmap/roadmap-tracker.md)
- [Foundation Model Naming](foundation-model-naming.md)
- Every other foundation engine — `foundation.auth`, `foundation.communication`, `foundation.notifications`, `foundation.approval`, `foundation.presentation`, `foundation.security`, `foundation.workflow`, `foundation.i18n`, `foundation.l10n`, … — and every domain module depend on this one.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-25 (Enhancement 06 Team Substrate ✅ **Delivered same-day** — flipped 🔴 → ✅ across 4 sites. Ships 4 new platform models (`res.team`, `res.team.type`, `ir.team.role` decoupled from RBAC `ir.role`, `res.team.role.assignment` with sequence) + 2 new `res.user` fields (`team_ids`, `primary_team_id`) + shared `TeamRoleService.resolve/is_assigned/all_assignments_for_user` consumed by foundation.approval Enh 01 and foundation.workflow Enh 01 (siblings). New `Settings → Teams & Roles` menu group with 4 child leaves + 16 RBAC permission rows + admin views for all 4 models. Foundation ships zero seed rows for types/roles — consumers seed their own. Alembic migration `a2e7f4b3c819` (4 new tables + `res_user.primary_team_id` additive column + `res_user_res_team_rel` M2M join). **Bundled platform fix**: M2M soft-FK degradation in `metadata_builder._build_join_tables` (skip join table when either FK target absent from build batch — same contract as Reference-field soft-degrade shipped by Phase 2 SDK). 20 new pytest cases under `src/tests/foundation/test_team_substrate.py`. **2617 pytest passed + 564 vitest passed + `bun run build` clean**. Developer guide: [foundation-base-team-substrate.md](foundation-base-team-substrate.md). **Unblocks foundation.approval Enh 01, foundation.workflow Enh 01, logistics.sales-crm Enh 09, and eventual sales-crm Phase 2 · 02 Pricing Approval feature.** Prior sync earlier today: Enhancement 06 stub added to Known Gaps. Prior sync this session: Enhancement 05 `res.user` Org-Unit Assignment Fields + Cascade Validators stub added to Known Gaps — 4 new fields on `res.user` for explicit org/unit assignment + 4 cascade validators + auto-resolution for single-unit orgs + reactive admin form + `PrincipalEnricher` direct-read of `allowed_organization_ids` + one-release deprecation shim for legacy `organization_ids` M2M. Depends on Enhancement 04. Prior sync this session: Enhancement 04 added — `res.organization.unit` model + per-org `base.allow_multiple_units` toggle + auto-stamp + migration off `ir.org.unit`. Together enhancements 04 + 05 form the foundation half of the branch-switching design — `foundation.security/enhancements/01` carries the auth + runtime + BranchSwitcher half.) To refresh, invoke the `syncing-roadmap-to-docs` skill.*
