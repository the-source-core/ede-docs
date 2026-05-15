<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Engine Substrate (`src/ede/core/engines/`) — Implementation Docs

**Module:** `ede.core.engines` + `ede.core.api` (cross-cutting kernel)
**Roadmap:** [roadmap/platform/05-engine-substrate.md](../roadmap/platform/05-engine-substrate.md)
**Status:** 🟡 In Progress (Phase 1 substrate Stage 1A + 1B shipped on 2026-05-13)
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A new top-level kernel directory `src/ede/core/engines/` at peer level with `kernel/`, `orm/`, `bus/`, `services/`, `adapters/`, `tenancy/`. Houses **renderer-agnostic computational engines** — dataset compiler, metric registry, formula evaluator, chip / period kernel, integration spine, plus the report / document / dashboard engines. Each engine subdirectory contains ONLY core engine code (algorithms + registries + runners + types) — no ORM models, no admin UI, no HTTP routes, no menus. Those live in the foundation module shell that wraps the engine.

Also covers the **decorator-surface additions to `src/ede/core/api.py`** that foundation modules use to register into the engines: `@api.metric`, `@api.chip`, `@api.kpi` (joining the existing family of `@api.model`, `@api.on_command`, `@api.scheduled_job`).
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Today every kernel capability lives in one of four parallel trees under `src/ede/core/`. New kernel capabilities have generally landed inside `services/` when they don't fit `kernel/`, `orm/`, or `bus/`. The reporting / dashboard / document substrate introduces four substantial computational engines that share structural traits not fitting `services/`:
- They're computational pipelines (typed input → typed output), not request-handling services.
- They're renderer-agnostic — same engine, multiple consumers (HTTP / SSE / WebSocket / event / webhook / future MCP).
- They register decorators on `core/api.py` that domain modules call.

Wedging them under `services/` would crowd that directory and blur its role. A sibling directory `engines/` is the cleaner home and establishes a stable pattern for future engines (search engine, expression engine, anything else that's pure computation).
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Engine authors:** drop a subdirectory under `src/ede/core/engines/` with pure-computation code. Declare the public entry function (e.g. `Compiler.build(spec) → CompiledQuery`). Add the corresponding decorator to `src/ede/core/api.py` if foundation modules will register into the engine.
- **Foundation module authors:** import the engine class, register via decorator. NEVER reach into another engine's private state. NEVER put ORM models / views / HTTP routes / RBAC inside `core/engines/`.
- **PR reviewers:** when reviewing a change that touches `src/ede/core/engines/` or adds a new `@api.<name>` decorator to `core/api.py`, confirm a foundation module's roadmap claims it.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
src/ede/core/
├── kernel/        — Field descriptors, schema builder, DomainModel base, UoW
├── orm/           — RecordSet, ModelProxy, repo helpers
├── bus/           — Command + event bus
├── services/      — Request infrastructure (HTTP, auth, tenancy, presentation DSL, persistence contracts)
├── adapters/      — FastAPI / SQLAlchemy / Kafka adapters
├── tenancy/       — Tenant context + resolver
└── engines/       ← NEW. Renderer-agnostic computational engines.
    ├── dataset/        — JSON spec → SQL compiler
    ├── metric/         — Registry + executor + 3 engines (dataset/plan/formula) + cache
    ├── formula/        — Safe AST evaluator (shared by metric formulas + DML <var formula="...">)
    ├── chip/           — Toolbar contract + auto-injecting providers
    ├── period/         — Period resolver with per-line date_scope
    ├── integration/    — JSON contract types + runner + HTTP / SSE / WS / webhook / export
    ├── report/         — Report runner + body builders + write-back + streaming + snapshot
    ├── document/       — DML parser + RelaxNG + inheritance + render plan + PDF/HTML/print
    └── dashboard/      — KPI registry + evaluator + runner + insight rules
```

Dependency rule: engines depend on `kernel/`, `bus/`, `orm/`, `services/persistence/` only. Engines can compose with each other (`metric/` imports `dataset/`; `document/dml/variables.py` imports `formula/safe_eval.py`). No engine imports from `foundation/<module>/`.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none_ | Engines hold no ORM models. Models live in each foundation module's shell. | n/a |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `DatasetCompiler` | 13-step pipeline: parse spec → resolve fields → assemble SELECT/WHERE/GROUP BY/ORDER BY/LIMIT via SQLAlchemy Core `select()`. Reuses kernel `_build_metadata`, `_build_where_clause`, `model_key_to_table_name`. Auto-applies `active_test` via `apply_active_filter`. | `src/ede/core/engines/dataset/compiler.py` |
| `FieldResolver` | Validates spec field references against registered DomainModel field specs; enforces projectable vs non-projectable field-type rules; resolves aliases. | `src/ede/core/engines/dataset/field_resolver.py` |
| Expression parsers | Regex-based parsers for `sum(...)` / `avg(...)` / `min(...)` / `max(...)` / `count(...)` / `count_distinct(...)` aggregates and `field asc|desc` sort directives. Strict whitespace rejection. | `src/ede/core/engines/dataset/expressions.py` |
| Typed exceptions | `DatasetSpecError`, `DatasetTimeoutError`, `DatasetRowLimitExceeded`, `MetricRegistrationError`, `MetricExecutionError`. | `src/ede/core/engines/dataset/errors.py` |
| `Metric` dataclass + `metric_registry` | Frozen `Metric` records + global registry with `register / get_metric / all_metrics / has_metric / clear_registry`; invariant validation at registration. | `src/ede/core/engines/metric/base.py`, `src/ede/core/engines/metric/registry.py` |
| `@api.metric` decorator | Class-decorator surface mirroring `@api.model`; harvests class attributes into a `Metric` and registers it. | `src/ede/core/engines/metric/decorator.py`, exported from `src/ede/core/api.py` |
| `MetricExecutor` | Executes a registered metric: RBAC gate first (`AuthorizationService.require(model_key, "read")`), then compiles spec → executes via `env.session.execute(stmt)` → projects `DatasetResult`. | `src/ede/core/engines/metric/executor.py` |
| Integration contract | `DatasetResult`, `DatasetMeta`, `SchemaColumn` TypedDicts + `ColumnKind` Literal closed enum + `CONTRACT_VERSION = 1` + `make_meta()` factory + `now_iso_utc()` helper. | `src/ede/core/engines/integration/contract.py` |
| `run_metric` central dispatch | One-line entrypoint that every HTTP/SSE/MCP adapter calls. Looks up metric, executes, returns `DatasetResult`. | `src/ede/core/engines/integration/runner.py` |
| _Phase 1 remainder pending_ | `engines/formula/safe_eval.py`, `engines/chip/`, `engines/period/`, plan/formula metric engines, per-run cache. | (planned) — Stage 1C onward |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | Engines do not register their own commands directly. Each foundation module wires its engine to commands via `@api.on_command` in its shell. | n/a |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ | Engines are pure computation. Events come from foundation modules around them. | n/a |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none_ | Engines hold no HTTP routes; controllers live in foundation shells. | n/a |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
*Not applicable — engines are stateless computation pipelines.*
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> This is a directory + decorator-surface convention. No env vars, no `ir.config` keys, no `<settings>` panels, no seed data. Engine-specific settings live in each foundation module's manifest.

### Activation
<!-- SYNC-BLOCK: activation -->
- No new `ACTIVE_MODULES` entry — the directory is kernel code, not a foundation app.
- No manifest changes.
<!-- /SYNC-BLOCK -->

### Foundation-level settings
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
| Phase 1 | `engines/` namespace + dataset + metric + formula + chip + period + integration | 🟡 | [roadmap/platform/05-engine-substrate.md](../roadmap/platform/05-engine-substrate.md) — Stage 1A (errors + expressions + field_resolver) and Stage 1B (compiler + metric registry + executor + integration contract + `@api.metric`) shipped 2026-05-13; chip/period/formula and plan/formula metric engines pending |
| Phase 2 | Report engine subdirectory | 🔴 | Same file — lands with `foundation.reporting` Waves W3b/W6/W7 |
| Phase 3 | Document engine subdirectory | 🔴 | Same file — lands with `foundation.document` Waves W4/W6/W7 |
| Phase 4 | Dashboard engine subdirectory + `@api.kpi` | 🔴 | Same file — lands with `foundation.dashboard` Waves W5/W7 |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| `engines/` namespace created (sibling to `kernel/` / `bus/` / `orm/` / `services/`) | n/a | `src/ede/core/engines/__init__.py` | Phase 1 / Stage 1A |
| Dataset compiler — JSON spec → SQLAlchemy Core `select()` (parameterized) | n/a | `src/ede/core/engines/dataset/compiler.py` + `field_resolver.py` + `expressions.py` + `errors.py` | Phase 1 / Stage 1B |
| RBAC gate inside metric executor — calls `AuthorizationService(env).require(model_key, "read")` before any SQL is compiled or emitted | n/a | `src/ede/core/engines/metric/executor.py` | Phase 1 / Stage 1B |
| `active_test` auto-filter respected — compiler delegates to kernel's `apply_active_filter(domain, env, model_cls)` so archived rows are hidden unless `env.with_active_test(False)` | n/a | `src/ede/core/engines/dataset/compiler.py` `_apply_active_test` | Phase 1 / Stage 1B |
| Metric registry + `@api.metric` class-decorator surface | n/a | `src/ede/core/engines/metric/{base,registry,decorator,executor}.py`; exported from `src/ede/core/api.py` | Phase 1 / Stage 1B |
| Universal JSON contract v1 — `DatasetResult` / `DatasetMeta` / `SchemaColumn` TypedDicts + `ColumnKind` closed enum + `CONTRACT_VERSION = 1` | n/a | `src/ede/core/engines/integration/contract.py` | Phase 1 / Stage 1B (also tracked under platform-06) |
| Central `run_metric` dispatch | n/a | `src/ede/core/engines/integration/runner.py` | Phase 1 / Stage 1B |
| Tracer-bullet end-to-end proof — 10 pytest cases covering smoke / default-value / unknown-metric / RBAC gate / `active_test` gate | n/a | `src/tests/core/engines/dataset/test_tracer_bullet.py` + `test_expressions.py` (30) + `test_field_resolver.py` (15) | Phase 1 / Stage 1B |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| `engines/formula/safe_eval.py` shared AST evaluator not yet shipped (consumed by metric formula engine + DML `<var formula="...">`) | 🟡 Phase 1 remainder | [roadmap/platform/07-shared-safe-ast-evaluator.md](../roadmap/platform/07-shared-safe-ast-evaluator.md) |
| Plan + Formula metric engines (only Dataset engine ships in Stage 1B) | 🟡 Phase 1 remainder | [roadmap/foundation/dataset/phase-2-implementation.md](../roadmap/foundation/dataset/phase-2-implementation.md) |
| Per-run metric cache (so a 5-line × 3-period × 4-metric report executes ≤12 unique queries instead of 60) | 🟡 Phase 1 remainder | [roadmap/foundation/dataset/phase-2-implementation.md](../roadmap/foundation/dataset/phase-2-implementation.md) |
| `@api.chip` decorator + `engines/chip/` + `engines/period/` (per-line `date_scope` resolution) | 🟡 Phase 1 remainder | [roadmap/foundation/dataset/phase-3-implementation.md](../roadmap/foundation/dataset/phase-3-implementation.md) |
| `engines/report/` directory | 🔴 Phase 2 | [roadmap/foundation/reporting/](../roadmap/foundation/reporting/) |
| `engines/document/` directory | 🔴 Phase 3 | [roadmap/foundation/document/](../roadmap/foundation/document/) |
| `engines/dashboard/` directory + `@api.kpi` decorator | 🔴 Phase 4 | [roadmap/foundation/dashboard/](../roadmap/foundation/dashboard/) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
*Will be populated as the substrate is built and engineers integrate.*

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
*Pre-build. No schema changes; directory creation is a code-only change.*
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ | RBAC is owned by each foundation module's shell, never by the engine. |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.dataset`](./foundation-dataset.md) — Phase 1 consumer; introduces 6 of the substrate engines.
- [`foundation.reporting`](./foundation-reporting.md) — Phase 2 consumer; introduces `engines/report/`.
- [`foundation.document`](./foundation-document.md) — Phase 3 consumer; introduces `engines/document/`.
- [`foundation.dashboard`](./foundation-dashboard.md) — Phase 4 consumer; introduces `engines/dashboard/` + `@api.kpi`.
- [`platform-06-universal-json-contract`](./platform-06-universal-json-contract.md) — the JSON contract every engine produces; type definitions live in `engines/integration/contract.py`.
- [`platform-07-shared-safe-ast-evaluator`](./platform-07-shared-safe-ast-evaluator.md) — `engines/formula/safe_eval.py` — shared by metric formulas + DML variables.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-13. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
