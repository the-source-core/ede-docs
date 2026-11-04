# Demo Use-Case — `logistics_nvocc_agency.agency_voyage`

**Module:** `agency_voyage` (BRD N8 — Feeder Vessel, Voyage & Port Call Handling)
**Demo file:** `src/domains/logistics_nvocc_agency/agency_voyage/demo/demo_agency_voyage.xml`
**Scenario:** Acme Forwarding Ltd. (Mumbai HQ) — see [the unifying scenario](../../README.md)
**Requires first:** the `agency_booking` demo (this file references its principal, its depot and its booking)

---

## The story this data tells

The `agency_booking` demo left Globex Inc. with a confirmed booking under **Sealink Container Line**,
Nhava Sheva → New York, and a release order authorising two 40ft boxes out of the Nhava Sheva depot.

This module adds what Acme's **port desk** actually works when those boxes reach the quay:

> Sealink runs **IGX — India Gulf Express**, a weekly feeder loop: Nhava Sheva → Jebel Ali →
> Singapore. Sailing **2609W** on the *Sealink Gulf* is loading at Nhava Sheva on 2 September. Three
> containers are on the load list. At Jebel Ali the boxes for Europe connect to a mainline vessel,
> the *Maersk Madrid*, on voyage 2612E — and that connection has a cut-off.
>
> One of the three boxes does not make the ship.

Everything after that — the short load, its owner, the detention exposure it leaves running, the
exception case, the refusal to close — is the reviewer's own doing. None of it ships as data, and
that is the point.

---

## Records shipped (24)

| # | Record | Model | Notes |
|---|---|---|---|
| 1 | `demo_config_closure_documents` | `ir.config` | `nvocc.port_call_closure_requires_documents = false`. **The only setting this file changes.** On by default, and with the checklist engine unreachable from this domain it refuses every closure — correct, but a dead end for step 8. See below |
| 2–3 | `demo_binding_admin_port_operations` / `_feeder_operations` | `ir.rbac.role.binding` | Admin ↔ the two roles this module adds. `AssignmentService` resolves a ROLE approval step by exact code and does **not** walk inheritance, so without these the reopen approval in step 9 has nobody to assign to |
| 4 | `demo_partner_vendor_marine` | `res.partner` | JNPT Marine Services — the pilotage/towage vendor. A party of this module's own: reusing a customer as a vendor would make the vendor dimension on every cost report meaningless |
| 5 | `demo_service_igx` | `nvocc.service` | `IGX`, feeder, weekly, Nhava Sheva → Jebel Ali. `principal_id` **is** set — this loop is Sealink's own |
| 6 | `demo_vessel_sealink_gulf` | `logistics.vessel` | 1 118 TEU feeder. A vessel of its own rather than reusing the 20 568 TEU *Maersk Madrid*, which would make nonsense of a feeder rotation |
| 7 | `demo_voyage_igx_2609w` | `nvocc.voyage` | The operated sailing. **No `sailing_schedule_id`** — opened by the agency, not synced. **No `status`** — lands in `planned` |
| 8–10 | `demo_rotation_1_innsa` / `_2_aedxb` / `_3_sgsin` | `nvocc.voyage.rotation.line` | Sequence 1..3, contiguous. Step 1 breaks it on purpose |
| 11–12 | `demo_port_call_innsa` / `_aedxb` | `nvocc.port.call` | Two calls for the first two positions. **Position 3 deliberately has none** — step 9's voyage closure has to refuse for a reason the reviewer can see |
| 13 | `demo_plan_innsa_load` | `nvocc.port.call.operation.plan` | 3 planned moves, 2 gangs, 1 crane |
| 14 | `demo_booking_link_sealink` | `nvocc.voyage.booking.link` | 6 TEU / 3 containers committed. A **load** call only — the boxes tranship at Jebel Ali, and that asymmetry is why the linkage is its own object |
| 15 | `demo_list_innsa_load` | `nvocc.load.discharge.list` | v1, `draft` |
| 16–18 | `demo_list_line_001` … `_003` | `nvocc.load.discharge.list.line` | Two 40GP and one 20GP, all `planned`. **The 20GP is the one step 7 leaves unstamped** — it becomes the short load the rest of the walkthrough follows |
| 19–21 | `demo_voyage_mainline_ne2`, `demo_rotation_mainline_1`, `demo_port_call_mainline_aedxb` | voyage / rotation / port call | The mainline leg. A connection links two voyages and a self-link is refused, so the outbound leg is a voyage of its own |
| 22 | `demo_connection_aedxb` | `nvocc.feeder.connection` | Cut-off 8 Sep 12:00, planned dwell 34 h, `planned`. Step 10 pushes the inbound ATA past the cut-off |
| 23–25 | `demo_service_pilotage_innsa` / `_towage_` / `_attendance_` | `nvocc.port.call.service` | Three states: `confirmed`, `requested`, `requested`. Agency attendance has **no vendor** — the seed sets `requires_vendor` false on it |
| 26–27 | `demo_cost_pilotage_innsa` / `_towage_` | `nvocc.port.call.cost` | PDA estimates only, because that is what an agency has before the vessel sails |
| 28 | `demo_communication_arrival_notice` | `nvocc.port.call.communication` | **Outbound**, to the **principal** — the combination the closure gate counts |

Plus the module's own seeds, which load as ordinary data rather than demo: 15
`nvocc.port.service.type.master` rows and 12 `ops.exception.code.master` rows covering N8-22's
exception vocabulary.

---

## The walkthrough

1. **Agency → Operations → Voyages → SLK IGX 2609W.** Press **Release to Operations** — it
   succeeds, because the rotation runs 1..3 with no gaps. Now delete rotation line 2 and try again:
   the refusal **names position 2 by number**. Re-add it before continuing.
2. **Agency → Operations → Port Calls → the Nhava Sheva call.** The baselines were stamped from the
   rotation and are read-only. Press **Arrived** and give it a time. Look back at the voyage: it has
   moved to `in_progress` on its own, because a recorded ATA is what makes a voyage under way.
3. **Revise the schedule twice** with different reasons (`nvocc.port_call.revise_schedule`). The
   Schedule History page shows two rows, `baseline_etd` has not moved, and `etd` shows the newest
   value — BR-N8-13. Then try a revision with no reason: refused,
   `nvocc.schedule_revision_reason_required` being on by default.
4. **Press Sailed before Operations Completed.** Refused, naming BR-N8-05. Record Berthed, Operations
   Started and Operations Completed first.
5. **Correct the ATA**: a new actual with `is_correction`, a reason and `supersedes_id`. Both rows
   stay on the Actual Events page and `ata` shows the correction. Now try editing the original row
   directly — the refusal explains why Phase 7 needs it to survive.
6. **Load / Discharge Lists → the Nhava Sheva load list.** Three containers, all `planned`. Run
   `nvocc.port_call.reconcile`: all three become `short_loaded` and three discrepancies appear,
   because no LOADED events carry this port call yet. **That is the correct answer** for an unworked
   call.
7. **Stamp two of them** — create LOADED movement events on the two 40GP boxes and set their
   `port_call_id` — then reconcile again. Two lines become `loaded`; the 20GP stays `short_loaded`,
   with an owner slot, a continuing CRO reference and a days-out figure. **One** exception case was
   opened, not three.
8. **Try to close the port call.** Refused, and the refusal names every outstanding item at once. Turn
   `nvocc.port_call_closure_requires_documents` back **on** first to see the document leg refuse and
   say why — that is [carried dependency D1](../../../../roadmap/logistics/nvocc-agency/phase-8/README.md#d1--n8-19--the-port-call-document-checklist-blocked-on-a-peer-domain-boundary),
   and the fail-closed behaviour is deliberate.
9. **Roll the short load** to the next voyage (`nvocc.discrepancy.roll_over`), close the port call,
   then try to close the **voyage** — refused while Jebel Ali is open and while position 3 has no
   call at all.
10. **Feeder Connections → the Jebel Ali connection.** Set the inbound call's ATA later than the
    cut-off and run the `nvocc.voyage.operations.sweep` job. The connection resolves to `missed`, its
    reason is written for you, and an exception case opens (BR-N8-18).

---

## What this demo deliberately does not ship, and why

| Not shipped | Why |
|---|---|
| Any `status` on a voyage or port call | Both are `workflow=True`; the ORM guard rejects a direct write. More to the point, a fixture that pre-set the stages would show port calls `arrived` with **no ATA** — exactly the state the design exists to make impossible. `test_the_demo_file_sets_no_workflow_status` asserts it |
| Any recorded actual | `nvocc.port.call.actual`'s create hook enforces chronology (BR-N8-04), the departure policy (BR-N8-05) and the correction contract (BR-N8-17). Pre-recorded actuals would skip all three; recording them in the walkthrough exercises all three |
| Any discrepancy | Short load, short landed and over landed are **derived** by reconciliation. A short load shipped as a fixture is precisely the hand-typed discrepancy the design refuses |
| Any `logistics.sailing.schedule` row | This sailing was opened by the agency, not synced from a feed. Leaving `sailing_schedule_id` empty makes the module's founding distinction visible |
| A port call for rotation position 3 | So that step 9's voyage-closure refusal has a reason the reviewer can see. "Position 3 has no call" is a different and more interesting failure than "position 3 is open" |
| A vendor bill, payable or posting | BR-N8-15. EDE owns the charge data, not the GL — `test_port_cost_produces_no_accounting_model` asserts the absence |

---

## Loading

```bash
ede migrate upgrade -t <tenant> \
    --with-demo=logistics_nvocc_agency.agency_booking,logistics_nvocc_agency.agency_voyage
```

Order matters: this file references `agency_booking.demo_principal_sealink`,
`agency_booking.demo_facility_nsict_depot` and `agency_booking.demo_booking_sealink_001`.

## Smoke test — ✅ green and idempotent, 2026-08-19

Verified on PostgreSQL against a reference tenant at head: zero errors, every model populated, and a
re-run changes nothing.

| Model | Rows |
|---|---|
| `nvocc.service` | 1 |
| `nvocc.voyage` | 2 |
| `nvocc.voyage.rotation.line` | 4 |
| `nvocc.port.call` | 3 |
| `nvocc.port.call.operation.plan` | 1 |
| `nvocc.voyage.booking.link` | 1 |
| `nvocc.load.discharge.list` (+ 3 lines) | 1 |
| `nvocc.feeder.connection` | 1 |
| `nvocc.port.call.service` | 3 |
| `nvocc.port.call.cost` | 2 |
| `nvocc.port.call.communication` | 1 |
| `nvocc.port.service.type.master` (seed) | 15 |
| `ops.exception.code.master` — `N8_*` (seed) | 12 |

**It took seven runs to get there, and that is the point of the gate.** The first five found defects
no unit test had: two seed CSVs that failed to load entirely on an unquoted comma, two controls
placed at the wrong moment in a record's life, four branch-scoped children with no way to obtain a
branch, a set of frozen columns that made the record unloadable on the *second* run — and, worst,
**twenty stored computes that silently returned NULL** because a compute cannot call a helper method
on its RecordSet `self`. Every one of those shipped green tests and a rendering screen. Full write-up:
[phase-8/README.md](../../../../roadmap/logistics/nvocc-agency/phase-8/README.md#five-defects-the-demo-smoke-test-caught--and-one-worth-the-whole-exercise).
