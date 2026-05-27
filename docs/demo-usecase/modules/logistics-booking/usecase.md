# `logistics.booking` — Demo Use-Case

**Module:** `domains.logistics.booking`
**App key:** `logistics.booking`
**Demo manifest entries** (Phase 1 e2e): none — tests seed programmatically via `env` per the canonical Phase 1 fixture discipline.
**Status:** ✅ Phase 1 Delivered (2026-05-27) including Integration phase

---

## Use-case

A forwarder running on EDE wants to demo the Booking module — the conditional provider-reservation layer between Handover (sales→ops bridge) and Shipments (cargo in motion). This use case exercises the Booking app surface after the Phase 1 + Integration phase ship: app-switcher entry visible, booking list renders, form view opens with the 6-state status workflow header + 11-section sheet (Identity / Source / Route / Schedule / Cut-offs / Primary Provider / Equipment / Totals / Operational Ownership / Cancellation / Notes), and the Y-fork command (`logistics.booking.create_from_handover`) is routable through the live command bus.

The Phase 1 demo deliberately keeps the data narrow — the goal is to prove the wire from clicked button → FastAPI → command bus → SQLAlchemy → React webclient render. Each test method seeds its own minimal records via the live env (using `booking.allow_unhandover=true` config to bypass handover requirement for the bare-bones smoke), asserts the UI surface, then tears down.

## Recorded e2e tests

| Test file | Steps | Use case |
|---|---|---|
| `tests/e2e/test_booking_smoke.py` | App-switcher → Booking → All Bookings list renders · Direct creation via env (allow_unhandover=true) · ORM proofs of registry + cross-module FK to equipment.usage | Phase 1 smoke — proves the Booking app loads against the live tenant after the 7-table Alembic migration `d7a3f0bcb4f7` + Integration migration `cd256b07f55b` apply. ORM-level assertions on `logistics.booking`, `logistics.booking.equipment.allocation`, and the cross-module FK to `logistics.equipment.usage`. Plus the @api.extend_model contribution of `path_taken` / `booking_id` / `path_decided_at` to `crm.handover`. |

## Notable Phase 1 design intent recorded for testing

- **Y-fork is intentional**: `crm.handover` is a source→destination mapping (quote → booking-or-shipment), not an operational record. Operational data lives on booking. Tests for `create_from_handover` accept caller-supplied overrides for transport_mode / POL / POD / cut-offs until `logistics.sales-crm` Enhancement 10 enriches the handover.
- **Slice 09 (equipment.allocation) is delivered in Integration phase**: the cross-module FK to `logistics_equipment_usage.record_uuid` lives on the new `logistics_booking_equipment_allocation` table. `mark_ready` dispatches `allocate_equipment_usage` inside its all-pass branch; `_apply_cancellation` dispatches `release_equipment_usage`. Both degrade cleanly if the equipment_operations module isn't loaded.
- **Feasibility check auto-creates on `booking.created`**: every new booking gets a pending feasibility row via the `@api.on_event` handler. Tests assert this side effect.

## What this demo does NOT cover (yet)

- Full booking lifecycle browser walk (handover → create → provider request → confirm → mark_ready → cancel) — requires accepted handover + carrier partner with role; deferred to a follow-up rich demo session
- Provider confirmation workflow + booking-header cascade end-to-end via UI (covered at registry layer by 9/9 smoke tests)
- Amendment workflow with commercial review approval — requires foundation.approval flow seeded with the `booking.amendment.commercial_review` template
- Cancellation outside grace window opening an `ir.approval.case` — requires the foundation.approval engine in active state during the test session
- Browser-driven feasibility check commit + escalation event consumed by sales-crm (covered by the cross-module event contract; not exercised in browser yet)
