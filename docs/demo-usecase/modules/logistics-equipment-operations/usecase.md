# `logistics.equipment-operations` — Demo Use-Case

**Module:** `domains.logistics.equipment_operations`
**App key:** `logistics.equipment_operations`
**Demo manifest entries** (Phase 1 e2e): none — tests seed programmatically via `env` per the canonical Phase 1 fixture discipline.
**Status:** ✅ Phase 1 Delivered (2026-05-27)

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
