<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# BI Dashboard Engine — Implementation Docs

**Module:** `foundation.dashboard` (`src/ede/core/engines/dashboard/` + `src/ede/foundation/dashboard/` + `src/frontend/src/workspace/views/dashboard/`)
**Roadmap:** [roadmap/foundation/dashboard/](../roadmap/foundation/dashboard/README.md)
**Status:** 🔴 Not Started (roadmap drafted 2026-05-12)
**Layer:** Foundation engine — consumes `foundation.dataset` substrate + `foundation.jobs` for scheduled insights

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
The **at-a-glance KPI surface** — the Dashboards app in the app-switcher, the React widget grid, the KPI registry (a Metric plus targets / thresholds / visualisation hints / alert channels), and a scheduled **rule-based insight surface** that evaluates every active KPI nightly and posts findings to a feed.

Five widget types ship in Phase 1: scorecard, trend_sparkline, vs_target_bar, table, simple_chart. Phase 2 adds rule-based insight detection + alerting via `foundation.notifications`. Phase 3 adds subscriptions, public embed, mobile layouts, and drill-through chains.

**AI is explicitly future scope** under the internal MCP server project, not part of this module. Phase 1+2+3 ship deterministic rule-based dashboards.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Reports answer "what is the value?" Dashboards answer "is this number on track?" The two surfaces share a substrate but render differently — dashboards prioritise glanceability (scorecards, sparklines, gauges), thresholds (target vs actual visual), and rule-based intelligence (insight surface that auto-flags KPIs crossing thresholds).

Two load-bearing design choices:
1. **KPI as a first-class abstraction on top of metric.** A `Metric` is "the value"; a `KPI` is "the value + what's good vs bad + how to show it + who gets alerted". Both need different lifecycles, so both get registries.
2. **Rule-based insight surface, not AI.** Threshold-crossing, trend anomaly (MAD z-score), target achievement, stale-KPI detection — all deterministic, all testable. AI insight authoring is future scope under the internal MCP server, which will consume this module's KPI registry as MCP tools without requiring substrate changes.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX** — Dashboards app in the app-switcher. Each dashboard category gets a dynamic menu entry. The grid renders KPIs + widgets with auto-refresh (60s default), drill-through navigation, mobile-responsive layouts (Phase 3). Phase 2 adds an insight feed right-rail panel.
- **Authoring**:
  - Code authors register KPIs via `@api.kpi("namespace.key")` with metric_key + target + thresholds + direction + visualization + alert_channels.
  - Domain modules author Dashboard XML in their `data/` manifest using `<Dashboard>` / `<DashboardWidget>` / `<KPI>` DSL.
- **Programmatic entry points:**
  - `env.dispatch(Command("ede.dashboard.run", payload={"key": ..., "params": {...}}))` — execute a dashboard.
  - `env.dispatch(Command("ede.kpi.evaluate", payload={"key": ..., "params": {...}}))` — evaluate a single KPI.
  - `@api.on_event("ede.dashboard.insight.created")` — react to new findings (Phase 2).
  - HTTP: `POST /api/dashboard/<key>/run`, `POST /api/kpi/<key>/evaluate`, `GET /api/dashboard/embed/<token>` (Phase 3).
- **Integration boundary** — PRODUCES `list[WidgetResult]` JSON, `ir.dashboard.insight` rows (Phase 2), `ede.kpi.threshold.crossed` events. CONSUMES `foundation.dataset` (every KPI references a metric), `foundation.jobs` (Phase 2 scheduled insight job; Phase 3 digest jobs), `foundation.notifications` (alert dispatch), `foundation.email` (Phase 3 digest delivery), `foundation.document` Phase 2 HTML (Phase 3 digest body).
- **Future internal MCP server** — will enumerate `ir.kpi.definition` rows as MCP tools alongside the dataset/metric registry. No changes required here; the KPI registry is exposed via the standard `ede.kpi.evaluate` command.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Code author]                                [foundation.dashboard]                         [User]
──────────────                               ──────────────────────                         ──────
@api.kpi("sales.pipeline_value", ...)   ─►  ir.kpi.definition (registry mirror)
                                                       │
ir.dashboard.definition (layout)          ─►  DashboardRunner.run(def, params)
   ir.dashboard.widget (per cell)                      │
                                                       ▼
                                          For each widget:
                                             resolve metric_key | kpi_key | dataset_key
                                             dispatch ede.metric.run with chip-derived params
                                             evaluate against KPI thresholds (if KPI)
                                                       │
                                                       ▼
                                          list[WidgetResult]  ──►  React DashboardView
                                                                       ▼
                                                                 scorecard / sparkline /
                                                                 vs_target_bar / table /
                                                                 simple_chart

Phase 2 — Scheduled insight surface:
   @api.scheduled_job("dashboard.insight_surface_nightly")
       evaluate every active KPI via 4 rules:
         ThresholdRule  → threshold_breach
         TrendRule      → trend_anomaly  (MAD z-score)
         TargetRule     → target_achievement
         StaleRule      → stale_kpi
       ─►  ir.dashboard.insight (append-only)
       ─►  ir.dashboard.alert.rule → notification.send

Core engine:        src/ede/core/engines/dashboard/
  kpi/registry.py      — @api.kpi decorator + frozen KPI
  kpi/evaluator.py     — threshold + direction + band check
  runner.py            — DashboardRunner
  insight/surface.py   — Phase 2 scheduled job entry
  insight/rules/       — Phase 2 rule library
  insight/dispatcher.py — Phase 2 channel router

Foundation shell:   src/ede/foundation/dashboard/
  models/              — ir.kpi.definition + ir.dashboard.definition + .widget
                         + Phase 2: insight + alert.rule
                         + Phase 3: subscription + saved_filter + share.token + drillthrough
  views/               — admin forms for KPIs / dashboards / widgets / alert rules
  jobs/                — Phase 2 insight_job; Phase 3 digest_job
  data/                — RBAC seed + menus + notification templates

Frontend:           src/frontend/src/workspace/views/dashboard/
  DashboardView.tsx    — page wrapper
  WidgetGrid.tsx       — responsive 12-col grid
  widgets/             — 5 widget components
  ChipBar.tsx          — reuses chip system
  InsightFeed.tsx      — Phase 2 right-rail
  EmbedDashboardView.tsx — Phase 3 public-token embed
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none yet — all 🔴 Not Started_ | Phase 1: `ir.kpi.definition` (registry mirror — one row per `@api.kpi` plus admin-authored ad-hoc KPIs), `ir.dashboard.definition` (layout root with chip config + refresh interval + `(company, country, priority)` resolution), `ir.dashboard.widget` (per cell in grid — type + source + position + viz_config + drill_target_key). Phase 2: `ir.dashboard.insight` (append-only findings feed), `ir.dashboard.alert.rule` (channel routing with cooldown). Phase 3: `ir.dashboard.subscription` (digest cadence), `ir.dashboard.saved.filter` (per-user chip sets), `ir.dashboard.share.token` (public embed), `ir.kpi.history` (audit), `ir.dashboard.drillthrough` (cross-dashboard nav). | (planned) `src/ede/foundation/dashboard/models/...` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ | Phase 1: `KpiRegistry`, `KpiEvaluator` (threshold/direction/band), `DashboardRunner`, `WidgetResolver`. Phase 2: `InsightSurface` (scheduled job entry), 4 rule classes (`ThresholdRule` / `TrendRule` / `TargetAchievementRule` / `StaleKpiRule`), `AlertDispatcher`. Phase 3: `SubscriptionDigestJob`, `ShareTokenResolver`, `KpiAuditHook`. | (planned) `src/ede/core/engines/dashboard/...` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.dashboard.run` | HTTP `/api/dashboard/<key>/run`, programmatic dispatch | Resolves layout → runs widgets → returns `list[WidgetResult]` |
| `ede.kpi.evaluate` | HTTP `/api/kpi/<key>/evaluate`, programmatic dispatch | Resolves KPI → executes metric → evaluates threshold → returns `KpiResult` |
| `ede.create`/`ede.update`/`ede.delete` on `ir.dashboard.*`, `ir.kpi.definition` | Admin form | Standard CRUD |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.dashboard.executed` | After dashboard run | Audit log |
| `ede.kpi.evaluated` | After KPI evaluation | Audit log |
| `ede.kpi.threshold.crossed` (Phase 2) | When KPI value crosses warn/alert threshold | Alert dispatcher → `foundation.notifications` |
| `ede.dashboard.insight.created` (Phase 2) | When a rule fires | Insight feed UI subscribers |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/dashboard/<key>/run` | Execute a dashboard | `src/ede/foundation/dashboard/controllers.py` |
| `POST /api/kpi/<key>/evaluate` | Evaluate a single KPI | same |
| `GET /api/dashboard/_list` | List dashboards visible to caller | same |
| `GET /api/kpi/_list` | List KPIs visible to caller | same |
| `GET /api/dashboard/embed/<token>` (Phase 3) | Token-gated public dashboard view (no JWT) | same |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.ir.dashboard.insight.update` (Phase 2) | Always raises — append-only |
| `pre.ir.dashboard.insight.delete` (Phase 2) | Always raises — append-only (dismissal sets `dismissed_at_utc` instead) |
| `pre.ir.kpi.definition.update` (Phase 3) | Snapshots before/after to `ir.kpi.history` (audit) |
| `pre.ir.kpi.history.update` / `delete` (Phase 3) | Always raises — append-only |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
*No state machine. Dashboards are stateless renders; insights are append-only events. KPI evaluator returns one of `{ok, warn, alert}` per call but that's a derived computation, not a persistent state.*
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES`: `"dashboard"` (added in Phase 1)
- Manifest `depends`: `["base", "presentation", "dataset", "jobs", "notifications"]`
<!-- /SYNC-BLOCK -->

### Foundation-level settings
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `DASHBOARD_DEFAULT_REFRESH_INTERVAL_SECONDS` | int | `60` | `DASHBOARD_DEFAULT_REFRESH_INTERVAL_SECONDS` | Default auto-refresh interval. |
| `DASHBOARD_INSIGHT_RETENTION_DAYS` (Phase 2) | int | `180` | `DASHBOARD_INSIGHT_RETENTION_DAYS` | Auto-cleanup of older insights. |
| `DASHBOARD_INSIGHT_JOB_CRON` (Phase 2) | str | `"0 2 * * *"` | `DASHBOARD_INSIGHT_JOB_CRON` | Nightly insight evaluation cron. |
| `DASHBOARD_TREND_WINDOW_DAYS` (Phase 2) | int | `7` | `DASHBOARD_TREND_WINDOW_DAYS` | Trend rule rolling window. |
| `DASHBOARD_TREND_ZSCORE_THRESHOLD` (Phase 2) | float | `2.5` | `DASHBOARD_TREND_ZSCORE_THRESHOLD` | Trend anomaly cutoff. |
| `DASHBOARD_STALE_KPI_DAYS` (Phase 2) | int | `7` | `DASHBOARD_STALE_KPI_DAYS` | Stale-KPI threshold. |
| `DASHBOARD_EMBED_DEFAULT_TTL_DAYS` (Phase 3) | int | `30` | `DASHBOARD_EMBED_DEFAULT_TTL_DAYS` | Embed token default TTL. |
| `DASHBOARD_DIGEST_DEFAULT_FORMAT` (Phase 3) | str | `"inline_html"` | `DASHBOARD_DIGEST_DEFAULT_FORMAT` | Subscription default format. |
| `DASHBOARD_MOBILE_BREAKPOINT_PX` (Phase 3) | int | `768` | `DASHBOARD_MOBILE_BREAKPOINT_PX` | Mobile layout cutoff. |
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
| `data/dashboard_rbac.csv` | 3 RBAC roles — `ir.dashboard.viewer`, `ir.dashboard.designer`, `ir.dashboard.admin` |
| `data/dashboard_menus.xml` | Dashboards app in app-switcher |
| `data/dashboard_insight_job.xml` (Phase 2) | Scheduled job definition |
| `data/dashboard_notification_templates.xml` (Phase 2) | 4 notification templates (threshold_breach / trend_anomaly / target_achievement / stale_kpi) |
| `data/dashboard_phase2_rbac.csv` (Phase 2) | RBAC delta — alert rule admin |
| `data/dashboard_phase3_rbac.csv` (Phase 3) | RBAC additions for subscriptions / embeds / drill-through |
| `demo/demo_kpis.xml` | 5 demo KPIs |
| `demo/demo_dashboards.xml` | 2 demo dashboards (Sales Executive + Pricing Performance) with 8 widgets |
| `demo/demo_alert_rules.xml` (Phase 2) | Demo alert rules wiring 2 KPIs to email channel |
| `demo/demo_dashboard_subscriptions.xml` (Phase 3) | Demo weekly digest subscription |
| `demo/demo_dashboard_embed_tokens.xml` (Phase 3) | Demo public embed token |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | KPIs + Widgets + React Grid | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/dashboard/phase-1-implementation.md) |
| Phase 2 | Rule-Based Insight Surface + Alerting | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/dashboard/phase-2-implementation.md) |
| Phase 3 | Sharing + Digests + Mobile + Embed | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/dashboard/phase-3-implementation.md) |
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
| Entire module not yet built | 🔴 Not Started | [roadmap/foundation/dashboard/](../roadmap/foundation/dashboard/README.md) |
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
| `ir.dashboard.viewer` | Read dashboards / KPIs visible to caller; call `POST /api/dashboard/<key>/run` |
| `ir.dashboard.designer` | Viewer + CRUD on `ir.dashboard.*` (admin-authored ad-hoc KPIs in Phase 3) |
| `ir.dashboard.admin` | Designer + manage alert rules (Phase 2) + manage embed tokens (Phase 3) + cross-tenant visibility |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.dataset`](./foundation-dataset.md) — substrate; every KPI/widget references a metric or dataset.
- [`foundation.jobs`](./foundation-jobs.md) — Phase 2 scheduled insight job; Phase 3 digest jobs.
- [`foundation.notifications`](./foundation-notifications.md) — Phase 2 alert dispatch.
- [`foundation.email`](./foundation-jobs.md) — Phase 3 digest delivery.
- [`foundation.document`](./foundation-document.md) — Phase 3 digest HTML rendering.
- [`foundation.reporting`](./foundation-reporting.md) — cross-engine drill-through (widget → report).
- [`foundation.presentation`](./foundation-presentation.md) — DSL parser branch + admin form.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-12. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
