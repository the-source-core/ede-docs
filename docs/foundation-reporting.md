<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Reporting Engine — Implementation Docs

**Module:** `foundation.reporting` (`src/ede/core/engines/report/` + `src/ede/foundation/reporting/` + `src/frontend/src/workspace/views/report/`)
**Roadmap:** [roadmap/foundation/reporting/](../roadmap/foundation/reporting/README.md)
**Status:** 🔴 Not Started (roadmap drafted 2026-05-12)
**Layer:** Foundation engine — consumes `foundation.dataset` substrate

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
The **on-screen reporting surface** — the Reports app in the app-switcher, the React grid component, and the orchestration layer that turns a `ReportDefinition` + chip selections into a `ReportResult` JSON payload. Three rendering modes: **statement** (lines × periods × metrics — the financial-report standard), **pivot** (rows × columns dynamic from a single table-mode metric), and **table** (single table-mode metric drives the grid directly).
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Domains needing tabular cross-period reports (margin by quarter, pipeline by stage, partner ledger, P&L) have nowhere to put them today — the existing kanban / list / form views render single-model records, not statement-style grids with periods as columns. `foundation.reporting` is the dedicated rendering layer.

The single most important architectural commitment: **per-line `date_scope`** from day one. A balance sheet's "Cash" line uses cumulative-to-date; "Current Period Revenue" uses within-period activity; "Retained Earnings" uses FY-to-date. The reference reporting stack we evaluated applied **one** `root_filter` per period uniformly to every line, a permanent architectural gap that left financial reports half-right. This module avoids that trap by making `date_scope` a first-class line attribute.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX** — Reports app in the app-switcher. Each report category gets a dynamic menu entry. The grid renders with sticky headers, frozen first column, comparison columns (when `comparison_enabled=True`), cell-level drill-down on click. Phase 2 adds editable cells for write-back and live re-fetch via SSE.
- **Authoring** — Domain modules author Report XML in their `data/` manifest using the `<Report>` / `<ReportLine>` / `<ReportMetric>` DSL.
- **Programmatic entry points:**
  - `env.dispatch(Command("ede.report.run", payload={"key": ..., "params": {...}}))` — execute a report by key.
  - `@api.on_event("ede.report.executed")` — react to report runs.
  - HTTP: `POST /api/report/<key>/run`, `GET /api/report/<key>/stream` (Phase 2 SSE), `POST /api/report/<key>/cell` (Phase 2 write-back).
- **Integration boundary** — PRODUCES `ReportResult` JSON. CONSUMES `foundation.dataset` (every metric flows through `ede.metric.run`), `foundation.presentation` (admin form), Phase 3 also consumes `foundation.document` (PDF snapshots), `foundation.email` + `foundation.jobs` (subscriptions).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Author]                  [foundation.reporting]                      [React webclient]
─────────                 ──────────────────────                      ──────────────────
Report XML in module's    ReportDefinition row in              ──►   Reports app
data/ manifest            ir.report.definition                        opens grid view
       │                          │                                          │
       ▼                          ▼                                          ▼
DslParser <Report>...      ReportRunner.run(def, params)            ReportGrid.tsx
parses to RenderPlan         1. ChipDispatcher.compute_deltas              │
                             2. PeriodResolver.build_periods                │
                             3. For each line × period:                    │
                                PeriodResolver.resolve_for_line             ▼
                                MetricExecutor.run                   sticky headers,
                             4. body_builder.assemble                frozen columns,
                                  │                                  drill-down on click,
                                  ▼                                  comparison columns
                          ReportResult JSON (universal contract)

Core engine:        src/ede/core/engines/report/
  runner.py         — orchestration
  body_builder.py   — statement / pivot / table modes
  writeback.py      — Phase 2: editable cells (external values)
  streaming.py      — Phase 2: SSE/WebSocket
  snapshot.py       — Phase 3
  fx.py             — Phase 3: multi-currency
  rbac.py           — Phase 3: row-level RBAC
  resolution.py     — definition resolution by (company, country, priority)

Foundation shell:   src/ede/foundation/reporting/
  models/           — ir.report.category + definition + line + metric
                      + Phase 2: external_value + pivot_config
                      + Phase 3: saved_filter + snapshot + subscription + drillthrough
  controllers.py    — HTTP wiring
  views/            — admin forms for managing definitions
  data/             — menus + RBAC seed
  demo/             — 3 demo reports

Frontend:           src/frontend/src/workspace/views/report/
  ReportView.tsx    — page wrapper
  ReportGrid.tsx    — statement-mode grid
  PivotGrid.tsx     — Phase 2 pivot
  ChipBar.tsx       — toolbar
  EditableCell.tsx  — Phase 2 write-back
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none yet — all 🔴 Not Started_ | Phase 1: `ir.report.category` (stable entry-point), `ir.report.definition` (body structure with `(company, country, priority)` resolution), `ir.report.definition.line` (statement rows with `date_scope` + `data_filter_domain` + sign + rounding + visibility), `ir.report.definition.metric` (column declarations). Phase 2: `ir.report.external.value` (write-back), `ir.report.pivot.config`. Phase 3: `ir.report.saved.filter`, `ir.report.snapshot`, `ir.report.subscription`, `ir.report.drillthrough`. | (planned) `src/ede/foundation/reporting/models/...` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ | Phase 1: `ReportRunner` (chip → period → metric orchestration), `StatementBodyBuilder` (lines × periods × metrics with per-line `date_scope`), `DefinitionResolver` (`(company, country, priority)` cascade). Phase 2: `PivotBodyBuilder`, `WritebackHandler` (append-only external_value), `LiveStreamSubscriber`. Phase 3: `SnapshotCapture`, `FXConverter`, `RowLevelRbacFilter`, `SubscriptionDispatcher` (consumes `foundation.jobs`). | (planned) `src/ede/core/engines/report/...` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.report.run` | HTTP `/api/report/<key>/run`, programmatic dispatch | Resolves definition → runs ReportRunner → returns `ReportResult` |
| `ede.report.snapshot.capture` (Phase 3) | Manual capture or scheduled job | Captures current state to `ir.report.snapshot`; optionally generates PDF |
| `ede.create`/`ede.update` on `ir.report.definition*` | Admin form | Standard CRUD |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.report.executed` | After a report runs successfully | Audit log; analytics |
| `ede.report.cell.written` (Phase 2) | After a write-back cell save | Audit log |
| `ede.report.snapshot.captured` (Phase 3) | After a snapshot is captured | Email subscription dispatcher |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/report/<key>/run` | Execute a report by key | `src/ede/foundation/reporting/controllers.py` |
| `GET /api/report/<key>/stream` (Phase 2) | SSE channel for live re-fetch | same |
| `POST /api/report/<key>/cell` (Phase 2) | Write a single editable cell value | same |
| `POST /api/report/<key>/snapshot` (Phase 3) | Capture point-in-time snapshot | same |
| `GET /api/report/category/<key>/definitions` | List definitions in a category visible to caller | same |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.ir.report.external.value.update` (Phase 2) | Always raises — external_value rows are append-only. |
| `pre.ir.report.external.value.delete` (Phase 2) | Always raises — append-only invariant. |
| `pre.ir.report.snapshot.update` (Phase 3) | Always raises — snapshots are immutable. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
*No persistent state machine. Report execution is stateless — every call produces a fresh `ReportResult`. Phase 2 introduces append-only state via `ir.report.external.value` and Phase 3 introduces immutable snapshots; neither uses workflow stages.*
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES`: `"reporting"` (added in Phase 1)
- Manifest `depends`: `["base", "presentation", "dataset"]`
<!-- /SYNC-BLOCK -->

### Foundation-level settings
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `REPORTING_DEFAULT_LOCALE` | str | `"en"` | `REPORTING_DEFAULT_LOCALE` | Locale fallback for value formatting. |
| `REPORTING_GRID_PAGE_SIZE` | int | `200` | `REPORTING_GRID_PAGE_SIZE` | Default React grid page size. |
| `REPORTING_WRITEBACK_REQUIRE_NOTE` (Phase 2) | bool | `False` | `REPORTING_WRITEBACK_REQUIRE_NOTE` | Audit-strict mode for editable cells. |
| `REPORTING_SNAPSHOT_RETENTION_DAYS` (Phase 3) | int | `365` | `REPORTING_SNAPSHOT_RETENTION_DAYS` | Auto-cleanup snapshot age. |
| `REPORTING_SUBSCRIPTION_DEFAULT_FORMAT` (Phase 3) | str | `"pdf"` | `REPORTING_SUBSCRIPTION_DEFAULT_FORMAT` | Default subscription output format. |
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
| `data/reporting_rbac.csv` | 3 RBAC roles — `ir.report.viewer`, `ir.report.designer`, `ir.report.admin` |
| `data/reporting_menus.xml` | Reports app in app-switcher |
| `data/reporting_phase2_rbac.csv` (Phase 2) | RBAC delta — `ir.report.writeback` permission |
| `data/reporting_phase3_rbac.csv` (Phase 3) | RBAC additions for snapshot / subscription / drill-through |
| `demo/demo_reports.xml` | 3 demo reports (Phase 1) |
| `demo/demo_reports_categories.xml` | 2 demo categories — `sales.pipeline`, `logistics.financial` |
| `demo/demo_reports_live.xml` (Phase 2) | Live demo report `sales.daily_revenue_live` |
| `demo/demo_subscriptions.xml` (Phase 3) | Demo email subscription |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Statement Mode + React Grid | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/reporting/phase-1-implementation.md) |
| Phase 2 | Live + Pivot + Write-Back | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/reporting/phase-2-implementation.md) |
| Phase 3 | Sharing + Snapshots + Multi-Currency | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/reporting/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Entire module not yet built | 🔴 Not Started | [roadmap/foundation/reporting/](../roadmap/foundation/reporting/README.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
*Will be populated as the module is built and first consumers integrate.*

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
*Pre-build — no migrations yet.*
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `ir.report.viewer` | Read `ir.report.*`; call `POST /api/report/<key>/run` |
| `ir.report.designer` | Viewer + CRUD on definition (admin form); Phase 2 grants write-back |
| `ir.report.admin` | Designer + manage categories + cross-tenant report visibility |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.dataset`](./foundation-dataset.md) — substrate; every metric flows through `ede.metric.run`.
- [`foundation.document`](./foundation-document.md) — Phase 3 PDF snapshots use DML templates.
- [`foundation.email`](./foundation-jobs.md) — Phase 3 subscriptions.
- [`foundation.jobs`](./foundation-jobs.md) — Phase 3 scheduled snapshot + subscription jobs.
- [`foundation.presentation`](./foundation-presentation.md) — admin form + DSL parser branch.
- [`foundation.base`](./foundation-customization.md) — `res.currency`, `res.organization`, `res.user.language_id`.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-12. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
