<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Computed Field Runtime — Implementation Docs

**Module:** `ede.core.kernel` + `ede.core.orm` (cross-cutting)
**Roadmap:** [roadmap/platform/02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md)
**Status:** ✅ Delivered (Phase 1 + Phase 2 + Phase 3 — all shipped 2026-05-11)
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A kernel-level runtime that materialises the declared-but-unwired `method=` / `depends_on=` / `store=` API on `fields.Field`. Today the descriptor accepts those kwargs ([kernel/fields.py:110–186](../src/ede/core/kernel/fields.py)) and the schema builder uses `store=True` to decide column inclusion ([kernel/schema.py:185-188](../src/ede/core/kernel/schema.py)), but no runtime invokes the compute method or watches dependencies — every "auto-computed header field" in the codebase is a stored regular field with an imperative post-write hook on its dependency model.

Phase 1 closes the loop: declaring `method="_compute_x"` + `depends_on=["other_field", "line_ids.unit_rate"]` (and optionally `store=True`) registers the field in a dependency graph at module-load time, and a write-time invoker recomputes affected fields in the same Unit-of-Work when their dependencies change. Stored mode persists the value to the column; unstored mode computes on read with no column.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every domain that today writes a "derived header field" duplicates the same hook plumbing. `pricing.rate` has four such fields (`calculated_margin_amount`, `calculated_margin_percent`, `margin_risk_level`, `calculated_sell_amount`) all updated by a `_recalculate_margin` helper called from `post.pricing.rate.line.{create,update,delete}` hooks. Future analytics, finance, reporting, and CRM modules will all need at least one similar field. The hook pattern has four real problems:

1. **Imperative coupling.** Each line-side hook has to name the parent's field. Renaming the field means touching the hook.
2. **No dependency declaration in the field.** Readers of `pricing.rate.calculated_sell_amount` have no way to see "this is recomputed from `line_ids.unit_rate`" without grepping for the hook.
3. **Easy to drift.** Add a new dependency (e.g. `currency_id` change should trigger FX-aware recompute) and the hook has to be edited too; declarative `depends_on=["line_ids.unit_rate", "line_ids.currency_id"]` would catch the new edge automatically.
4. **No transient/unstored mode.** Want a field that's never persisted but always reflects current state of dependencies? Today you'd write a service method and call it from every read site.

Wiring the kernel runtime collapses all four into a one-line field declaration.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Application developer (declaring a computed field):** add `method="_compute_x"` + `depends_on=[...]` (and `store=True` if persistence is wanted) to the field declaration. Implement `_compute_x(self, env) -> Any` on the same DomainModel class.
- **Application developer (reading the value):** `record.x` or `record.read()` returns the up-to-date value. No need to call the compute method directly.
- **No end-user UX surface.** This is a developer-time feature; admins never see it.
- **No HTTP endpoints.** Pure kernel runtime.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
Module load
   field declaration with method= + depends_on=
            │
            ▼
DependencyGraphBuilder            ──►   inverse map { (model_key, field) : [(dependent_model, dependent_field, kind), ...] }
   (scans __ede_field_specs__)         where kind ∈ {same_record, o2m_membership, o2m_field}
                                       cycle detection → raises at boot
                                            │
                                            ▼
Write-time trigger                  Read-time shim
   _ede_handle_update                 RecordSet.read()
   _ede_handle_create                       │
   post.{line_model}.{create,delete}        ▼
            │                          invoke method for unstored
            ▼                          computed fields on demand
   walk inverse graph
   topo-sort affected stored fields
   invoke method → write column
            │
            ▼
   end of UoW: dependent stored
   values are now consistent with the
   triggering write
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none_ — runtime only, no new models | | |

The runtime annotates the host model class with a cached `__ede_unstored_compute_fields__` attribute on first read of an unstored compute (set by `_unstored_compute_field_names` in [recordset.py](../src/ede/core/orm/recordset.py)). Not a model in the persistence sense — just a class-level memoization tag.
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `build_compute_graph(registry)` | Scans every registered model's `__ede_field_specs__` at boot; resolves same-record / O2M dotted / O2M bare / M2O dotted dependency expressions; produces an inverse map keyed on `(trigger_model, field_or_trigger)`; topo-sorts compute targets; raises `ComputeCycleError` on cycle. | [src/ede/core/orm/compute_graph.py](../src/ede/core/orm/compute_graph.py) |
| `register_compute_hooks(registry)` | Installs the four lifecycle hooks per trigger model: `post.{m}.create` / `post.{m}.update` / `pre.{m}.delete` (snapshot backref FK values) / `post.{m}.delete`. Returns `None` (and registers nothing) when no model declares a compute field. | [src/ede/core/orm/compute_runtime.py](../src/ede/core/orm/compute_runtime.py) |
| `execute_compute_chain` | Per write: collects direct dependents, transitively expands same-record cascades, dedupes, topo-sorts, invokes each compute method, writes through `_acquire_repo_for_env` (reuses the active UoW so we see uncommitted INSERTs). Wrapped in a `_compute_dispatch` token so our own writes don't re-trigger us. | [compute_runtime.py](../src/ede/core/orm/compute_runtime.py) |
| `_resolve_target_uuids` | Translates a triggering record into the list of target record UUIDs per dep kind: same_record → identity; o2m_field/o2m_membership → read backref FK (or pre-delete snapshot); m2o_field → reverse-search the owning model. | [compute_runtime.py](../src/ede/core/orm/compute_runtime.py) |
| `_unstored_compute_field_names(model_cls)` | Lists `store=False` compute fields for a model; cached on the class. Used by `RecordSet.read()` to materialise unstored values on demand. | [recordset.py](../src/ede/core/orm/recordset.py) |
| `fields.Field._build_compute_metadata` | Already produces the `compute` dict on FieldSpec — unchanged. | [src/ede/core/kernel/fields.py](../src/ede/core/kernel/fields.py) |
| `kernel/schema._is_persisted_field` | Honours `store=True` for column inclusion — unchanged. | [src/ede/core/kernel/schema.py](../src/ede/core/kernel/schema.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ — no new commands; runtime hooks into existing CRUD bus | | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ — recomputes are silent (no `ede.record.updated` thrash for system-internal recomputes; this is a Phase 1 decision documented in the roadmap) | | |
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
| `post.{model}.update` | Walks the inverse map for the keys of `cmd.payload["values"]` and recomputes dependent **stored** computed fields in topological order. Same-UoW; no separate transaction. Unstored computes are skipped at write time — they evaluate on read. |
| `post.{model}.create` | Same logic as update, plus the `__create__` trigger so O2M-membership deps fire (parent's sum-over-lines updates when a new line is born). |
| `pre.{model}.delete` | Snapshots the to-be-deleted record's backref FK values into `cmd._compute_delete_snapshot` so the post-delete recompute can find the parent after the row is gone. |
| `post.{model}.delete` | Fires the `__delete__` trigger using the pre-delete snapshot; recomputes affected parents (the line is already gone from the DB at this point). |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- No new `ACTIVE_MODULES` / `ACTIVE_DOMAINS` entry — this is kernel runtime.
- No manifest `depends` change.
- Opt-in is **per field**: declare `method=` on a `fields.*` descriptor. Fields without `method` keep their current behaviour.
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
| _none_ — runtime + tests only | |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Compute runtime + same-record + O2M `depends_on` | ✅ Delivered (2026-05-11) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| Phase 2 | Retrofit `pricing.rate`'s 4 hook-driven fields to declarative form | ✅ Delivered (2026-05-11) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| Phase 3 | M2O dotted dependency paths + unstored (`store=False`) computes | ✅ Delivered (2026-05-11) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Dependency graph builder + cycle detection at boot (Phase 1) | n/a — kernel | [src/ede/core/orm/compute_graph.py](../src/ede/core/orm/compute_graph.py) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| Compute invoker + write-time `post.{m}.{create,update,delete}` hooks + transitive cascading + idempotency check (Phase 1) | n/a — kernel | [src/ede/core/orm/compute_runtime.py](../src/ede/core/orm/compute_runtime.py) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| `_ede_handle_create` populates `cmd.model_id` post-INSERT (side fix; unblocks post-create hooks across the codebase) | n/a — kernel | [src/ede/core/kernel/model.py](../src/ede/core/kernel/model.py) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| Boot integration via `register_compute_hooks(registry)` (Phase 1) | n/a — kernel | [src/ede/core/bootstrap.py](../src/ede/core/bootstrap.py) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| Pricing.rate retrofit: 3 hook-driven fields → declarative `method=` + `depends_on=`; legacy `_recalculate_margin` + 3 line post-hooks deleted (Phase 2) | `pricing.rate` | [src/domains/logistics/pricing/models/rate.py](../src/domains/logistics/pricing/models/rate.py) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| M2O dotted paths (`depends_on=["currency_id.symbol"]` triggers on target row update via reverse-search resolver, plus owning record m2o reassignment) (Phase 3) | n/a — kernel | [compute_graph.py](../src/ede/core/orm/compute_graph.py), [compute_runtime.py](../src/ede/core/orm/compute_runtime.py) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| Unstored computes (`store=False`) evaluate on `RecordSet.read()` with no DB column (Phase 3) | n/a — kernel | [src/ede/core/orm/recordset.py](../src/ede/core/orm/recordset.py) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| Test suite — 21 integration tests against a real SQLite tenant covering all 3 phases incl. cycle detection, FK-detach at delete, transitive same-record cascade, M2O reverse-search, unstored-on-read | n/a — tests | [src/tests/orm/test_compute_runtime.py](../src/tests/orm/test_compute_runtime.py) | [02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| _none — all Phase 1, 2, 3 gaps closed 2026-05-11. Out-of-scope items (caching for unstored computes, async/background recompute, runtime UI for compute fields) are deliberate Phase 3 deferrals; reopen as a new platform entry if a real workload pushes back._ | | |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
_(populated as integration learnings emerge after Phase 1 lands)_

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- When a stored computed field is **added** to an existing model, the migration owner is responsible for recomputing legacy rows. The runtime does NOT scan at boot. Recommended pattern: write a one-off Alembic data migration that iterates rows and triggers the compute.
- `workflow=True` fields are excluded from compute writes — they remain writable only via `env.workflow.transition()`. Declaring a compute on a workflow field is a usage error and would be flagged at registration in Phase 1.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ — compute writes happen as the active principal that triggered the dependency change | |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Platform Execution Rules](platform-execution-rules.md) — universal design rules
- [Foundation Model Naming](foundation-model-naming.md) — `ir.*` vs `res.*` conventions
- [src/ede/core/kernel/fields.py](../src/ede/core/kernel/fields.py) — current compute-metadata declaration surface
- [src/domains/logistics/pricing/models/rate.py](../src/domains/logistics/pricing/models/rate.py) — `_recalculate_margin` hook (Phase 2 retrofit target)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-11 (all 3 phases ✅ Delivered — kernel runtime shipped end-to-end, pricing.rate retrofitted off hook-driven margin computation, M2O dotted paths + unstored computes live, 21 compute_runtime tests, 1487 backend tests total). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
