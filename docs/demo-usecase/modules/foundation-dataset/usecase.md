# `foundation.dataset` — Demo Use-Case

**Module:** `ede.foundation.dataset`
**App key:** `foundation.dataset`
**Demo manifest entries** (planned, Phase 1): `demo/demo_blueprints.xml`, `demo/demo_metrics_seed_data.xml`
**Status:** 🔴 Not authored (module itself is 🔴 Not Started — see [roadmap/foundation/dataset/](../../../../roadmap/foundation/dataset/))

---

## Use-case

The dataset substrate is mostly engine code, but two **headline UI features** demand demo data so admins can immediately experience the value at first login:

1. **Low-code Blueprint UI** — `Settings → Technical → Datasets` should show three pre-built dataset blueprints that demonstrate join + group + filter + sort across the Acme Trading demo data. An admin can clone any of them, tweak in the form, and see live SQL preview update on save.
2. **Metric Registry browser** — `Settings → Technical → Metrics` should show registered metrics with "Run sample" buttons so admins can see the universal JSON result contract immediately.

The substrate also ships **2 code-authored demo metrics** (`sales.pipeline_value` + `sales.opportunity_count_by_stage`) used by the reporting + dashboard modules' demo data. These ride on the foundation.base + logistics.sales-crm demo records that already exist.

## Records produced (planned, Phase 1)

### `demo/demo_blueprints.xml`

| External ID | Model | Notes |
|---|---|---|
| `dataset.demo_blueprint_partners_with_country` | `ir.dataset.blueprint` | Base model = `res.partner`; 1 JOIN to `res.country`; 3 fields (`p.name`, `p.email`, `c.iso_code_2:country_code`); 1 group by country. **Ships in `state="locked"`** to demo the lock UX. |
| `dataset.demo_blueprint_rates_expiring_30d` | `ir.dataset.blueprint` | Base model = `pricing.rate`; 1 JOIN to `logistics.trade.lane`; filter `validity_end ≤ today + 30d`; sort by `validity_end ASC`. Designed to surface pricing operational priority. |
| `dataset.demo_blueprint_quote_lines_with_currency` | `ir.dataset.blueprint` | Base model = `crm.quote.line`; 2 JOINs (to `crm.quote` + `res.currency`); 4 fields including currency code; demonstrates nested connections. |

### `demo/demo_metrics_seed_data.xml`

| External ID | Model | Notes |
|---|---|---|
| _none_ | _The demo metrics themselves are **code-authored** (`@api.metric`), so they don't need XML demo rows._ The XML file is empty in Phase 1, kept as a placeholder for future ad-hoc admin-authored metrics that the demo-data path may surface in Phase 2+. |

## Out of scope

- Plan + formula demo metrics — those land with `foundation.dataset` Phase 2 (Win Rate, Avg Deal Size, Partner Ledger).
- Webhook outbound demos — Phase 3.
- Streaming SSE demos — Phase 3 (overlaps with the reporting module's `sales.daily_revenue_live` demo).

## Dependencies

- `foundation.base` demo data (organizations, users, partners) for blueprint joins.
- `logistics.sales-crm` demo data (opportunities, quotes, quote lines) for the metric demos.
- `logistics.pricing` demo data (rates, trade lanes) for the rates-expiring blueprint.

## Verification (once Phase 1 lands)

```
ede migrate upgrade -t demo --with-demo=foundation.dataset
```

Then:
1. Open `Settings → Technical → Datasets` — three blueprints visible, one locked.
2. Click `demo_blueprint_partners_with_country` → form shows resolved SQL preview, "Run now" button returns ≥3 rows.
3. Open `Settings → Technical → Metrics` — `sales.pipeline_value` + `sales.opportunity_count_by_stage` visible, "Run sample" returns non-zero values.

## Authoring order

1. `foundation.dataset` Phase 1 ships (gates this doc moving past 🔴).
2. Code-authored demo metrics ship in `src/ede/foundation/dataset/tools/metrics/sales.py`.
3. `demo_blueprints.xml` authored after `foundation.base` + `logistics.sales-crm` + `logistics.pricing` demo data is verified.

---

*Back to [demo-usecase index](../../README.md).*
