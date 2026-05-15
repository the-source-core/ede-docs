<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# ORM Soft-Archive Auto-Filter — Implementation Docs

**Module:** `ede.core.orm` + `ede.core.services.persistence` + `foundation.presentation`
**Roadmap:** [roadmap/platform/01-orm-active-filter.md](../roadmap/platform/01-orm-active-filter.md)
**Status:** ✅ Delivered (2026-05-08)
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

> **Companion:** [Archivable Models — Developer Guide](foundation-archivable-models.md) covers the *how-to* (declaring `active`, opt-outs, frontend behaviour) for application developers. This doc covers *what shipped* and *technical anatomy*.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A framework-wide soft-archive convention for any DomainModel that opts in by declaring a Boolean `active` field. The ORM then automatically hides `active=False` rows from `search`, `count`, `read_group`, and relational reads (`One2Many` / `Many2Many`) — and the React webclient auto-renders an "Archived" toggle in the search panel for those models. Opt-in per model; not auto-injected. By-UUID lookups (`browse`, `read`, `exists`) stay unfiltered so callers asking for a specific UUID always get the data.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Many EDE models declare `active` as a soft-archive flag (`res.country`, `res.uom`, `res.partner`, `res.uom.category`, `res.currency`, `res.partner.address`, `res.partner.role.master`, several logistics rate models). Before this change, every search/read site had to remember to include `("active", "=", True)` in its domain. Archived rows leaked into list views, dropdowns, relational reads, and reports whenever a callsite forgot. Centralising the filter at the ORM level eliminates the bleed and makes "archived" mean "invisible by default" framework-wide.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX:** open any list view of an archivable model (e.g. `res.country`); the search bar's filter dropdown shows an **"Archived"** chip alongside other filters. Default state hides archived rows. Toggle on → archived rows appear (alongside active ones).
- **Application developer (declaring archivable):** add `active = fields.Boolean(default=True)` to the model — the framework detects it and activates the auto-filter for that model.
- **Application developer (reading archived data in code):** either include an `active` leaf in the search domain (`("active", "in", [True, False])` for both, `("active", "=", False)` for archived only), or wrap the call in `with env.with_active_test(False):` for cross-cutting bypasses (admin tools, exports, bulk maintenance).
- **Frontend integrator:** `SearchViewDef.has_active?: boolean` is exposed on the load-action payload; the search panel reads it and renders the toggle when `true`.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
Model class declaration
   active = fields.Boolean(default=True)
            │
            ▼
DomainModel.__init_subclass__   ──►   cls.__ede_has_active__ = True
                                            │
                                            ▼
ModelProxy.search / count / read_group
RecordSet._get_one2many / _get_many2many       ──►   apply_active_filter(env, model_cls, domain)
Kernel CRUD _ede_handle_search / count / read_group       │
                                                          ▼
                                    domain_filter.augment_with_active_filter
                                       (skip if has_active=False, active_test=False, or domain mentions "active")
                                                          │
                                                          ▼
                                    repo.search(model_key, domain=[...active=True..., *caller_domain])
```
Frontend sister flow:
```
presentations.load_action  ──► views.search.has_active = cls.__ede_has_active__
                                            │
                                            ▼
                          ControlPanelSearchView synthesises an "Archived"
                          IrFilter when has_active === true; user toggles it
                          ──► domain `[["active","in",[true,false]]]` injected,
                              backend's per-call opt-out activates.
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none_ — pure ORM behaviour change, no new persistence models | | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `DomainModel.__init_subclass__` | Sets `cls.__ede_has_active__` flag at class creation when an `active` Boolean field is declared. | [src/ede/core/kernel/model.py](../src/ede/core/kernel/model.py) |
| `domain_filter.domain_mentions_active` | Per-call opt-out detection; True if any flat-list leaf names `active`. | [src/ede/core/services/persistence/domain_filter.py](../src/ede/core/services/persistence/domain_filter.py) |
| `domain_filter.augment_with_active_filter` | Pure injector — prepends `["active","=",True]` unless caller opted out. | same file |
| `domain_filter.apply_active_filter` | Convenience wrapper used at every read callsite; reads `__ede_has_active__` + `env._active_test` and delegates. | same file |
| `Env.with_active_test(bool)` | Clones env with the auto-filter toggled. Mirrored shape of `with_tenant_id` / `with_principal`. | [src/ede/core/env.py](../src/ede/core/env.py) |
| `ModelProxy.search` / `search_count` / `read_group` | Top-level ORM read entry points; inject active filter before calling the repo. | [src/ede/core/orm/model_proxy.py](../src/ede/core/orm/model_proxy.py) |
| `RecordSet._get_one2many` / `_get_many2many` | Relational reads; inject active filter on the target model so archived children are hidden from parents. | [src/ede/core/orm/recordset.py](../src/ede/core/orm/recordset.py) |
| `_ede_handle_search` / `_ede_handle_count` / `_ede_handle_read_group` | Kernel CRUD command-bus handlers; inject active filter before repo call. | [src/ede/core/kernel/model.py](../src/ede/core/kernel/model.py) |
| `presentations.load_action` | Stamps `has_active` on the search-view payload from `__ede_has_active__`. | [src/ede/foundation/presentation/models/presentations.py](../src/ede/foundation/presentation/models/presentations.py) |
| `ControlPanelSearchView` (React) | Renders synthetic "Archived" filter chip when `searchView.has_active === true`. | [src/frontend/src/workspace/components/search/ControlPanelSearchView.tsx](../src/frontend/src/workspace/components/search/ControlPanelSearchView.tsx) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.search` | CRUD bus dispatch | Now auto-filters `active=False` rows when target model has `__ede_has_active__`. |
| `ede.count` | CRUD bus dispatch | Same auto-filter as `ede.search`. |
| `ede.read_group` | CRUD bus dispatch | Same auto-filter as `ede.search`. |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ — read-only behaviour change, no new events | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none_ — change is internal to the ORM/CRUD layer; existing endpoints inherit the new behaviour transparently | | |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_ — boolean toggle on a per-row field; not a multi-state lifecycle.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- No new module activation; behaviour change is in `ede.core.*` and applies framework-wide as soon as a model declares `active`.
- Per-model opt-in: declare `active = fields.Boolean(default=True)` on the DomainModel.
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
| _none_ | |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| n/a | Single-shot platform change | ✅ Delivered (2026-05-08) | [roadmap/platform/01-orm-active-filter.md](../roadmap/platform/01-orm-active-filter.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Registry-driven `__ede_has_active__` detection | every DomainModel subclass | [kernel/model.py](../src/ede/core/kernel/model.py) `__init_subclass__` | [roadmap/platform/01-orm-active-filter.md](../roadmap/platform/01-orm-active-filter.md) |
| Domain helpers (`domain_mentions_active`, `augment_with_active_filter`, `apply_active_filter`) | n/a | [services/persistence/domain_filter.py](../src/ede/core/services/persistence/domain_filter.py) | same |
| `Env.with_active_test(bool)` per-scope opt-out | n/a | [core/env.py](../src/ede/core/env.py) | same |
| 8-callsite injection (ModelProxy x3, RecordSet x2, kernel CRUD x3) | n/a | model_proxy.py, recordset.py, kernel/model.py | same |
| Backend `has_active` payload on `SearchViewDef` | n/a | [foundation/presentation/models/presentations.py](../src/ede/foundation/presentation/models/presentations.py) | same |
| Frontend "Archived" toggle in search panel | n/a | [frontend/.../ControlPanelSearchView.tsx](../src/frontend/src/workspace/components/search/ControlPanelSearchView.tsx) | same |
| 18 integration tests | n/a | [src/tests/orm/test_active_filter.py](../src/tests/orm/test_active_filter.py) | same |
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
- Calling `record.read()` on an archived RecordSet expecting an empty result. By-UUID reads stay unfiltered — the caller already knows the UUID, so they get the data.
- Forgetting that `is_active` is **not** the canonical name. Models still using `is_active` do not get the auto-filter; only an exact `active` Boolean field triggers detection.
- Adding the auto-filter to a transactional model (e.g. order, shipment, invoice). Transactional records have a `state`/`status` field that drives their lifecycle — they should not have an `active` field. Adding one would let users "archive" a half-completed order out of view.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- No schema changes — opt-in by virtue of the `active` field a model already declares.
- Models currently using `is_active` are unaffected; a separate naming-unification effort tracks renaming `is_active` → `active`.
- Audit any callsite that *expected* archived rows to be visible without explicit filtering — those should add an `active` leaf to the domain or use `env.with_active_test(False)`. Example landed in this change: `src/tests/data_loader/test_data_loader.py::test_bool_and_int_coercion` was searching for an `active=False` record and was updated to wrap in `with_active_test(False)`.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ — no new RBAC; existing read permissions still apply, the filter is orthogonal | |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Archivable Models — Developer Guide](foundation-archivable-models.md) — hand-authored "how-to" companion.
- [docs/05-orm-layer.md](05-orm-layer.md) — broader ORM architecture (legacy guide).
- [docs/03-domain-model.md](03-domain-model.md) — DomainModel base class (legacy guide).
- [CLAUDE.md](../CLAUDE.md) — one-line rule pointing to the developer guide.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-08. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
