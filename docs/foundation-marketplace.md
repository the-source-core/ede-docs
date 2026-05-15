<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Extension Marketplace — Implementation Docs

**Module:** `foundation.marketplace` (`src/ede/foundation/marketplace/`)
**Roadmap:** [roadmap/foundation/marketplace/README.md](../roadmap/foundation/marketplace/README.md)
**Status:** 🔴 Not Started (roadmap drafted 2026-05-13)
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A four-layer system that takes a third-party Python package from a vendor's laptop to a tenant's active feature: extension SDK, per-tenant lifecycle, marketplace control plane, and storefront. Architecture is **curated marketplace + metadata activation** — vendors submit packages, platform vets and deploys them to infrastructure, tenants click Activate to flip metadata and run a per-tenant migration. No code download at click-time.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
EDE is a platform. The platform thesis only works if third parties can build extensions the way in-house teams build country packs — declarative additions to existing models, gated by activation, with their own migrations, demo data, and tests. Today there is no extension SDK, no vendor portal, no per-tenant activation registry, no marketplace storefront. This module ships all four. The same model used by every enterprise SaaS marketplace.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Tenant admin (end-user UX):** Marketplace app in app-switcher (Phase 4) — browse, search, click Activate; Settings → Marketplace → Installed Extensions (Phase 2) to manage activated extensions and see migration status.
- **Vendor:** Vendor portal at `/vendor/` (Phase 3, gated by `MARKETPLACE_VENDOR_PORTAL_ENABLED`) for submission + review queue. CLI `python -m ede_marketplace_sdk package --sign` to package locally (Phase 4).
- **Reviewer (platform staff):** approval inbox shows extension submissions; multi-step approval flow (security review + product review + compatibility check) before publish.
- **Programmatic entry points for other modules:**
  - `@api.extend_model("model.key", extension_key="vendor.key")` — vendors use the same decorator pattern country packs do.
  - `env.l10n.is_in_extension_scope(record, "vendor.key")` — predicate to check if an extension is active for the record's tenant.
- **Integration boundary:** the marketplace produces an *extension-scope predicate* peer to l10n's country-scope predicate. Workflow, approval, validation, view rendering, and required enforcement all read from the same evaluator (via the shared `foundation.l10n.scope_evaluator` service).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
Code travels at deploy time. Activation is metadata + per-tenant migration only.

```
[Vendor]                          [Marketplace Control Plane]              [Platform Infra]              [Tenant]
─────────                         ──────────────────────────              ──────────────              ──────
build + sign package
    │
    │ python -m ede_marketplace_sdk
    │    package --sign && upload
    ▼
vendor portal submission   ─►   automated scan (manifest valid,
                                signature verified, deps allowed)
                                │
                                ▼
                                review board (uses foundation.approval —
                                security + product + compat reviewers)
                                │
                                ▼
                                approved → version-tagged → push to
                                extension registry → deploy pipeline
                                rolls package out to all app servers
                                                                       │
                                                                       ▼
                                                                       extension visible in marketplace catalog
                                                                                                                  │
                                                                                                                  ▼
                                                                                                                  tenant admin clicks Activate
                                                                                                                  │
                                                                                                                  ▼
                                                                                                                  gateway.tenant_extension row written
                                                                                                                  │
                                                                                                                  ▼
                                                                                                                  per-tenant migration runs
                                                                                                                  (extension's tables created in this
                                                                                                                  tenant's DB only)
                                                                                                                  │
                                                                                                                  ▼
                                                                                                                  menus / models / workflows / RBAC
                                                                                                                  become eligible for this tenant;
                                                                                                                  user reloads → AEB menu appears
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.extension` | Catalog row per published extension. State (Submitted/Vetting/Approved/Published/Deprecated). | (Phase 1) `src/ede/foundation/marketplace/models/extension.py` |
| `ir.extension.version` | One row per version of an extension. | (Phase 1) `src/ede/foundation/marketplace/models/extension_version.py` |
| `ir.extension.vendor` | Vendor master. | (Phase 1) `src/ede/foundation/marketplace/models/vendor.py` |
| `gateway.tenant_extension` | Per-tenant activation registry. | (Phase 2) `src/ede/foundation/gateway/models/tenant_extension.py` |
| `ir.extension.submission` | Vendor portal submission queue. | (Phase 3) `src/ede/foundation/marketplace/models/submission.py` |
| `ir.extension.review` | Per-reviewer record on a submission. | (Phase 3) `src/ede/foundation/marketplace/models/review.py` |
| `ir.extension.subscription` | Per-tenant billing stub. | (Phase 4) `src/ede/foundation/marketplace/models/subscription.py` |
| `ir.extension.review.rating` | Tenant-side rating of an installed extension. | (Phase 4) `src/ede/foundation/marketplace/models/rating.py` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `ScopeEvaluator` (generalized) | Shared with `foundation.l10n` — adds `is_in_extension_scope(record, extension_key)` peer to `is_in_country_scope`. | (Phase 1) `src/ede/foundation/l10n/services/scope_evaluator.py` |
| `ExtensionRegistry` | Tracks extension catalog + per-tenant activation; provides the gating predicate. | (Phase 1) `src/ede/foundation/marketplace/services/extension_registry.py` |
| `TenantExtensionRunner` | Runs per-tenant Alembic migration for one extension's version_location. | (Phase 2) `src/ede/foundation/marketplace/services/tenant_runner.py` |
| `VendorPortalController` | Vendor-side REST endpoints — submission, draft management, version history. | (Phase 3) `src/ede/foundation/marketplace/controllers/vendor_portal.py` |
| `ExtensionDeployPipeline` | Approved-extension push to ops cluster — updates `ACTIVE_EXTENSIONS`, triggers rolling restart. | (Phase 3) `src/ede/foundation/marketplace/services/deploy_pipeline.py` |
| `MarketplaceCatalogService` | REST API tenants consume to discover extensions; respects platform-version compat. | (Phase 3) `src/ede/foundation/marketplace/services/catalog.py` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ir.extension.activate_for_tenant` | Tenant admin clicks Activate | Writes `gateway.tenant_extension`; runs per-tenant migration; fires `extension.activated` event |
| `ir.extension.deactivate_for_tenant` | Tenant admin clicks Deactivate | Marks `gateway.tenant_extension` row deactivated; hides extension's surface from this tenant |
| `ir.extension.submission.submit` | Vendor uploads package | Validates package, creates `ir.extension.submission`, starts vetting workflow |
| `ir.extension.submission.approve` | Reviewer approves | Advances workflow; on full approval, triggers deploy pipeline |
| `ir.extension.submission.reject` | Reviewer rejects | Returns submission to vendor with notes |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `extension.submission.created` | Vendor submits | Notify reviewer queue |
| `extension.approved` | Final approver approves | Trigger deploy pipeline; notify vendor |
| `extension.published` | Deploy pipeline completes | Notify all tenants subscribed to new-extension alerts |
| `extension.activated` | Tenant activates an extension | Cache invalidation; tenant audit log; vendor analytics |
| `extension.deactivated` | Tenant deactivates | Cache invalidation; tenant audit log |
| `extension.migration.failed` | Per-tenant migration fails | Notify tenant admin + platform ops; mark `gateway.tenant_extension.last_error` |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `GET /api/marketplace/catalog` | Browse available extensions | `MarketplaceController.catalog` (Phase 4) |
| `GET /api/marketplace/extension/{key}` | Extension detail | `MarketplaceController.detail` (Phase 4) |
| `POST /api/marketplace/extension/{key}/activate` | Activate for current tenant | `MarketplaceController.activate` (Phase 2) |
| `POST /api/marketplace/extension/{key}/deactivate` | Deactivate for current tenant | `MarketplaceController.deactivate` (Phase 2) |
| `GET /api/marketplace/installed` | List installed extensions for current tenant | `MarketplaceController.installed` (Phase 2) |
| `POST /vendor/submissions` | Submit a packaged extension | `VendorPortalController.submit` (Phase 3) |
| `GET /vendor/submissions` | List my submissions | `VendorPortalController.list_submissions` (Phase 3) |
| `POST /vendor/submissions/{id}/version` | Upload a new version | `VendorPortalController.add_version` (Phase 3) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.ir.extension.activate_for_tenant` | Validates platform-version compat; refuses activation if compat-range doesn't include this cluster's version |
| `post.ir.extension.activate_for_tenant` | Triggers per-tenant migration; if migration fails, automatically reverses the activation |
| `pre.gateway.tenant.delete` | Refuses tenant delete if extensions are active (must deactivate first) |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
`ir.extension.state` is workflow-managed:
```
Submitted ──► SecurityReview ──► ProductReview ──► Approved ──► Published ──► Deprecated
     │              │                  │
     └──► Rejected ◄┴──────────────────┘   (any review can reject; vendor can re-submit)
```
Backed by `workflow_extension_lifecycle.xml` consumed by `foundation.workflow`. Approval rules wired via `foundation.approval`.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` (in `src/ede/foundation/settings.py`): add `"marketplace"` after `"l10n"`. (Phase 1)
- `ACTIVE_EXTENSIONS` (new key, in `src/extensions/settings.py`): list of extension keys to load, e.g. `["aeb.customs"]`. Populated by the deploy pipeline; ops-managed. (Phase 1)
- Manifest `depends`: `foundation.marketplace` depends on `["base", "l10n", "approval", "workflow", "gateway"]`. Each extension depends on `["marketplace", ...whatever domain modules it extends...]`.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `MARKETPLACE_CATALOG_URL` | `str` | `""` | `MARKETPLACE_CATALOG_URL` | URL of the marketplace catalog service. Empty = self-hosted in this cluster's control plane (Phase 3). |
| `MARKETPLACE_PLATFORM_VERSION` | `str` | `"1.0.0"` | `MARKETPLACE_PLATFORM_VERSION` | The platform version this deployment advertises. Extensions whose `compatible_with_platform_version_range` doesn't satisfy this are hidden in the storefront. (Phase 1) |
| `MARKETPLACE_VENDOR_PORTAL_ENABLED` | `bool` | `False` | `MARKETPLACE_VENDOR_PORTAL_ENABLED` | When `True`, vendor portal routes mount under `/vendor/`. Off by default — only the central marketplace cluster needs this. (Phase 3) |
| `MARKETPLACE_EXTENSION_PACKAGE_SIGNING_KEY` | `str` | `""` | `MARKETPLACE_EXTENSION_PACKAGE_SIGNING_KEY` | Public key the marketplace uses to verify extension package signatures. (Phase 3) |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `marketplace.tenant_self_service_enabled` | system | bool | `True` | When `True`, tenant admins can activate extensions themselves. When `False`, only platform admin can activate (managed-services mode). (Phase 2) |
| `marketplace.auto_deactivate_on_uninstall` | system | bool | `False` | When `True`, an extension globally deprecated by ops auto-deactivates on all tenants. (Phase 3) |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| Settings → Marketplace → Installed Extensions | `src/ede/foundation/marketplace/views/marketplace_settings.xml` | Per-tenant: list of active extensions with deactivate buttons, version info, migration status. (Phase 2) |
| Settings → Marketplace → Vendor Portal | `src/ede/foundation/marketplace/views/vendor_portal_settings.xml` | Off-tenant: vendor management, submission queue, security review board. Only visible when `MARKETPLACE_VENDOR_PORTAL_ENABLED=True`. (Phase 3) |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `src/ede/foundation/marketplace/data/ir.rbac.permission.csv` | RBAC: `marketplace.read`, `marketplace.activate_extension`, `marketplace.manage_extensions`, `vendor_portal.submit_extension`, `vendor_portal.review_extension` |
| `src/ede/foundation/marketplace/data/marketplace_menus.xml` | Marketplace app (Phase 4) + Settings → Marketplace → Installed Extensions (Phase 2) + Vendor Portal (Phase 3) |
| `src/ede/foundation/marketplace/data/workflow_extension_lifecycle.xml` | Workflow definition: Submitted → SecurityReview → ProductReview → Approved → Published → Deprecated (Phase 3) |
| `src/ede/foundation/marketplace/data/approval_policy_extension_review.xml` | Approval policy: security reviewer + product reviewer required for Submitted → Approved (Phase 3) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Extension SDK + Activation Substrate | 🔴 Not Started | [phase-1](../roadmap/foundation/marketplace/phase-1-implementation.md) |
| Phase 2 | Per-Tenant Activation + Lifecycle UI | 🔴 Not Started | [phase-2](../roadmap/foundation/marketplace/phase-2-implementation.md) |
| Phase 3 | Marketplace Control Plane (off-tenant) | 🔴 Not Started | [phase-3](../roadmap/foundation/marketplace/phase-3-implementation.md) |
| Phase 4 | Storefront Frontend + First Partner Onboarding | 🔴 Not Started | [phase-4](../roadmap/foundation/marketplace/phase-4-implementation.md) |
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
| Extension SDK substrate not built — `ir.extension*` catalog absent | 🔴 | [Phase 1](../roadmap/foundation/marketplace/phase-1-implementation.md) |
| Per-tenant activation registry absent — `gateway.tenant_extension` not modelled | 🔴 | [Phase 2](../roadmap/foundation/marketplace/phase-2-implementation.md) |
| Per-tenant migration runner scoping to a single extension not implemented | 🔴 | [Phase 2](../roadmap/foundation/marketplace/phase-2-implementation.md) |
| Vendor portal not built — no submission UI, no vetting workflow | 🔴 | [Phase 3](../roadmap/foundation/marketplace/phase-3-implementation.md) |
| Marketplace storefront not built — no Marketplace app in app-switcher | 🔴 | [Phase 4](../roadmap/foundation/marketplace/phase-4-implementation.md) |
| Vendor SDK package not published — no `ede-marketplace-sdk` on PyPI | 🔴 | [Phase 4](../roadmap/foundation/marketplace/phase-4-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- (none yet — populate as integration learnings emerge)
- Anticipated: confusing "code activation" (ops, deploys the package) with "tenant activation" (in-product, flips a flag and runs migration). These are two distinct steps; the second is impossible without the first.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1 adds `ir.extension`, `ir.extension.version`, `ir.extension.vendor` tables. Additive.
- Phase 2 adds `gateway.tenant_extension` table. Additive.
- Phase 2's per-tenant migration is the critical safety surface: schema changes for activated extensions are applied to one tenant's DB at a time. Refuse to run ALTER on huge tables without explicit confirmation.
- Phase 3 adds `ir.extension.submission` + `ir.extension.review` tables. Additive.
- Phase 4 adds `ir.extension.subscription` + `ir.extension.review.rating` tables. Additive.
- **Deprecation policy**: an extension marked Deprecated remains usable by tenants that already activated it. Activation by new tenants is blocked. Force-deactivate only with `marketplace.auto_deactivate_on_uninstall=True`.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Any user | `marketplace.read` — browse marketplace catalog |
| Tenant admin | `marketplace.activate_extension` — activate / deactivate for own tenant |
| Platform admin | `marketplace.manage_extensions` — install / uninstall packages cluster-wide |
| Vendor user | `vendor_portal.submit_extension` — submit new versions in vendor portal |
| Reviewer | `vendor_portal.review_extension` — approve / reject submissions |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [foundation.l10n](./foundation-l10n.md) — peer module providing the scope predicate this module generalizes. **Hard prerequisite** (Phase 1 of l10n).
- [foundation.approval](./foundation-approval.md) — engine the vetting workflow runs on.
- [foundation.workflow](./foundation-workflow.md) — engine the extension lifecycle runs on.
- [foundation.gateway](./foundation-gateway.md) — tenant lifecycle; `gateway.tenant_extension` lands beside `gateway.tenant`.
- [foundation.jobs](./foundation-jobs.md) — Phase 2's per-tenant migration runner uses this when it lands; otherwise inline.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-13. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
