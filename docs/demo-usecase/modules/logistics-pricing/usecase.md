# `logistics.pricing` — Demo Use-Case

**Module:** `domains.logistics.pricing`
**App key:** `logistics.pricing`
**Demo manifest entries** (target): `demo/demo_rates.xml`
**Status:** 🔴 Not yet authored (module itself 🟡 In Progress — see [roadmap/logistics/pricing/](../../../../roadmap/logistics/pricing/))

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

*Back to [demo-usecase index](../../README.md).*
