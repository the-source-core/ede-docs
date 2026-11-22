# Demo Use-Case — `logistics_nvocc_agency.agency_release`

**Module:** `agency_release` (BRD N6 — Delivery Order & Cargo Release Management)
**Demo file:** `src/domains/logistics_nvocc_agency/agency_release/demo/demo_release.xml`
**Scenario:** Acme Forwarding Ltd. (Mumbai HQ) — see [the unifying scenario](../../README.md)

---

## The story this data tells

`agency_documentation` left Globex's bill of lading under **Sealink Container Line** on the tenant:
two 40ft dry containers, Nhava Sheva → New York, consigned to Stark Industries. This module picks
the story up at the far end, where Acme's New York desk has to release the cargo:

> Stark's nominated haulier, **Hudson Drayage**, arrives to collect. The delivery order is drafted
> and looks complete — the release location is set, the parties are named, the lines match the B/L.
>
> The readiness check refuses it three times over. The B/L has not been released. The container has
> not been discharged. Customs has not cleared the entry. **Three refusals, three different desks,
> one message** — and each one names the record that failed rather than saying "not ready".
>
> The desk clears all three, issues the DO, and then somebody in compliance places a **customs hold**.
> The release refuses again, naming the hold and quoting its reason. Clearing it without a reason is
> refused. Overriding it without an approval case is refused. With both, the override goes through —
> and **the hold stays open**, because the row's job is to record that somebody released in spite of
> it.
>
> The cargo goes. That writes a `gate_out_full` movement event on the container's own stream, and it
> creates an **empty-return obligation** the DO cannot be closed over.

**The pending customs entry is the point of the fixture.** Everything else about this delivery order
is correct, so the first readiness run fails on exactly one *interesting* condition rather than on a
wall of missing data. A demo whose data is already perfect never shows you the control.

---

## Why the release end of the existing B/L, and not a new import shipment

A delivery order is an import instrument, so the obvious fixture would be a second booking and a
second B/L moving the other way. It is deliberately **not** built:

* an NVOCC agency handles **both ends of its own line**, and the release desk at the discharge port
  is a real Acme desk;
* reusing the existing chain means the reviewer watches **one shipment** run from quote to empty
  return, instead of two half-shipments that never meet;
* a second booking and B/L would be forty records restating facts the N5 fixture already carries
  correctly, and every one of them a place for the two to drift apart.

---

## Records shipped (13)

| # | Record | Model | Notes |
|---|---|---|---|
| 1 | `demo_facility_usnyc_terminal` | `logistics.facility.master` | New York Container Terminal — where the laden box is discharged and collected |
| 2 | `demo_facility_usnyc_depot` | `logistics.facility.master` | New York Empty Depot. **Two facilities, not one**: the release location and the return location are different places, and using one for both would hide that the empty-return obligation is about a different gate from the one the cargo left by |
| 3–4 | `demo_binding_admin_do_executive` / `_release_manager` | `ir.rbac.role.binding` | Admin ↔ the two new roles. **Both are needed, and the split is the whole of BRD §20.1 here**: the executive may create, validate, issue and record a release but may **not** clear or override a hold. Unbind the manager role and walk step 6 again to watch the override refuse with a `PermissionDeniedError` naming the roles it wanted. **Demo only**: a module must never grant release authority on install |
| 5 | `demo_numbering_rule_do_sealink` | `nvocc.numbering.rule` | `delivery_order` × Sealink × default org → the `DO` sequence, `prefix_override = SLKD`. Deactivate it and `nvocc.do.issue` refuses **by name**, saying which rule is missing (BR-NAM-12) |
| 6 | `demo_partner_stark_haulier` | `res.partner` | Hudson Drayage LLC, the collecting haulier |
| 7 | `demo_partner_role_haulier_pickup` | `res.partner.role` | Hudson as **pickup party**. `nvocc.do.party` refuses a `release_to` party holding none of the configured roles, **at create** — delete this row and try to add them and the refusal names the partner and the roles it wanted |
| 8 | `demo_customs_release_globex` | `nvocc.customs.release` | Entry `IMP-2026-DEMO-0001`, status **`pending`** on purpose. Try setting it to `hold` with no reason: refused, because the release refusal quotes the reason verbatim and "customs hold" alone tells the desk nothing |
| 9 | `demo_do_globex_001` | `nvocc.delivery.order` | **Draft**, no number, no validity. `bl_number_snapshot`, `shipment_id`, `booking_id` and `document_id` are deliberately **not set** — the pre-create hook copies them off the B/L, and letting it do so is the point |
| 10–11 | `demo_do_line_box_001` / `_002` | `nvocc.do.line` | One per container. `package_count` is the ceiling BR-N6-12 measures a release against. With partial release off, try releasing 200 of 240: refused, and the message says which switch to turn on |
| 12 | `demo_do_party_consignee` | `nvocc.do.party` | Stark — who the **B/L** names |
| 13 | `demo_do_party_release_to` | `nvocc.do.party` | Hudson — who may actually **collect**. Distinct from the consignee, which is the whole of N6-16 |
| 14 | `demo_do_party_transporter` | `nvocc.do.party` | Hudson again, in a second role. One party holding two roles on one DO is exactly what four columns on the header could not express |

---

## What this demo deliberately does not ship

**1. No issued delivery order.**
`nvocc.delivery.order.status` is `workflow=True`, so the ORM guard rejects a direct status write. The
DO lands in `draft` and the reviewer drives it with the real commands. A fixture that issued a release
authority by writing a column would demonstrate a readiness gate that had never run — which is the one
thing this module exists for.

**2. No release hold.**
A hold hangs off the B/L's `logistics.document` envelope, and that envelope does not exist until the
B/L is issued. A hold written here would have nothing to point at. Placing one by hand during the
walkthrough is better anyway: the reviewer sees the refusal name the hold they just created.

**3. No movement events.**
Discharge, full gate-out and empty return are Phase 4's append-only stream with its own sequence and
duplicate detection. Hand-writing them would show a release chain that skipped the validation that
makes the dates defensible in a demurrage dispute.

**4. No cleared customs entry.**
See above — the pending entry is the control, not an oversight.

---

## Walkthrough

Load with:

```
ede migrate upgrade -t <TENANT> --with-demo=logistics_nvocc_agency.agency_release
```

**Prerequisite:** walk the `agency_documentation` demo first (its steps 1–7). This fixture starts
where that one stops.

| Step | Do this | What it shows |
|---|---|---|
| 1 | Agency → Cargo Release → Delivery Orders → the draft DO. Run `nvocc.do.check_readiness` | **NOT READY**, naming three unmet conditions with the record that failed each. The B/L status comes from **Phase 5's own contract** — this module never writes a domain against `nvocc.bl.status` (N6-01, N6-02) |
| 2 | Walk the B/L to `released` with the N5 commands. Re-run readiness | The documents condition clears, and `document_id` is populated — which is what gives a hold something to hang off in step 6 |
| 3 | Record the discharge via `equipment-operations.log.movement.event` | The arrival condition clears. Note the module read **Phase 4's date contract**, not the event table, so a tenant adding a second discharge variant changes one seed row and nothing here (N6-04) |
| 4 | Set the customs entry to `cleared` with a release date. Re-run readiness | **READY.** Try `hold` with no reason first: refused (N6-06) |
| 5 | `nvocc.do.mark_ready`, then `nvocc.do.issue` | A number from the Sealink DO series, seven-day validity, and a rendered document envelope. The number was **not** burned at create — abandon a draft and the series has no hole in it (N6-09, N6-11, N6-13) |
| 6 | `logistics.document.place_hold` (type CUSTOMS), then try to release. Then try to clear it with no reason. Then override with no approval case. Then raise a case and override | Four refusals and one success, and **the hold stays open** after the override. Then look at Agency → Cargo Release → Hold Override Audit (N6-07, BR-N6-15, BR-N6-16) |
| 7 | `nvocc.do.record_cargo_release` | Writes a `gate_out_full` event **through Phase 4's own command** and links it to the DO. Grep the schema: there is no second event table (N6-18) |
| 8 | Try `nvocc.do.complete` | Refused — the container is still owed as an empty return, and Phase 7 charges detention against exactly that gap. See Agency → Cargo Release → Empty Returns Owed (N6-23) |
| 9 | Record the empty return as a movement event, then re-run `nvocc.do.complete` | It passes. **Nobody closed an obligation row**, because there is no obligation row — the event closed it |
| 10 | `nvocc.release.date_feed` on the DO | One contract, both halves: the release-side dates this phase owns and Phase 4's container dates composed in verbatim, with one merged `missing` list. That is what Phase 7 bills from (N6-24) |

---

## Refusals worth walking into on purpose

Each of these is the control working, not a defect:

* deactivate `demo_numbering_rule_do_sealink` → issuance refuses, naming the missing rule;
* delete `demo_partner_role_haulier_pickup` → adding Hudson as the release-to party refuses, naming
  the partner and the roles it wanted;
* release to **Globex** (the shipper, not a release party) → refused, naming them;
* release 200 of 240 packages with partial release off → refused, naming the switch;
* unbind `agency_release_manager` from admin → the override refuses with a role-named 403;
* set the DO's `valid_until` into the past → release refuses, quoting the expiry date and telling you
  to extend or reissue.

---

## Enhancement 01 — demo-data expansion

**Status:** ✅ Delivered 2026-11-22 — 8 records in 1 file; `--with-demo=all` smoke green and idempotent (see the end of this section).
**Roadmap:** [nvocc-agency Enhancement 01](../../../../roadmap/logistics/nvocc-agency/enhancements/01-demo-data-expansion.md). The cross-module walkthrough order lives there, and step IDs match it.
**New demo file** (appended **after** `demo/demo_release.xml`): `demo/demo_release_bluewater_import.xml` (8)

### Use case 1 — a delivery order at Nhava Sheva, through a customs examination

> Wayne Imports wants its two boxes of LED driver boards out of the JNPT container freight station.
> The B/L was telex-released at Singapore, and the IGM is filed. Customs, however, orders a
> **100% examination**. The examiner finds the carton marks do not match the B/L and places the
> entry on **hold**. Three days later customs clears it. The hold does not lift itself, and that is
> the point.

This is the import end the existing release demo deliberately did not build. That demo releases
the Sealink export at New York, at the far end of one shipment. This one is a destination agency
doing its own job: the DO is refused until the **IGM is accepted** (Phase 5's readiness contract,
consumed by command name), until the box is **discharged** (Phase 4's date contract), and until
**customs** says so. All three are refused in one message.

#### Records produced — `demo/demo_release_bluewater_import.xml` (8)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_release.demo_facility_innsa_cfs` | `logistics.facility.master` | `code` INNSA-CFS · "JNPT Container Freight Station" · `unlocode_id` `masters.demo_port_innsa` · description | The release location: an examined box is held at a CFS, not at the terminal. The empty return goes to a **different** gate, `agency_booking.demo_facility_nsict_depot` |
| `agency_release.demo_numbering_rule_do_bluewater` | `nvocc.numbering.rule` | `document_type=delivery_order` · Bluewater · default org · `agency_numbering.sequence_delivery_order` · `prefix_override` BLWD · draft | Issuance refuses by name until it is active (BR-NAM-12) |
| `agency_release.demo_customs_release_bluewater` | `nvocc.customs.release` | `bl_id` `agency_documentation.demo_bl_bluewater_import` · **`customs_status=pending`** · `declaration_number` BE-2026-INNSA-4471182 (the Bill of Entry) · `broker_partner_id` `base.demo_partner_co_001` (Bharat Logistics, the CHA) · `inspection_required=false` · `country_reference` "Bill of Entry filed on ICEGATE; awaiting RMS assessment" | Pending, not hold; see the first row of the next table. The principal is inherited off the B/L by the create hook |
| `agency_release.demo_do_bluewater_import` | `nvocc.delivery.order` | `bl_id` the import B/L · Bluewater · default org · `branch_id` MUM · `cargo_mode=fcl` · `is_partial=false` · `release_location_type=cfs` · `release_location_id` the CFS · `port_id` INNSA · `release_instruction` "Release to Wayne Imports' transport against the CHA's out-of-charge copy; empties to DEP-NSICT" | Draft, with no number and no validity. `bl_number_snapshot`, `booking_id`, `shipment_id` and `document_id` are left for the pre-create hook to copy off the B/L |
| `agency_release.demo_do_line_bluewater_box_1` / `_box_2` | `nvocc.do.line` | `equipment_usage_id` → `agency_booking.demo_usage_bluewater_box_1` / `_2` · 480 packages · 9 200 kg · `release_type=full` | One line per container. Each line's `package_count` is the ceiling a release is measured against |
| `agency_release.demo_do_party_bluewater_consignee` | `nvocc.do.party` | `role=consignee` · Wayne · contact | Who the B/L names |
| `agency_release.demo_do_party_bluewater_release_to` | `nvocc.do.party` | `role=release_to` · Wayne · contact · `identification_reference` "IEC 0000000000" (synthetic) | Wayne collects with its own transport. It holds `customer`, one of `nvocc.release_to_required_roles`, so no new role row is needed |

**Block split, same as `demo_release.xml`:** the facility goes in `noupdate="0"`; everything else
goes in `noupdate="1"`. A reload must not put a cleared customs entry back to `pending`, or rewrite
an issued DO.

### What this slice deliberately does NOT ship

| Not shipped | Why |
|---|---|
| **A customs hold**, although `customs_status` is a plain Enum and *could* be seeded as `hold` | The CUSTOMS hold row on the B/L's `logistics.document` envelope is raised only by `nvocc.customs.release.record`, and that envelope does not exist until the B/L is issued. A seeded `hold` status would be a hold with no hold row behind it: readiness would refuse on the status while the hold register stayed empty. Recording it in the walkthrough raises both |
| An issued DO | `nvocc.delivery.order.status` is `workflow=True` |
| Discharge, gate-out or empty-return events | The movement-event stream is append-only and duplicate-checked. The voyage slice records the discharge; this slice records the gate-out through `nvocc.do.record_cargo_release` |
| A FINANCE hold | It is **derived**: `FINANCE_CLEAR` places one when credit enforcement meets a held customer. See the optional refusal below |

### Walkthrough — the steps this module owns

| Step | Do this | What it shows |
|---|---|---|
| UC1-4 | After UC1-3 (B/L telex-released) and before the IGM is filed: Delivery Orders → the Bluewater DO → `nvocc.do.check_readiness` | **NOT READY, three conditions, three desks**: the B/L is not on an accepted manifest (read from Phase 5's contract, never from `nvocc.bl.status`); the box has not been discharged; customs is `pending` |
| UC1-10 | After UC1-5 and UC1-9 (IGM accepted, both boxes discharged): `nvocc.customs.release.record` with `customs_status=exam` and `inspection_required=true`. Then `record` with `customs_status=hold` and `hold_reason` "RMS 100% examination at JNPT CFS — carton marks do not match the B/L". Try `hold` with **no** reason first | The reasonless hold is **refused** (N6-06). With a reason, readiness refuses **twice**: the customs status, and a CUSTOMS hold now standing on the B/L's document, with the reason quoted verbatim |
| UC1-11 | `record` with `customs_status=cleared` and `release_date` 2026-10-15T17:00, then re-run readiness | **Still NOT READY.** Clearing customs does not clear the hold, because lifting a block is a decision with a name attached (BR-N6-15). `logistics.document.release_hold` with a reason → **READY** |
| UC1-12 | `nvocc.do.mark_ready` → `nvocc.do.issue` → `nvocc.do.record_cargo_release` for line 1 with `released_at` 2026-10-16T10:00 and line 2 with 2026-10-16T15:00 | A BLWD number with seven-day validity. Each release writes a `gate_out_full` event **through Phase 4's command**, and demurrage stops there |
| UC1-13 | `nvocc.do.complete` | **Refused**: both empties are owed. Log `empty_return` at DEP-NSICT, box 1 at 2026-10-19T11:00 and box 2 at 2026-10-26T09:00, then complete again. It passes, and no obligation row was closed because none exists |

**Optional refusals, no extra records:**

* tick Credit Hold on Wayne before UC1-12 → readiness places a **FINANCE** hold on the document
  (N6-05). This is the release-side twin of the booking-side credit refusal in Use case 3;
* try `nvocc.do.issue` before activating `demo_numbering_rule_do_bluewater` → refused, naming the
  rule;
* after issue, request a validity extension → refused while `nvocc.do_max_extensions = 0`.

### Dependencies and load order

`agency_documentation` (the import B/L) · `agency_booking` (usages; the empty depot). This
module reaches `agency_booking` **transitively** through `agency_documentation`, which declares
it in `depends` (the edge was added by Enhancement 01), so booking is guaranteed to load first ·
`agency_numbering` (`sequence_delivery_order`) · `sales_crm` (Wayne) · `foundation.base`
(`base.demo_partner_co_001`). The discharge that satisfies *cargo available* comes from the
`agency_voyage` walkthrough (UC1-8 / UC1-9). Load with `--with-demo=all`.

### Smoke test — ✅ green and idempotent, 2026-11-22

`ede migrate upgrade -t agencydemo3 --with-demo=all` on a fresh PostgreSQL tenant, run twice, exit
0, zero loader errors. Actual counts match the prediction exactly:

| File | Pass 1 | Pass 2 |
|---|---|---|
| `demo_release_bluewater_import.xml` | created=8 | created=0 · updated=1 · skipped=7 |

All 8 external ids are `is_demo = true`, and the customs entry loads as `pending`. The readiness,
customs and DO-issue steps above were **not** driven live in this delivery. They use the same
commands the existing, verified release demo walks.

Then check that **Delivery Orders** shows a second draft (Bluewater, CFS release location, two
lines, Wayne twice in two roles), and that **Customs Releases** shows BE-2026-INNSA-4471182 as
`pending`.
