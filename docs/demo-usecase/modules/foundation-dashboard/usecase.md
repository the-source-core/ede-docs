# `foundation.dashboard` — Demo Use-Case

**Module:** `ede.foundation.dashboard`
**App key:** `foundation.dashboard`
**Demo manifest entries** (planned, Phase 1): `demo/demo_kpis.xml`, `demo/demo_dashboards.xml`
**Status:** 🔴 Not authored (module itself is 🔴 Not Started — see [roadmap/foundation/dashboard/](../../../../roadmap/foundation/dashboard/))

---

## Use-case

The dashboard engine ships **2 demo dashboards backed by 5 demo KPIs** demonstrating the headline capabilities:

1. **Sales Executive Dashboard** — five widgets exercising all five widget types: scorecard (Pipeline Value), vs_target_bar (Pipeline vs Target), trend_sparkline (Win Rate over 12 weeks), table (Opportunities by Stage with drill-through to the Pipeline by Stage report), simple_chart bar (Revenue YTD by month).
2. **Pricing Performance Dashboard** — three scorecards (Active Rates Count / Avg Margin % / Rates Expiring 30d) for the pricing operations desk.

Both dashboards exercise:
- KPI registry mirror via `ir.kpi.definition` rows.
- Threshold evaluation — Pipeline Value's `threshold_warn=0.80, threshold_alert=0.50` cause amber/red badges based on current values vs target.
- Auto-refresh — both dashboards default to 60s; observable in the browser without page reload.
- Cross-engine drill-through — clicking the Opportunities-by-Stage table row navigates to the reporting module's `sales.pipeline_by_stage` report pre-filtered by stage.
- Per-run cache benefit — the 5 KPIs share 3 underlying metrics, so a dashboard render hits the DB 3 times not 5.

## The unifying scenario fit

All five KPIs evaluate against the Acme Forwarding demo data:
- Sales KPIs aggregate `crm.opportunity` + `crm.quote` records owned by `demo_user_sales_rep`.
- Pricing KPIs aggregate `pricing.rate` records on lane INMUM ↔ SGSIN.

## Records produced (planned, Phase 1)

### `demo/demo_kpis.xml`

| External ID | Model | Key | Target | Direction | Visualization |
|---|---|---|---|---|---|
| `dashboard.demo_kpi_pipeline_value` | `ir.kpi.definition` | `sales.pipeline_value` | 5,000,000 | higher_better | scorecard, vs_target_bar, trend_sparkline |
| `dashboard.demo_kpi_win_rate` | `ir.kpi.definition` | `sales.win_rate` | 0.35 | higher_better | scorecard, trend_sparkline |
| `dashboard.demo_kpi_avg_deal_size` | `ir.kpi.definition` | `sales.avg_deal_size` | 50,000 | higher_better | scorecard |
| `dashboard.demo_kpi_active_rate_count` | `ir.kpi.definition` | `pricing.active_rate_count` | 100 | higher_better | scorecard |
| `dashboard.demo_kpi_avg_margin_pct` | `ir.kpi.definition` | `pricing.avg_margin_pct` | 0.05 | higher_better | scorecard |

### `demo/demo_dashboards.xml`

| External ID | Model | Key | Widgets |
|---|---|---|---|
| `dashboard.demo_sales_executive` | `ir.dashboard.definition` | `sales.executive_dashboard` | 5 widgets (scorecard Pipeline / vs_target_bar Pipeline / trend Win Rate / table Opps by Stage / chart Revenue YTD) |
| `dashboard.demo_pricing_performance` | `ir.dashboard.definition` | `pricing.performance_dashboard` | 3 widgets (scorecard Active Rates / scorecard Avg Margin / table Expiring 30d) |

Each dashboard contains 3–5 `ir.dashboard.widget` child rows. Plus ~20 RBAC permission rows linking demo users to `ir.dashboard.viewer`.

## Phase 2 + Phase 3 additions (planned)

| Phase | File | What it adds |
|---|---|---|
| Phase 2 | `demo/demo_alert_rules.xml` | 2 demo alert rules wiring Pipeline Value + Avg Margin KPIs to email channel |
| Phase 2 | _seeded ir.dashboard.insight rows_ | 3 sample insights (one threshold_breach, one trend_anomaly, one target_achievement) so the feed UI is exercisable at first login |
| Phase 3 | `demo/demo_dashboard_subscriptions.xml` | Weekly digest subscription for `demo_user_sales_rep` on Sales Executive |
| Phase 3 | `demo/demo_dashboard_embed_tokens.xml` | Public read-only embed token for Pricing Performance (TTL 30 days) |

## Out of scope

- AI-driven insight authoring — future scope under the internal MCP server project, not part of this module.
- Cross-tenant dashboard sharing — never (each tenant gets its own).
- Live WebSocket push (sub-second updates) — Phase 3 candidate, not committed.

## Dependencies

- `foundation.dataset` Phase 1+2+3 (metric registry + per-run cache + chip system).
- `foundation.jobs` Phase 1 (Phase 2's insight surface scheduled job).
- `foundation.notifications` Phase 1 ✅ (alert dispatch in Phase 2).
- `foundation.base` + `logistics.sales-crm` + `logistics.pricing` demo data.

## Verification (once Phase 1 lands)

```
ede migrate upgrade -t demo --with-demo=foundation.dashboard
```

Then:
1. Open Dashboards app from app-switcher.
2. **Sales Executive Dashboard** — 5 widgets render with non-zero values.
3. Pipeline Value scorecard badge reflects target status (green/amber/red).
4. Win Rate sparkline shows 12 data points.
5. Click the table widget's Opportunities-by-Stage row → drill-through to reporting's Pipeline by Stage report opens pre-filtered.
6. Wait 60 seconds → widgets auto-refresh without page reload.

## Authoring order

1. `foundation.dataset` Phase 1+2+3 ship (substrate prereqs).
2. `foundation.dashboard` Phase 1 ships (KPI registry + dashboard runner + 5 widget types).
3. Demo KPIs authored as `@api.kpi` decorators in `src/ede/foundation/dashboard/tools/kpis/`.
4. Demo dashboards authored as `demo_dashboards.xml`.
5. Smoke-test with `--with-demo=foundation.dashboard`.

---

*Back to [demo-usecase index](../../README.md).*
