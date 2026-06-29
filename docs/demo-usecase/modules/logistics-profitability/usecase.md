# `logistics.profitability` — Demo Use-Case

**Module:** `domains.logistics.profitability`
**App key:** `logistics.profitability`
**Demo manifest entries** (target): `demo/demo_profitability.xml`
**Status:** ✅ Delivered (2026-06-29)

---

## Use-case

Operational profitability over the unifying scenario's flagship shipment — the
**Globex Inc. INMUM → SGSIN** ocean export (`shipments.demo_shipment_globex_001`).
The demo stands up one `logistics.shipment.profitability` header for that shipment
and a realistic charge breakdown (freight + origin + destination + customs +
documentation), computes per-line and rolled-up gross profit / margin in the
USD base currency, shows **one multi-currency line** (documentation bought in INR,
sold in USD) with its `logistics.profitability.currency.line` FX record, a
**pending manual adjustment** (detention recovery awaiting approval), and the
**audit trail** those records produce. It illustrates the Phase-1 promise: a usable
Profitability tab running entirely on estimated cost + inherited revenue (no
external finance sync).

## Records produced

### `demo/demo_profitability.xml`

| External ID | Model | Notes |
|---|---|---|
| `profitability.demo_header_globex_001` | `logistics.shipment.profitability` | Shipment-level header on `demo_shipment_globex_001`; base = USD; stage `estimated`; totals rolled up |
| `profitability.demo_line_freight` | `logistics.profitability.charge.line` | Ocean freight — rev 3200 / cost 2400 USD (from `demo_shipment_charge_freight`) → profit 800, margin 25% |
| `profitability.demo_line_origin` | `logistics.profitability.charge.line` | Origin THC — rev 400 / cost 250 USD |
| `profitability.demo_line_destination` | `logistics.profitability.charge.line` | Destination handling — rev 300 / cost 200 USD |
| `profitability.demo_line_customs` | `logistics.profitability.charge.line` | Customs clearance — rev 150 / cost 100 USD |
| `profitability.demo_line_docs` | `logistics.profitability.charge.line` | Documentation — rev 100 USD, cost 5000 INR (converted 60.24 USD) — the multi-currency line |
| `profitability.demo_currency_docs` | `logistics.profitability.currency.line` | FX for the docs line — INR→USD @ 0.01205, converted cost 60.24 USD |
| `profitability.demo_adjustment_detention` | `logistics.profitability.adjustment` | Detention recovery +200 USD on the freight line, `pending_approval` |
| `profitability.demo_audit_inherit` | `logistics.profitability.audit.entry` | `revenue_change` — freight line inherited from the shipment charge |
| `profitability.demo_audit_adjustment` | `logistics.profitability.audit.entry` | `manual_override` — detention adjustment proposed |

## Out of scope

- Actual invoice / vendor-bill sync, billing/cost status, variance, closure, exceptions (Phase 2 — finance integration).
- Master/console + leg-wise profitability and AI insights (Phase 2 / Phase 3).
- The adjustment is left `pending_approval` (illustrative) — the demo loader creates rows but does not run the apply engine, so no line mutation is implied.

## Dependencies

- `foundation.base` demo (`base.currency_usd`, `base.currency_inr`, `base.demo_partner_co_001`).
- `logistics.shipments` demo (`shipments.demo_shipment_globex_001`, `shipments.demo_shipment_charge_freight`).

## Verification

`ede migrate upgrade -t <tenant> --with-demo=logistics.profitability` — expected
**10 created**; re-run idempotent (`updated`, not `created`). Header rolls up
total revenue 4150 / est cost ~3010 / gross profit ~1140 / margin ~27% (healthy).

## Authoring order

Header → charge lines → currency line → adjustment → audit entries (refs cascade
in that order). Manifest: `demo: ["demo/demo_profitability.xml"]`.

---

*Back to [demo-usecase index](../../README.md).*
