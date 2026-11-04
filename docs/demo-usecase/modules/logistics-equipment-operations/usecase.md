# `logistics.equipment-operations` — Demo Use-Case

**Module:** `domains.logistics_base.equipment_operations`
**App key:** `logistics_base.equipment_operations`
**Demo manifest entries:** `demo/demo_equipment.xml`, `demo/demo_operations.xml` (Enhancement 02 — retrofit 2026-05-28), `demo/demo_n4_container_control.xml` (BRD N4 — 2026-08-18)
**Status:** ✅ Phase 1 Delivered (2026-05-27); ✅ Demo data Delivered (2026-05-28, Enhancement 02); ✅ BRD N4 demo data Delivered (2026-08-18, smoke-tested on a fresh tenant 2026-08-19)

---

## Demo data records produced (Enhancement 02 — `--with-demo=logistics_base.equipment_operations`)

Anchored to the unifying scenario in [docs/demo-usecase/README.md](../../README.md): **Acme Forwarding Ltd. (Mumbai HQ)** as an asset-owning forwarder running its own fleet on the `INMUM → SGSIN` lane. Custodian is the seeded `base.default_organization`; every CECS master referenced (`masters.eq_type_*`, `masters.eq_status_*`, `masters.eq_cond_*`, `masters.usage_status_*`, `masters.seal_type_*`, `masters.movement_event_*`) ships from the `logistics.masters` production seed CSVs — the demo references them, never recreates them.

### `demo/demo_equipment.xml` — the Acme fleet (7 units)

| External ID | Model | Notes |
|---|---|---|
| `equipment_operations.demo_equipment_42gp_001` | `logistics.equipment` | 40ft dry (MSKU4200001) · owned · Good · **status derived** — this is the unit the BRD N4 seven-event chain runs on |
| `equipment_operations.demo_equipment_42gp_002` | `logistics.equipment` | 40ft dry (MSKU4200002) · owned · Good · **status derived** — the unit the BRD N4 customs hold blocks |
| `equipment_operations.demo_equipment_42rt_001` | `logistics.equipment` | 40ft reefer (MSKU4250001) · owned · Good · **status derived** — the unit the BRD N4 damage + repair records attach to |
| `equipment_operations.demo_equipment_22gp_001` | `logistics.equipment` | 20ft dry (MSKU2200001) · owned · Good · **status derived** |
| `equipment_operations.demo_equipment_tractor_001` | `logistics.equipment` | 3-axle tractor (GJ01-AB-1234) · owned |
| `equipment_operations.demo_equipment_chassis_001` | `logistics.equipment` | 40ft chassis (CHAS-40-001) · owned |
| `equipment_operations.demo_equipment_carrier_42gp_001` | `logistics.equipment` | Carrier-supplied 40ft (TCLU5500001) · `lease_type=carrier_supplied` — for contrast |

### `demo/demo_operations.xml` — operational lifecycle (6 records)

| External ID | Model | Notes |
|---|---|---|
| `equipment_operations.demo_usage_22gp_inmum_job` | `logistics.equipment.usage` | Binds the in-use 20ft to a job window; status Active |
| `equipment_operations.demo_usage_42rt_reefer_job` | `logistics.equipment.usage` | Reefer usage window; status Reserved |
| `equipment_operations.demo_seal_42rt_primary` | `logistics.equipment.seal.application` | BOLT seal, primary slot, applied to the reefer usage |
| `equipment_operations.demo_movement_22gp_gate_out` | `logistics.equipment.movement.event` | GATE_OUT, manual source |
| `equipment_operations.demo_movement_22gp_gate_in` | `logistics.equipment.movement.event` | GATE_IN, manual source |
| `equipment_operations.demo_loadcalc_42gp_stuffing` | `logistics.load.calculation` | Single-unit-fit stuffing plan for a 40ft container (`algorithm_version=phase1.v1`, `result_status=single_unit_fit`) |

> **BRD N4 changed `demo_equipment.xml`.** Every `current_status_id` was removed from it. The
> column is now derived from the movement-event stream, and a direct write is refused once a unit has
> movement history — so hand-setting it in demo data would both demonstrate the behaviour the phase
> retired and make the file unloadable on a second run for MSKU4200001, which now carries a
> seven-event chain. The create hook defaults each unit to the first `is_available_state` row, which
> is the truthful answer for a box that has not moved.

### `demo/demo_n4_container_control.xml` — BRD N4 container control (17 records)

The Phase 1 demo proved the wire. This one shows the **rules**: a container's whole commercial life as seven timestamps, the three states that stop a box moving, and the exception register that makes a bad event visible before Phase 7 bills from it.

Nothing in it sets a derived value. `current_status_id`, `current_facility_id`, `last_movement_at` and `damage_flag` are all absent from these records on purpose — they are computed from the movement events and the damage rows, and a demo that hand-set them would be demonstrating the bug the phase removed.

| External ID | Model | Notes |
|---|---|---|
| `equipment_operations.demo_facility_nhava_depot` | `logistics.facility.master` | The empty depot — releases and returns |
| `equipment_operations.demo_facility_jnpt_terminal` | `logistics.facility.master` | The terminal — laden gate-in, load, discharge, laden gate-out |
| `equipment_operations.demo_n4_usage_export_cycle` | `logistics.equipment.usage` | The allocation the seven events hang off (USG-DEMO-N4-01) |
| `equipment_operations.demo_n4_event_empty_release` | `logistics.equipment.movement.event` | 1/7 — released empty at the depot |
| `equipment_operations.demo_n4_event_gate_out_empty` | `logistics.equipment.movement.event` | 2/7 — gated out empty · **detention starts** |
| `equipment_operations.demo_n4_event_gate_in_full` | `logistics.equipment.movement.event` | 3/7 — gated in laden · **detention stops**; the unit is laden from here |
| `equipment_operations.demo_n4_event_loaded` | `logistics.equipment.movement.event` | 4/7 — loaded · no facility (a box on a vessel is not at the terminal it sailed from) |
| `equipment_operations.demo_n4_event_discharged` | `logistics.equipment.movement.event` | 5/7 — discharged · **demurrage starts** |
| `equipment_operations.demo_n4_event_gate_out_full` | `logistics.equipment.movement.event` | 6/7 — gated out laden · **demurrage stops** |
| `equipment_operations.demo_n4_event_empty_return` | `logistics.equipment.movement.event` | 7/7 — returned empty · the usage closes and the box is allocatable again |
| `equipment_operations.demo_n4_hold_customs` | `logistics.equipment.hold` | Open, blocking — allocating MSKU4200002 is refused naming this reason |
| `equipment_operations.demo_n4_hold_commercial_released` | `logistics.equipment.hold` | Released — the row and its reason survive the clear |
| `equipment_operations.demo_n4_damage_reefer` | `logistics.equipment.damage` | Reefer machinery fault, `under_repair` — what `damage_flag` derives from |
| `equipment_operations.demo_n4_repair_reefer` | `logistics.equipment.repair` | WO-2026-0451, in progress · a cost **reference**, never a payable |
| `equipment_operations.demo_n4_damage_cosmetic_closed` | `logistics.equipment.damage` | Closed cosmetic damage — so the register shows both states |
| `equipment_operations.demo_n4_exception_duplicate` | `logistics.equipment.event.exception` | Two feeds, one gate move — merged, resolved, both references retained |
| `equipment_operations.demo_n4_exception_out_of_sequence` | `logistics.equipment.event.exception` | Discharge with no preceding load — **open**, owned, naming the expected predecessor |

## Demo dependencies

- `base.default_organization` (seeded res.organization) — custodian + organization on every record.
- `logistics.masters` production seed CSVs — all CECS taxonomy / status / condition / usage-status / seal-type / movement-event-type rows referenced by `masters.*` xml-id.
- No foundation.base demo dir required for the Phase 1 files — they are self-contained against the default org + seeded masters.
- The BRD N4 file additionally references `base.demo_partner_co_001` / `base.demo_partner_co_002` (the responsible party on a damage record and the repair vendor) and the N4 taxonomies seeded by this module's own `logistics.equipment.hold.type.master.csv` / `logistics.equipment.damage.type.master.csv`.

## Demo verification

```bash
ede migrate upgrade -t <tenant> \
    --with-demo=foundation.base,logistics_base.masters,logistics_base.equipment_operations
```

**Both upstream scopes are required, not optional.** The BRD N4 file references
`base.demo_partner_co_001` / `_002` for the responsible party on a damage record and the repair
vendor, and those rows only exist when `foundation.base`'s demo is in the same load — a
module-only `--with-demo` fails those three records with `Cannot resolve ref`.

Verified 2026-08-19 on a fresh tenant: **30 records** under the `equipment_operations` module
(7 equipment + 6 operations + 17 BRD N4), `0 skipped`, no errors. A re-run reports `updated` for the
equipment and facility rows (`noupdate="0"`) and **skips** the operations and N4 lifecycle rows
(`noupdate="1"` — those models are append-only, so an update is not merely unnecessary, it is
rejected).

The derivation is visible in the loaded data rather than asserted:

| Unit | Derived status | Derived location | `damage_flag` |
|---|---|---|---|
| `MSKU4200001` — ran the full seven-event chain | `AVAILABLE` | `INMUM-DEPOT` | false |
| `MSKU2200001` — mid-chain, last event a load | `RESERVED` | *(none — a load does not move the box to a facility)* | false |
| `MSKU4250001` — open reefer damage | `RESERVED` | — | **true**, rolled up from the damage row |

Not one of those four values is set anywhere in the demo XML. UI: Equipment Control → Equipment → All Units shows the 7-unit fleet; Operations → Usage / Movement Events / (seals) render the lifecycle rows; Planning → Load Calculator shows the stuffing plan.

## Demo out of scope

- Multi-leg / cross-module allocation into `logistics.booking` (covered by the booking module's own demo + e2e).
- Maintenance & Inspection records — masters exist but no consuming model in Phase 1 (see Enhancement 01 menu note).
- Carrier-API / EDI movement ingestion (Phase 2).

---

## Use-case

A forwarder running on EDE wants to demo the CECS (Centralized Equipment Control System) physical-truth layer — the registry where every container / truck / wagon / ULD lives + the binding records that connect equipment to operations. This use case exercises the Equipment Control app surface end-to-end after the Phase 1 ship: app-switcher entry visible, list views render, form view opens, lifecycle commands are routable through the live command bus, and the registry-only invariants hold against the running ORM.

The Phase 1 demo deliberately keeps the data narrow — the goal is to prove the wire from clicked button → FastAPI → command bus → SQLAlchemy → React webclient render, not to ship a rich dataset. Each test method seeds its own minimal records via the live env, asserts the UI surface, then tears down. Module-local conftest provides session-scoped helpers but no XML demo files.

## Recorded e2e tests

| Test file | Steps | Use case |
|---|---|---|
| `tests/e2e/test_equipment_operations_smoke.py` | App-switcher → Equipment Control → All Units list renders · Movement Events list renders · Load Calculator screen reachable | Phase 1 smoke — proves the Equipment Control app loads against the live tenant after the 5-table Alembic migration `ca60f868d5c8` applies. ORM-level assertions on `logistics.equipment` + `logistics.equipment.usage` registration. |

## Notable Phase 1 architectural deviations recorded for testing

- `branch_id` dropped from usage + load.calculation (Enh 07 removed `ir.org.unit`) — `organization_id` carries branch identity via parent_id tree
- Identifier-type master uses `pattern` not `format_regex` — BR-EQ-01 regex validation reads the `pattern` column
- Default-status lookup uses `is_available_state=True` (no `is_initial_default` on the masters)
- BR-EVT-05 category-fit approximated via `transport_mode_id` match
- ~~BR-EVT-10 event-code → usage-status mapping hardcoded~~ — **retired by BRD N4 (2026-08-18)**. The mapping is now seed data on `logistics.movement.event.rule`, one row per event type; adding an event type is a data change, not a release.
- BR-CALC-05 hazmat/temperature compatibility gates pass-through pending `equipment.type.master` schema enhancement

These deviations are stable surfaces — the e2e tests verify the registry contracts and the UI flows that depend on them, not the deviation-specific shortcuts.

## What this demo does NOT cover (yet)

- ~~Full equipment lifecycle browser walk~~ — the **data** now exists (the seven-event chain in `demo_n4_container_control.xml`); the browser walk itself is still to be recorded as an e2e use case.
- Carrier-API / EDI event ingestion (Phase 2 only)
- Load calculator BRS §5 scenario walk-through via UI (algorithm is unit-tested at the model layer)
- Browser-driven cross-module flow into Booking app to allocate equipment.usage on `mark_ready` — covered in the booking module's e2e suite
