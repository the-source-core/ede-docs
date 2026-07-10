# `foundation.dashboard` — Demo Use-Case

**Module:** `ede.foundation.dashboard`
**App key:** `foundation.dashboard`
**Demo manifest entries:** `demo/demo_kpis.xml`, `demo/demo_dashboards.xml`
**Status:** ✅ Delivered (2026-07-10)

---

## Use-case

The dashboard engine ships **2 demo dashboards backed by 4 demo KPIs** that
demonstrate the headline capabilities against the Acme Forwarding demo tenant —
**without depending on any domain module.** All KPIs evaluate self-contained
`foundation` metrics (`base.partner_count`, `base.organization_count`,
`base.entity_total`, `base.partner_to_org_ratio`, `base.entity_summary` — shipped
by `foundation.dataset`), so the demo is coherent on a `--with-demo=all` tenant
regardless of which domains are active.

1. **Platform Overview** (`demo.platform_overview`) — five widgets exercising **all
   five widget types**: scorecard (Active Partners, with target + warn/alert
   thresholds), vs_target_bar (Active Organizations), trend_sparkline (Total
   Entities), table (Entity Summary), simple_chart bar (Entity Summary). Visible
   to everyone.
2. **Ops Desk** (`demo.ops_desk`) — two scorecards demonstrating **team-scoped
   object visibility**: the Active-Partners scorecard is visible to all, while the
   Partner/Org-Ratio scorecard is **owned by the Pricing Desk – West team**
   (`base.demo_team_pricing_west`, code `PDW`) — only that team's members (and
   admins) see it.

Both dashboards exercise:
- KPI registry mirror via `ir.kpi.definition` rows (`is_decorated=False` — demo-authored).
- Threshold evaluation — `threshold_warn=0.80, threshold_alert=0.50` drive amber/red status from current value vs target.
- Auto-refresh (60s default).
- Team-scoped object visibility via `team_scope="optional"` + `team_id` (Enh 13 read-filter).

## The unifying scenario fit

- Metrics aggregate the Acme Forwarding tenant's `res.partner` + `res.organization` rows (loaded by `foundation.base` demo).
- The team-scoped widget attaches to `base.demo_team_pricing_west` (Pricing Desk – West), a team the base demo already ships.

## Records produced

### `demo/demo_kpis.xml`

| External ID | Model | Key | Metric | Target | Direction |
|---|---|---|---|---|---|
| `dashboard.demo_kpi_active_partners` | `ir.kpi.definition` | `demo.active_partners` | `base.partner_count` | 6 | higher_better |
| `dashboard.demo_kpi_active_orgs` | `ir.kpi.definition` | `demo.active_orgs` | `base.organization_count` | 3 | higher_better |
| `dashboard.demo_kpi_total_entities` | `ir.kpi.definition` | `demo.total_entities` | `base.entity_total` | 10 | higher_better |
| `dashboard.demo_kpi_partner_ratio` | `ir.kpi.definition` | `demo.partner_ratio` | `base.partner_to_org_ratio` | 150 | higher_better |

### `demo/demo_dashboards.xml`

| External ID | Model | Notes |
|---|---|---|
| `dashboard.demo_dashboard_platform` | `ir.dashboard.definition` | Platform Overview (category `platform`) |
| `dashboard.demo_widget_platform_partners` | `ir.dashboard.widget` | scorecard · kpi `demo.active_partners` |
| `dashboard.demo_widget_platform_orgs` | `ir.dashboard.widget` | vs_target_bar · kpi `demo.active_orgs` |
| `dashboard.demo_widget_platform_total` | `ir.dashboard.widget` | trend_sparkline · kpi `demo.total_entities` |
| `dashboard.demo_widget_platform_table` | `ir.dashboard.widget` | table · metric `base.entity_summary` |
| `dashboard.demo_widget_platform_chart` | `ir.dashboard.widget` | simple_chart · metric `base.entity_summary` |
| `dashboard.demo_dashboard_ops` | `ir.dashboard.definition` | Ops Desk (category `operations`) |
| `dashboard.demo_widget_ops_partners` | `ir.dashboard.widget` | scorecard · visible to all |
| `dashboard.demo_widget_ops_ratio` | `ir.dashboard.widget` | scorecard · **team-scoped to `base.demo_team_pricing_west` (PDW)** |

## Out of scope

- Domain (sales / pricing) KPIs — those belong to the owning domain module's demo (a foundation module must not depend on domain metrics).
- Phase 2 insight/alert demo records; Phase 3 subscription/embed demo records.

## Dependencies

- `foundation.base` demo (partners, organizations, `base.demo_team_pricing_west`).
- `foundation.dataset` metrics (`base.*`) — registered at boot.

## Verification

`ede migrate upgrade -t <tenant> --with-demo=all` → `foundation.dashboard/demo`: `demo_kpis.xml` **4 created**, `demo_dashboards.xml` **9 created**; re-run is idempotent (updated, not created). The `demo.ops_desk` ratio widget carries `team_id` = the PDW team.

## Authoring order

`demo/demo_kpis.xml` before `demo/demo_dashboards.xml` (widgets `source_key` → KPI keys).

---

*Back to [demo-usecase index](../../README.md).*
