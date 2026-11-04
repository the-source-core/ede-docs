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
