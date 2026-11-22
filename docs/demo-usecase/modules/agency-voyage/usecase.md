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

---

## Enhancement 01 — demo-data expansion

**Status:** ✅ Delivered 2026-11-22 — 16 records in 2 files; `--with-demo=all` smoke green and idempotent (see the end of this section).
**Roadmap:** [nvocc-agency Enhancement 01](../../../../roadmap/logistics/nvocc-agency/enhancements/01-demo-data-expansion.md). The cross-module walkthrough order lives there, and step IDs match it.
**New demo files** (appended **after** `demo/demo_agency_voyage.xml`, whose Jebel Ali port calls and feeder connection Use case 4 hangs off):
`demo/demo_voyage_bluewater_import.xml` (6) · `demo/demo_voyage_transshipment.xml` (10)

This module does not depend on `agency_documentation`, so neither file references a B/L. The
discharge list carries `bl_number` as **text**, as the model's own help explains (carried dependency
D1). The containers come from `agency_booking` (Use case 1) or are this file's own (Use case 4).

### Use case 1 — vessel arrival and discharge at Nhava Sheva

> *Maersk Madrid* arrives at Nhava Sheva on voyage **2614W** on 6 October. Acme attends the call
> as **Maersk Demo Line's** port agent, and two of the boxes it discharges are **Bluewater's**,
> consigned to Wayne Imports. Acme is their destination agent as well. The tally clerk records
> one discharge and misses the other.

The existing demo has a **load** list at Nhava Sheva and has never shown the import side of a port
call. One discharge event here has three consumers, and none of them keeps its own copy: the
port-call reconciliation, the DO's *cargo available* condition (Phase 4's date contract), and the
demurrage clock.

#### Records produced — `demo/demo_voyage_bluewater_import.xml` (6)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_voyage.demo_voyage_mm_2614w` | `nvocc.voyage` | `voyage_number` 2614W · `masters.demo_vessel_maersk_madrid` · `principal_id` `agency_masters.demo_principal_maersk` · `direction=import` · default org · planned 2026-10-01 → 2026-10-07 | The vessel's principal is Maersk; the cargo's principal is Bluewater. It uses the **same vessel and voyage number** as the import B/L and the IGM, because that is what the IGM matches on. No `status`, no `sailing_schedule_id` |
| `agency_voyage.demo_rotation_mm_2614w_innsa` | `nvocc.voyage.rotation.line` | sequence 1 · INNSA · ETA 2026-10-06 06:00 · ETD 2026-10-07 20:00 · `operation_type=discharge` · `is_transshipment=false` | A one-port rotation, as the existing mainline voyage has. Acme does not attend Singapore |
| `agency_voyage.demo_port_call_mm_2614w_innsa` | `nvocc.port.call` | the voyage · the rotation line · sequence 1 · INNSA · Maersk · default org · `operation_type=discharge` · baseline ETA 06 Oct 06:00 / ETB 10:00 / ETD 07 Oct 20:00 · `port_authority_reference` JNPT/2026/10/0388 | The attended call |
| `agency_voyage.demo_list_innsa_discharge_2614w` | `nvocc.load.discharge.list` | the port call · `list_type=discharge` · version 1 · draft | The import-side plan |
| `agency_voyage.demo_list_line_bluewater_box_1` / `_box_2` | `nvocc.load.discharge.list.line` | `equipment_id` `agency_booking.demo_equipment_bluewater_box_1` / `_2` · 42GP · seal BLW-771201 / BLW-771202 · 9 200 kg · `booking_id` the Bluewater import booking · `consignee_partner_id` Wayne · POL SGSIN · POD INNSA · `operation=discharge` · `special_cargo=none` · **`bl_number` empty** | The B/L number is allocated at issue, which happens in the walkthrough. Reconciliation matches on `equipment_id`, never on the text |

### Use case 4 — transshipment via Jebel Ali: the sailing connects, one box does not

> The IGX feeder *Sealink Gulf* 2609W reaches Jebel Ali at 09:30 on 7 September, well inside the
> 12:00 cut-off on the 8th for *Maersk Madrid* 2612E. The **connection is made**. Two Sealink 40ft
> boxes for New York are transhipping. **SEAU4200206** comes off at 18:00 on the 7th and makes
> the mainline. **SEAU4200211** was stowed under hatch 4, comes off at 13:30 on the 8th, ninety
> minutes after the cut-off, and is left on the quay.

The existing file plans this connection, and its step 10 walks the **voyage-level** miss (inbound
ATA after the cut-off). This use case is the other half of the design the feeder-connection
docstring describes: *"one says which sailings connect, the other says which boxes made it."* Both
halves of a transshipped box's movement, its discharge row at the hub and its load row on the
connecting voyage, carry the same `feeder_connection_id`. Reconciling the mainline call is what
finds the box that did not make it.

#### Records produced — `demo/demo_voyage_transshipment.xml` (10)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_voyage.demo_equipment_ts_box_a` / `_box_b` | `logistics.equipment` | `SEAU4200206` / `SEAU4200211` (valid ISO 6346 check digits) · `masters.eq_type_42gp` · `principal_id` `agency_booking.demo_principal_sealink` · custodian default org · `lease_type=owned` | Own boxes ([README rule](../../README.md#shared-fixtures-that-are-deliberately-unavailable)). They are **not** on the Nhava Sheva load list, so the original three-box walkthrough and its counts are untouched |
| `agency_voyage.demo_usage_ts_box_a` / `_box_b` | `logistics.equipment.usage` | default org · 2026-09-01 → 2026-10-10 | Movement events require a usage |
| `agency_voyage.demo_list_aedxb_ts_discharge` | `nvocc.load.discharge.list` | `agency_voyage.demo_port_call_aedxb` (the feeder at the hub) · `list_type=discharge` · version 1 | The hub-side plan. The call's `operation_type=both`, so a discharge list is admitted |
| `agency_voyage.demo_list_line_ts_discharge_a` / `_b` | `nvocc.load.discharge.list.line` | box A / B · 42GP · `booking_id` `agency_booking.demo_booking_sealink_001` · POL INNSA · POD USNYC · **`operation=transshipment`** · **`feeder_connection_id` `agency_voyage.demo_connection_aedxb`** | — |
| `agency_voyage.demo_list_aedxb_ts_load_mainline` | `nvocc.load.discharge.list` | `agency_voyage.demo_port_call_mainline_aedxb` · `list_type=load` · version 1 | The connecting voyage's plan |
| `agency_voyage.demo_list_line_ts_load_a` / `_b` | `nvocc.load.discharge.list.line` | same boxes · `operation=transshipment` · the same connection | With both halves pointing at one connection, a transshipped box is **one movement**, not a short landing followed by an over-landing |

### What this slice deliberately does NOT ship

| Not shipped | Why |
|---|---|
| Any `status` on the new voyage or port call, or any recorded actual | Same reasons as the original file: `workflow=True`, and the actual-event create hook enforces chronology, the departure policy and the correction contract |
| Any discrepancy, or a connection status | Short landed and short loaded are **derived** by reconciliation. `connected` / `missed` are derived by the sweep |
| Any movement event, or a `transshipment_of_event_id` pairing | The stream is append-only. The connecting load is always recorded *after* the discharge it completes, which is why pairing is a command (`equipment-operations.link.transshipment`) and not a column a fixture can fill |
| A LOADED event at Singapore or Nhava Sheva for these boxes | The origin load is outside these use cases. Under the default `logistics.equipment.event_sequence_enforcement = warn`, the first discharge carries a sequence **warning**, which is the honest outcome |
| A next mainline sailing | `nvocc.discrepancy.roll_over` needs a target voyage. The use case ends with the short load owned and the customer told |

### Walkthrough — the steps this module owns

| Step | Do this | What it shows |
|---|---|---|
| UC1-6 | Activate `PRN-MAERSK` if not done. Voyages → *Maersk Madrid* 2614W → **Release to Operations** | Activation reads the principal's status. Planning a sailing for a draft principal is legitimate; releasing it is not |
| UC1-7 | Port Calls → the Nhava Sheva call → Arrived (ATA 2026-10-06 06:40) → Berthed → Operations Started | The voyage moves to `in_progress` by itself |
| UC1-8 | Log a DISCHARGED event (`equipment-operations.log.movement.event`) for **box 1 only**: its usage, `port_call_id` = this call, `event_at` 2026-10-06T14:00. Then `nvocc.port_call.reconcile` | Box 1 is `discharged`. Box 2 is **`short_landed`**, with one discrepancy and one exception case: a short landing triggers a manifest review (BR-N8-09) |
| UC1-9 | Log box 2's discharge (14:20) and reconcile again | The line moves to `discharged`, but the **discrepancy stays open**. Reconciliation never closes what a person must explain. `nvocc.discrepancy.resolve` with notes ("tally missed it — discharged 14:20") closes it |
| UC4-1 | Release IGX 2609W (original step 1) **and** *Maersk Madrid* 2612E. Jebel Ali feeder call → Arrived, ATA 2026-09-07 09:30. Run the `nvocc.voyage.operations.sweep` job | The connection resolves to **`connected`**: the ATA is before the 12:00 cut-off on the 8th |
| UC4-2 | At the **feeder's** Jebel Ali call, log DISCHARGED for box A (2026-09-07T18:00) and box B (2026-09-08T13:30), then reconcile that call | Both transshipment discharge lines are `discharged` |
| UC4-3 | At the **mainline's** Jebel Ali call, log LOADED for box A only (2026-09-09T15:00). Run `equipment-operations.link.transshipment` with box A's load and, first, **box B's** discharge; then box A's | Box B's pairing is **refused** (*"the discharge and the connecting load are on different containers (N4-10)"*); box A's is linked |
| UC4-4 | Reconcile the mainline call | Box A is `loaded`. Box B is **`short_loaded`**, with one discrepancy and one exception case. The connection still reads `connected`. **The sailing made it; the box did not.** That is two facts on two objects, and neither overwrites the other |
| UC4-5 | Discrepancy → `nvocc.discrepancy.assign` an owner; set `reason=cutoff_missed` with detail "Discharged 13:30, 90 min after the 12:00 connection cut-off — under hatch 4"; `nvocc.discrepancy.notify_customer` | The short-load register groups on reason, which answers "how many of our short loads are missed cut-offs" |
| UC4-6 | Try to close the mainline call | **Refused**, naming the open discrepancy. Box A's discharge/load pair reads as a transshipment. Box B's discharge is still **unpaired**: an open gap at the hub, where box A's dwell is not |

**Ordering with the original walkthrough.** Walk UC4 **before** the original step 10. If step 10
comes first, the connection is already `missed` (the sweep only judges `planned` connections), and
UC4-1 shows that instead. The per-box short load in UC4-4 happens either way. Until UC4 is walked,
the original closure steps (8–9) also name these lists' `planned` lines among the outstanding items.
That is additive and correct.

### Dependencies and load order

`agency_booking` (the Bluewater containers and booking; the Sealink principal and booking) ·
`agency_masters` (Maersk) · this module's own `demo_agency_voyage.xml` (`demo_port_call_aedxb`,
`demo_port_call_mainline_aedxb`, `demo_connection_aedxb`) · `sales_crm` (Wayne) ·
`logistics_base.masters` (ports, vessel, equipment type). Load with `--with-demo=all`.

### Smoke test — ✅ green and idempotent, 2026-11-22

`ede migrate upgrade -t agencydemo3 --with-demo=all` on a fresh PostgreSQL tenant, run twice, exit
0, zero loader errors. Actual counts match the prediction exactly:

| File | Pass 1 | Pass 2 |
|---|---|---|
| `demo_voyage_bluewater_import.xml` | created=6 | created=0 · updated=6 |
| `demo_voyage_transshipment.xml` | created=10 | created=0 · updated=10 |

All 16 external ids are `is_demo = true`. The existing `demo_agency_voyage.xml` still reads
`updated=28` on pass 2. The arrival, reconciliation and transshipment steps above were **not**
driven live in this delivery. They use the same commands the existing, verified voyage demo walks.

| Model | Rows added by Enhancement 01 |
|---|---|
| `nvocc.voyage` | +1 |
| `nvocc.voyage.rotation.line` | +1 |
| `nvocc.port.call` | +1 |
| `nvocc.load.discharge.list` | +3 |
| `nvocc.load.discharge.list.line` | +6 |
| `logistics.equipment` / `.usage` | +2 / +2 |
