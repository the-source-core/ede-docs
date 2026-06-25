# `logistics.operations_execution` — Demo Use-Case

**Module:** `src/domains/logistics/operations_execution`
**App key:** `logistics.operations_execution`
**Demo manifest entries** (target): `demo/demo_execution.xml`, `demo/demo_operations.xml`
**Status:** ✅ Delivered 2026-06-25

---

## Use-case

The Control Tower takes over once the Globex ocean export shipment
(`shipments.demo_shipment_globex_001`, Mumbai → Singapore, FCL) is activated for
execution. The demo seeds **one live execution** for that shipment, mid-flight
(`in_execution`), so the Control Tower app shows a realistic operational picture
on a fresh `--with-demo` tenant: a generated execution plan, a pre-carriage road
leg plus the main sea leg, the task checklist (some done, some in progress),
planned-vs-actual milestones, the cargo pickup at the Globex factory, the
onward delivery to the Singapore consignee, the trucking dispatch with its
driver, the gate-in cargo event, the SI / VGM cut-offs, the nominated trucking
provider, and one open exception (a vessel rollover delay) being worked by the
operations manager. A customer SOP master for Globex captures the standing
handling instructions the plan was built against.

Every actor, lane and shipment is the same one used across the rest of the
unifying scenario in [docs/demo-usecase/README.md](../../README.md) — no new
customer, port or carrier is introduced.

## Records produced

### `demo/demo_execution.xml`

| External ID | Model | Notes |
|---|---|---|
| `operations_execution.demo_sop_globex` | `logistics.customer.sop.master` | Globex standing SOP — door pickup, SI 48h before cut-off |
| `operations_execution.demo_execution_globex` | `logistics.execution` | Control-tower record for the Globex shipment, `in_execution` |
| `operations_execution.demo_plan_globex` | `logistics.execution.plan` | Generated plan — direct / port-to-port, 4 tasks + 5 milestones |
| `operations_execution.demo_leg_precarriage` | `logistics.execution.leg` | Seq 1 road pre-carriage (factory → INMUM), completed |
| `operations_execution.demo_leg_main` | `logistics.execution.leg` | Seq 2 sea main-carriage (INMUM → SGSIN), in progress; binds `shipments.demo_shipment_leg_001` |
| `operations_execution.demo_task_confirm_booking` | `logistics.execution.task` | Confirm booking + equipment — done |
| `operations_execution.demo_task_pickup_order` | `logistics.execution.task` | Generate pickup order — done |
| `operations_execution.demo_task_submit_si` | `logistics.execution.task` | Submit shipping instructions — in progress |
| `operations_execution.demo_task_customs_docs` | `logistics.execution.task` | Customs documentation — not started |
| `operations_execution.demo_ms_booking_confirmed` | `logistics.milestone` | Booking Confirmed — reached |
| `operations_execution.demo_ms_gated_in` | `logistics.milestone` | Gated In at Origin — reached |
| `operations_execution.demo_ms_departed` | `logistics.milestone` | Vessel Departed — pending |
| `operations_execution.demo_ms_arrived` | `logistics.milestone` | Vessel Arrived — pending |
| `operations_execution.demo_ms_delivered` | `logistics.milestone` | Delivered — pending |
| `operations_execution.demo_completion_globex` | `logistics.execution.completion` | Completion tracker — pending (shipment still in flight) |

### `demo/demo_operations.xml`

| External ID | Model | Notes |
|---|---|---|
| `operations_execution.demo_provider_trucker` | `logistics.execution.provider` | Nominated pre-carriage trucker — confirmed |
| `operations_execution.demo_pickup_factory` | `logistics.cargo.pickup` | Cargo pickup at the Globex Mumbai factory — confirmed |
| `operations_execution.demo_delivery_consignee` | `logistics.destination.delivery` | Onward delivery to the Singapore consignee — planned |
| `operations_execution.demo_dispatch_precarriage` | `ops.dispatch.order` | Pre-carriage dispatch (gate-in tractor + chassis) — completed |
| `operations_execution.demo_driver_precarriage` | `ops.driver.assignment` | Driver assigned to the pre-carriage dispatch — completed |
| `operations_execution.demo_event_gate_in` | `logistics.cargo.handover.event` | Gate-in cargo event at INMUM, on time |
| `operations_execution.demo_cutoff_si` | `logistics.execution.cutoff` | Shipping-Instructions cut-off — met |
| `operations_execution.demo_cutoff_vgm` | `logistics.execution.cutoff` | VGM cut-off — at risk |
| `operations_execution.demo_exception_rollover` | `ops.exception.case` | Vessel rollover — transport delay, medium severity, open |

## Out of scope

- Production-seeded reference masters (`ops.sla.policy.master`, `ops.exception.code.master`,
  `logistics.milestone.template.master`, `logistics.task.template.master`,
  `logistics.visibility.rule.master`) ship in the `data: [...]` channel for every
  tenant — they are referenced by the demo records but not re-authored here.
- Approval routing for the exception (the `pending_approval` state) is exercised by
  the command surface + approval engine, not pre-seeded.
- Multi-shipment / consolidation execution (house + console fan-out) — the demo
  covers the single direct Globex ocean shipment only.

## Dependencies

Loaded (demo pass, dependency order) before this module:

- `foundation.base` demo — organisation (`base.default_organization`).
- `logistics.masters` demo + seeds — `masters.demo_port_inmum`, `masters.demo_port_sgsin`,
  `masters.transport_mode_sea`, `masters.transport_mode_road`.
- `logistics.sales_crm` demo — `sales_crm.demo_partner_customer_globex` (customer),
  `sales_crm.demo_partner_customer_stark` (consignee).
- `logistics.equipment_operations` demo — `equipment_operations.demo_equipment_tractor_001`,
  `equipment_operations.demo_movement_22gp_gate_in`.
- `logistics.shipments` demo — `shipments.demo_shipment_globex_001`,
  `shipments.demo_shipment_leg_001`.
- This module's own production seeds — `operations_execution.ops_*` template / SLA /
  exception-code masters (loaded from `data/*.csv` before the demo pass).

## Verification

`ede migrate upgrade -t <tenant> --with-demo=logistics.operations_execution` — expected
**24 created** (15 in `demo_execution.xml` + 9 in `demo_operations.xml`), then re-run
idempotent (`0 created, 24 updated`).

```sql
SELECT model_key, count(*)
  FROM ir_data_reference
 WHERE is_demo = true AND module = 'operations_execution'
 GROUP BY model_key ORDER BY model_key;
```

UI: log in as the operations manager and open the Control Tower → the Globex
execution shows the plan, two legs, the task board, the milestone timeline, and
the open rollover exception.

## Authoring order

`demo_execution.xml` first (the execution + plan are `ref=`d by every operations
record), then `demo_operations.xml`. Within `demo_execution.xml`: SOP master →
execution → plan → legs → tasks → milestones → completion. Manifest `demo: [...]`
order mirrors this.

---

*Back to [demo-usecase index](../../README.md).*
