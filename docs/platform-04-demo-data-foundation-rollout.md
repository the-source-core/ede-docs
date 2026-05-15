<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Demo Data Rollout — Active Foundation Modules — Implementation Docs

**Module:** `ede.foundation.base` + `auth` + `approval` + `communication` + `customization` + `i18n` + `notifications` (demo channel only)
**Roadmap:** [roadmap/platform/04-demo-data-foundation-rollout.md](../roadmap/platform/04-demo-data-foundation-rollout.md)
**Status:** 🔴 Not Started
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
One-off retrofit that authors `demo/*.xml` files for the delivered foundation modules so `ede migrate upgrade -t <tenant> --with-demo=all` produces a coherent demo tenant matching the [unifying scenario](../docs/demo-usecase/README.md) (Acme Forwarding Ltd. — Mumbai HQ). The platform mechanism that loads these files already shipped in [Platform Change 03](./platform-03-demo-data-loader.md); this change is pure data authoring on top of it.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Platform 03 shipped the loader and the per-module specs in `docs/demo-usecase/`, but no actual demo XMLs exist yet — every spec row in the index is 🔴 Not yet authored. Without this rollout `--with-demo=all` produces empty tables and the freshly-shipped catalogue is aspirational. Logistics modules can't ship their demo data either, because their files cross-reference foundation demo IDs declared here.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Platform operator (CLI):** `ede migrate upgrade -t <tenant> --with-demo=foundation.base` — or `--with-demo=all` to include every retrofitted foundation app at once. Re-running updates rather than duplicates.
- **App author / reviewer:** demo records are tagged `is_demo=True` in `ir.data.reference` and visible alongside production seeds.
- **End-user demo / sanity walkthrough:** log in as one of the four demo users with password `demo` (admin / sales rep / ops manager / finance approver) and the unifying scenario records are already populated.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
Pure data authoring — no new platform code. Each app gains:

```
<app>/__manifest__.py        +  "demo": ["demo/<file>.xml", ...]
<app>/demo/<file>.xml        +  records using the same DSL as data/*.xml
```

The shipped `DataLoader.load_demo_for_apps` (Platform 03) iterates `registry.sorted_app_specs()`, runs the demo pass strictly after the production data pass, shares `_local_refs` across both passes, and writes `is_demo=True` rows to `ir.data.reference`. `OrphanCleanup` filters those rows so flag-less migrates leave them alone.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none new_ | All affected models already shipped. Demo records are instances, not definitions. | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none new_ | Existing `DataLoader.load_demo_for_apps` does all the work. | [src/ede/core/services/data_loader/loader.py](../src/ede/core/services/data_loader/loader.py) |
<!-- /SYNC-BLOCK -->

### Commands / Events / REST / Hooks / State machine
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none new_ | | (Demo records emit the standard `ede.record.created` events via the loader's normal CRUD path.) |
<!-- /SYNC-BLOCK -->

<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none new_ | |
<!-- /SYNC-BLOCK -->

<!-- SYNC-BLOCK: state-machine -->
_none_
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> No runtime configuration. Demo data is fully covered by the existing `--with-demo` CLI flag.

### Activation
<!-- SYNC-BLOCK: activation -->
- _None new_ — the `--with-demo` flag already shipped in [Platform Change 03](./platform-03-demo-data-loader.md).
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
| `src/ede/foundation/base/demo/demo_org.xml` | 1 `res.organization` (Acme Forwarding — Mumbai) |
| `src/ede/foundation/base/demo/demo_users.xml` | 4 `res.user` rows + role bindings; one `language_id` pref |
| `src/ede/foundation/base/demo/demo_partners.xml` | 3 `res.partner` rows + addresses |
| `src/ede/foundation/base/demo/demo_language_extras.xml` | 5 additional `res.language` rows |
| `src/ede/foundation/customization/demo/demo_property_definitions.xml` | 1 `ir.model.property.definition` |
| `src/ede/foundation/approval/demo/demo_approval_rules.xml` | 1 rule + 1 flow (2 steps) |
| `src/ede/foundation/communication/demo/demo_followers.xml` | Followers + 1 partner-scoped chatter note |
| `src/ede/foundation/notifications/demo/demo_inbox.xml` | 4 read welcome `ir.notification.outbox` rows |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Single delivery | Author demo XMLs for 6 foundation modules; flip per-module usecase docs + index + foundation doc mirrors | 🔴 Not Started | [roadmap/platform/04-demo-data-foundation-rollout.md](../roadmap/platform/04-demo-data-foundation-rollout.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap.

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| `foundation.base` demo (org / users / partners) — not yet authored. | 🔴 Not Started | [roadmap/platform/04-demo-data-foundation-rollout.md](../roadmap/platform/04-demo-data-foundation-rollout.md) |
| `foundation.base` extra-language demo records — not yet authored. | 🔴 Not Started | (same) |
| `foundation.customization` demo property definition — not yet authored. | 🔴 Not Started | (same) |
| `foundation.approval` demo rule + flow — not yet authored. | 🔴 Not Started | (same) |
| `foundation.communication` demo followers — not yet authored. | 🔴 Not Started | (same) |
| `foundation.notifications` demo welcome inbox — not yet authored. | 🔴 Not Started | (same) |
| Logistics demo files (`logistics.masters`, `logistics.pricing`, `logistics.sales-crm`) — gated by this rollout because they cross-reference foundation demo IDs. | 🟠 Blocked | [docs/demo-usecase/](../docs/demo-usecase/) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _Section reserved — will be populated once the rollout lands and integration learnings emerge._

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- No schema changes; no Alembic revisions. This is pure data authoring on top of the Platform 03 loader.
- Existing tenants are unaffected unless `--with-demo` is explicitly passed.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ | (Demo users bind to existing production role definitions seeded in `data/rbac_roles.xml`.) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Platform Change 03 — Demo Data Loader](platform-03-demo-data-loader.md) — ships the mechanism this change consumes.
- [docs/demo-usecase/](../docs/demo-usecase/) — per-module spec catalogue, status flips alongside this rollout.
- `preparing-demo-data` skill — gate every future ✅ flip must pass.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-12. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
