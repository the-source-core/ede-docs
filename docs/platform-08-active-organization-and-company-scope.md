<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Platform — Active-Organization Context + `company_scope` ORM Opt-In

**Module:** `ede.core.env` + `ede.core.kernel` + `ede.core.services.persistence` + `ede.foundation.base.services` (cross-cutting)
**Roadmap:** [roadmap/platform/08-active-organization-and-company-scope.md](../roadmap/platform/08-active-organization-and-company-scope.md)
**Status:** 🔴 Not Started — landing in lockstep with [`foundation.security`](foundation-security.md) Phases 1 + 2.
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state* — currently "nothing shipped". Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A kernel-level cross-cutting design that adds three things to the framework:
1. **`Env.active_organization_id` slot + `with_active_organization_id` clone method** in [src/ede/core/env.py](../src/ede/core/env.py).
2. **`@api.model(..., company_scope="strict|optional|multi"|None)` decorator surface** in [src/ede/core/kernel/decorators.py](../src/ede/core/kernel/decorators.py) that triggers auto-injection of `organization_id` (strict / optional) or `organization_ids` Many2Many (multi) at class-init time.
3. **`apply_company_filter(domain, env, model_cls)` helper** at `src/ede/core/services/persistence/company_filter.py`, structurally identical to `apply_active_filter`, wired into every ORM read callsite that already runs `apply_active_filter`.

Plus three foundation-side companions in `foundation.base`:
4. **`PrincipalEnricher` extension** with 5 new `$principal.*` keys (`active_organization_id`, `allowed_organization_ids`, split `org_ids` / `branch_ids` / `department_ids`).
5. **`register_company_scope_hooks(registry)`** registrar wired from `bootstrap_environment()` next to `register_authorization_hooks`.
6. **Audit-log column additions** on `ir.rbac.decision.log` + `ir.rbac.binding.change.log` for forensic queries on cross-org access.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Today's platform has a hard tenant boundary but no concept of an active organization within a tenant. The frontend has an `OrganizationSwitcher` that updates a URL slug only — the JWT, the principal, and every search domain are unchanged. The RBAC engine's `$principal.org_unit_ids` is an aggregated list of all orgs the user has bindings in, not the one they have currently selected.

Cross-org data leakage protection within a single tenant relies on every domain author remembering to hand-write `[["organization_id", "in", "$principal.org_unit_ids"]]` on every permission row. Miss it once and the model leaks. Branch-RBAC features in domain roadmaps (sales-crm Phase 3) cannot be implemented coherently because they assume a platform primitive that doesn't yet exist.

This doc pins the kernel changes that close the loop. The work is delivered as `foundation.security` Phases 1–4, but the platform-level rationale lives here (alongside `platform/02-compute-field-runtime.md` and `platform/05-engine-substrate.md`).
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
**Domain author.** One decorator argument:
```python
@api.model("crm.opportunity", company_scope="strict")
class Opportunity(DomainModel):
    ...
```
After delivery, this single line auto-injects `organization_id`, adds the read filter, stamps creates, and rejects writes whose target org is outside the user's allowed list.

**Platform / system code.** `env.with_active_organization_id(org_id)` for stamping into a specific org; `env.with_company_test(False)` to bypass the auto-filter; `env.sudo().with_active_organization_id(None)` for truly org-blind operations.

**Security admin.** Settings → Security → Access Decision Log with the "Cross-Org Write Attempts" saved search surfaces every cross-boundary access (allowed or denied).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
See [docs/foundation-security.md § Architecture](foundation-security.md#architecture-at-a-glance) for the end-to-end flow diagram. This platform doc pins the **kernel boundary** of that flow:

```
src/ede/core/
├── env.py                          ← +1 slot (active_organization_id), +1 clone (with_active_organization_id), +1 flag slot (_company_test), +1 clone (with_company_test)
├── kernel/decorators.py            ← +1 kwarg on @api.model(...)
├── kernel/model.py                 ← __init_subclass__ block to auto-inject organization_id / organization_ids
├── services/persistence/
│   └── company_filter.py (NEW)     ← apply_company_filter + augment_with_company_filter + domain_mentions_company_field
└── orm/recordset.py                ← +1 callsite (right after apply_active_filter) at lines 270 + 306
src/ede/foundation/base/
├── services/
│   ├── principal_enricher.py       ← +5 keys returned; cache key extended with active_organization_id
│   ├── authorization_service.py    ← _resolve_value implicitly supports new keys; _write_decision_log threads org context through
│   └── company_scope_registry.py (NEW)  ← register_company_scope_hooks (stamping + write guard)
└── models/audit.py                 ← +2 columns on ir.rbac.decision.log, +1 on ir.rbac.binding.change.log
src/ede/foundation/auth/
├── models/session.py               ← +1 column (active_organization_id)
├── services/session_service.py     ← create_session(..., active_organization_id=); switch_active_organization(...)
└── api/switch_organization.py (NEW) ← POST /api/auth/switch-organization
src/ede/core/services/auth/
└── jwt_service.py                  ← encode_access_token(..., active_organization_id=)
src/ede/core/adapters/http/fastapi/
└── auth_middleware.py              ← lift active_organization_id onto principal; defense-in-depth re-validation
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _no new models_ — extends `ir.session`, `ir.rbac.decision.log`, `ir.rbac.binding.change.log`, `ir.model` | | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File (planned) |
|---|---|---|
| `Env.with_active_organization_id` + slot | Per-request active-org context | `src/ede/core/env.py` |
| `Env.with_company_test(bool)` + slot | Toggle company-scope auto-filter per env clone | `src/ede/core/env.py` |
| `@api.model(..., company_scope=)` | Decorator surface | `src/ede/core/kernel/decorators.py` |
| Class-init auto-injection of `organization_id` / `organization_ids` | Mirrors `__ede_has_active__` computation | `src/ede/core/kernel/model.py` |
| `apply_company_filter` | Auto-filter on every read | `src/ede/core/services/persistence/company_filter.py` |
| `register_company_scope_hooks` | Boot-time stamping + write-guard registrar | `src/ede/foundation/base/services/company_scope_registry.py` |
| `PrincipalEnricher.load_principal` (extended) | 5 new `$principal.*` keys | `src/ede/foundation/base/services/principal_enricher.py` |
| `_write_decision_log` (extended) | Stamp `active_organization_id` + `target_organization_id` | `src/ede/foundation/base/services/authorization_service.py` |
| `JwtService.encode_access_token` (extended) | `active_organization_id` claim | `src/ede/core/services/auth/jwt_service.py` |
| `SessionService.create_session` + `switch_active_organization` | Session row persistence + token rotation | `src/ede/foundation/auth/services/session_service.py` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ — work flows through existing CRUD + new hooks | | |
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
| `POST /api/auth/switch-organization` | Validate target ∈ `res.user.organization_ids`; update `ir.session.active_organization_id`; mint fresh access token | `src/ede/foundation/auth/api/switch_organization.py` |
| `GET /api/auth/me` (extended) | Adds `active_organization_id` to response | `src/ede/foundation/auth/api/me.py` |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.{model_key}.create` | Auto-registered for every model with `__ede_company_scope__`; stamps `organization_id` from `env.active_organization_id` when missing; strict mode rejects null. |
| `pre.{model_key}.update` | Auto-registered for same models; rejects writes whose target org is outside `$principal.allowed_organization_ids`; gates `organization_id` reassignment behind `res.organization.reassign` permission. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_ — pure auto-filter + hook + audit-column work; no new state machines.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Mirrors `roadmap/foundation/security/README.md` Section "Configuration Introduced". This platform doc adds no new config of its own — it pins the kernel decisions; the foundation module ships the settings.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES`: no new entry — code lands inside existing `foundation.base` + `foundation.auth` + `src/ede/core/`.
- `ACTIVE_DOMAINS`: n/a.
- Manifest `depends`: no change.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `SECURITY_REQUIRE_ACTIVE_ORG` | Boolean | `True` | `EDE_SECURITY_REQUIRE_ACTIVE_ORG` | Fail closed when active-org absent on a `company_scope` ∈ {strict, optional} model. |
| `SECURITY_DECISION_LOG_RETAIN_DAYS` | Integer | `365` | `EDE_SECURITY_DECISION_LOG_RETAIN_DAYS` | Retention horizon for `ir.rbac.decision.log` rows. |
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
| `src/ede/foundation/base/data/security_permissions.csv` | `res.organization.reassign` + `ir.rbac.{decision,binding.change}.log.read` permissions (system_admin only). |
| `src/ede/foundation/base/data/security_menus.xml` | Settings → Security parent menu + audit-log child menus + 2 `ir.action.window`. |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Wave | Lands in | Status |
|---|---|---|
| W1 — Active-org context + JWT/session/principal/middleware | `foundation.security` Phase 1 | 🔴 Not Started |
| W2 — `company_scope` decorator + auto-filter + stamping | `foundation.security` Phase 2 | 🔴 Not Started |
| W3 — Write guard + reassign permission | `foundation.security` Phase 3 | 🔴 Not Started |
| W4 — Decision-log enrichment + forensic admin views | `foundation.security` Phase 4 | 🔴 Not Started |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Key Files | Roadmap Source |
|---|---|---|
| _none yet_ | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Kernel has no `Env.active_organization_id` slot | 🔴 | [phase-1](../roadmap/foundation/security/phase-1/README.md) |
| No `@api.model(company_scope=)` opt-in on the decorator | 🔴 | [phase-2](../roadmap/foundation/security/phase-2/README.md) |
| `_BYPASS_MODELS` / `_EXEMPT_MODELS` frozensets duplicated across two services | 🔴 | [phase-2](../roadmap/foundation/security/phase-2/README.md) — deduplication ships with the work |
| Decision-log rows carry no org context | 🔴 | [phase-4](../roadmap/foundation/security/phase-4/README.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
*(populated post-delivery)*

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- W1 ships `ir.session.active_organization_id` migration; existing rows backfill to NULL.
- W2 ships `ir.model.company_scope` migration; populated by `RegistrySync` on every `ede migrate upgrade`. Consumer-model migrations (adding `organization_id` / `organization_ids` to opt-in models) are authored per-model by domain teams.
- W4 ships `ir.rbac.decision.log.{active,target}_organization_id` + `ir.rbac.binding.change.log.scope_organization_id`; existing rows keep NULL; optional one-time backfill SQL for binding-change-log.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `rbac.role_system_admin` | `res.organization.reassign`, `ir.rbac.decision.log.read`, `ir.rbac.binding.change.log.read` |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [foundation-security.md](foundation-security.md) — module-level companion; this platform doc is the kernel rationale; that one is the consumer-side surface.
- [platform-02-compute-field-runtime.md](platform-02-compute-field-runtime.md) — precedent for a platform-doc-pins-kernel-design pattern.
- [docs/18-permissions.md](18-permissions.md) — ABAC reference; updated by W1 with 5 new `$principal.*` keys.
- [docs/foundation-archivable-models.md](foundation-archivable-models.md) — architectural twin (`active=True` auto-filter).
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-13. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
