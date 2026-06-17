# `logistics.shipments` — Demo Use-Case

**Module:** `domains.logistics.shipments`
**App key:** `logistics.shipments`
**Demo manifest entries** (target): `demo/demo_shipment.xml`
**Status:** ✅ Delivered (2026-06-17) — postgres `--with-demo` smoke green: 8 records (1 shipment + 1 leg + 4 parties + 1 cargo + 1 charge), composed `shipment_ref` `CO-SEA-D-2026-000001`, idempotent re-run. Note: full `--with-demo` load is postgres-only (upstream booking/pricing/sales-crm demos ship date strings the sqlite adapter rejects — pre-existing).

---

## Use-case

The operational backbone for the unifying scenario's **Globex Inc.** ocean export
(Mumbai → Singapore, FCL). After the Globex booking is confirmed
(`booking.demo_booking_globex_001`), operations creates the shipment file — the
long-lived record that owns the journey, parties, cargo, charges, and the single
equipment-usage binding through its leg. The demo seeds one direct, single-mode
shipment in the `created` state with its single leg, the three mandatory parties
(shipper / consignee / billing) plus the main-carriage carrier, one cargo line
(2×40HC cotton garments), and the frozen ocean-freight charge carried forward
from the booking. It illustrates the booking→shipment no-re-keying handoff and
the classification + ownership + routing surface a fresh tenant sees immediately
on the Shipments app.

## Records produced

### `demo/demo_shipment.xml`

| External ID | Model | Notes |
|---|---|---|
| `shipments.demo_shipment_globex_001` | `logistics.shipment` | Direct single-mode ocean shipment from `booking.demo_booking_globex_001`; Mumbai → Singapore; `status=created`, composed `shipment_ref` auto-stamped by the numbering engine |
| `shipments.demo_shipment_leg_001` | `logistics.shipment.leg` | Single sea leg (sequence 1), `leg_status=planned` |
| `shipments.demo_shipment_party_shipper` | `logistics.shipment.party` | Globex Inc. as shipper |
| `shipments.demo_shipment_party_consignee` | `logistics.shipment.party` | Stark Industries as consignee |
| `shipments.demo_shipment_party_billing` | `logistics.shipment.party` | Globex Inc. as billing party |
| `shipments.demo_shipment_party_carrier` | `logistics.shipment.party` | Main-carriage carrier |
| `shipments.demo_shipment_cargo_001` | `logistics.shipment.cargo.line` | 2×40HC cotton garments, 18 t / 128 CBM |
| `shipments.demo_shipment_charge_freight` | `logistics.shipment.charge` | Frozen ocean-freight charge (buy 2400 / sell 3200 USD) |

## Out of scope

- Master/house structure + multi-leg journeys (Phase 2).
- Reports / KPIs / milestone templates (Phase 3).
- POD records, attachments, and amendments — exercised by the command surface +
  e2e flows, not pre-seeded (the demo shipment stays at `created`).
- The numbering-rule master + partner-role masters are **production seeds**
  (`data/`), not demo records.

## Dependencies

- `logistics.booking` demo (`booking.demo_booking_globex_001`) — the source booking.
- `logistics.sales_crm` demo (`sales_crm.demo_partner_customer_globex`, `_stark`) — parties.
- `logistics.masters` demo (`masters.demo_port_inmum`, `masters.demo_port_sgsin`,
  `masters.transport_mode_sea`) — routing + mode.
- `foundation.base` (`base.default_organization`, `base.currency_usd`,
  `base.demo_partner_co_001`) — org, currency, carrier party.
- Production seeds (`data/`): the default numbering rule + the carrier partner-role.

## Verification

`ede migrate upgrade -t <tenant> --with-demo=logistics.shipments` — expected
`8 created, 0 updated, 0 skipped`. Re-run is idempotent (`8 updated`). The
shipment row carries a composed `shipment_ref` (e.g. `…-SEA-D-2026-000001`) and
is reachable in the Shipments app list/form.

## Authoring order

`demo/demo_shipment.xml`: shipment header first (others `ref=` it), then leg,
parties, cargo, charge. Manifest `demo: ["demo/demo_shipment.xml"]`.

---

*Back to [demo-usecase index](../../README.md).*
