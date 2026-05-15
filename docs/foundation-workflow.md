<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Workflow Engine — Implementation Docs

**Module:** `foundation.workflow` (`src/ede/foundation/workflow/`)
**Roadmap:** [roadmap/foundation/workflow/](../roadmap/foundation/workflow/README.md)
**Status:** 🟡 Phase 2 In Progress — Phase 1 ✅ Delivered (engine, DSL, ORM guard, approval bridge, pricing.rate proof migration shipped); Phase 2 in flight (form-view statusbar + `WorkflowController` HTTP routes; kanban drag-drop moved out, now scoped under [`foundation.presentation` Phase 1](../roadmap/foundation/presentation/phase-1-implementation.md)); Phase 3 not started
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A generic, DB-configurable workflow engine that becomes the only path for changing any field marked `workflow=True`. Stage progressions, guard checks, and transition actions are declared as `ir.workflow.*` rows — not embedded in command handlers. The engine plugs into the existing lifecycle-hook system; domains gain statusbar UIs, kanban drag-drop transitions, and a runtime visual editor for free.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every domain that has a lifecycle today re-invents the same five concerns: defining stages, enforcing legal transitions, evaluating guards, gating on approval, and auditing changes. Five domains × the same five concerns = 25 ad-hoc implementations that drift. The platform owns it once; domains declare a `workflow=True` field and get all five for free. This module also resolves the `ir.state.machine` row in [platform/00-execution-rules.md](../roadmap/platform/00-execution-rules.md) that has been outstanding since the rules were authored.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX (Phase 2+)**: Statusbar in form view auto-renders the current stage and allowed transition buttons; kanban drag-drop fires guarded transitions.
- **Admin UX (Phase 3)**: Settings → Technical → Workflows opens a `@xyflow/react`-based visual editor where stages and transitions are drawn as nodes and edges; "Manage Workflow" shortcut on every action's top bar deep-links to the editor for that model.
- **Programmatic entry**: `env.workflow.transition(record, "submit")` — the only way to change a field with `workflow=True`. Direct ORM writes raise `StageWriteForbidden`.
- **Integration boundary**: Workflow consumes the approval engine (`ir.approval.case.request` is the canonical async action) and emits its own `workflow.stage.entered` / `workflow.stage.exited` / `workflow.transition.pending` / `workflow.transition.cancelled` events for downstream subscribers.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Domain]                            [foundation.workflow]
─────────                           ─────────────────────
fields.Enum(workflow=True)   ──►   boot-time scan validates
                                    ir.workflow.definition exists
                                            │
                                            ▼
record.write({"status": "..."})  ──►  StageWriteForbidden
                                       (engine is the only path)
                                            │
                                            ▼
env.workflow.transition(record,    ──►   check → validate → match
                        "submit")        → guard → trigger
                                            │
                                            ▼
                                    sync action: write new stage value
                                    async action: dispatch approval,
                                                  set pending_transition_id
                                            │
                                            ▼
                          approval.case.approved  ──►  bridge advances
                          approval.case.rejected  ──►  bridge moves to failure stage
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.workflow.definition` | One per `(model_key, field_name)` pair; names the workflow and holds editor layout JSON | `src/ede/foundation/workflow/models/definition.py` (Phase 1) |
| `ir.workflow.stage` | Stage node; polymorphic `target_kind` (`enum` / `reference`) with `target_value` | `src/ede/foundation/workflow/models/stage.py` (Phase 1) |
| `ir.workflow.transition` | Directed edge with trigger, guard (inline `guard_ast` OR reference to `ir.workflow.guard.policy`), action, action_config | `src/ede/foundation/workflow/models/transition.py` (Phase 1) |
| `ir.workflow.instance` | Per-record runtime tracker; current_stage_id + pending_transition_id | `src/ede/foundation/workflow/models/instance.py` (Phase 1) |
| `ir.workflow.event.log` | Append-only audit ledger | `src/ede/foundation/workflow/models/event_log.py` (Phase 1) |
| `ir.workflow.guard.policy` | Named, reusable guard expression scoped to one workflow definition; holds `expression_text` (human-readable source) + `expression_ast` (compiled JSON). Mirrors `ir.approval.policy.set` + `rule` for approval-flow selection — except guards return a boolean to gate transitions. | `src/ede/foundation/workflow/models/guard_policy.py` (Phase 1) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `WorkflowEngine` | `transition()`, `bulk_transition()`, `available_transitions()`, `cancel_pending()`. Resolves `guard_policy_id` first, falls back to inline `guard_ast`, then to always-true. | `src/ede/foundation/workflow/services/engine.py` (Phase 1) |
| `OrmGuard` | `pre.{model}.write` hook that rejects direct writes to `workflow=True` fields | `src/ede/foundation/workflow/services/orm_guard.py` (Phase 1) |
| `ApprovalBridge` | `@api.on_event("approval.case.approved" / "rejected")` listeners that complete async transitions | `src/ede/foundation/workflow/services/approval_bridge.py` (Phase 1) |
| `Bootstrap` | Boot-time scan for `workflow=True` fields, validation of definitions, hook installation | `src/ede/foundation/workflow/bootstrap.py` (Phase 1) |
| `AstEvaluator` (shared) | Guard expression evaluator lifted from `ir.approval.rule` | `src/ede/foundation/base/services/ast_evaluator.py` (Phase 1) |
| `WorkflowXmlParser` | Parses `<workflow>` DSL documents from `data/*.xml` into ordered `ir.workflow.*` create-commands. Resolves stage codes locally within the workflow scope; resolves `guard="<code>"` to `ir.workflow.guard.policy` rows. | `src/ede/foundation/workflow/data_loader/workflow_xml_parser.py` (Phase 1, WS-W5) |
| `ExpressionCompiler` | Compiles human-readable guard expressions (e.g., `subject.margin_pct < 0.15`) into the JSON AST consumed by `AstEvaluator`. Invoked by the XML parser at install time and by a `pre.ir.workflow.guard.policy.update` hook on admin edits. | `src/ede/foundation/workflow/data_loader/expression_compiler.py` (Phase 1, WS-W5) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ — workflow does not introduce its own commands | | The engine listens to existing domain commands via `pre.{trigger_command}` hooks |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `workflow.stage.entered` | After a transition completes (sync or async-completed) | Audit, analytics, downstream consumers |
| `workflow.stage.exited` | Paired with `workflow.stage.entered` | Same as above |
| `workflow.transition.pending` | An async transition has dispatched the action and is awaiting completion | UI badge ("Awaiting approval") |
| `workflow.transition.cancelled` | Pending transition was withdrawn via `cancel_pending()` | UI clears the pending badge |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/workflow/transition` | Run a transition (record + code) | `src/ede/foundation/workflow/controllers/workflow.py` (Phase 2) |
| `GET /api/workflow/available?model=&uuid=` | List allowed transitions for a record + principal | (Phase 2) |
| `GET /api/workflow/definition/:code` | Fetch full definition with stages, transitions, layout | (Phase 3) |
| `PUT /api/workflow/definition/:code` | Atomic save from visual editor | (Phase 3) |
| `GET /api/workflow/by-model?model_key=&field_name=` | Resolve definition for action-bar shortcut | (Phase 3) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.{model}.write` (per workflowed model) | Rejects writes that target `workflow=True` fields unless `env._workflow_dispatch` is set by the engine |
| `pre.{model}.create` (per workflowed model) | Auto-fills the field with the `is_initial` stage's `target_value`; ignores user-supplied value |
| `pre.{trigger_command}` (per command-triggered transition) | Routes the command into `WorkflowEngine.transition()` |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
The engine itself does not have a state machine — it executes whatever state machines domains declare via `ir.workflow.definition` rows. Per-record runtime state is tracked by `ir.workflow.instance.current_stage_id` (mirrors the parent's field value) and `ir.workflow.instance.pending_transition_id` (set during async transitions, blocks new transitions, cleared on completion or cancel).
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Empty rows fine; missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` (in `src/ede/foundation/settings.py`): add `"workflow"` after `"approval"`.
- `ACTIVE_DOMAINS`: n/a (foundation engine).
- Manifest `depends`: `["base", "approval"]`.
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
| `data/ir.rbac.permission.csv` | Permission rows: `workflow.read`, `workflow.manage` (Phase 3) |
| `data/workflow_menus.xml` | Settings → Technical → Workflows menu (Phase 3) |
| `src/domains/logistics/pricing/data/workflow.pricing.rate.xml` | (Domain-side, Phase 1 proof migration) Seed `pricing.rate.lifecycle` definition + 2 named guard policies (`rate_complete`, `is_admin`) + 6 stages + 5 transitions, authored in the `<workflow>` XML DSL with `noupdate="true"` |
| `docs/foundation-workflow-dsl.md` | Authoring guide for the `<workflow>` XML DSL (Phase 1, WS-W5) — end-to-end example, expression grammar, named-guard vs inline-expression rule of thumb, raw-`<record>`-to-DSL migration recipe |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Engine + Enforcement + DSL + Pricing.Rate Migration | ✅ Delivered | [phase-1-implementation.md](../roadmap/foundation/workflow/phase-1-implementation.md) |
| Phase 2 | UI Integration (statusbar) | 🟡 In Progress | [phase-2-implementation.md](../roadmap/foundation/workflow/phase-2-implementation.md) |
| Phase 3 | Visual Editor + Action-Bar Shortcut + Cron + Bulk | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/workflow/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Kernel `workflow=False` kwarg + `FieldSpec.workflow` propagation (WS-W1) | n/a | [src/ede/core/kernel/fields.py](../src/ede/core/kernel/fields.py) | [phase-1-implementation.md WS-W1](../roadmap/foundation/workflow/phase-1-implementation.md) |
| AST evaluator lifted to shared `foundation.base` service (WS-W1) | n/a | [src/ede/foundation/base/services/ast_evaluator.py](../src/ede/foundation/base/services/ast_evaluator.py) | [phase-1-implementation.md WS-W1](../roadmap/foundation/workflow/phase-1-implementation.md) |
| Six `ir.workflow.*` models + Alembic migrations (WS-W2) | `ir.workflow.definition`, `.stage`, `.transition`, `.instance`, `.event.log`, `.guard.policy` | [src/ede/foundation/workflow/models/](../src/ede/foundation/workflow/models/), [migrations/versions/ed403d6818f9_…](../src/ede/foundation/workflow/migrations/versions/ed403d6818f9_foundation_workflow_initial.py), [c8a4f2d9e7b1_…](../src/ede/foundation/workflow/migrations/versions/c8a4f2d9e7b1_foundation_workflow_guard_policy.py) | [phase-1-implementation.md WS-W2](../roadmap/foundation/workflow/phase-1-implementation.md) |
| `WorkflowEngine` (`transition`, `transition_by_command`, `bulk_transition`, `available_transitions`, `cancel_pending`) + ORM write guard + bootstrap scan (WS-W3) | n/a | [services/engine.py](../src/ede/foundation/workflow/services/engine.py), [services/orm_guard.py](../src/ede/foundation/workflow/services/orm_guard.py), [bootstrap.py](../src/ede/foundation/workflow/bootstrap.py) | [phase-1-implementation.md WS-W3](../roadmap/foundation/workflow/phase-1-implementation.md) |
| Approval bridge (consumer of `approval.case.approved/rejected` events) (WS-W4) | n/a | [services/approval_bridge.py](../src/ede/foundation/workflow/services/approval_bridge.py) | [phase-1-implementation.md WS-W4](../roadmap/foundation/workflow/phase-1-implementation.md) |
| `<workflow>` XML DSL parser + named-guard registry + expression compiler (WS-W5) | n/a | [data_loader/xml_parser.py](../src/ede/foundation/workflow/data_loader/xml_parser.py), [data_loader/expression_compiler.py](../src/ede/foundation/workflow/data_loader/expression_compiler.py), [data_loader/schemas/workflow.rng](../src/ede/foundation/workflow/data_loader/schemas/workflow.rng) | [phase-1-implementation.md WS-W5](../roadmap/foundation/workflow/phase-1-implementation.md) |
| DSL authoring guide | n/a | [docs/foundation-workflow-dsl.md](./foundation-workflow-dsl.md) | [phase-1-implementation.md WS-W5](../roadmap/foundation/workflow/phase-1-implementation.md) |
| **Pricing.rate proof migration (WS-W6)** — `pricing.rate.status` declared `workflow=True`; the four hardcoded transition handlers (`handle_submit/approve/reject/suspend`) refactored to validation + `env.workflow.transition_by_command(...)` delegation + post-transition side effects (rate-number generation, rejection notification). The `pricing_rate_workflow.xml` seed (2 named guards, 6 stages, 7 transitions) is now load-bearing. | `pricing.rate` | [src/domains/logistics/pricing/models/rate.py](../src/domains/logistics/pricing/models/rate.py), [src/domains/logistics/pricing/data/pricing_rate_workflow.xml](../src/domains/logistics/pricing/data/pricing_rate_workflow.xml) | [phase-1-implementation.md WS-W6](../roadmap/foundation/workflow/phase-1-implementation.md) |
| Engine + handler test coverage (WS-W7) | n/a | [src/tests/foundation/workflow/](../src/tests/foundation/workflow/) (10 files incl. `test_transition_by_command.py`), [src/tests/pricing/test_margin_engine.py](../src/tests/pricing/test_margin_engine.py) (handler delegation tests) | [phase-1-implementation.md WS-W7](../roadmap/foundation/workflow/phase-1-implementation.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Phase 2 (statusbar + WorkflowController routes) in flight 2026-05-10 | 🟡 | [phase-2-implementation.md](../roadmap/foundation/workflow/phase-2-implementation.md) |
| Kanban drag-drop moved out of workflow Phase 2 — now scoped under `foundation.presentation` Phase 1 (KanbanView MVP). The `WorkflowController` routes shipping in workflow Phase 2 are the contract the kanban calls. | 🔴 | [foundation/presentation phase-1-implementation.md](../roadmap/foundation/presentation/phase-1-implementation.md) |
| Phase 3 (visual editor + cron + bulk + admin permission) not started | 🔴 | [phase-3-implementation.md](../roadmap/foundation/workflow/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
_(populated as integration learnings emerge after Phase 1 lands)_

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1 introduces six new `ir.workflow.*` tables via Alembic migration: `definition`, `stage`, `transition`, `instance`, `event.log`, `guard.policy`.
- Phase 1 adds `ir.workflow.transition.guard_policy_id` reference (nullable) alongside the existing `guard_ast` JSON column. A pre-write hook rejects transitions that set both fields.
- Phase 1 lifts the AST evaluator out of `foundation.approval` into `foundation.base`. Approval imports refactored; no behavior change. No data migration needed.
- Phase 1 ships the `<workflow>` XML DSL (WS-W5) as the canonical seeding format. Domains author one `data/workflow.<model>.xml` per workflow with named `<guards>`, `<stage>`s, and `<transition>`s. The parser compiles human-readable guard expressions into AST JSON. Raw `<record>` rows and CSV remain technically possible but are not encouraged for new workflows.
- `noupdate="true"` on a `<workflow>` element follows the existing `ir.menu` / `ir.action.*` convention — on module upgrade, rows are skipped to preserve admin edits made via the Phase 3 Visual Editor.
- Phase 1 migrates `pricing.rate` lifecycle to engine-driven, authored entirely in the new XML DSL. Existing rate records keep their current `status` value, which the engine's boot-time validation maps to seeded `ir.workflow.stage` rows.
- Phase 1 adds `workflow: bool = False` kwarg to base `Field` class — backwards-compatible default, no existing field declarations affected.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Admin | `workflow.read`, `workflow.manage` (Phase 3) |
| Standard user | `workflow.read` (Phase 3) — sufficient to see statusbar transitions allowed by guards; cannot edit workflow definitions |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Approval](foundation-approval.md) — peer engine; workflow's async action dispatches into it
- [Platform Execution Rules](platform-00-execution-rules.md) — the `ir.state.machine` row this module resolves
- [Foundation Model Naming](foundation-model-naming.md) — `ir.*` vs `res.*` conventions
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-10 (Phase 2 🔴 → 🟡 In Progress — UI integration started; kanban drag-drop split out of Phase 2 since no `KanbanView.tsx` exists yet). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
