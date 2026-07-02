# `logistics.pricing` — Demo Use-Case

**Module:** `domains.logistics.pricing`
**App key:** `logistics.pricing`
**Demo manifest entries:** `demo/demo_customization.xml`, `demo/demo_vendors.xml`, `demo/demo_rules.xml`, `demo/demo_charge_codes.xml`, `demo/demo_rates.xml`, `demo/demo_spot_rfqs.xml`, `demo/demo_phase2.xml`
**Status:** ✅ Delivered (2026-07-02) — Phase 1 rate/contract demo + Phase 2 demo (`demo_phase2.xml`: version history, branch override, contract volume-utilization) shipped and smoke-tested.

---

## Use-case

Ship **three demo rates** that exercise the full pricing → approval → active lifecycle on the unifying scenario lane:

1. `pricing.demo_rate_inmum_sgsin_lcl` — Mumbai → Singapore LCL, status `pending_approval` (so the approval inbox demo has a real subject).
2. `pricing.demo_rate_sgsin_inmum_lcl` — Singapore → Mumbai LCL, status `active` (so the rate-card demo screen has a fully-approved record).
3. `pricing.demo_rate_inmum_sgsin_fcl_draft` — Mumbai → Singapore FCL 20 ft, status `draft` (so the sales rep can see a draft awaiting submission).

Each rate carries a small line set (base ocean freight + 2–3 surcharges) so the line-form demo is meaningful, and one rate carries a value for the custom `vendor_code` property defined by `foundation.customization` demo.

## Records produced

### `demo/demo_rates.xml`

| External ID | Model | Status | Lane | Notes |
|---|---|---|---|---|
| `pricing.demo_rate_inmum_sgsin_lcl` | `pricing.rate` | `pending_approval` | INMUM → SGSIN | currency=USD, valid_from=today, valid_to=today+90d, owner=`demo_user_ops_manager`, `properties={"vendor_code":"ACME-OPS-001"}` |
| `pricing.demo_rate_sgsin_inmum_lcl` | `pricing.rate` | `active` | SGSIN → INMUM | currency=USD, valid_from=today-7d, valid_to=today+83d |
| `pricing.demo_rate_inmum_sgsin_fcl_draft` | `pricing.rate` | `draft` | INMUM → SGSIN | currency=USD, equipment=`20GP` |

Plus 2-3 `pricing.rate.line` rows per rate (base freight + BAF + THC).

A chatter note (`communication.demo_msg_rate_inmum_sgsin_submit`) is appended on the first rate at submit time — declared inline in this XML rather than in `foundation.communication/demo` because the rate UUID is only known after this file runs.

A notification row (`notifications.demo_inbox_rate_pending_ops`) is appended in this XML for the same reason.

## Out of scope

- Bulk rate-upload demo — Phase 2 of pricing (upload UI).
- Margin / MSP-floor / dedupe-hash demo — those engines fire automatically on the rates above; no separate seed records needed.
- Multi-currency conversion demo — Phase 2.
- Multi-leg / segment rate — Phase 2.

## Dependencies

- `foundation.base` demo (users, partners — owner + customer references)
- `logistics.masters` demo (ports, equipment)
- `foundation.customization` demo (property definition `vendor_code` for the first rate)
- Production data: `res.currency.USD`, `res.uom.*`

## Verification

```
ede migrate upgrade -t demo --with-demo=all
```

Settings → Pricing → Rate list shows three demo rates in three different statuses. Click the pending one → header is locked, chatter shows submission note, approvals inbox shows the case for ops_manager.

## Authoring order

1. Masters demo loaded (ports, equipment present).
2. `demo_rates.xml` — header records first, then lines, then chatter, then notification rows.

Manifest:
```python
"demo": [
    "demo/demo_rates.xml",
],
```

---

## Phase 2 demo records (`demo/demo_phase2.xml` + a field on the Maersk contract)

| External ID | Model | Demonstrates | Notes |
|---|---|---|---|
| `pricing.demo_rate_version_globex_v1` | `pricing.rate.version` | WS4/WS5 — version compare, rollback, VER-010 audit | v1 snapshot of the Globex sell rate (ocean 1100 + doc 75) |
| `pricing.demo_rate_version_globex_v2` | `pricing.rate.version` | WS4/WS5 | v2 after a GRI (ocean 1100→1250); `change_summary` populated → the compare view shows the delta, rollback restores v1 |
| `pricing.demo_rate_override_air_branch` | `pricing.rate` | WS2 / ME-007 | branch tariff override of the global air sell rate, linked via `override_of_rate_id`, `branch_ids=[base.default_organization]` |
| `pricing.demo_rate_line_override_air_freight` | `pricing.rate.line` | WS2 | the override's sharper branch-local air freight line |
| `pricing.demo_contract_maersk_sea` (field) | `pricing.contract` | WS3 / CON-005 | `current_utilization=17.6` of 20 MT = 88% → the sweep raises a one-time 80%-band owner alert |

### Demo-exempt Phase 2 items (documented skips)

- **WS1 branch visibility** — enforced by `ir.rbac.record.rule` rows shipped as **`data/` seeds** (load for every tenant); visibility is a behaviour, not a sample record. The demo override above exercises the branch-scoping fields.
- **WS7 configurable margin thresholds (PM-008)** — the thresholds + `pricing.branch_sharing_mode` are **settings whose panel defaults are the config**; no sample row needed. Change them under Settings → Pricing to demo.
- **WS8 quote predictive-margin snapshot (PM-011)** — the snapshot lands on the **auto-created** `crm.quote.version` (no external ID), which the single-pass demo loader cannot cross-ref — the same documented limitation as quote charge lines. Captured at runtime via `crm.quote.version.capture_predictive_margin` from the rate picker.

## Phase 2 use-cases (in addition to Phase 1 above)

The Phase 2 backend lands these new flows on top of the Phase 1 record set.
The deep behaviour is covered by unit tests; the end-to-end smoke surface
(under `src/domains/logistics/pricing/tests/e2e/`) proves the wire still
works.

| Flow | Trigger | Outcome | E2E coverage |
|---|---|---|---|
| Recalculate predictive margin | `env.dispatch(pricing.rate.recalculate_predictive_margin, model_id=<rate>)`, or implicitly via post-create / post-amend / line CRUD hooks | `PredictiveMarginService.evaluate(rate_id)` writes back `predictive_margin_amount`, `predictive_margin_percent`, `margin_risk_level`, `risk_reason` on the rate row | `test_pricing_smoke.py::test_recalculate_predictive_margin_command_is_registered` |
| Publish to branches (ME-006) | `env.dispatch(pricing.rate.publish_to_branches, model_id=<rate>, payload={"branch_ids": [...]})` — requires `pricing.rate:publish_to_branches` permission | `branch_ids` M2M on the rate is set to the supplied org list; unauthorised principals get a PermissionError | `test_pricing_smoke.py::test_publish_to_branches_command_is_registered_and_guarded` |
| Direct-update branch_ids veto | Any other path that tries to write `branch_ids` (e.g. a generic `ede.update` from the form) | `pre.pricing.rate.update` hook rejects unless the principal holds `pricing.rate:publish_to_branches` — the publish command is the one authorised entry point | Same smoke test (the veto fires via the unauthorised-dispatch path) |

The smoke entry point — runs headed locally:

```
pytest src/domains/logistics/pricing/tests/e2e/test_pricing_smoke.py --headed -s
```

---

*Back to [demo-usecase index](../../README.md).*
