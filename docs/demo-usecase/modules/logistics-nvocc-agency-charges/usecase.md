# `logistics_nvocc_agency.agency_charges` — Demo Use-Case

**Module:** NVOCC / Agency — Detention, Demurrage & Container Charges (`src/domains/logistics_nvocc_agency/agency_charges/`)
**Manifest key:** `demo: ["demo/demo_agency_charges.xml"]`
**Load:** `ede migrate upgrade -t <tenant> --with-demo=logistics_nvocc_agency.agency_charges`
**Prerequisite demos:** `logistics_nvocc_agency.agency_booking` (the Sealink principal, the booking, the Globex customer) · `logistics_base.equipment_operations` (the containers)
**BRD:** N7 — Detention, Demurrage & Container Charges
**Roadmap:** [roadmap/logistics/nvocc-agency/phase-7/README.md](../../../../roadmap/logistics/nvocc-agency/phase-7/README.md)

---

## The story this data tells

Acme Forwarding Ltd. represents **SEALINK CONTAINER LINE** out of Nhava Sheva (the third
principal in the unifying scenario, added by the BRD N3 demo). One of Sealink's 40ft dry
boxes lands on the 1st of August, is collected on the 12th and comes back empty on the
25th. Nobody disputes the dates. Everything else about that box is disputed:

* the customer was sold **ten** detention free days but was never sold demurrage free
  time at all, so the two clocks on the same container resolve from **two different
  sources** — which is exactly the conflict N7-02's priority order exists to settle;
* the overrun lands in the **second tariff band** on both clocks, so the charge is not a
  flat multiplication and the breakdown has something to show;
* a second box has been discharged and collected but **never returned**, so calculating
  detention against it is refused rather than answered with zero.

That last one is the whole phase in one record. A zero charge and an uncalculable charge
must never look the same.

## Records shipped

| Model | Records | Illustrates |
|---|---|---|
| `logistics.free.time.term` | 1 — 10 detention days on the booking | N7-01, and a `customer_contract` source that beats the standard |
| `logistics.equipment.usage` | 2 — one complete cycle, one still open | the subject of both clocks |
| `logistics.equipment.movement.event` | 5 — discharge / full gate-out / empty return, plus an incomplete pair | N7-03, N7-24 |
| `pricing.rate` + `.line` + `.charge.slab` | 1 rate, 2 lines, 5 slabs | N7-09 — **no D&D tariff model exists**; the delivered triple carries every dimension |
| `ir.rbac.role.binding` | 3 — demo only | the three ROLE approval steps this module adds |

**Nothing else is seeded, and that is deliberate.**

* **No calculation.** `nvocc.dnd.calculation.status` is `workflow=True`, and the number
  itself is the engine's output rather than a fixture. Seeding one would demonstrate a
  system that does not exist. The reviewer runs the engine — that is the exercise.
* **No waiver, dispute, adjustment or hold.** Each is a lifecycle, and a pre-approved
  fixture teaches nothing about the control it is supposed to demonstrate.
* **The tariff loads in `draft`.** `pricing.rate.status` is `workflow=True` and the
  resolver accepts only an approved rate, so the first calculation is *refused*. That
  refusal is the control, not a defect.

## Walkthrough

### 1 — Walk into the tariff refusal on purpose

Dispatch `nvocc.charge.calculate` before approving anything:

```json
{
  "free_time_type_id":   "<commercial_terms.free_time_type_demurrage>",
  "trade_direction":     "import",
  "customer_partner_id": "<Globex>",
  "principal_id":        "<Sealink>",
  "booking_id":          "<the Sealink booking>",
  "equipment_ids":       ["<MSKU4200002>"],
  "currency_id":         "<USD>"
}
```

It is refused naming the missing approved tariff. Then: **Pricing → Rates → "Sealink D&D
Tariff INNSA" → `pricing.rate.submit` → `pricing.rate.approve`**, and re-run.

### 2 — Read the explanation, not the number

Discharged 2026-08-01 09:00 → full gate-out 2026-08-12 14:00 is **11 calendar days**,
minus the 5-day demurrage standard = **6 chargeable**, priced 3 days at USD 60 and 3 at
USD 95 = **USD 465**.

Open the charge line. Its `explanation` names the two events, the dates, the free days and
the slab, and its `slab_breakdown_json` shows the arithmetic band by band. That sentence
is the engine's real product; the number is only its conclusion.

### 3 — Two clocks, two sources

Run the same command with `free_time_type_id = free_time_type_detention`. Full gate-out
2026-08-12 → empty return 2026-08-25 is **13 days**, minus the customer's negotiated
**10** = 3 chargeable.

Now open both free-time resolutions side by side. Detention resolved from
`customer_contract` at priority 3; demurrage fell through to `company_default` at priority
6, and its `explanation` names what it beat. One box, two sources, both defensible.

### 4 — Recalculation supersedes, it never overwrites

Edit the booking's free-time term from 10 days to 8 and re-run the detention calculation.
The charge changes; the previous calculation moves to `superseded` and stays readable, and
the two are linked in both directions. Re-run again **without** changing anything: the
input fingerprint matches, and nothing is written at all.

### 5 — The billing gate is computed, not ticked

Place a hold: `nvocc.charge.place_hold` with `hold_reason: "finance_review"`. The charge's
`is_billing_ready` flips to false and `blocking_reason` names the hold. Release it with
`nvocc.charge.release_hold` (a release reason is mandatory) and the gate re-opens by
itself. Nobody ticked anything, because there is nothing to tick.

Try the same with a dispute: `nvocc.charge.dispute.open`, then resolve it. Same behaviour,
different reason text.

### 6 — A waiver reduces, it does not erase

Raise a waiver for USD 200, submit it (a Commercial Manager approval case opens — the demo
role binding is what makes it assignable), approve it, then `nvocc.charge.waiver.apply`.

`calculated_amount` does **not** move. `waived_amount` and `net_amount` do. That is what
makes the waiver-leakage report mean anything: if the waiver had edited the charge, the
denominator would have disappeared along with the evidence.

### 7 — The refusal that is the point

Run detention against **MSKU4200003** (the box that has not come back). The calculation is
refused with a named exception listing the missing feed key — not returned as zero. Pass
`"allow_estimate": true` and it produces a visibly-estimated line instead, which shows
exposure and can never be marked billing-ready (BR-N7-02).

## What this demo deliberately cannot show

| Not demonstrated | Why |
|---|---|
| The finance hand-off actually reaching an accounting system | `foundation.connectors` needs a registered `finance` provider. The category is registered by this module and the hand-off ledger works; there is no provider to talk to in a demo tenant, and the attempt is recorded as `failed` with its message — which is itself the correct, visible behaviour. |
| Tax on a charge line | Requires `res.tax.code` rows on the seeded charge products and a fiscal position. The mechanism is `foundation.tax`'s and is composed, not reimplemented; a tenant with the tax demo loaded sees it resolve. |
| Working-day counting against a real holiday calendar | `foundation.l10n` is not delivered. Weekday counting works and an explicit holiday list is accepted; there is no country or port calendar to draw from. |
| Charge statements and alerts | BRD-N7-Phase-2 (N7-22, N7-23). |
