# `foundation.reporting` — Demo Use-Case

**Module:** `ede.foundation.reporting`
**App key:** `foundation.reporting`
**Demo manifest entries** (planned, Phase 1): `demo/demo_reports.xml`, `demo/demo_reports_categories.xml`
**Status:** 🔴 Not authored (module itself is 🔴 Not Started — see [roadmap/foundation/reporting/](../../../../roadmap/foundation/reporting/))

---

## Use-case

The reporting engine ships **3 demo reports** demonstrating the three load-bearing features of the module:

1. **Per-line `date_scope` mixing** — a single Sales Pipeline report renders section headers using `same_period` while the "Pipeline Total" line uses `inception_to_date`. Proves the fix to the reference-stack's architectural gap on the day of delivery.
2. **Period comparison columns** — Quarterly Margin report renders 3 columns (Q4-26 / Q3-26 / Q4-25). Demonstrates the auto-injecting comparison chip + the period resolver building 3 periods from one chip selection.
3. **Formula-metric column** — Quote Conversion Rate report consumes a `sales.win_rate` formula metric (from `foundation.dataset` Phase 2). Demonstrates that derived KPIs work as report columns without special-casing.

Reports also exercise the React grid renderer (sticky headers, frozen columns, drill-down), the `(company_id, country_id, priority)` definition resolution cascade, and the per-run cache (45+ metric executions per report dedup'd to ~5 DB queries).

## The unifying scenario fit

All three reports run against the Acme Forwarding scenario:
- Sales Pipeline ranges over `crm.opportunity` records owned by `demo_user_sales_rep`.
- Quarterly Margin ranges over `pricing.rate` records on lane `INMUM ↔ SGSIN`.
- Quote Conversion Rate computes from `crm.quote` records with statuses `ACCEPTED` / `REJECTED` / `LOST`.

## Records produced (planned, Phase 1)

### `demo/demo_reports_categories.xml`

| External ID | Model | Name | Notes |
|---|---|---|---|
| `reporting.demo_category_sales_pipeline` | `ir.report.category` | Sales Pipeline | Groups demo sales reports |
| `reporting.demo_category_logistics_financial` | `ir.report.category` | Logistics Financial | Groups demo logistics financial reports |

### `demo/demo_reports.xml`

| External ID | Model | Key | Notes |
|---|---|---|---|
| `reporting.demo_sales_pipeline_by_stage` | `ir.report.definition` | `sales.pipeline_by_stage` | Statement-mode; columns This-Quarter / Last-Quarter / YoY; mixed `date_scope` (sections=`same_period`, "Pipeline Total" line=`inception_to_date`) |
| `reporting.demo_quarterly_margin` | `ir.report.definition` | `logistics.quarterly_margin` | Columns Q4-26 / Q3-26 / Q4-25; rows grouped by trade lane; sign-inverted cost rows |
| `reporting.demo_quote_conversion_rate` | `ir.report.definition` | `sales.quote_conversion_rate` | Scalar formula metric with trend column |

Plus ~30 RBAC permission rows linking the demo users to `ir.report.viewer` / `ir.report.designer` roles.

## Out of scope

- Live SSE demo — that belongs to Phase 2 (`sales.daily_revenue_live`).
- Write-back editable cells — Phase 2.
- Snapshot PDFs — Phase 3 (requires `foundation.document` Phase 1).
- Email subscriptions — Phase 3.

## Dependencies

- `foundation.dataset` Phase 1, 2, 3 must ship first.
- `foundation.base` + `logistics.sales-crm` + `logistics.pricing` demo data (the records the reports aggregate over).
- Reporting module's own Phase 1 ship.

## Verification (once Phase 1 lands)

```
ede migrate upgrade -t demo --with-demo=foundation.reporting
```

Then:
1. Open Reports app from app-switcher.
2. **Sales Pipeline by Stage** — chip the date range, observe section header values (same period) differ from the "Pipeline Total" (inception-to-date). Toggle the comparison chip and a second column appears.
3. **Quarterly Margin** — 3 columns render; click any cost row cell → drill modal opens filtered to that lane × quarter.
4. **Quote Conversion Rate** — single scalar with trend; refresh after toggling a chip; cell value re-computes in <500ms (per-run cache active).

## Authoring order

1. `foundation.dataset` Phase 1 + 2 + 3 ship (per-line date_scope + chip system require Phase 3).
2. `foundation.reporting` Phase 1 ships the runner + React grid.
3. Demo records authored last, smoke-tested with `--with-demo=foundation.reporting`.

---

*Back to [demo-usecase index](../../README.md).*
