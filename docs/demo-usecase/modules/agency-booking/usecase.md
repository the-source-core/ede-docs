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
