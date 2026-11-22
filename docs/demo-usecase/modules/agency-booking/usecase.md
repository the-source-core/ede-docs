# Demo Use-Case — `logistics_nvocc_agency.agency_booking`

**Module:** `agency_booking` (BRD N3 — Customer Booking / CRO / Release Order)
**Demo file:** `src/domains/logistics_nvocc_agency/agency_booking/demo/demo_agency_booking.xml`
**Scenario:** Acme Forwarding Ltd. (Mumbai HQ) — see [the unifying scenario](../../README.md)

---

## The story this data tells

Acme does not only forward freight; it also acts as **local shipping agency**. BRD N1 gave
it two principals to act for. This module adds the third and the first *transaction* under
one of them:

> **Sealink Container Line** is an NVOCC brand Acme represents out of Nhava Sheva. Globex
> Inc. books two 40ft dry containers with Acme under Sealink, Nhava Sheva → New York. The
> boxes will be collected from the Nhava Sheva Empty Container Depot, by Globex's own
> transporter, inside a one-week window.

Everything a release order needs is on the tenant after this demo loads. The release order
itself is the reviewer's first act — see *What this demo deliberately does not ship*.

---

## Records shipped (13)

| # | Record | Model | Notes |
|---|---|---|---|
| 1 | `demo_config_manual_booking` | `ir.config` | `nvocc.manual_booking_allowed = true`, system scope. **Demo only** — off by default in a real tenant, and the booking below would then be refused, which is what N3-02 intends |
| 2 | `demo_binding_admin_container_control` | `ir.rbac.role.binding` | Admin ↔ `agency_container_control`. Needed because `AssignmentService` resolves a ROLE approval step by exact code and does **not** walk role inheritance |
| 3 | `demo_binding_admin_operations_admin` | `ir.rbac.role.binding` | Admin ↔ `agency_operations_admin`, for the amendment approval step |
| 4a | `demo_entitlement_admin_sealink` | `nvocc.principal.entitlement` | Admin, branch-agnostic, `full`. **Without an entitlement the principal is unusable** — `PrincipalScope`'s create guard refuses any record tagged to a principal the caller holds no grant for (BR-NAM-02), so every command refuses long before reaching the control it was meant to exercise. A live run is what surfaced this |
| 4b | `demo_entitlement_priya_sealink` | `nvocc.principal.entitlement` | Priya, branch-scoped, `booking` — the desk case, and the one that shows an access level narrower than `full` |
| 4 | `demo_principal_sealink` | `nvocc.principal` | `PRN-SEALINK`, type `nvocc`, prefix `SLK`. Lands in **draft** — `status` is workflow=True |
| 5 | `demo_facility_nsict_depot` | `logistics.facility.master` | `DEP-NSICT`, the empty depot the release is drawn from |
| 6 | `demo_numbering_rule_cro_sealink` | `nvocc.numbering.rule` | `cro` × Sealink × default org → the `CRO` sequence, `prefix_override = SLK-CRO`. Lands in **draft** on purpose (see below) |
| 7 | `demo_booking_sealink_001` | `logistics.booking` | Principal-tagged, manual (no quote), Nhava Sheva → New York, vessel + voyage on the header so it can confirm from `draft` |
| 8–10 | `demo_booking_party_customer` / `_consignee` / `_pickup` | `logistics.booking.party` | Globex as customer **and** as pickup party; Stark as consignee. Uses the `pickup_party` role this phase added to `logistics_base.booking` |
| 11 | `demo_booking_line_sealink` | `logistics.booking.line` | 2 × 40GP, FAK commodity, 21.5 t, 40 pallets, HS code — complete enough to satisfy the readiness condition set |

---

## The walkthrough

1. **Agency → Setup → Principals → Sealink** — `nvocc.principal.submit` opens a
   maker-checker case; approve it on the approval task surface; the principal becomes
   `active`.
2. **Agency → Configuration → Numbering Rules → the Sealink CRO rule** — submit and
   approve it. *Until it is `active`, issuing a release order is refused naming this rule
   (BR-NAM-12).* Walk into that refusal once on purpose: it is the control, not a defect.
3. **Bookings → the Sealink booking → Confirm.** The three confirmation gates run here.
   Tick `credit_hold` on Globex first and watch the refusal name it *and* any other
   standing hold in the same message — then untick and confirm.
4. **Agency → Operations → Container Release Orders → Create.** Pick the booking; the
   principal, customer and branch are inherited and immutable. Add a line for 2 × 40GP
   from the depot, then `submit` → `issue`. Try this **before** step 3 to see BR-N3-06
   refuse it.
5. **Record a communication, then `send`.** Sending is refused while the communication log
   is empty (BR-N3-16).
6. **`record_release` with three usages against a line authorising two** — refused.
7. **Shorten `valid_until` to yesterday, run `nvocc.cro.expiry.sweep`.** The stage follows
   the date — and a collection was already refused *before* the sweep ran, because the
   guard reads the dates, not the stage.

---

## What this demo deliberately does NOT ship

- **No container release order.** BR-N3-06 refuses a CRO against an unconfirmed booking,
  and `logistics.booking.status` is `workflow=True`, so demo data cannot confirm it. A
  demo CRO would only be possible by bypassing the control this phase exists to build —
  a fixture that walks around its own control demonstrates a system that does not exist.
  Creating it is step 4, and creating it is the point.
- **No `active` principal or numbering rule.** Same reason: both are `workflow=True`.
  Driving them forward is steps 1 and 2, and it exercises the real lifecycle including
  the approval bridge.
- **No equipment usage rows.** Collection is BRD N4 / Phase 4 work. Step 6 uses whatever
  usages the `logistics_base.equipment_operations` demo already provides.
- **No second Maersk principal.** `agency_reports` loads *last* (its blueprints need every
  other module's `ir.model` rows), so its demo principals are not resolvable from here —
  and duplicating `PRN-MAERSK` would collide on `principal_code`.

---

## Cross-module references consumed

| Ref | Owner | Used for |
|---|---|---|
| `base.default_organization` | foundation.base | Issuing branch on the booking and the numbering rule |
| `base.admin_user` | foundation.base | Subject of the two demo role bindings |
| `base.currency_usd` | foundation.base | Booking currency |
| `base.country_sg` | foundation.base | Principal country of registration (required on the delegated `res.partner`) |
| `masters.transport_mode_sea` | logistics_base.masters | Ocean mode |
| `masters.demo_port_innsa` / `masters.demo_port_usnyc` | logistics_base.masters | POL / POD, and the depot's UN/LOCODE |
| `masters.eq_type_42gp` | logistics_base.masters | 40ft dry container type |
| `masters.demo_commodity_fak` | logistics_base.masters | Commodity on the booking line |
| `sales_crm.demo_partner_customer_globex` / `_stark` | logistics_base.sales_crm | Customer / pickup party, consignee |
| `sales_crm.role_master_customer` / `_consignee` | logistics_base.sales_crm | Booking party roles |
| `booking.role_master_pickup_party` | logistics_base.booking | The pickup-party role added by this phase |
| `agency_numbering.sequence_cro` | logistics_nvocc_agency.agency_numbering | The `CRO` number series the rule binds to |

---

## Smoke test

```bash
ede migrate upgrade -t <tenant> --with-demo=logistics_nvocc_agency.agency_booking
```

Expect **13 records** created on a fresh tenant; a re-run is idempotent. The cross-module refs above mean this must run as part of
`--with-demo=all`, or after the `foundation.base` / `logistics_base` demos — on its own
they cannot resolve.

Then check **Agency → Operations → Container Release Orders** is reachable and empty, and
**Bookings** shows the Sealink booking in `draft` with its principal, its three parties and
its cargo line.

---

## Enhancement 01 — demo-data expansion

**Status:** ✅ Delivered 2026-11-22 — 19 records in 3 files; `--with-demo=all` smoke green and idempotent (see the end of this section).
**Roadmap:** [nvocc-agency Enhancement 01](../../../../roadmap/logistics/nvocc-agency/enhancements/01-demo-data-expansion.md). The cross-module walkthrough order lives there, and step IDs (`UC1-…`) match it.
**New demo files** (appended to `demo: [...]` **after** `demo/demo_agency_booking.xml`, whose Sealink principal and `nvocc.manual_booking_allowed` row they rely on):
`demo/demo_booking_bluewater_import.xml` (9) · `demo/demo_booking_special_cargo.xml` (6) · `demo/demo_booking_credit_hold.xml` (4)

This module is the lowest point every later file is guaranteed to load after. Data load is
topological over manifest `depends`, with an alphabetical tiebreak. `agency_documentation`,
`agency_charges` and `agency_voyage` each declare `agency_booking` in `depends`
(documentation's edge was added by this enhancement), and `agency_release` inherits it through
documentation. **`agency_charges` is not ordered relative to documentation or release** by
`depends`; under the tiebreak it can load before both. So the import's **containers, usages and
seals** ship here, not in the module that first prints them.

The admin's Bluewater entitlement is **not** in this module. It ships from `agency_masters`
(`agency_masters.demo_entitlement_admin_bluewater`), which owns both the entitlement model and
the principal. Before it, admin held Maersk and Sealink grants only, so every Bluewater command
would have been refused by PrincipalScope (BR-NAM-02) before reaching the control it was meant
to show.

### Use case 1 — Bluewater import into Nhava Sheva (the booking end)

> **Bluewater NVOCC** (`PRN-BLUEWATER`, seeded by `agency_masters` and unused until now) ships two
> 40ft dry boxes of LED driver boards from **Lion Asia Trading**, Singapore, to **Wayne Imports**,
> Mumbai. They travel on *Maersk Madrid* voyage **2614W**, discharging at Nhava Sheva on
> 6 October 2026. Acme is Bluewater's destination agent.

A B/L cannot exist without a booking (BR-N5-01), so the origin office's booking is recorded here.
This follows the release demo's precedent: an agency works both ends of its own line. The booking
**stays in draft**. Nothing at the import end reads its status, because B/L issue validates the
B/L, not the booking.

#### Records produced — `demo/demo_booking_bluewater_import.xml` (9)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_booking.demo_booking_bluewater_import` | `logistics.booking` | `principal_id` Bluewater · `manual_booking_reason` · `masters.demo_port_sgsin` → `masters.demo_port_innsa` · `vessel_name` MAERSK MADRID · `voyage_number` 2614W · `cargo_ready_date` 2026-09-26 · `expected_etd` 2026-10-01 · `expected_eta` 2026-10-06 · `base.currency_usd` · `equipment_availability=available` | The booking BR-N5-01 requires behind the import B/L. Draft on purpose |
| `agency_booking.demo_booking_party_bluewater_customer` | `logistics.booking.party` | `sales_crm.demo_partner_customer_wayne` · `sales_crm.role_master_customer` | On collect terms the importer is the customer at this end: the party D&D bills, and whose credit position release readiness reads |
| `agency_booking.demo_booking_line_bluewater_import` | `logistics.booking.line` | 2 × `masters.eq_type_42gp` · `masters.demo_commodity_electronics` · "LED driver boards" · HS 8504.40 · `carton` 960 · 18 400 kg · 110 m³ | The cargo totals the B/L repeats |
| `agency_booking.demo_equipment_bluewater_box_1` / `_box_2` | `logistics.equipment` | `BLWU4200308` / `BLWU4200313` (valid ISO 6346 check digits) · 42GP · `principal_id` Bluewater · `custodian_organization_id` default org · `lease_type=owned` · `masters.eq_cond_good` | Own boxes rather than shared fixtures ([README rule](../../README.md#shared-fixtures-that-are-deliberately-unavailable)). Box 2 is the one customs examines |
| `agency_booking.demo_usage_bluewater_box_1` / `_box_2` | `logistics.equipment.usage` | default org · 2026-09-25 → 2026-11-05 | A B/L container line, a DO line, a discharge-list line and every movement event hang off the usage |
| `agency_booking.demo_seal_bluewater_box_1` / `_box_2` | `logistics.equipment.seal.application` | `BLW-771201` / `BLW-771202` · `masters.seal_type_bolt` · `seal_slot=primary` · applied 2026-09-29 (Singapore stuffing) | B/L issue blocks on `seal_missing`. These sit in a **`noupdate="1"` block**: seal rows refuse every update, which is the defect the documentation smoke run found |

### Use case 2 — Special cargo: reefer + DG (the booking end)

> Globex books two special-cargo shipments under Sealink on **SLV-2649E**, Nhava Sheva →
> Singapore, consigned to Stark. One is a 40ft reefer of frozen shrimp carried at **−18 °C**; the
> other is a 20ft of flammable industrial paint, **UN 1263, class 3**. Both bookings are correct.
> What goes wrong goes wrong later, on the customer's two shipping instructions: the DG one drops
> the UN number, the reefer one drops the set-point (see the `agency_documentation` section).

There is no reefer flag. The line is temperature-controlled because `temperature_setting_c` is
set (the base model's own Q4 note), and the box is a reefer because its equipment type is 42RT.
`nvocc.cro.create_from_booking` copies **equipment type and quantity only**, because a release
order authorises empty equipment. So the set-point reaches the depot through the CRO line's
`special_conditions`, which the desk types in. The walkthrough makes that gap visible rather than
hiding it in a fixture.

#### Records produced — `demo/demo_booking_special_cargo.xml` (6)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_booking.demo_booking_reefer_sealink` | `logistics.booking` | Sealink · manual · INNSA → `masters.demo_port_sgsin` · SEALINK VOYAGER / SLV-2649E · cargo ready 2026-12-01 · ETD 2026-12-08 · ETA 2026-12-14 · USD | **Future dates on purpose**: a CRO's validity window is checked against today, and a September sailing would make every collection step refuse on expiry. **They expire around 2026-12-08**; after that the CRO walkthrough refuses on expiry until the dates are moved forward |
| `agency_booking.demo_booking_party_reefer_customer` | `logistics.booking.party` | Globex · `role_master_customer` | With no pickup party named, the CRO is issued to the customer (N3-15) |
| `agency_booking.demo_booking_line_reefer` | `logistics.booking.line` | 1 × `masters.eq_type_42rt` · `masters.demo_commodity_perishable` · "Frozen shrimp, IQF, glazed" · HS 0306.17 · `carton` 1 200 · 24 000 kg · 58 m³ · `temperature_setting_c` −18 · `temperature_min_c` −20 · `temperature_max_c` −16 · `special_handling_notes` "Pre-trip inspection before release; genset on the road leg; plug in within 2 h of gate-in" | The set-point and the tolerance band, as two distinct facts |
| `agency_booking.demo_booking_dg_sealink` | `logistics.booking` | Same sailing and lane as the reefer | The booking the DG shipping instruction hangs off |
| `agency_booking.demo_booking_party_dg_customer` | `logistics.booking.party` | Globex · `role_master_customer` | — |
| `agency_booking.demo_booking_line_dg` | `logistics.booking.line` | 1 × `masters.eq_type_22gp` · "Paint, flammable (industrial coatings in steel drums)" · HS 3208.90 · `drum` 80 · 16 000 kg · 28 m³ · `hazardous=true` · `hazmat_un_number` 1263 · `hazard_class` 3 · `stackable=false` | **Correct on purpose.** UN and class are both present, so allocation readiness passes. The error is on the SI, which is the document the B/L is built from |

### Use case 3 — A credit-hold customer refused at confirmation

> **Konkan Agro Exports Pvt Ltd**, a Navi Mumbai exporter of dehydrated onion, books one 40ft box
> under Sealink to Singapore. Finance put them on hold last week: two invoices are past 60 days.
> The booking is complete, and confirming it is refused with the reason quoted.

#### Records produced — `demo/demo_booking_credit_hold.xml` (4)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_booking.demo_partner_konkan_credit_hold` | `res.partner` | "Konkan Agro Exports Pvt Ltd" · `member_type=company` · `base.country_in` · `code` KONKAN-AGRO · email · **`credit_hold=true`** · `credit_hold_reason` "Two invoices past 60 days (INR 4.2 lakh) — block placed by Accounts 2026-11-12; release on receipt of payment" | A **new** party, part of the proposed scenario extension. It ships here because `credit_hold` is this module's extension column, and no demo file in the repo writes another module's record. A deliberately-blocked fixture, so it gets a row in the README's unavailable-fixtures table |
| `agency_booking.demo_booking_konkan_credit_hold` | `logistics.booking` | Sealink · manual · INNSA → SGSIN · SEALINK VOYAGER / SLV-2649E · vessel and voyage on the header so it can confirm straight from draft | Complete in every other respect, so the refusal has exactly one cause (two if Sealink is still draft) |
| `agency_booking.demo_booking_party_konkan_customer` | `logistics.booking.party` | Konkan · `role_master_customer` | The hold gate reads the customer through this role row |
| `agency_booking.demo_booking_line_konkan` | `logistics.booking.line` | 1 × 42GP · `masters.demo_commodity_fak` · "Dehydrated onion flakes" · HS 0712.20 · `bag` 1 000 · 20 000 kg · 60 m³ | — |

**Block split in the two files above.** The reefer, DG and Konkan **booking headers** sit in
`noupdate="1"`. Once the walkthrough confirms a booking, its route and mode are locked
(BR-BK-04), so a reload of an updatable header would fail. Their parties and lines carry no such
lock and stay updatable. The Konkan **party** stays updatable on purpose, so a reload re-ticks its
credit hold. The Bluewater import booking stays updatable too, because the walkthrough never
confirms it. **Known gap, unchanged:** the older Sealink booking in `demo_agency_booking.xml` is
updatable, so re-running `--with-demo` after its own walkthrough confirms it would hit the same
lock.

### What this slice deliberately does NOT ship

| Not shipped | Why |
|---|---|
| A confirmed booking, or any CRO | `logistics.booking.status` and `nvocc.cro.status` are `workflow=True`, and BR-N3-06 refuses a CRO against an unconfirmed booking |
| A temperature on a CRO | There is no such column, by design. The walkthrough types the set-point into `special_conditions` |
| A free-time term on the Bluewater booking | The company defaults (DEM 5 days, DET 7) are what the import is meant to show. The `agency_charges` extension adds days on top of a default, and the existing Sealink file already shows `customer_contract` beating the standard |
| A LOADED event at Singapore | The origin load happened outside this tenant. Under the default `warn` sequence mode the first discharge carries a sequence warning, which is the honest outcome |
| A credit hold on Globex or Wayne | Both drive other walkthroughs. A seeded hold would break every one of them |

### Walkthrough — the steps this module owns

| Step | Do this | What it shows |
|---|---|---|
| UC3-1 | Bookings → the Konkan booking → **Confirm** | **Refused** (verified live): *"booking cannot be confirmed — 1 blocking condition: Konkan Agro Exports Pvt Ltd: the customer is on credit hold — Two invoices past 60 days (INR 4.2 lakh). Block placed by Accounts 2026-11-12; release on receipt of payment."* If Sealink is still draft, the principal refusal arrives **in the same message**, because the gates collect every standing reason. A row lands in `nvocc.principal.usage.refusal` with `refusal_type=booking_hold` |
| UC3-2 | Clear Credit Hold on Konkan (a finance user mirroring the accounting system), then Confirm again | Passes. Nobody overrode anything; the mirror changed |
| UC2-1 | Confirm the reefer booking and the DG booking | Both pass: complete cargo, an active principal, a clean customer |
| UC2-2 | Switch the principal switcher to **Sealink** first: with `nvocc.principal_mandatory` on, a create from a booking is refused without an active principal context (*"No active principal… (BR-NAM-01)"*, seen live on the B/L equivalent). Then `nvocc.cro.create_from_booking` on the reefer booking, passing `pickup_facility_id` = DEP-NSICT (the one field the handler leaves for the desk) | A draft CRO with one 1 × 42RT line and **no temperature on it**. Write "Set point −18 °C, band −20/−16, vents closed, PTI certificate required, genset for road haul" into the line's Special Conditions, then submit → issue (Sealink CRO rule) |
| UC2-3 | Try `nvocc.cro.send`, then log a communication and send again | Refused while the communication log is empty (BR-N3-16); then sent. The depot now holds the set-point in writing |

### Dependencies and load order

`agency_masters` (Bluewater), `logistics_base.sales_crm` (Globex, Stark, Wayne, the role masters),
`logistics_base.masters` (ports, equipment types, commodities, seal type) and `foundation.base`
(org, admin, currencies, countries). Within this module, the existing `demo_agency_booking.xml`
loads first. Run as part of `--with-demo=all`: `--with-demo` takes app keys literally and does not
expand along `depends`.

### Smoke test — ✅ green and idempotent, 2026-11-22

`ede migrate upgrade -t agencydemo3 --with-demo=all` on a fresh PostgreSQL tenant, run twice, exit
0, zero loader errors. Actual counts match the prediction exactly:

| File | Pass 1 | Pass 2 |
|---|---|---|
| `demo_booking_bluewater_import.xml` | created=9 | created=0 · updated=7 · skipped=2 (seals) |
| `demo_booking_special_cargo.xml` | created=6 | created=0 · updated=4 · skipped=2 (booking headers) |
| `demo_booking_credit_hold.xml` | created=4 | created=0 · updated=3 · skipped=1 (booking header) |

All 19 external ids are in `ir_data_reference` with `is_demo = true`. The existing
`demo_agency_booking.xml` still reads `updated=13` on pass 2.

**Reload after the walkthrough (agencydemo3):** with the Bluewater tariff approved, the reefer booking confirmed and the DG SI validated through the real commands, a third `--with-demo=all` run exits 0 with zero loader errors, and all three keep their walked state.

Then check that **Bookings** shows four new drafts (Bluewater import, reefer, DG, Konkan), and that
**Konkan Agro Exports** shows Credit Hold ticked with its reason.
