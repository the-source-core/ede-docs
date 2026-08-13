# `logistics.profitability` — Demo Use-Case

**Module:** `domains.logistics.profitability`
**App key:** `logistics.profitability`
**Demo manifest entries**: `demo/demo_billing_readiness.xml`, `demo/demo_profitability.xml`, `demo/demo_proforma.xml`
**Status:** ✅ Delivered (2026-06-29) · proforma billing (WS-3) ✅ 2026-07-27 · commercial surface (WS-4) ✅ 2026-07-27 · creation & linkage (WS-4.1) ✅ 2026-07-29 · line currency translation + FX provenance (WS-4.3) ✅ 2026-08-10 · operational tax (WS-5 / WS-13) ✅ 2026-08-12 — **inputs only, computed on demand** (see the tax section below) · billing readiness (WS-8) ✅ 2026-08-12 — the gate's two dials ship as `ir.config` rows; the evaluator itself seeds nothing · proforma numbering (WS-1) 🟡 — **no demo rows**, which is what holds it off ✅

---

## Use-case

Operational profitability over the unifying scenario's flagship shipment — the
**Globex Inc. INMUM → SGSIN** ocean export (`shipments.demo_shipment_globex_001`).
The demo stands up one `logistics.shipment.profitability` header for that shipment
and a realistic charge breakdown (freight + origin + destination + customs +
documentation + BAF), computes per-line and rolled-up gross profit / margin in the
USD base currency, shows **one multi-currency line** (documentation bought in INR,
sold in USD) with its `logistics.profitability.currency.line` FX record, a
**pending manual adjustment** (detention recovery awaiting approval), and the
**audit trail** those records produce. It illustrates the Phase-1 promise: a usable
Profitability tab running entirely on estimated cost + inherited revenue (no
external finance sync).

The **proforma-billing (WS-3 / WS-4)** demo layers four `logistics.proforma.document`
records on top of that same charge breakdown — two customer proforma **invoices**
that bill the sell side (the ocean freight split 2,000 + 1,200 across the two, to
show partial + split billing, the second carrying a document-level discount) plus
two vendor proforma **bills** on the buy side, an independent quota: the freight
cost (2,400) and the documentation cost (60.24 USD translated from 5,000 INR,
which is the demo's one cross-currency proforma line). Because the loader dispatches
`ede.create`, the WS-3 hooks run on load: line amounts compute, document totals
roll up (PFM-12), and each invoice line syncs its source charge's sell-side
billing coverage (PFM-20) — so after load the freight charge reads
`fully_invoiced`, remaining 0.

## Records produced

### `demo/demo_profitability.xml`

| External ID | Model | Notes |
|---|---|---|
| `profitability.demo_header_globex_001` | `logistics.shipment.profitability` | Shipment-level header on `demo_shipment_globex_001`; base = USD; stage `estimated`; org `base.demo_org_west_branch`; `billing_status=partially_billed` (five of six charges consumed by the demo proformas); totals rolled up |
| `profitability.demo_line_freight` | `logistics.profitability.charge.line` | Ocean freight — rev 3200 / cost 2400 USD (from `demo_shipment_charge_freight`) → profit 800, margin 25% |
| `profitability.demo_line_origin` | `logistics.profitability.charge.line` | Origin THC — rev 400 / cost 250 USD |
| `profitability.demo_line_destination` | `logistics.profitability.charge.line` | Destination handling — rev 300 / cost 200 USD |
| `profitability.demo_line_customs` | `logistics.profitability.charge.line` | Customs clearance — rev 150 / cost 100 USD |
| `profitability.demo_line_docs` | `logistics.profitability.charge.line` | Documentation — rev 100 USD, cost 5000 INR (converted 60.24 USD) — the multi-currency line |
| `profitability.demo_line_baf` | `logistics.profitability.charge.line` | Bunker Adjustment Factor — rev 250 / cost 180 USD. **Deliberately left out of every demo proforma** so the WS-4.1 "Create Proforma Invoice" button has a billable charge on first click; without it every charge is `fully_invoiced` and the button can only refuse (BR-09 / PFM-21) |
| `profitability.demo_currency_docs` | `logistics.profitability.currency.line` | FX for the docs line — INR→USD @ 0.01205, converted cost 60.24 USD |
| `profitability.demo_adjustment_detention` | `logistics.profitability.adjustment` | Detention recovery +200 USD on the freight line, `pending_approval` |
| `profitability.demo_audit_inherit` | `logistics.profitability.audit.entry` | `revenue_change` — freight line inherited from the shipment charge |
| `profitability.demo_audit_adjustment` | `logistics.profitability.audit.entry` | `manual_override` — detention adjustment proposed |

### `demo/demo_proforma.xml` (WS-3 / WS-4)

**13 records — 4 documents + 9 lines.**

| External ID | Model | Notes |
|---|---|---|
| `profitability.demo_proforma_invoice_1` | `logistics.proforma.document` | Customer proforma **invoice** → Globex; freight 2,000 (of 3,200, partial) + origin 400 + customs 150 = payable 2,550 USD |
| `profitability.demo_proforma_invoice_1_freight` | `logistics.proforma.line` | ← `demo_line_freight`; billed 2,000, remaining 1,200 (partial) |
| `profitability.demo_proforma_invoice_1_origin` | `logistics.proforma.line` | ← `demo_line_origin`; the one **RATED** line (WS-4 / D-13) — 2 × 40HC @ 200, not 400 × 1. Its billable base is the source charge's *quantity* (2), where the amount-form lines carry their amount. |
| `profitability.demo_proforma_invoice_1_customs` | `logistics.proforma.line` | ← `demo_line_customs`; billed 150 (full) |
| `profitability.demo_proforma_invoice_2` | `logistics.proforma.document` | Customer proforma **invoice** → Globex; freight remainder 1,200 (SPLIT) + destination 300 + docs 100 = 1,600, less a 100 document discount = payable **1,500 USD** |
| `profitability.demo_proforma_invoice_2_freight` | `logistics.proforma.line` | ← `demo_line_freight`; billed 1,200 (prev 2,000 → completes 3,200, fully_invoiced) |
| `profitability.demo_proforma_invoice_2_destination` | `logistics.proforma.line` | ← `demo_line_destination`; billed 300 (full) |
| `profitability.demo_proforma_invoice_2_docs` | `logistics.proforma.line` | ← `demo_line_docs`; billed 100 (full) |
| `profitability.demo_proforma_invoice_2_discount` | `logistics.proforma.line` | **Document-level discount** (WS-4 / D-14) — −100 USD carrying the seeded `DISCOUNT` charge product, a required reason, no source charge, and `sequence` 9000 so it always prints last |
| `profitability.demo_proforma_bill_1` | `logistics.proforma.document` | Vendor proforma **bill** → carrier; freight COST 2,400 (buy side, independent quota) = payable 2,400 USD |
| `profitability.demo_proforma_bill_1_freight` | `logistics.proforma.line` | ← `demo_line_freight`; billed 2,400 on the cost/buy side |
| `profitability.demo_proforma_bill_2` | `logistics.proforma.document` | Second vendor proforma **bill** → carrier; documentation cost = payable **60.24 USD**. Exists to prove the buy side is not a single-document special case, and to carry the **cross-currency** line below |
| `profitability.demo_proforma_bill_2_docs` | `logistics.proforma.line` | ← `demo_line_docs`; the **multi-currency** line — source 5,000 INR (`source_currency_id` INR, `source_unit_rate` 1.00) translated onto a USD document at `unit_rate` 0.012048, so `source_*` and the document-currency figures are both retained (WS-4.3 / PFM-10). This is the only demo line where document currency ≠ source currency |

### Operational tax (WS-5 / WS-13) — inputs only, computed on demand

This module ships **no tax rows of its own.** What the tax layer needs is a
*source* set on the charge products and a *jurisdiction map*, and both live in
other modules' demo files — so the demo contribution here is the wiring that
makes them meet, not new records.

| Where | What | Notes |
|---|---|---|
| `pricing/demo/demo_charge_codes.xml` (extended) | `tax_ids = [IN-GST-18]` on the six sea/ocean charge products — OF · BAF · THC-O · THC-D · DOC · CUS | The **source** set (D-10). A line's taxes resolve through the same `product_id` FK that already derives its `charge_code`, so the tax layer inherits Enhancement 01's single-identity guarantee rather than adding a second one. |
| `foundation.tax` demo | `tax.demo_fiscal_position_in_intra` · `_in_inter` · `_in_export`, and the India tax codes | The **map** (D-11). Ships over the same INMUM→SGSIN lane this catalogue prices, so no demo-only fixture is needed to exercise the remap. |
| `demo_proforma.xml` | `tax_amount = 0.00` on all four documents | **Deliberate.** Tax is not computed at demo load: `…apply_taxes` is an explicit operator command, not an automatic recompute on every line change (a draft edited line by line would pay the masters + fiscal-position chain on every keystroke). The demo ships a document in exactly the state an operator meets it — priced, grouped, untaxed — and the figure materialises on **Apply Tax**. |

Running Apply Tax on `demo_proforma_invoice_1` remaps each line's `IN-GST-18`
through the resolved position and writes the result to the line's `tax_ids` +
`tax_amount`, which the header totals roll up: intra-state yields
`[IN-CGST-9, IN-SGST-9]`, inter-state `[IN-IGST-18]`, both at the same 18% total
— the split is presentational, not arithmetical (D-11). Per **D-38** there is no
per-borne-tax breakdown row, no override and no snapshot to inspect afterwards;
the figure on the line and the header is the whole of the delivered output, and
it is **indicative, never statutory** (BR-20).

### `demo/demo_billing_readiness.xml` (WS-8) — the gate's configuration

| External ID | Model | Notes |
|---|---|---|
| `profitability.demo_config_billing_readiness_enforced` | `ir.config` | `proforma.billing_readiness_enforced = false` — the gate runs and reports on every grouping, but **advises rather than refuses**. Shipped explicitly rather than left to the code default so the setting is visible and switchable on the Settings → Proforma → Billing Readiness panel in a demo tenant. |
| `profitability.demo_config_zero_total_blocks` | `ir.config` | `proforma.zero_total_blocks = false` — a zero-total proforma warns rather than blocks at grouping (PFM-12). Same reasoning: make the pair of readiness dials real rows an operator can see and flip. |

**Why the gate is advisory in the demo, and why that is the point.** The scenario's
Globex shipment is deliberately mid-lifecycle — `status=created`,
`operational_status=not_started`, no POD — because the rest of the demo needs a
shipment you can still act on. Readiness therefore evaluates to **not ready**, and
that is the interesting state: `evaluate_readiness` returns a structured verdict
naming each failing check and what to do about it, rather than a bare `False`.
Turning enforcement on in demo data would make `create_from_charges` refuse, and
`demo_proforma.xml` — 13 records that depend on grouping succeeding — would fail
to load. The gate is shipped in the mode the scenario can actually demonstrate.

**No records of its own.** WS-8 is an evaluator: it reads shipment state, charges,
parties, the legal entity and the seeded `customer_billing` document gate, and
writes nothing. Its demo contribution is the two dials above plus the verdict the
existing records produce — there is no `logistics.*.readiness` table to seed. The
`customer_billing` gate point it invokes is a **production seed** shipped by
`logistics.documentation` (`logistics.document.gate.point.csv`, `is_enforced=true`),
not demo data, so the gate contributes a real result under `--with-demo=all`.

## Out of scope

- Actual invoice / vendor-bill sync, billing/cost status, variance, closure, exceptions (Phase 2 — finance integration).
- Master/console + leg-wise profitability and AI insights (Phase 2 / Phase 3).
- The adjustment is left `pending_approval` (illustrative) — the demo loader creates rows but does not run the apply engine, so no line mutation is implied.

## Dependencies

- `foundation.base` demo (`base.currency_usd`, `base.currency_inr`, `base.demo_partner_co_001`, `base.demo_org_west_branch`).
- `logistics.shipments` demo (`shipments.demo_shipment_globex_001`, `shipments.demo_shipment_charge_freight`).
- `logistics.sales_crm` demo (`sales_crm.demo_partner_customer_globex` — the invoice billing party). **WS-3 proforma demo only.**
- `demo/demo_proforma.xml` depends on `demo/demo_profitability.xml` (the charge lines it bills) — declared after it in the manifest.
- **`logistics.pricing` demo ([Enhancement 01](../../../../roadmap/logistics/profitability/enhancements/01-product-anchored-charge-identity.md) / D-20/D-21) — a hard demo→demo dependency.** Every charge line and every proforma line anchors its identity to a `logistics.product.master` row (`charge_code` is *derived* from `product_id`, and the tax layer resolves a line's taxes through the same FK), and those products ship in `pricing/demo/demo_charge_codes.xml`: `pricing.demo_charge_code_ocean_freight` (OF), `_thc_origin` (THC-O), `_thc_dest` (THC-D), `_documentation` (DOC), plus `_customs` (CUS) added by Enhancement 01. `--with-demo` loads **only the app keys it is given** — it does *not* pull a dependency's demo files (`data_loader/loader.py::load_demo_for_apps` skips any app not named in `app_keys`) — so `logistics.pricing` now joins the cross-app demo closure this module already required (`shipments` / `sales_crm` / `base`), and `--with-demo=all` remains the verification command. Verified 2026-07-29: with `--with-demo=logistics.pricing,logistics.profitability` alone, `demo_charge_codes.xml` loads 10 products but `demo_profitability.xml` loads **0** records — it cannot resolve `shipments.demo_shipment_globex_001`. The products stay demo rather than seed because a charge catalogue is deployment-specific.
- **`foundation.tax` demo (WS-5 / WS-13) — a demo→demo dependency for the tax path only.** The charge products' `tax_ids` reference the India tax codes, and the fiscal-position remap needs `tax.demo_fiscal_position_in_intra` / `_in_inter` / `_in_export`. Without that demo loaded, Apply Tax still runs and still succeeds — it resolves no position, maps nothing, and every line comes back with an empty `tax_ids` and zero tax, which is a valid answer and never an error (BR-20). `--with-demo=all` covers it.
- The WS-4 discount line references `masters.product_charge_discount` and `base.payment_term_net_30`, both of which are **production seeds** (`logistics.masters` / `foundation.base` `data/`), not demo. That is deliberate: `proforma.discount_product_code` defaults to the `DISCOUNT` code, so a deployment that never loads demo data still gets a working discount line. The same reasoning seeds `masters.product_charge_adjustment` (`ADJ`, Enhancement 01 / D-22), which `services/adjustment.py` resolves by code for every manual-adjustment charge line.

## Verification

`ede migrate upgrade -t <tenant> --with-demo=all` (the proforma demo refs
cross-app `pricing` / `sales_crm` / `shipments` demo records, so the full closure
is loaded). `logistics.pricing` is now part of that closure (Enhancement 01): without it
every `pricing.demo_charge_code_*` product reference is unresolved.
- Every charge line and proforma line resolves a real `logistics.product.master`
  row, so `charge_code` reads back derived (OF / THC-O / THC-D / CUS / DOC / BAF /
  DISCOUNT) with no column behind it (Enhancement 01 / D-20).
- `demo_profitability.xml` — **11** rows (header + 6 lines + currency + adjustment + 2 audit).
- `demo_proforma.xml` — **13** rows (4 `logistics.proforma.document` + 9 `logistics.proforma.line`, one of which is the WS-4 document-discount line).
- Header roll-up: revenue 4400 / est cost 3190.24 / gross profit 1209.76 / margin 27.49% (PRF-BR-02 — grouping never alters it; the figures moved because the BAF charge was added, not because billing changed them).
- After load, `demo_line_freight` reads `billing_status=fully_invoiced`, `remaining_quantity=0` (2,000 + 1,200 across the two invoices); the invoice document totals read 2,550 / 1,500 (invoice 2 after its 100 document discount); the bills 2,400 and 60.24.
- Re-run idempotent (`updated`, not `created`).
- **Tax (WS-5 / WS-13):** all six sea/ocean charge products read back `tax_ids = [IN-GST-18]`, and the three India fiscal positions are present from `foundation.tax`'s demo. The four proforma documents load with `tax_amount = 0.00` **by design** — dispatch `logistics.proforma.document.apply_taxes` on `demo_proforma_invoice_1` to resolve the position and populate each line's mapped `tax_ids` + `tax_amount` into the header totals. The command is idempotent: running it twice on an unchanged document is a no-op, not a doubling.
- Last verified 2026-07-29 on tenant `btntest2` (WS-4.1 button smoke): `--with-demo=all` → demo_load **441** created / 0 errors; `demo_profitability.xml` 11 created, `demo_proforma.xml` 13 created, `sales_crm/demo_partners.xml` 15 created. Dispatching `logistics.proforma.document.create_from_profitability` on the demo header **succeeds end-to-end**: it bills the BAF line (250.00), skips the other five naming `already fully billed (BR-09 / PFM-21)` per charge, and returns a draft invoice stamped with the Globex billing address, Net 30 payment term (`payment_terms_days=30`) and `base.demo_org_west_branch`. This required three PFM-08 prerequisites the demo had never shipped — partner-level `res.partner.role` assignments (distinct from the shipment's `logistics.shipment.party` roles, which check 3 does **not** read), a billing `res.partner.address`, and a `payment_term_id` on the partner — all added to `logistics.sales_crm`'s demo, which owns the partners.
- Last verified 2026-07-29 on tenant `ws042smokeall` (Enhancement 01): `--with-demo=all` → demo_load 430 created / 0 errors; `demo_profitability.xml` 10 created, `demo_proforma.xml` 11 created; `orphan_cleanup` ran (not skipped, i.e. no load errors). All **6** charge lines and all **9** proforma lines carry a non-null `product_id`, deriving OF / THC-O / THC-D / CUS / DOC and DISCOUNT on the document-discount line. `ADJ` and `DISCOUNT` present as standard-locked **seed** products.
- Last verified 2026-07-27 on tenant `ws3final`: 10 created → re-run 0 created / 10 updated; totals 2,550 / 1,600 / 2,400; header roll-up 4,150 / 3,010.24 / 1,139.76; all four proforma direction record rules `active`.

## Recorded e2e tests

| Test | Covers |
|---|---|
| [`tests/e2e/test_proforma_reachability.py`](../../../../src/domains/logistics/profitability/tests/e2e/test_proforma_reachability.py) | WS-3 — both proforma menus reachable under Shipments → Profitability; each action's `document_type` domain keeps the other direction out of its list; invoice form renders the derived-totals block + Lines tab; the grouping command is routable. **5 passed** (2026-07-27, chromium). Its fixture loads the full demo closure — a hand-picked subset breaks the ref chain (the shipments demo shipment refs a booking demo record). |

## Authoring order

`demo_profitability.xml`: header → charge lines → currency line → adjustment →
audit entries. `demo_proforma.xml`: PINV-1 doc → its lines → PINV-2 doc → its
lines → PBILL-1 doc → its line (the freight line on PINV-1 must precede PINV-2's
so the split guard has the prior 2,000 to measure against). Manifest:
`demo: ["demo/demo_profitability.xml", "demo/demo_proforma.xml"]`.

---

*Back to [demo-usecase index](../../README.md).*
