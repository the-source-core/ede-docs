<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Security & Authorization — Implementation Docs

**Module:** `foundation.security` (theme — code lands across `src/ede/core/`, `src/ede/foundation/base/`, `src/ede/foundation/auth/`)
**Roadmap:** [roadmap/foundation/security/](../roadmap/foundation/security/README.md)
**Status:** ✅ Delivered 2026-05-13 — all five phases shipped same-day.
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — which today is "nothing shipped". The planned scope below mirrors the four-phase roadmap. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A three-layer active-organization security stack that turns multi-company from a hand-crafted convention into a declarative platform contract:

1. **Active-org propagation (Phase 1).** The user's chosen organization flows from the React webclient to the backend as a JWT claim, is persisted on `ir.session`, surfaces on the runtime principal, and is exposed on a new `env.active_organization_id` slot. New ABAC variables (`$principal.active_organization_id`, `$principal.allowed_organization_ids`, plus split `$principal.org_ids` / `$principal.branch_ids` / `$principal.department_ids`) become available to existing `ir.rbac.permission.domain` filters.
2. **`company_scope` ORM opt-in (Phase 2).** Models declare `@api.model("...", company_scope="strict|optional|multi")`. The kernel auto-injects the right ownership field (`organization_id` required / nullable / `organization_ids` M2M), an auto-filter on every read scoped to `env.active_organization_id`, a create-time default stamp, and a `with_company_test(False)` env switch for admin bypass.
3. **Allowed-org write guard + reassign permission (Phase 3).** A universal pre-create / pre-update hook rejects mutations whose target `organization_id` is not in `$principal.allowed_organization_ids`; mutating `organization_id` post-create requires the new `res.organization.reassign` permission.
4. **Decision-log enrichment + forensic surface (Phase 4).** Adds `active_organization_id` / `target_organization_id` columns to `ir.rbac.decision.log` and `scope_organization_id` to `ir.rbac.binding.change.log`, plus admin views for "cross-org write attempts" and "denials by active-org × resource × action" pivots.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Today's RBAC engine in `foundation.base` enforces *who* can do *what*, but the framework has no concept of the user's currently active organization on the backend:
- The frontend `OrganizationSwitcher` writes to a URL slug only — the JWT, the principal, and every search domain are unchanged. A user with two organizations sees the same records whichever one they "switch" to.
- The JWT carries `tenant_id` (the deployment boundary) but no active-organization claim.
- The principal exposes `org_unit_ids` as an aggregated list of all orgs the user is bound to via role-bindings, not the *one* org they have currently selected.
- Cross-organization data-leakage protection within a single tenant relies on every domain author remembering to hand-write `[["organization_id", "in", "$principal.org_unit_ids"]]` on every permission row. Miss it once and the model leaks.
- Branch-RBAC features in domain roadmaps (sales-crm Phase 3) cannot be implemented coherently because they assume a platform primitive that doesn't yet exist.

This module is the platform-level fix. Phases 1–3 are tightly coupled (each builds on the previous); Phase 4 can ship in parallel once the Phase 1 schema lands.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
**End user (web client).** Phase 1 makes the existing `OrganizationSwitcher` real: clicking an organization in the dropdown calls `POST /api/auth/switch-organization`, which validates the target against `res.user.organization_ids`, updates the persisted session row, and returns a fresh access token. After Phase 2 ships, switching the active org instantly narrows every list view, kanban, and search panel to records owned by that org (for `company_scope="strict"` models), or to records owned by that org plus globally-shared records (for `optional`), or to records shared with that org (for `multi`).

**Domain author.** One decorator argument:
```python
@api.model("crm.opportunity", company_scope="strict")
class Opportunity(DomainModel):
    ...
```
After Phase 2 ships, this single line:
- Auto-injects `organization_id = fields.Reference("res.organization", required=True, on_delete="restrict", index=True)` if not declared.
- Adds an implicit `[["organization_id", "=", env.active_organization_id]]` filter to every `search` / `count` / `read_group` / relational read.
- Stamps `organization_id` with `env.active_organization_id` on every create where the caller didn't pass it.
- Rejects writes whose `organization_id` is outside `$principal.allowed_organization_ids` (Phase 3).

**Programmatic / admin.** `env.with_company_test(False)` bypasses the auto-filter (for cross-org reports, migrations, system jobs). `env.sudo()` bypasses RBAC entirely as today. `env.with_active_organization_id(org_id)` clones an env to act as if the named org were active — useful for backfill scripts that should stamp records into a specific company.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
React webclient                  /api/auth/switch-organization
   OrganizationSwitcher ──────────────────► AuthController
        │                                       │
        │ POST {organization_id}                 ▼
        │                                  SessionService
        │                                  - validates against res.user.organization_ids
        │                                  - updates ir.session.active_organization_id
        │                                  - mints fresh JWT with active_organization_id claim
        │                                       │
        │ ◄─── { access_token, active_organization_id }
        ▼

Every subsequent request
   Bearer <JWT>
        │
        ▼
   AuthMiddleware._resolve_principal           Env (per-request)
   - decode JWT                           ──►  - tenant_id
   - load principal                            - principal{ user_id, active_organization_id, ... }
                                               - active_organization_id
                                               - active_test, company_test flags
        │
        ▼
   CommandBus.dispatch
        │
        ▼
   Pre-hooks (universal):
   1. RBAC AuthorizationService.require()
        - $principal.* resolver pulls active_organization_id, allowed_organization_ids,
          org_ids, branch_ids, department_ids from PrincipalEnricher
        - ABAC domain filter evaluated against the record
   2. Company-scope create-default + write-guard
        - if create: stamp organization_id from env.active_organization_id when missing
        - if update: reject if target org not in $principal.allowed_organization_ids
        - if changing organization_id: require res.organization.reassign permission
        │
        ▼
   Handler (model command)
        │
        ▼
   Post-hooks (decision-log writer)
        - ir.rbac.decision.log row gains active_organization_id + target_organization_id
```

The architecture deliberately mirrors today's `active=False` soft-archive auto-filter:
- `__ede_has_active__` flag ↔ `__ede_company_scope__` flag
- `apply_active_filter(domain, env, model_cls)` ↔ `apply_company_filter(domain, env, model_cls)`
- `env.with_active_test(False)` ↔ `env.with_company_test(False)`
- `permission_registry.register_authorization_hooks(registry)` ↔ `register_company_scope_hooks(registry)` (separate registrar)
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _no new models_ — Phase 1 adds one column on `ir.session`; Phase 4 adds two columns on `ir.rbac.decision.log` + one column on `ir.rbac.binding.change.log` | | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File (planned) |
|---|---|---|
| `JwtService.encode_access_token(..., active_organization_id=)` | Bake the active-org claim into the JWT (Phase 1). | `src/ede/core/services/auth/jwt_service.py` |
| `SessionService.create_session(..., active_organization_id=)` + `switch_active_organization(...)` | Persist the chosen org on `ir.session`, mint a new access token (Phase 1). | `src/ede/foundation/auth/services/session_service.py` |
| `AuthMiddleware._resolve_principal` | Lift the new claim onto `request.state.principal["active_organization_id"]` (Phase 1). | `src/ede/core/adapters/http/fastapi/auth_middleware.py` |
| `Env.with_active_organization_id(org_id)` + `env.active_organization_id` | Shallow-clone Env with active org set (Phase 1). | `src/ede/core/env.py` |
| `PrincipalEnricher.load_principal` (extended) | Resolve 5 new `$principal.*` keys: `active_organization_id`, `allowed_organization_ids`, `org_ids` (split), `branch_ids` (split), `department_ids` (split). Keep `org_unit_ids` as union for back-compat (Phase 1). | `src/ede/foundation/base/services/principal_enricher.py` |
| `AuthorizationService._resolve_value` (extended) | New `$principal.*` keys resolve in domain filters (Phase 1). | `src/ede/foundation/base/services/authorization_service.py` |
| `apply_company_filter(domain, env, model_cls)` | Mirrors `apply_active_filter`; injects the right scope filter per `company_scope` mode (Phase 2). | `src/ede/core/services/persistence/company_filter.py` |
| `register_company_scope_hooks(registry)` | Boot-time registrar that walks every model with `__ede_company_scope__` set and registers pre-create / pre-update hooks. Mirrors `register_authorization_hooks` (Phase 2). | `src/ede/foundation/base/services/company_scope_registry.py` |
| `Env.with_company_test(bool)` | Toggle company-scope auto-filter per env clone (Phase 2). | `src/ede/core/env.py` |
| Write-guard hook (universal pre-create / pre-update) | Reject writes whose target `organization_id` is not in `$principal.allowed_organization_ids`; gate reassignment behind `res.organization.reassign`; emits `PermissionDeniedError` with `reason ∈ {not_in_allowed_organizations, reassign_permission_required, organization_inactive}` (Phase 3). | `src/ede/foundation/base/services/company_scope_registry.py` |
| Decision-log writer (extended) | Stamp `active_organization_id` + `target_organization_id` on every ALLOW/DENY row (Phase 4). | `src/ede/foundation/base/services/authorization_service.py` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _no new commands_ — every operation is plumbed through existing CRUD + the existing AuthorizationService check pipeline | | |
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
| `POST /api/auth/switch-organization` | Validate target is in `res.user.organization_ids`; update `ir.session.active_organization_id`; mint fresh access token with new claim; return token + active-org payload. (Phase 1) | `src/ede/foundation/auth/api/switch_organization.py` |
| `GET /api/auth/me` (extended) | Response gains `active_organization_id` field so the frontend doesn't decode the JWT. (Phase 1) | `src/ede/foundation/auth/api/me.py` |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.{model_key}.create` | Universal — auto-registered by `register_company_scope_hooks` on every model with `__ede_company_scope__` set. Stamps `organization_id` (or `organization_ids[]`) from `env.active_organization_id` when caller did not pass it. Strict mode rejects null. (Phase 2) |
| `pre.{model_key}.update` | Universal — same registrar. Rejects writes whose target `organization_id` is not in `$principal.allowed_organization_ids`; rejects mutation of `organization_id` post-create unless principal has `res.organization.reassign` permission. (Phase 3) |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_ — no new state machines. The existing RBAC decision flow (ALLOW / DENY in `AuthorizationService`) is unchanged; this module enriches the inputs, the auto-filter, the write guard, and the audit row.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES`: no new entry — code lands inside existing `foundation.base` + `foundation.auth` + `src/ede/core/`.
- `ACTIVE_DOMAINS`: n/a (foundation theme).
- Manifest `depends`: no change to `foundation.base` or `foundation.auth` manifests.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `SECURITY_REQUIRE_ACTIVE_ORG` | Boolean | `True` | `EDE_SECURITY_REQUIRE_ACTIVE_ORG` | When True, authenticated requests without an `active_organization_id` claim fail closed on any model whose `company_scope` is `strict` or `optional`. Phase 1 introduces the flag; Phase 2 consumes it. |
| `SECURITY_DECISION_LOG_RETAIN_DAYS` | Integer | `365` | `EDE_SECURITY_DECISION_LOG_RETAIN_DAYS` | Retention horizon for `ir.rbac.decision.log` rows. Phase 4 introduces the flag and a corresponding pruning job (deferred to `foundation.jobs` Phase 1). |
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
| `src/ede/foundation/base/data/security_permissions.csv` (Phase 3) | 1 RBAC permission: `res.organization.reassign` (system_admin only) — required to mutate `organization_id` on a `company_scope="strict"` model post-create. |
| `src/ede/foundation/base/data/security_menus.xml` (Phase 4) | Settings → Security parent menu + Access Decision Log child menu + ir.action.window targeting `ir.rbac.decision.log`. |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Active-Organization Propagation | ✅ Delivered 2026-05-13 | [phase-1/](../roadmap/foundation/security/phase-1/README.md) |
| Phase 2 | `company_scope` Decorator + ORM Auto-Filter + Default Stamping | ✅ Delivered 2026-05-13 | [phase-2/](../roadmap/foundation/security/phase-2/README.md) |
| Phase 3 | Allowed-Org Write Guard + Reassignment Permission | ✅ Delivered 2026-05-13 | [phase-3/](../roadmap/foundation/security/phase-3/README.md) |
| Phase 4 | Decision-Log Enrichment + Forensic Admin Surface | ✅ Delivered 2026-05-13 | [phase-4/](../roadmap/foundation/security/phase-4/README.md) |
| Phase 5 | Record-Rule Engine (`ir.rbac.record.rule`) | ✅ Delivered 2026-05-13 | [phase-5/](../roadmap/foundation/security/phase-5/README.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| JWT `active_organization_id` claim | — | [`src/ede/core/services/auth/jwt_service.py`](../src/ede/core/services/auth/jwt_service.py) | [phase-1/01](../roadmap/foundation/security/phase-1/01-active-org-claim-and-session.md) |
| `ir.session.active_organization_id` column + Alembic migration | `ir.session` | [`session.py`](../src/ede/foundation/auth/models/session.py), [migration `c91d2e7a4b85`](../src/ede/foundation/auth/migrations/versions/c91d2e7a4b85_session_active_organization_id.py) | [phase-1/01](../roadmap/foundation/security/phase-1/01-active-org-claim-and-session.md) |
| `SessionService.switch_active_organization` + login org-validation | — | [`session_service.py`](../src/ede/foundation/auth/services/session_service.py), [`auth.py`](../src/ede/foundation/auth/api/auth.py) | [phase-1/01](../roadmap/foundation/security/phase-1/01-active-org-claim-and-session.md) |
| `AuthMiddleware` claim surface + `X-Active-Organization-Id` header override | — | [`auth_middleware.py`](../src/ede/core/adapters/http/fastapi/auth_middleware.py) | [phase-1/01](../roadmap/foundation/security/phase-1/01-active-org-claim-and-session.md) |
| `Env.active_organization_id` slot + `with_active_organization_id` clone | — | [`env.py`](../src/ede/core/env.py), [`handler.py`](../src/ede/core/adapters/http/fastapi/handler.py) | [phase-1/02](../roadmap/foundation/security/phase-1/02-env-active-organization-context.md) |
| `PrincipalEnricher` split scopes + active-org + allowed-orgs | — | [`principal_enricher.py`](../src/ede/foundation/base/services/principal_enricher.py) | [phase-1/03](../roadmap/foundation/security/phase-1/03-principal-abac-variables.md) |
| `POST /api/auth/switch-organization` + `/me` extension + frontend wiring | — | [`switch_organization.py`](../src/ede/foundation/auth/api/switch_organization.py), [`me.py`](../src/ede/foundation/auth/api/me.py), [`OrganizationSwitcher.tsx`](../src/frontend/src/workspace/components/header/OrganizationSwitcher.tsx), [`WorkspaceClient.tsx`](../src/frontend/src/workspace/components/WorkspaceClient.tsx), [`AuthService.ts`](../src/frontend/src/services/auth/AuthService.ts) | [phase-1/04](../roadmap/foundation/security/phase-1/04-switch-organization-endpoint.md) |
| `@api.model(..., company_scope="strict\|optional\|multi")` decorator kwarg + post-`__init_subclass__` auto-injection of `organization_id` (strict/optional) or `organization_ids` M2M (multi) + mode-conflict validation at decoration time | — | [`decorators.py`](../src/ede/core/kernel/decorators.py) | [phase-2/01](../roadmap/foundation/security/phase-2/01-company-scope-decorator-and-field-injection.md) |
| `apply_company_filter` next to `apply_active_filter` + 8 ORM read-callsite wirings (ModelProxy search/count/read_group · RecordSet O2M/M2M · kernel CRUD handlers) | — | [`domain_filter.py`](../src/ede/core/services/persistence/domain_filter.py), [`model_proxy.py`](../src/ede/core/orm/model_proxy.py), [`recordset.py`](../src/ede/core/orm/recordset.py), [`model.py`](../src/ede/core/kernel/model.py) | [phase-2/02](../roadmap/foundation/security/phase-2/02-company-filter-search-and-relational-injection.md) |
| `Env.with_company_test(bool)` clone + propagation through every existing clone method | — | [`env.py`](../src/ede/core/env.py) | [phase-2/02](../roadmap/foundation/security/phase-2/02-company-filter-search-and-relational-injection.md) |
| `register_company_scope_hooks` pre-create stamping registrar + bootstrap wire-up (strict-mode `organization_id_required` rejection; multi-mode `RelationalCommand.link()` shape) | — | [`company_scope_hooks.py`](../src/ede/foundation/base/services/company_scope_hooks.py), [`bootstrap.py`](../src/ede/core/bootstrap.py) | [phase-2/03](../roadmap/foundation/security/phase-2/03-create-time-default-stamping.md) |
| `ir.model.company_scope` registry column + `RegistrySync` extension + Alembic | `ir.model` | [`ir_model.py`](../src/ede/foundation/base/models/ir_model.py), [`registry_sync.py`](../src/ede/foundation/base/services/registry_sync.py), [migration `b7e2a8f3c5d1`](../src/ede/foundation/base/migrations/versions/b7e2a8f3c5d1_ir_model_company_scope.py) | [phase-2/01](../roadmap/foundation/security/phase-2/01-company-scope-decorator-and-field-injection.md) |
| `res.partner` opted into `company_scope="optional"` (proof-of-life — nullable `organization_id` + index + FK) | `res.partner` | [`partner.py`](../src/ede/foundation/base/models/partner.py), [migration `c2a5b9d4e8f3`](../src/ede/foundation/base/migrations/versions/c2a5b9d4e8f3_res_partner_organization_id.py) | [phase-2/04](../roadmap/foundation/security/phase-2/04-verification.md) |
| Developer guide: `docs/foundation-company-scope.md` (decision tree across 3 modes + migration recipe for opting an existing model into `strict`) | — | [`foundation-company-scope.md`](foundation-company-scope.md) | [phase-2/04](../roadmap/foundation/security/phase-2/04-verification.md) |
| Phase 3 — Allowed-org write guard (pre-create + pre-update hooks reject writes whose `organization_id` ∉ `$principal.allowed_organization_ids`; multi-mode validates every LINK target) | — | [`company_scope_hooks.py`](../src/ede/foundation/base/services/company_scope_hooks.py) | [phase-3/01](../roadmap/foundation/security/phase-3/01-allowed-orgs-write-guard.md) |
| Phase 3 — `res.organization.reassign` permission seed + strict-mode post-create reassign gate (`reason="reassign_permission_required"`) | `ir.rbac.permission` (seed row) | [`company_scope_hooks.py`](../src/ede/foundation/base/services/company_scope_hooks.py), [`ir.rbac.permission.csv`](../src/ede/foundation/base/data/ir.rbac.permission.csv) | [phase-3/02](../roadmap/foundation/security/phase-3/02-organization-reassign-permission.md) |
| Phase 4 — `ir.rbac.decision.log.{active,target}_organization_id` columns + indexes + `AuthorizationService._write_decision_log` extension threading record context | `ir.rbac.decision.log` | [`audit.py`](../src/ede/foundation/base/models/audit.py), [`authorization_service.py`](../src/ede/foundation/base/services/authorization_service.py), [migration `d5e9a1c3b7f2`](../src/ede/foundation/base/migrations/versions/d5e9a1c3b7f2_phase4_decision_log_org_columns.py) | [phase-4/01](../roadmap/foundation/security/phase-4/01-decision-log-enrichment.md) |
| Phase 4 — `ir.rbac.binding.change.log.scope_organization_id` column + writer extension (populated when `scope_type='ORG'`) | `ir.rbac.binding.change.log` | [`audit.py`](../src/ede/foundation/base/models/audit.py), [`role_binding.py`](../src/ede/foundation/base/models/role_binding.py) | [phase-4/02](../roadmap/foundation/security/phase-4/02-binding-change-log-enrichment.md) |
| Phase 4 — admin list/form/search views (`Cross-Org Attempts` saved search; groupby active-org / resource / action) + `SECURITY_DECISION_LOG_RETAIN_DAYS` setting (default 365; pruner deferred to `foundation.jobs` Phase 1) | — | [`ir_rbac_audit_views.xml`](../src/ede/foundation/base/views/ir_rbac_audit_views.xml), [`settings.py`](../src/ede/foundation/settings.py) | [phase-4/03](../roadmap/foundation/security/phase-4/03-forensic-admin-views.md) |
| Phase 5 — `ir.rbac.record.rule` model + Alembic + auto M2M join to `ir.rbac.role` + 3 indexes (model_id / active / composite model+active+read) | `ir.rbac.record.rule` | [`record_rule.py`](../src/ede/foundation/base/models/record_rule.py), [migration `f4c8a1e9b6d7`](../src/ede/foundation/base/migrations/versions/f4c8a1e9b6d7_phase5_ir_rbac_record_rule.py) | [phase-5/01](../roadmap/foundation/security/phase-5/01-record-rule-model.md) |
| Phase 5 — `RecordRuleEngine` composes `GLOBAL_1 AND ... AND ((ROLE_A_OR_BLOCK) OR (ROLE_B_OR_BLOCK))` per `(user_id, model_key, action)` with `$principal.*` resolution + per-request env cache + system_admin/sudo/platform-models bypass + fail-closed `[["record_uuid","=",None]]` sentinel | — | [`record_rule_engine.py`](../src/ede/foundation/base/services/record_rule_engine.py) | [phase-5/02](../roadmap/foundation/security/phase-5/02-rule-engine-and-filter.md) |
| Phase 5 — `apply_record_rules_filter` wired into all 8 ORM read callsites + `AuthorizationService._enforce_record_rules` per-record gate (`reason="record_rule_violation"`) for read_one/create/update/delete | — | [`domain_filter.py`](../src/ede/core/services/persistence/domain_filter.py), [`model_proxy.py`](../src/ede/core/orm/model_proxy.py), [`recordset.py`](../src/ede/core/orm/recordset.py), [`kernel/model.py`](../src/ede/core/kernel/model.py), [`authorization_service.py`](../src/ede/foundation/base/services/authorization_service.py) | [phase-5/03](../roadmap/foundation/security/phase-5/03-wire-callsites-and-per-record-gate.md) |
| Phase 5 — Settings → Security → Roles & Permissions → Record Rules admin UI (list+form+search views with "Gates Read/Create/Update/Delete" filters + model_id groupby) + 4 RBAC seed permissions + ir.action + menu | — | [`ir_rbac_record_rule_views.xml`](../src/ede/foundation/base/views/ir_rbac_record_rule_views.xml), [`base_menus.xml`](../src/ede/foundation/base/data/base_menus.xml), [`ir.rbac.permission.csv`](../src/ede/foundation/base/data/ir.rbac.permission.csv) | [phase-5/04](../roadmap/foundation/security/phase-5/04-admin-ui.md) |
| Phase 5 — Developer guide: `docs/foundation-record-rules.md` (mental model + composition algorithm + UI walkthrough + worked example + bypass cookbook + perf characteristics + debugging cookbook) | — | [`foundation-record-rules.md`](foundation-record-rules.md) | [phase-5/05](../roadmap/foundation/security/phase-5/05-verification.md) |
| Enhancement 01 — Active-Unit Propagation + BranchSwitcher — **Retired 2026-05-26 by foundation.base Enh 07 Slice 2**. The unit axis is gone: branches are now child organizations in the recursive `res.organization` tree. Every artifact in this row was deleted in lockstep (`ir.session.active_unit_id` column + JWT claim + `Env.active_unit_id` slot + `with_active_unit_id`/`with_unit_test` clone methods + `POST /api/auth/switch-unit` + `SessionService.switch_active_unit` + `_assert_unit_allowed` + `_allowed_unit_ids` + `_resolve_initial_unit_for_login` + `@api.model(unit_scope=...)` + `apply_unit_filter` + `_UNIT_SCOPE_BYPASS_MODELS` + `unit_scope_hooks` + `register_unit_scope_hooks` + `BranchSwitcher.tsx` + URL `?u=` reconciliation + `WorkspaceContext` unit slots + `authService.switchUnit` + `SECURITY_REQUIRE_ACTIVE_UNIT` + `res.partner.unit_id` + `res_user.default_unit_id` + `res_user_allowed_unit_rel` M2M + 12 vitest + 3 E2E + 17 unit_scope pytest cases). `active_organization_id` infrastructure (Phase 1 / 2 / 3 / 4 / 5) stays — only the per-unit parallel axis is gone. | ✅ Delivered 2026-05-25 (retired 2026-05-26 by foundation.base Enh 07 Slice 2) | [enhancements/01-active-unit-propagation.md](../roadmap/foundation/security/enhancements/01-active-unit-propagation.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| _none yet_ | | |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
*(populated as integration learnings emerge after Phase 1 ships)*

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- **Phase 1** adds `active_organization_id` to `ir.session` (nullable Char(36)). Existing rows backfill to `NULL`; clients re-login or the next refresh-token rotation populates the column.
- **Phase 2** is opt-in per model. No global migration for consumer models. Each model that opts into `company_scope="strict"` ships its own Alembic revision that adds the `organization_id` column with `NOT NULL` and includes a per-tenant backfill recipe (default to the creator's default org or a tenant-wide fallback). `optional` and `multi` modes are additive — no NOT NULL constraint. Phase 2 itself ships one schema delta on the platform side: `ir.model.company_scope` (`Char(20)`, nullable) added to the customization-Phase-1 mirror so admin tooling can introspect which models are scoped; populated by `RegistrySync` on every `ede migrate upgrade`.
- **Phase 3** adds one seed row to `ir.rbac.permission` for `res.organization.reassign`. Idempotent — runs through the standard data-loader path.
- **Phase 4** adds `active_organization_id` + `target_organization_id` to `ir.rbac.decision.log` and `scope_organization_id` to `ir.rbac.binding.change.log`. All nullable; existing rows keep `NULL`. New rows from Phase 1 onward fill in.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `rbac.role_system_admin` | `res.organization.reassign` (Phase 3) — required to mutate `organization_id` on a `company_scope="strict"` model after create. |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Approval Workflow Engine](foundation-approval.md) — consumes principal + ABAC; unaffected by this module beyond gaining org-context in decision logs.
- [Customization (Properties + Persistent Model Registry)](foundation-customization.md) — the `ir.model` mirror this module's RBAC seed validates against (no stale `resource` values for non-existent models).
- [Internationalization (i18n)](foundation-i18n.md) — `res.user.language_id` lives next to `res.user.organization_ids`; both are per-user identity fields with no overlap.
- [Notifications Engine](foundation-notifications.md) — switch-organization may emit a `web.client.reload` event (deferred decision; currently only `res.organization.code` triggers reload).
- [`platform/08-active-organization-and-company-scope.md`](platform-08-active-organization-and-company-scope.md) — cross-cutting kernel + ORM doc; one stop for "what's the platform contract here?"
- [`docs/18-permissions.md`](18-permissions.md) — ABAC domain-filter reference; Phase 1 extends the `$principal.*` variable list.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-26 (Cross-reference sync — foundation.base Enhancement 07 (Organization Hierarchy) ✅ **Delivered (both slices)** retired Enhancement 01's full unit surface in lockstep. **Built Capabilities row** for Enh 01 updated to ✅ Delivered (retired 2026-05-26 by foundation.base Enh 07 Slice 2) with the full retirement inventory. **Known Gaps row** cleared (the planned retirement was the only entry; it's now done). All `active_organization_id` infrastructure (Phase 1 / 2 / 3 / 4 / 5) stays — only the per-unit parallel axis is gone. **No retroactive status change** on the Enh 01 entry itself — it shipped on 2026-05-25 and was retired 2026-05-26 by a superseding design (foundation.base Enh 07 Slice 2), not by a defect or rollback. Prior sync: 2026-05-26 (Cross-reference sync — foundation.base Enhancement 07 (Organization Hierarchy) drafted at 🔴 Not Started; Slice 2 of that enhancement will retire Enh 01's surface (BranchSwitcher.tsx + `ir.session.active_unit_id` column + JWT claim + `Env.active_unit_id` slot + `with_active_unit_id` clone + `$principal.active_unit_id` / `allowed_unit_ids` / `default_unit_id` enrichment + `POST /api/auth/switch-unit` controller + `SessionService.switch_active_unit` + `_assert_unit_allowed` + `_allowed_unit_ids` + `_resolve_initial_unit_for_login` + `@api.model(unit_scope=...)` decorator + auto-injected `unit_id` field + `apply_unit_filter` + `_UNIT_SCOPE_BYPASS_MODELS` + `unit_scope_hooks.py` + `register_unit_scope_hooks` boot registration + 17 `test_unit_scope.py` cases + 12 `BranchSwitcher.test.tsx` vitest cases + 3 E2E `test_branch_switcher.py` cases) because branches become child orgs in the recursive `res.organization` tree, making the per-unit parallel axis unnecessary. `active_organization_id` infrastructure (Phase 1-5) stays — only the per-unit mirror retires. **Known Gaps row added** flagging the planned retirement at 🟢 Low backlog severity (depends on foundation.base Enh 07 reaching Slice 2). **No status flips** on Enh 01 — it remains ✅ Delivered until Slice 2 actually executes the removal; Slice 1 (Shadow Mode) preserves the existing surface fully functional. Implementation work awaits HARD GATE approval per the roadmap-driven-delivery skill before any `src/` edits.) (previous: 2026-05-25 (Enhancement 01 ✅ **Delivered same-day — all 3 slices A + B + C landed end-to-end**. **Slice 1** (backend, commit `5ec0d13`): `ir.session.active_unit_id` Char(36) + JWT `active_unit_id` claim + `Env.active_unit_id` + `with_active_unit_id()` / `with_unit_test()` clones propagated through every existing clone method + `SessionService.switch_active_unit` + `SessionService._resolve_initial_unit_for_login` + `switch_active_organization` cascade-reset of `active_unit_id` to default-or-first-allowed unit in new org + `POST /api/auth/switch-unit` controller (validates allowed-list + org-match; accepts null as the All Units sentinel; reject codes `unit_not_in_allowed_units` / `unit_org_mismatch` / `unit_inactive`) + `GET /api/auth/me` extended with `active_unit_id` / `allowed_unit_ids` / `allowed_units_for_active_org` + AuthMiddleware lifts the JWT claim onto `request.state.principal` + `handler.py` threads it onto the request Env + PrincipalEnricher sets `$principal.active_unit_id` (with fallback to `res.user.default_unit_id`) + `$principal.allowed_unit_ids` + `$principal.default_unit_id` + cache key extended + `SECURITY_REQUIRE_ACTIVE_UNIT` FoundationSettings (default True; mirrors `SECURITY_REQUIRE_ACTIVE_ORG`) + Alembic `d4f9a23e7b16` (ir_session.active_unit_id Char(36) + idx via batch_alter_table, SQLite-compatible; no FK to res_organization_unit because the auth app doesn't depend on foundation.base schema — application-layer validation in `_assert_unit_allowed`). **Slice 2** (ORM unit_scope, commit `57d3b52`): `@api.model(..., unit_scope="strict"|"optional")` decorator kwarg + auto-injects `unit_id` Reference field at class init (mirrors the company_scope injection pattern) + `__ede_unit_scope__` + `__ede_has_unit_scope__` class metadata + `apply_unit_filter` parallel to `apply_company_filter` in `src/ede/core/services/persistence/domain_filter.py` (`_UNIT_SCOPE_BYPASS_MODELS` + `domain_mentions_unit` per-call opt-out + `augment_with_unit_filter` strict/optional modes with fail-closed semantics when `active_unit_id` is None) wired into all 8 ORM read callsites (search/search_count/read_group in model_proxy + kernel CRUD + _get_one2many/_get_many2many in recordset) composing AND with `apply_company_filter` + `register_unit_scope_hooks` for pre-create stamping from `env.active_unit_id` (system principal bypass + caller-supplied wins; strict-mode raises `ValueError("unit_id_required")` when slot still empty) + `env.with_unit_test(False)` admin bypass clone + `res.partner` first-adopter (`unit_scope="optional"` alongside the existing `company_scope="optional"`) + Alembic `e5a8c4d2f793` (res_partner.unit_id Char(36) + FK to res_organization_unit restrict + idx) + 17 pytest cases (TestDecoratorInjection × 2 + TestAugmentWithUnitFilter × 9 + TestPartnerFirstAdopter × 5 + TestComposesWithCompanyScope × 1 with 4-row matrix proof). **Slice 3** (frontend): NEW `BranchSwitcher.tsx` header chip placed before the notification bell per UX spec — hidden entirely when `currentOrganization.allow_multiple_units !== true` (single-branch tenants pay no UX cost), static label when exactly one allowed unit, dropdown chip with chevron when multiple + WorkspaceHeader.tsx integration + WorkspaceContext.tsx extended with `currentUnit` / `allowedUnitsForActiveOrg` / `switchUnit()` closure + URL `?u=<unit-code>` query-param wiring with **option (b) URL-driven persistence** (URL is the truth-of-the-moment; URL-set unit param fires `POST /api/auth/switch-unit` to stamp the session so the chip + ORM scope persist; the SET op short-circuits when current URL `u=` matches; invalid `?u=` silently drops the param + reverts) + types extended (`OrganizationInfo.allow_multiple_units` + new `UnitInfo` + `WorkspaceBootstrap` adds `active_unit_id` / `allowed_unit_ids` / `allowed_units_for_active_org`) + `authService.switchUnit()` returning `{accessToken, activeOrganizationId, activeUnitId}` (accepts null for All Units sentinel) + 10 new vitest cases (`src/__tests__/BranchSwitcher.test.tsx` — switchUnit happy path / null sentinel / 403 unit_not_in_allowed_units / 403 unit_org_mismatch / no-session error + navigateToUnit URL helper × 5) + 1 existing AuthService.switchOrganization vitest case updated to expect the new `activeUnitId` field in the cascade-stamped response. **Verification**: full repo suite **2830 pytest + 574 vitest green** end-to-end; zero regressions. `cd src/frontend && bun run build` exit 0 (tsc + vite); `bun run test` 574/574 passed. **Status sites updated in lockstep across 4 sites**: enhancement file 01 header (🔴 → ✅ Delivered), foundation/security README Enhancements row, roadmap-tracker Last-refreshed block (pending in next edit), docs/foundation-security.md Built Capabilities row added with full file inventory + Known Gaps cleared. **Three downstream consumers unblocked** by the end-to-end branch-switching design: this is the third and last enhancement in the chain; multi-branch tenants can now be shipped end-to-end. Prior sync: Enhancement 01 — Active-Unit Propagation + BranchSwitcher stub added to Known Gaps Mirrors Phase 1's active-org slice for the unit layer now that the typed `res.organization.unit` model is on the foundation.base roadmap. Backend: `ir.session.active_unit_id` column + JWT claim + `Env.active_unit_id` + `with_active_unit_id` clone + `$principal.active_unit_id` + `POST /api/auth/switch-unit` + `switch-organization` cascade-reset + `@api.model(unit_scope=...)` decorator + `apply_unit_filter` parallel to `apply_company_filter` + create-time stamping + admin bypass + `SECURITY_REQUIRE_ACTIVE_UNIT` flag + `res.partner` first-adopter proof. Frontend: `BranchSwitcher.tsx` rendered before the notification bell (hidden for orgs with `base.allow_multiple_units=False`, static label for single allowed unit, dropdown for multiple). Hard prereqs: foundation.base Enhancements 04 + 05. No status flips on the existing 5 delivered phases. Prior sync: 2026-05-13 — Phase 5 Record-Rule Engine ✅ Delivered same-day with Phases 1-4.) To refresh, invoke the `syncing-roadmap-to-docs` skill.*
