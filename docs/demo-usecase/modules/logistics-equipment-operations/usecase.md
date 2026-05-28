# `logistics.equipment-operations` — Demo Use-Case

**Module:** `domains.logistics.equipment_operations`
**App key:** `logistics.equipment_operations`
**Demo manifest entries:** `demo/demo_equipment.xml`, `demo/demo_operations.xml` (Enhancement 02 — retrofit 2026-05-28)
**Status:** ✅ Phase 1 Delivered (2026-05-27); ✅ Demo data Delivered (2026-05-28, Enhancement 02)

---

## Demo data records produced (Enhancement 02 — `--with-demo=logistics.equipment_operations`)

Anchored to the unifying scenario in [docs/demo-usecase/README.md](../../README.md): **Acme Forwarding Ltd. (Mumbai HQ)** as an asset-owning forwarder running its own fleet on the `INMUM → SGSIN` lane. Custodian is the seeded `base.default_organization`; every CECS master referenced (`masters.eq_type_*`, `masters.eq_status_*`, `masters.eq_cond_*`, `masters.usage_status_*`, `masters.seal_type_*`, `masters.movement_event_*`) ships from the `logistics.masters` production seed CSVs — the demo references them, never recreates them.

### `demo/demo_equipment.xml` — the Acme fleet (7 units)

| External ID | Model | Notes |
|---|---|---|
| `equipment_operations.demo_equipment_42gp_001` | `logistics.equipment` | 40ft dry (MSKU4200001) · owned · Available / Good |
| `equipment_operations.demo_equipment_42gp_002` | `logistics.equipment` | 40ft dry (MSKU4200002) · owned · Available / Good |
| `equipment_operations.demo_equipment_42rt_001` | `logistics.equipment` | 40ft reefer (MSKU4250001) · owned · Available / Good |
| `equipment_operations.demo_equipment_22gp_001` | `logistics.equipment` | 20ft dry (MSKU2200001) · owned · In-Use / Good |
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

## Demo dependencies

- `base.default_organization` (seeded res.organization) — custodian + organization on every record.
- `logistics.masters` production seed CSVs — all CECS taxonomy / status / condition / usage-status / seal-type / movement-event-type rows referenced by `masters.*` xml-id.
- No foundation.base demo dir required — the demo is self-contained against the default org + seeded masters.

## Demo verification

`ede migrate upgrade -t <tenant> --with-demo=logistics.equipment_operations` → 13 created (7 equipment + 6 operations); re-run reports `updated`, not `created` (records use `noupdate="0"`). UI: Equipment Control → Equipment → All Units shows the 7-unit fleet; Operations → Usage / Movement Events / (seals) render the lifecycle rows; Planning → Load Calculator shows the stuffing plan.

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
- BR-EVT-10 event-code → usage-status mapping hardcoded (`loaded → active`, `delivered → completed`, `cancelled → cancelled`)
- BR-CALC-05 hazmat/temperature compatibility gates pass-through pending `equipment.type.master` schema enhancement

These deviations are stable surfaces — the e2e tests verify the registry contracts and the UI flows that depend on them, not the deviation-specific shortcuts.

## What this demo does NOT cover (yet)

- Full equipment lifecycle browser walk (register → allocate → seal → move → deliver → complete) — requires richer seeded state; deferred to Phase 2 or follow-up
- Carrier-API / EDI event ingestion (Phase 2 only)
- Load calculator BRS §5 scenario walk-through via UI (algorithm is unit-tested at the model layer)
- Browser-driven cross-module flow into Booking app to allocate equipment.usage on `mark_ready` — covered in the booking module's e2e suite
