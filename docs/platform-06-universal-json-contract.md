<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Universal Result JSON Contract — Implementation Docs

**Module:** `ede.core.engines.integration.contract` (kernel cross-cutting type)
**Roadmap:** [roadmap/platform/06-universal-json-contract.md](../roadmap/platform/06-universal-json-contract.md)
**Status:** 🟡 In Progress (`DatasetResult` v1 shipped 2026-05-13; `ReportResult` / `WidgetResult` / `KpiResult` + JSON schema fixture pending)
**Layer:** Platform-wide

> Source of truth is the roadmap. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A single canonical JSON contract that every reporting / analytics / KPI / export engine in EDE produces. Four `TypedDict` shapes — `DatasetResult`, `ReportResult`, `WidgetResult`, `KpiResult` — share a common `meta` envelope and a type-aware `schema` descriptor on every column. Consumers (React grid, document `<rows datasource>`, HTTP / SSE / WebSocket / webhook / export adapters, future MCP server) read this contract and never need engine-specific renderers.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Without a shared shape, every engine emits its own JSON dialect and every consumer writes engine-specific renderers — a combinatorial mess (4+ producers × N consumers). The reference reporting stack we evaluated suffers this exact problem: different shapes for `dataset.run`, `metric.run`, `report.run`; no schema descriptor on columns so renderers guess types from values.

This contract:
1. Puts a typed `schema` descriptor on every column (`kind` ∈ {`ref`, `decimal`, `integer`, `enum`, `date`, `datetime`, `json`, `char`, `boolean`} + format hint). Renderers never guess.
2. Uses one shape that scales scalar → table → series → grouped grid via optional fields, not a discriminated union of incompatible shapes.
3. Carries a bounded `meta` envelope (tenant + principal + params + lang + cache-hit + timing) sufficient for audit/trace without parsing the payload.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Engine authors:** every command/HTTP route that returns "tabular data with type metadata" returns one of the four shapes. Validated by JSON schema in tests.
- **Consumer authors (frontend / external):** read `meta.contract_version` first; switch behaviour only across version boundaries. Within a version, unknown additive fields are ignored.
- **MCP tool builders (future):** the contract is what each enumerated MCP tool returns. No engine-specific adapter required.
- **Audit consumers:** every payload's `meta` block contains everything needed for an audit trail entry — no payload parsing required.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
src/ede/core/engines/integration/
├── contract.py            ← TypedDict definitions for the four shapes
├── contract_schema.json   ← JSON schema fixture (generated from TypedDicts at boot)
└── runner.py              ← central dispatch — ensures every command call returns a contract-valid payload

Type family:
  DatasetResult   ← base; produced by ede.dataset.run + ede.metric.run
  ReportResult    ← extends DatasetResult.meta + adds header[] (periods) + body[] (typed row entries)
  WidgetResult    ← per-widget; appears inside list[WidgetResult] from ede.dashboard.run
  KpiResult       ← scalar + threshold; produced by ede.kpi.evaluate
```

Every payload carries `meta.contract_version` (1 in v1). Adding optional fields → no version bump. Renaming / removing / semantic-changing a field → version bump with documented bump checklist in the roadmap.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none_ | The contract is a Python type, not an ORM model. | n/a |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `DatasetResult` TypedDict | Top-level payload shape: `meta` (envelope) + `schema` (per-column descriptor) + `rows` (list[dict]) + optional `value` (scalar projection). | `src/ede/core/engines/integration/contract.py` |
| `DatasetMeta` TypedDict | Bounded envelope: `contract_version`, `tenant_id`, `principal_uid`, `metric_key`, `params`, `lang`, `started_at_utc`, `completed_at_utc`, `duration_ms`, `cache_hit`, `row_count`. | `src/ede/core/engines/integration/contract.py` |
| `SchemaColumn` TypedDict | Per-column descriptor: `name`, `kind` (`ColumnKind` literal), optional `model_key` (for `ref`), optional `enum_values`, optional `format`. Renderers consume this — never guess types from values. | `src/ede/core/engines/integration/contract.py` |
| `ColumnKind` Literal | Closed enum of column kinds: `ref`, `decimal`, `integer`, `enum`, `date`, `datetime`, `json`, `char`, `boolean`. | `src/ede/core/engines/integration/contract.py` |
| `make_meta()` + `now_iso_utc()` | Factory that builds a `DatasetMeta` from `env` + timing observations. Single source of meta-shape truth so future v2 bumps touch one file. | `src/ede/core/engines/integration/contract.py` |
| `run_metric(env, key, params)` | Central dispatch — recasts `KeyError` to `DatasetSpecError`, returns a contract-valid `DatasetResult`. The one entrypoint every HTTP/SSE/MCP adapter calls. | `src/ede/core/engines/integration/runner.py` |
| _Phase 1 remainder pending_ | JSON schema fixture generator (validates every reporting endpoint's response in CI). | (planned) `src/ede/core/engines/integration/contract_schema.json` |
| _Phase 2 pending_ | `ReportResult` TypedDict — extends `meta` + adds `header[]` (periods) + `body[]` (typed row entries). | (planned) `src/ede/core/engines/integration/contract.py` |
| _Phase 3 pending_ | `WidgetResult` + `KpiResult` TypedDicts. | (planned) `src/ede/core/engines/integration/contract.py` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | The contract defines the *return shape* of every reporting/analytics command. The commands themselves live in their respective foundation modules. | n/a |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ | The contract is a type, not a runtime. | n/a |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none_ | The contract is consumed by every reporting endpoint but isn't an endpoint itself. | n/a |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
*Not applicable — type contract, no runtime state.*
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- No `ACTIVE_MODULES` entry — pure type contract.
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
| Phase 1 | `DatasetResult` v1 | 🟡 | [roadmap/platform/06-universal-json-contract.md](../roadmap/platform/06-universal-json-contract.md) — TypedDicts + `ColumnKind` enum + `CONTRACT_VERSION = 1` + `make_meta()` factory shipped 2026-05-13 via `foundation.dataset` Stage 1B; JSON schema fixture generator pending |
| Phase 2 | `ReportResult` v1 extending | 🔴 | Same file — lands with `foundation.reporting` Phase 1 (W3b) |
| Phase 3 | `WidgetResult` + `KpiResult` v1 | 🔴 | Same file — lands with `foundation.dashboard` Phase 1 (W5) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| `DatasetResult` / `DatasetMeta` / `SchemaColumn` TypedDicts | n/a | `src/ede/core/engines/integration/contract.py` | Phase 1 / Stage 1B |
| `ColumnKind` Literal closed enum (`ref` / `decimal` / `integer` / `enum` / `date` / `datetime` / `json` / `char` / `boolean`) | n/a | `src/ede/core/engines/integration/contract.py` | Phase 1 / Stage 1B |
| `CONTRACT_VERSION = 1` constant | n/a | `src/ede/core/engines/integration/contract.py` | Phase 1 / Stage 1B |
| `make_meta()` factory + `now_iso_utc()` helper — single source of meta-shape truth | n/a | `src/ede/core/engines/integration/contract.py` | Phase 1 / Stage 1B |
| `run_metric(env, key, params)` central dispatch — every HTTP / SSE / future-MCP adapter calls this and gets a contract-valid `DatasetResult` | n/a | `src/ede/core/engines/integration/runner.py` | Phase 1 / Stage 1B |
| Tracer-bullet end-to-end proof — smoke test asserts `meta.contract_version == 1`, every column declares a typed `kind`, and the payload shape matches the TypedDict | n/a | `src/tests/core/engines/dataset/test_tracer_bullet.py` | Phase 1 / Stage 1B |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| JSON schema fixture generator (`contract_schema.json`) + CI validator that every reporting endpoint's response matches the schema | 🟡 Phase 1 remainder | [roadmap/platform/06-universal-json-contract.md](../roadmap/platform/06-universal-json-contract.md) |
| `ReportResult` TypedDict — extends `DatasetMeta` + adds `header[]` (periods) + `body[]` (typed row entries) | 🔴 Phase 2 | [roadmap/foundation/reporting/phase-1-implementation.md](../roadmap/foundation/reporting/phase-1-implementation.md) |
| `WidgetResult` TypedDict — per-widget payload appearing inside `list[WidgetResult]` from `ede.dashboard.run` | 🔴 Phase 3 | [roadmap/foundation/dashboard/phase-1-implementation.md](../roadmap/foundation/dashboard/phase-1-implementation.md) |
| `KpiResult` TypedDict — scalar + threshold from `ede.kpi.evaluate` | 🔴 Phase 3 | [roadmap/foundation/dashboard/phase-1-implementation.md](../roadmap/foundation/dashboard/phase-1-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
*Will be populated as engines + consumers integrate against v1.*

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
*Pre-build. Bump checklist for future v2 lives inline in the roadmap file.*
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ | The contract is universal — RBAC happens in each consumer's HTTP layer. |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`platform-05-engine-substrate`](./platform-05-engine-substrate.md) — establishes `engines/integration/` where the contract lives.
- [`foundation.dataset`](./foundation-dataset.md) — first producer; introduces `DatasetResult` v1.
- [`foundation.reporting`](./foundation-reporting.md) — introduces `ReportResult` v1.
- [`foundation.dashboard`](./foundation-dashboard.md) — introduces `WidgetResult` + `KpiResult` v1.
- [`foundation.document`](./foundation-document.md) — consumes `DatasetResult` via `<rows datasource="X">` binding.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-13. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
