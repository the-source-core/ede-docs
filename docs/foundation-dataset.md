<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Dataset & Metric Engine — Implementation Docs

**Module:** `foundation.dataset` (`src/ede/core/engines/{dataset,metric,formula,chip,period,integration}/` + `src/ede/foundation/dataset/`)
**Roadmap:** [roadmap/foundation/dataset/](../roadmap/foundation/dataset/README.md)
**Status:** ✅ Phase 1 Delivered 2026-05-13 · ✅ Phase 2 Delivered 2026-05-13 (plan + formula + cache + safe AST evaluator; 50 new pytest cases)
**Layer:** Foundation engine — data substrate; sits below `foundation.reporting`, `foundation.dashboard`, `foundation.document`.

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **declarative data layer** that compiles JSON specs into tenant-safe, parameterised SQL and ships a renderer-agnostic JSON result contract. Two surfaces flow into the same compiler:

1. **Code-authored metrics** — Python modules register frozen `Metric` definitions via `@api.metric("key")`. Used for invariant business truths (`sales.pipeline_value`, `revenue_by_partner`, etc.) shared across every consumer.
2. **Low-code Blueprint UI** — admins design datasets in a form (`Settings → Technical → Datasets`); the Blueprint emits the same JSON spec at save time. Used for ad-hoc / customer-specific datasets that don't warrant a code change.

Both surfaces speak the same compiler and the same `DatasetResult` JSON contract. Three execution engines plug into the metric registry: **dataset** (single spec), **plan** (multi-spec + post-process merge for opening + period + closing), **formula** (DAG-resolved derived KPIs).
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every reporting / analytics / document use-case across EDE needs the same primitive: turn a declarative spec into a safe, tenant-scoped, ORM-compliant SQL query, then expose the result through a renderer-agnostic JSON contract. Without a shared substrate, every consumer (reports, dashboards, documents, exports, the future internal MCP server) would re-implement compilation inline — exactly the trap that produces 7,000-line monoliths where SQL, options management, column generation, and rendering are tangled together.

`foundation.dataset` lifts the data layer out as one shared substrate so reports, dashboards, documents, exports, and the future MCP server all consume the **same** compiled spec — never duplicate SQL. The Blueprint UI is the productisation deliverable: without it, every new dataset would require a Python change + deploy; with it, an analyst draws a query in a form.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX** — Two settings screens reveal the substrate to admins:
  - `Settings → Technical → Datasets` — list + form for `ir.dataset.blueprint`. The form has notebook tabs for Fields / Connections / Group By / Sort plus a Live SQL Preview pane that recompiles on every save. Draft/Locked state machine.
  - `Settings → Technical → Metrics` — read-only browser over the in-memory metric registry; each row has a "Run sample" button.
- **Programmatic entry points for other modules:**
  - `@api.metric("namespace.key", spec=..., result_mode="scalar"|"table")` — register a frozen metric (Phase 1: dataset engine; Phase 2: plan + formula engines too).
  - `env.dispatch(Command("ede.metric.run", payload={"key": ..., "params": {...}}))` — invoke any registered metric.
  - `env.dispatch(Command("ede.dataset.run", payload={"key": ..., "params": {...}}))` — invoke a Blueprint dataset.
  - `params["root_filter"]` — caller-injected extra domain merged into the spec's WHERE; this is the integration point consumers (reports, dashboards) use to apply period/data filters.
  - Lifecycle events: `@api.on_event("ede.metric.computed")`, `"ede.dataset.computed"` — fired after a successful run; subscribers (e.g. webhook dispatcher) react.
- **HTTP** — `POST /api/dataset/<key>/run`, `POST /api/metric/<key>/run`, `GET /api/dataset/_list`, `GET /api/metric/_list`, `POST /api/dataset/_preview`. All return the canonical `DatasetResult` JSON.
- **Phase 3 adds** SSE / WebSocket streaming endpoints for live re-fetch, webhook outbound delivery, and Excel / CSV / Parquet exports.
- **Future internal MCP server** — will enumerate the metric / dataset registry as MCP tools and dispatch through the same command surface. No substrate changes required when MCP lands; the contract is designed so the MCP adapter is a thin wrapper.
- **Integration boundary** — the engine PRODUCES `DatasetResult` JSON + `ede.{dataset,metric}.computed` events. It CONSUMES `ir.model` / `ir.model.field` registry from `foundation.customization` (so Blueprint pickers populate from the live model graph) and `foundation.jobs` for webhook delivery (Phase 3).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Code author]                              [Admin in browser]
──────────────                             ─────────────────
@api.metric("sales.pipeline_value", ...    Settings → Datasets → New
                spec={...})                    fill model + fields + joins + groups
       │                                       form auto-renders live SQL
       │                                            │
       ▼                                            ▼ on save
in-memory metric registry              ir.dataset.blueprint (DB row)
       │                                            │
       └──────────────► canonical JSON spec ◄───────┘
                                  │
                                  ▼
                       DatasetCompiler.build(spec)
            13-step pipeline: filter merge → flatten connections → access check
            → base query → select terms → filters → record rules → group by
            → sort → limit/offset → tenant scope → CompiledQuery
                                  │
                                  ▼ executes via SQLAlchemy
                       DatasetResult (universal JSON contract)
                                  │
                  ┌───────────────┼────────────────┐
                  ▼               ▼                ▼
            foundation.       foundation.      foundation.
            reporting         dashboard        document
            (grid)            (KPIs+widgets)  (DML <rows>)

Substrate engines (src/ede/core/engines/):
  dataset/     — compiler, field resolver, connection builder, domain compiler
  metric/      — registry, executor, dataset/plan/formula engines, per-run cache, DAG
  formula/     — safe AST evaluator (shared by metric formulas AND DML <var formula="">)
  chip/        — toolbar contract + auto-injecting providers (date / comparison)
  period/      — period resolver with per-line date_scope (the proven fix)
  integration/ — JSON contract types, runner, HTTP routes, SSE/WS, webhook, export

Foundation shell (src/ede/foundation/dataset/):
  models/      — ir.dataset.blueprint + 5 child models (fields/connections/checks/groups/sort)
  views/       — Blueprint form + admin browsers
  controllers/ — thin HTTP wiring on top of engine commands
  data/        — RBAC seed + menus
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.dataset.blueprint` | Root Blueprint record — `name`, `key`, `description`, `model_id → ir.model`, `model_alias`, `state` (draft/locked), `active`, 3 compute-runtime fields (`dataset_spec_json` / `dataset_sql_preview` / `last_compile_error`) | [blueprint.py](../src/ede/foundation/dataset/models/blueprint.py) |
| `ir.dataset.blueprint.field` | Projected field row — `blueprint_id`, `sequence`, `connection_alias`, `field_id → ir.model.field`, `output_alias`, `aggregate` (sum/avg/min/max/count/count_distinct) | [blueprint_field.py](../src/ede/foundation/dataset/models/blueprint_field.py) |
| `ir.dataset.blueprint.connection` | JOIN spec — `blueprint_id`, `parent_connection_id` (self-ref nested joins), `alias`, `model_id`, `join_type` (left/inner), `filter_domain` | [blueprint_connection.py](../src/ede/foundation/dataset/models/blueprint_connection.py) |
| `ir.dataset.blueprint.connection.check` | Per-connection join check — `connection_id`, `lhs_alias`, `lhs_field_id`, `rhs_field_id` (defaults to target's `record_uuid`) | [blueprint_check.py](../src/ede/foundation/dataset/models/blueprint_check.py) |
| `ir.dataset.blueprint.group` | GROUP BY clause — `blueprint_id`, `sequence`, `connection_alias`, `field_id` | [blueprint_group.py](../src/ede/foundation/dataset/models/blueprint_group.py) |
| `ir.dataset.blueprint.sort` | ORDER BY clause — `blueprint_id`, `sequence`, `connection_alias`, `field_id`, `direction` (asc/desc) | [blueprint_sort.py](../src/ede/foundation/dataset/models/blueprint_sort.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `DatasetCompiler` | 13-step JSON spec → SQLAlchemy Core `select()`; single-table Phase 1 scope; every aggregate; GROUP BY validation; filter via existing kernel `_build_where_clause`; `_apply_active_test` via kernel `apply_active_filter` | [src/ede/core/engines/dataset/compiler.py](../src/ede/core/engines/dataset/compiler.py) |
| `FieldResolver` | `model_key + field_name` → `ResolvedField` against live model graph; gates `PROJECTABLE_FIELD_TYPES` (rejects O2M/M2M) | [src/ede/core/engines/dataset/field_resolver.py](../src/ede/core/engines/dataset/field_resolver.py) |
| `parse_select_expression` / `parse_group_expression` / `parse_sort_expression` / `parse_check_token` | 4 regex parsers + 3 typed parsed-result dataclasses (ParsedField / ParsedSort / ParsedCheck); inner-whitespace rejected by design | [src/ede/core/engines/dataset/expressions.py](../src/ede/core/engines/dataset/expressions.py) |
| `Metric` (frozen dataclass) + `_REGISTRY` + `register` / `get_metric` / `all_metrics` | In-memory metric registry with invariant validation at registration time (duplicate key, multi-engine, scalar-without-value_field, formula-non-scalar, plan-without-post_process) | [src/ede/core/engines/metric/base.py](../src/ede/core/engines/metric/base.py) + [registry.py](../src/ede/core/engines/metric/registry.py) |
| `@api.metric("key")` class-decorator | Harvests class attrs into `Metric` + registers | [src/ede/core/engines/metric/decorator.py](../src/ede/core/engines/metric/decorator.py) — re-exported from [src/ede/core/api.py](../src/ede/core/api.py) |
| `MetricExecutor.run` | Param coercion + `{{placeholder}}` substitution + **RBAC gate via `authorize_fn` (defaults to soft-imported `AuthorizationService.require(model_key, "read")` firing BEFORE SQL emission)** + compile + execute + shape into universal contract | [src/ede/core/engines/metric/executor.py](../src/ede/core/engines/metric/executor.py) |
| `DatasetResult` / `DatasetMeta` / `SchemaColumn` `TypedDict`s + `ColumnKind` enum + `CONTRACT_VERSION` + `make_meta()` | Universal JSON contract (Phase 1 of platform/06) | [src/ede/core/engines/integration/contract.py](../src/ede/core/engines/integration/contract.py) |
| `run_metric(env, key, params)` | Central dispatch; recasts registry `KeyError` to typed `DatasetSpecError` | [src/ede/core/engines/integration/runner.py](../src/ede/core/engines/integration/runner.py) |
| `safe_eval_number(expr, scope, function_set_name)` | Shared safe AST evaluator (Phase 2). Closed AST + function whitelist; same impl used by `foundation.document` Phase 2 `<var formula="..."/>`. | [src/ede/core/engines/formula/safe_eval.py](../src/ede/core/engines/formula/safe_eval.py) + [functions.py](../src/ede/core/engines/formula/functions.py) |
| `execute_formula(metric, params, resolve_dep_fn)` | Phase 2 formula engine. Substitutes `{{dep_key}}` → scalar, calls safe_eval_number. Runtime cycle guard via `params["__metric_stack__"]`. | [src/ede/core/engines/metric/formula_engine.py](../src/ede/core/engines/metric/formula_engine.py) |
| `execute_plan(metric, params, execute_spec_fn)` | Phase 2 plan engine. Runs each spec via caller-supplied executor, merges via `metric.post_process(result_sets, params)`. | [src/ede/core/engines/metric/plan_engine.py](../src/ede/core/engines/metric/plan_engine.py) |
| `check_no_cycle_introduced(candidate, registry_view)` | Phase 2 DAG cycle detection at `register()` time. Raises `FormulaCycleError` (subclass of `MetricRegistrationError`) with the offending cycle path. | [src/ede/core/engines/metric/dag.py](../src/ede/core/engines/metric/dag.py) |
| `get_or_execute` + `cache_key` + `initialize_cache` | Phase 2 per-run cache. SHA-256 key from `(metric.key, engine, mode, lang, params)` with `__metric_*` keys stripped; deep-copy in AND out; `METRIC_CACHE_ENABLED` toggle. | [src/ede/core/engines/metric/cache.py](../src/ede/core/engines/metric/cache.py) |
| `ChipDispatcher` / `ToolbarRegistry` / `PeriodResolver` / `StreamRegistry` / `WebhookDispatcher` / exporters | Phase 3 interactive + integration primitives (chip · period · SSE/WS streaming · webhook · csv/xlsx/parquet export). | [chip/](../src/ede/core/engines/chip/) · [period/](../src/ede/core/engines/period/) · [integration/streaming/](../src/ede/core/engines/integration/streaming/) · [integration/export/](../src/ede/core/engines/integration/export/) |
| `QueryCache` + `build_cache_key` + `get_default_query_cache` | Phase 3 cross-request query cache in front of `run_dataset`/`run_metric`. In-memory + Redis backends; tenant-scoped key; TTL; opt-in via `DATASET_QUERY_CACHE_*`; stamps `meta.query_cache_hit`. | [src/ede/core/engines/integration/cache/](../src/ede/core/engines/integration/cache/) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.metric.run` (Stage 1B ✅) | Programmatic dispatch via `run_metric(env, key, params)`; HTTP `/api/metric/<key>/run` Stage 2 | Resolves metric by registry key → RBAC check on base model → param coercion → `{{placeholder}}` substitution → compile → execute → returns `DatasetResult` |
| `ede.dataset.run` (Stage 2) | HTTP `/api/dataset/<key>/run`, programmatic dispatch | Resolves Blueprint by key → compiles spec → executes → returns `DatasetResult` |
| `ede.create` / `ede.update` / `ede.delete` on `ir.dataset.blueprint*` (Stage 2) | Standard CRUD via the admin form | Cascades to children; triggers compute-runtime recompile of `dataset_spec_json` + `dataset_sql_preview` |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.dataset.computed` | After a Blueprint dataset successfully executes | Webhook dispatcher (Phase 3); audit logger |
| `ede.metric.computed` | After a registered metric successfully executes | Same as above; dashboard insight surface |
| `ir.dataset.blueprint.locked` | Admin runs `action_lock` | Audit log |
| `ir.dataset.blueprint.unlocked` | Admin runs `action_unlock` | Audit log + `revision` bump |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/dataset/<key>/run` | Execute a Blueprint dataset | `src/ede/foundation/dataset/controllers.py` |
| `POST /api/metric/<key>/run` | Execute a registered metric | same |
| `GET /api/dataset/_list` | List Blueprint datasets visible to caller | same |
| `GET /api/metric/_list` | List registered metrics visible to caller | same |
| `POST /api/dataset/_preview` | Compile a draft spec (used by Blueprint live SQL preview) | same |
| `GET /api/dataset/<key>/stream` (Phase 3) | SSE channel for live re-fetch | `src/ede/foundation/dataset/streaming.py` |
| `POST /api/dataset/<key>/export?format=xlsx\|csv\|parquet` (Phase 3) | Bulk export | same |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.ir.dataset.blueprint.update` | Blocks edits when `state="locked"` (admin must `action_unlock` first). |
| `pre.ir.dataset.blueprint.delete` | Blocks deletion when blueprint is referenced from a webhook (Phase 3). |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
`ir.dataset.blueprint.state` (Phase 1):

```
draft ──(action_lock; spec must compile cleanly)──► locked
locked ──(action_unlock; bumps revision)──────────► draft
```

Locking is a convention — the compiler itself does not enforce state, only the pre-update hook does. Programmatic invocation continues to work regardless of state (used by the demo-data loader).
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds across all phases. Empty rows are fine; missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` in [src/ede/foundation/settings.py](../src/ede/foundation/settings.py): `"dataset"` (✅ active).
- Manifest `depends`: `["foundation.base"]` (✅ shipped).
- Data files loaded at boot: `data/dataset_rbac_roles.xml` + `data/ir.rbac.permission.csv` + `views/blueprint_views.xml` + `data/dataset_menus.xml`.
- Demo file (opt-in via `--with-demo=foundation.dataset`): `demo/demo_blueprints.xml`.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `DATASET_DEFAULT_QUERY_TIMEOUT_SECONDS` | int | `30` | `DATASET_DEFAULT_QUERY_TIMEOUT_SECONDS` | Hard timeout per compiled SQL execution. Raises `DatasetTimeoutError`. |
| `DATASET_MAX_RESULT_ROWS` | int | `100000` | `DATASET_MAX_RESULT_ROWS` | Engine-level row cap. Raises `DatasetRowLimitExceeded`. |
| `METRIC_CACHE_ENABLED` | bool | `True` | `METRIC_CACHE_ENABLED` | Phase 2 — per-run metric cache toggle. |
| `DATASET_SSE_DEBOUNCE_MS` | int | `350` | `DATASET_SSE_DEBOUNCE_MS` | Phase 3 — debounce window for chip-change-driven re-fetch. |
| `DATASET_EXPORT_CHUNK_SIZE` | int | `10000` | `DATASET_EXPORT_CHUNK_SIZE` | Phase 3 — streaming export chunk size. |
| `DATASET_WEBHOOK_HMAC_HEADER` | str | `"X-EDE-Signature"` | `DATASET_WEBHOOK_HMAC_HEADER` | Phase 3 — outbound webhook signature header. |
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
| `data/dataset_rbac.csv` | 4 RBAC roles: `ir.dataset.viewer`, `ir.dataset.executor`, `ir.dataset.author`, `ir.dataset.admin` |
| `data/dataset_menus.xml` | `Settings → Technical → Datasets` + `Settings → Technical → Metrics` menus |
| `data/dataset_phase3_rbac.csv` (Phase 3) | RBAC delta: `ir.dataset.webhook.admin`, `ir.dataset.streaming.subscriber` |
| `demo/demo_blueprints.xml` | 3 demo blueprints — partners-with-country, expiring-rates-30d, quote-lines-with-currency |
| `demo/demo_metrics_seed_data.xml` | Backing demo records the demo metrics aggregate over (leans on `foundation.base` + `logistics.sales-crm` demo data) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Substrate Core + Blueprint Low-Code UI | ✅ **Delivered 2026-05-13** | [phase-1-implementation.md](../roadmap/foundation/dataset/phase-1-implementation.md) |
| Phase 2 | Plan + Formula + Per-Run Cache | ✅ **Delivered 2026-05-13** | [phase-2-implementation.md](../roadmap/foundation/dataset/phase-2-implementation.md) |
| Phase 3 | Chip + Period + Streaming + Bulk Export | ✅ **Delivered 2026-07-07** (WS-D15…D21; 70 pytest + 6 vitest cases) — chip · period · SSE/WS streaming · webhook (real-TLS e2e verified) · csv/xlsx/parquet export · Redis-backed cross-request query cache · live `dataset.stream` React client action | [phase-3-implementation.md](../roadmap/foundation/dataset/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| **Stage 1A substrate** (commit `0c495f0`, 2026-05-13) — engine foundational layer: typed exception classes, regex parsers, FieldResolver | _none — engine-layer only_ | [errors.py](../src/ede/core/engines/dataset/errors.py) · [expressions.py](../src/ede/core/engines/dataset/expressions.py) · [field_resolver.py](../src/ede/core/engines/dataset/field_resolver.py) · 45 tests | [phase-1-implementation.md WS-D1](../roadmap/foundation/dataset/phase-1-implementation.md) |
| **Stage 1B substrate** (commit `d3a4b82`, 2026-05-13) — DatasetCompiler 13-step pipeline; metric registry + `@api.metric` decorator; `MetricExecutor.run()` with RBAC + active_test gates; `DatasetResult` universal JSON contract; `run_metric(env, key, params)` central dispatch | _none — engine-layer only_ | [compiler.py](../src/ede/core/engines/dataset/compiler.py) · [metric/](../src/ede/core/engines/metric/) · [integration/contract.py](../src/ede/core/engines/integration/contract.py) · [integration/runner.py](../src/ede/core/engines/integration/runner.py) · 10 tracer-bullet + RBAC + active_test tests | [phase-1-implementation.md WS-D1/D2/D3/D9](../roadmap/foundation/dataset/phase-1-implementation.md) |
| **Stage 2 foundation shell** (2026-05-13) — 6 ORM models + Alembic migration `ce27d6cc1a91` applied to live Postgres tenant `devqa`; manifest declares all data + demo files; HTTP controllers ship 4 routes (`/api/dataset/ping`, `/api/dataset/run`, `/api/metric/<key>/run`, `/api/metric/list`) | `ir.dataset.blueprint` + 5 children | [models/](../src/ede/foundation/dataset/models/) · [controllers.py](../src/ede/foundation/dataset/controllers.py) · [migrations/versions/ce27d6cc1a91_foundation_dataset_blueprint_init.py](../src/ede/foundation/dataset/migrations/versions/ce27d6cc1a91_foundation_dataset_blueprint_init.py) | [phase-1-implementation.md WS-D3 + WS-D4](../roadmap/foundation/dataset/phase-1-implementation.md) |
| **Blueprint admin form** (2026-05-13) — `<FormView>` with notebook tabs (Fields / Connections / Groups / Sort / SQL Preview) + 5 inline child list/form views + `SqlPreviewField.tsx` widget registered in field registry; compute-runtime on save recomputes `dataset_spec_json` + `dataset_sql_preview` + `last_compile_error` | `ir.dataset.blueprint*` | [views/blueprint_views.xml](../src/ede/foundation/dataset/views/blueprint_views.xml) · [SqlPreviewField.tsx](../src/frontend/src/workspace/views/fields/SqlPreviewField.tsx) · [registry.ts](../src/frontend/src/workspace/views/fields/registry.ts) | [phase-1-implementation.md WS-D5](../roadmap/foundation/dataset/phase-1-implementation.md) |
| **RBAC + menus + demo + metrics** (2026-05-13) — 4 dataset roles (`dataset_viewer` / `_author` / `_publisher` / `_admin`) + 24 `ir.rbac.permission` grants on the 6 models + `Settings → Datasets → Blueprints` menu + 2 demo metrics (`base.partner_count`, `base.organization_count`) registered via `@api.metric` + 1 demo blueprint (`demo.partner_listing`) | All 6 + `ir.rbac.role` / `.permission` | [data/dataset_rbac_roles.xml](../src/ede/foundation/dataset/data/dataset_rbac_roles.xml) · [data/ir.rbac.permission.csv](../src/ede/foundation/dataset/data/ir.rbac.permission.csv) · [data/dataset_menus.xml](../src/ede/foundation/dataset/data/dataset_menus.xml) · [tools/metrics/base_metrics.py](../src/ede/foundation/dataset/tools/metrics/base_metrics.py) · [demo/demo_blueprints.xml](../src/ede/foundation/dataset/demo/demo_blueprints.xml) | [WS-D6 / D7 / D8](../roadmap/foundation/dataset/phase-1-implementation.md) |
| **Acceptance tests** (2026-05-13) — 6 module-load tests proving app registered, all 6 models in registry, 3 compute fields declared, 2 demo metrics registered, 4 HTTP routes wired | n/a | [src/tests/foundation/dataset/test_module_loads.py](../src/tests/foundation/dataset/test_module_loads.py) | [phase-1-implementation.md WS-D9](../roadmap/foundation/dataset/phase-1-implementation.md) |
| **Phase 2: Shared safe AST evaluator** (2026-05-13) — closed AST-node whitelist (Constant / UnaryOp / BinOp / Compare / BoolOp / IfExp / Name / Call-to-whitelisted-functions) + closed function whitelist split into `numeric` (math + aggregate + logic) and `full` (numeric ∪ date + format) sets; attribute access / subscript / lambda / comprehension / keyword-args / unknown-function all rejected with typed `FormulaEvalError`; same module hardens `foundation.dataset` formulas and `foundation.document` Phase 2 variables | n/a — kernel-shared engine | [formula/safe_eval.py](../src/ede/core/engines/formula/safe_eval.py) · [formula/functions.py](../src/ede/core/engines/formula/functions.py) · 22 tests | [phase-2-implementation.md WS-D12](../roadmap/foundation/dataset/phase-2-implementation.md) |
| **Phase 2: Formula engine + DAG cycle detection** (2026-05-13) — `engine="formula"` + `depends_on` + `expr="{{a}} - {{b}}"`; DAG cycle detection runs at `register()` time so bad formulas fail at boot; `result_mode="table"` rejected on formula metrics; runtime cycle guard via `params["__metric_stack__"]` as the secondary fail-safe | n/a — engine-layer only | [metric/formula_engine.py](../src/ede/core/engines/metric/formula_engine.py) · [metric/dag.py](../src/ede/core/engines/metric/dag.py) · 9 formula + 7 DAG tests | [phase-2-implementation.md WS-D11](../roadmap/foundation/dataset/phase-2-implementation.md) |
| **Phase 2: Plan engine + post_process contract** (2026-05-13) — multi-spec list + pure-Python `post_process(result_sets, params)` callable; `MetricExecutor._execute_plan` runs each spec through the same compile + RBAC path as the dataset engine; post_process exceptions surface as `MetricExecutionError` with the original chain preserved | n/a — engine-layer only | [metric/plan_engine.py](../src/ede/core/engines/metric/plan_engine.py) · 6 tests | [phase-2-implementation.md WS-D10](../roadmap/foundation/dataset/phase-2-implementation.md) |
| **Phase 2: Per-run deep-copy cache** (2026-05-13) — fresh `params["__metric_cache__"]` per top-level `run_metric()` call; deep-copy in AND out so caller mutations cannot poison the cached copy; `meta.cache_hit` flag set per call; toggleable via `METRIC_CACHE_ENABLED` setting (defaults True) | n/a — engine-layer only | [metric/cache.py](../src/ede/core/engines/metric/cache.py) · 6 tests | [phase-2-implementation.md WS-D13](../roadmap/foundation/dataset/phase-2-implementation.md) |
| **Phase 2: 3 new demo metrics** (2026-05-13) — `base.entity_total` (formula: `{{base.partner_count}} + {{base.organization_count}}`); `base.partner_to_org_ratio` (formula with `iff()` divide-by-zero guard); `base.entity_summary` (plan: 2 specs merged into a 2-row table) | n/a — registered via `@api.metric` | [tools/metrics/base_metrics.py](../src/ede/foundation/dataset/tools/metrics/base_metrics.py) | [phase-2-implementation.md WS-D14](../roadmap/foundation/dataset/phase-2-implementation.md) |
| **Phase 3: Chip system** (2026-07-03, WS-D15) — `@api.chip(key, scope, widget)` + frozen `Chip` + scope×widget×output matrix validated at registration (`ChipContractError`); `ChipDispatcher.compute_deltas` merges per-chip `{params, root_filter, flags}` in `(sequence, key)` order; required-chip enforcement | _none — engine-layer only_ | [core/engines/chip/](../src/ede/core/engines/chip/) · 19 tests | [phase-3-implementation.md WS-D15](../roadmap/foundation/dataset/phase-3-implementation.md) |
| **Phase 3: Auto-injecting chip providers** (2026-07-03, WS-D16) — `ToolbarRegistry` builds date_asof / date_range / comparison chips from a definition's `date_filter_mode` + `comparison_enabled` (no XML) | _none — engine-layer only_ | [core/engines/chip/auto/](../src/ede/core/engines/chip/auto/) · 9 tests | [phase-3-implementation.md WS-D16](../roadmap/foundation/dataset/phase-3-implementation.md) |
| **Phase 3: Period resolver w/ per-line `date_scope`** (2026-07-03, WS-D17) — `PeriodResolver.build_periods` + `resolve_for_line(line, base_period)`; five scopes (`same_period` / `previous_period` / `previous_year` / `fy_to_date` / `inception_to_date`≡`cumulative`) resolvable on the SAME report; FY-boundary aware | _none — engine-layer only_ | [core/engines/period/](../src/ede/core/engines/period/) · 15 tests | [phase-3-implementation.md WS-D17](../roadmap/foundation/dataset/phase-3-implementation.md) |
| **Phase 3: SSE + WebSocket streaming** (2026-07-03, WS-D18) — `StreamRegistry` (channel keyed on `(consumer_type, key, params_hash, tenant_id)` + debounced re-fetch, `DATASET_SSE_DEBOUNCE_MS`), `sse_stream` generator, `WebSocketStreamSession`; routes `GET /api/dataset/<key>/stream`, `GET /api/report/<key>/stream`, `POST …/refresh` | _none — engine + routes_ | [core/engines/integration/streaming/](../src/ede/core/engines/integration/streaming/) · [controllers.py](../src/ede/foundation/dataset/controllers.py) · 9 tests | [phase-3-implementation.md WS-D18](../roadmap/foundation/dataset/phase-3-implementation.md) |
| **Phase 3: Webhook outbound** (2026-07-03, WS-D19) — `ir.dataset.webhook` (migration `be240def5e23`, verified-applied on fresh Postgres) + `WebhookDispatcher` (HMAC-SHA256, `@api.background_job` delivery, https-only guard) firing on `ede.dataset.computed` / `ede.metric.computed`; 5 RBAC perms + demo webhook | `ir.dataset.webhook` | [models/webhook.py](../src/ede/foundation/dataset/models/webhook.py) · [services/webhook_dispatcher.py](../src/ede/foundation/dataset/services/webhook_dispatcher.py) · 6 tests | [phase-3-implementation.md WS-D19](../roadmap/foundation/dataset/phase-3-implementation.md) |
| **Phase 3: Bulk exporters** (2026-07-03, WS-D20; Parquet closed 2026-07-07) — CSV (stdlib) / Excel (openpyxl) / Parquet (pyarrow, now a **core dep**) writers over the `DatasetResult` envelope with per-kind coercion (datetime→ISO, decimal→float, ref→uuid str); `POST /api/dataset/<key>/export?format=` | _none — engine + route_ | [core/engines/integration/export/](../src/ede/core/engines/integration/export/) · 7 tests (all run, incl. Parquet) | [phase-3-implementation.md WS-D20](../roadmap/foundation/dataset/phase-3-implementation.md) |
| **Phase 3: Cross-request query cache** (2026-07-07) — result-envelope cache in front of `run_dataset`/`run_metric`, keyed by `(consumer_type, key, params, tenant_id, lang)` with TTL; in-memory (deepcopy + lazy expiry) default + Redis backend (pickle, namespaced, degrades to a miss on Redis error); opt-in via `DATASET_QUERY_CACHE_*`; stamps `meta.query_cache_hit` | _none — engine-layer only_ | [core/engines/integration/cache/](../src/ede/core/engines/integration/cache/) · 21 tests | [phase-3-implementation.md WS-D21](../roadmap/foundation/dataset/phase-3-implementation.md) |
| **Phase 3: Real-TLS webhook delivery e2e** (2026-07-07) — hermetic end-to-end proof: loopback HTTPS server (self-signed cert) + real httpx transport driving `deliver_webhook`; server re-verifies the HMAC over the received bytes. No external inspector dependency | `ir.dataset.webhook` | [src/tests/foundation/dataset/test_webhook_delivery_e2e.py](../src/tests/foundation/dataset/test_webhook_delivery_e2e.py) · 2 tests | [phase-3-implementation.md WS-D19](../roadmap/foundation/dataset/phase-3-implementation.md) |
| **Phase 3: Live `dataset.stream` client action** (2026-07-07) — React client action that opens the dataset SSE channel, renders the `DatasetResult` (schema + rows), re-subscribes on period/comparison chip change, and drives the live `Refresh` push; enriched `demo.partner_listing` blueprint (code/name/email → real rows) | consumes `demo.partner_listing` | [foundation/dataset/frontend/](../src/ede/foundation/dataset/frontend/) · [dataset_menus.xml](../src/ede/foundation/dataset/data/dataset_menus.xml) · [DatasetStreamPage.test.tsx](../src/frontend/src/managers/DatasetStreamPage.test.tsx) · 6 vitest | [phase-3-implementation.md WS-D18/D21](../roadmap/foundation/dataset/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Phase 1 Enhancement 01 — keystroke-live recompile of `dataset_sql_preview` (today: on-save recompute) | 🟢 Low backlog | [WS-D5](../roadmap/foundation/dataset/phase-1-implementation.md) |
| Phase 1 Enhancement 02 — full Metrics admin browser (today: `GET /api/metric/list` only) | 🟢 Low backlog | [WS-D6](../roadmap/foundation/dataset/phase-1-implementation.md) |
| Phase 1 Enhancement 03 — 2 additional domain-coupled demo blueprints (`sales.pipeline_by_stage` + `pricing.rate_validity`); today: 1 minimal demo + 2 demo metrics | 🟢 Low backlog | [WS-D8](../roadmap/foundation/dataset/phase-1-implementation.md) |
| Pre-existing dataset-baseline drift — the 6 `ir_dataset_blueprint*` migrations predate current metadata-builder conventions (uq_→uidx_, per-column indexes, audit-FK handling, VARCHAR→Text); every postgres autogenerate re-proposes it. The webhook migration excludes this noise; a separate baseline reconciliation is warranted | 🟠 Backlog | — |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
*Will be populated as integration learnings emerge from first consumers (`foundation.reporting`, `foundation.document`).*

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- **Phase 1** — `ce27d6cc1a91` creates the 6 `ir_dataset_blueprint*` tables (+ follow-ups `f51f9b96bab9`, `f7e837b6c539`, `f27442d33487`).
- **Phase 3** — `be240def5e23` creates `ir_dataset_webhook`. Generated via `ede migrate generate --app foundation.dataset` against a fresh Postgres reference (the sqlite reference is currently unbuildable at head due to an unrelated non-batch `ALTER` migration in `foundation.communication`); the pre-existing dataset-baseline drift the postgres autogenerate re-proposes on the blueprint tables is intentionally excluded. Verified-applied on a fresh Postgres tenant.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `ir.dataset.viewer` | Read `ir.dataset.blueprint*`; list datasets/metrics |
| `ir.dataset.executor` | Viewer + execute datasets/metrics via HTTP routes |
| `ir.dataset.author` | Executor + CRUD on `ir.dataset.blueprint*` (draft state only) |
| `ir.dataset.admin` | Author + lock/unlock + ad-hoc `/api/dataset/_preview` |
| `dataset_publisher` (Phase 3) | + read `ir.dataset.webhook` |
| `dataset_admin` (Phase 3) | + create / update / delete `ir.dataset.webhook` |
| `dataset_viewer` (Phase 3) | + subscribe to dataset streams (`ir.dataset.stream.read`) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.reporting`](./foundation-reporting.md) — first consumer; statement-mode grid renderer.
- [`foundation.dashboard`](./foundation-dashboard.md) — KPI + widget consumer.
- [`foundation.document`](./foundation-document.md) — DML `<rows datasource>` resolves through this registry; `<var formula="...">` reuses the shared AST evaluator.
- [`foundation.customization`](./foundation-customization.md) — Blueprint pickers populate from `ir.model` / `ir.model.field` registry mirror.
- [`foundation.jobs`](./foundation-jobs.md) — Phase 3 webhook outbound delivery runs as a `@api.background_job`.
- [`foundation.presentation`](./foundation-presentation.md) — Blueprint admin form is a normal `<FormView>`.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-07-03. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
