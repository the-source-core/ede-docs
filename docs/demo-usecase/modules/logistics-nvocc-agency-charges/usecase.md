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

---

## Enhancement 01 — demo-data expansion

**Status:** ✅ Delivered 2026-11-22 — 7 records in 2 files; `--with-demo=all` smoke green and idempotent. Four product defects surfaced by the live walkthrough are carried in the defects table below.
**Roadmap:** [nvocc-agency Enhancement 01](../../../../roadmap/logistics/nvocc-agency/enhancements/01-demo-data-expansion.md). The cross-module walkthrough order lives there, and step IDs match it.
**New demo files** (appended **after** `demo/demo_agency_charges.xml`):
`demo/demo_charges_bluewater_import.xml` (6) · `demo/demo_charges_exceptions.xml` (1)

**A load-order constraint shaped both files.** Data load is topological with an alphabetical
tiebreak, and this module does not depend on `agency_documentation` or `agency_release`, so it
can load **before** them. Neither file may `ref=` a B/L, a DO or a customs entry. Both reference
only `agency_booking` (booking, containers), `agency_masters` (Bluewater), `sales_crm` (Wayne) and
this module's own seeds.

### Use case 1 — import D&D at Nhava Sheva: the tariff is the only precondition

> Wayne's two boxes were discharged on 6 October and held at the CFS under a customs examination
> until the 15th. Both left on the 16th. Box 1 came back empty on the 19th. Box 2 was turned away
> from the empty depot twice during a two-day closure and came back on the 26th.

The free days are the **company defaults** (demurrage 5, detention 7 —
`logistics.free.time.type.master.default_free_days`), so no free-time term ships. The Sealink file
already shows a booking term beating the standard. What this file adds is a second principal's
tariff, and with it the rule that a principal-scoped tariff **does not leak**: the Sealink tariff
is excluded for a Bluewater charge, so until this one is approved the calculation is refused, not
priced from the wrong line.

#### Records produced — `demo/demo_charges_bluewater_import.xml` (6)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_charges.demo_rate_dnd_bluewater` | `pricing.rate` | `rate_type=sell` · `rate_source=tariff` · `masters.transport_mode_sea` · `principal_id` Bluewater · `destination_location_id` INNSA · USD · 2026-01-01 → 2026-12-31 · `entity_id` default org | Lands **draft** (`status` is `workflow=True`). The resolver accepts only `approved`, so the first calculation is refused on purpose. The location is for the reader; the resolver scopes on principal, customer and entity |
| `agency_charges.demo_rate_line_dem_bluewater` | `pricing.rate.line` | `agency_charges.product_charge_dem` · `masters.eq_type_42gp` · `unit_rate` 125 · USD · customer-visible | Demurrage, 40ft dry |
| `agency_charges.demo_slab_dem_bluewater_1_3` | `pricing.rate.charge.slab` | days 1–3 · USD 80 (cost 55) | — |
| `agency_charges.demo_slab_dem_bluewater_4_plus` | `pricing.rate.charge.slab` | day 4, open top band · USD 125 (cost 85) | The boxes land here, so the breakdown shows two bands |
| `agency_charges.demo_rate_line_det_bluewater` | `pricing.rate.line` | `agency_charges.product_charge_det` · 42GP · `unit_rate` 60 · USD | Detention, 40ft dry |
| `agency_charges.demo_slab_det_bluewater_1_plus` | `pricing.rate.charge.slab` | day 1, open · USD 60 (cost 40) | One flat band, so the extension's effect in Use case 3 is legible |

**Block split.** All six tariff records sit in **`noupdate="1"`**. Once the walkthrough approves
the tariff, the rate is locked behind its approval case, and so are its lines and slabs
(`_check_parent_rate_locked`). A reload of updatable rows would fail. The line notes use the
model's real field, `remarks`; `pricing.rate.line` has no `notes` column. **Known gap, unchanged:**
the older Sealink tariff in `demo_agency_charges.xml` is still updatable, so re-running
`--with-demo` after its own walkthrough approves it would hit the same lock. That file's two line
notes were corrected to `remarks` in the same pass.

**Expected numbers**, with the event times the walkthrough passes:

| Clock | Box 1 `BLWU4200308` | Box 2 `BLWU4200313` |
|---|---|---|
| Demurrage, discharged 06 Oct → gate-out 16 Oct: 10 days − 5 free = 5 chargeable | 3 @ 80 + 2 @ 125 = **USD 490** | **USD 490** |
| Detention, gate-out 16 Oct → empty return | 19 Oct: 3 days, inside 7 → **USD 0.00** (a *calculable* zero) | 26 Oct: 10 days − 7 = 3 @ 60 → **USD 180** |

### Use case 3 — a demurrage dispute settled by a waiver, and a free-time extension

> Wayne disputes the USD 980 demurrage: the boxes sat under a customs examination order, and the
> cargo was ready to collect on the 9th. Commercial concedes the band-2 days and holds the rest.
> Separately, Wayne asks for **three** more detention days on box 2 because the depot was closed.
> Commercial grants **two**.

These are two instruments on two clocks. A **waiver** reduces a correctly calculated charge; an
**extension** changes the inputs so the charge is calculated differently. They are reported
separately because a business that cannot tell "we forgave money" from "we gave away days" cannot
see its own leakage.

#### Records produced — `demo/demo_charges_exceptions.xml` (1)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_charges.demo_freetime_extension_wayne_box_2` | `nvocc.freetime.extension` | `scope=container` · `equipment_id` `agency_booking.demo_equipment_bluewater_box_2` · `booking_id` `agency_booking.demo_booking_bluewater_import` · `customer_partner_id` Wayne · `principal_id` Bluewater · default org · `free_time_type_id` `commercial_terms.free_time_type_detention` · `requested_days` 3 · `reason` "Empty depot DEP-NSICT refused empty returns 22–23 Oct (yard resurfacing); Wayne's haulier turned away twice — gate refusal slips attached" · `customer_reference` WAYNE-DND-2610-07 | The **one** exception record that can be a fixture: it needs no calculation, and `requested` is its initial stage. `status` and `approved_days` are deliberately not set. In a **`noupdate="1"`** block, because a request that has entered approval must not be rewritten by a reload |

### What this slice deliberately does NOT ship

| Not shipped | Why |
|---|---|
| Any calculation | `nvocc.dnd.calculation.status` is `workflow=True`, and the number is the engine's output, not a fixture |
| A dispute, waiver or charge hold | All three declare `calculation_id` `required=True`. None can exist before a calculation, and none of them is a fixture |
| A free-time term on the Bluewater booking | The company defaults are the point; see Use case 1 |
| An approved extension | `approved_days` is the approver's decision. The workflow declares a `days_granted` guard (> 0), but it is **not enforced today**: see defect D1 below |
| An approved tariff | The refusal in UC1-14 is the control |

### Walkthrough — the steps this module owns

The event times come from the voyage slice (discharge, UC1-8/9) and the release slice (gate-out
UC1-12, empty return UC1-13). The common payload for `nvocc.charge.calculate` is
`trade_direction: import`, customer Wayne, principal Bluewater, the Bluewater import booking,
`equipment_ids` = both boxes, and currency USD.

| Step | Do this | What it shows |
|---|---|---|
| UC1-14 | `nvocc.charge.calculate` with `free_time_type_id` = demurrage, **before** approving anything | *(Verified live.)* **Refused**: *"No approved sell rate covers this charge code for this customer, principal and date…"*. The Sealink tariff is **excluded** because its principal scope differs; it is never borrowed |
| UC1-15 | Pricing → Rates → the Bluewater D&D tariff → `pricing.rate.submit` → `pricing.rate.approve`, then re-run | *(Verified live.)* **USD 980**: two lines, each with 5 chargeable days and USD 490. The free days resolve from `company_default` (5) |
| UC1-16 | The same with `free_time_type_id` = detention | *(Verified live.)* Box 1 has 0 chargeable days → **USD 0.00**. Box 2 has 3 days → **USD 180** |
| UC3-5 | `nvocc.charge.dispute.open` against the demurrage calculation: customer Wayne, `dispute_reason=delay_not_ours`, reason "Boxes held 8–15 Oct under a Customs RMS examination order; cargo ready to collect from 9 Oct", `disputed_amount` 980, `customer_reference` WAYNE-DND-2610-05 | *(Verified live.)* The calculation moves to `disputed`, and `blocking_reason` adds *"1 open dispute(s) — resolve or reject them first (BR-N7-09)"*. Billing readiness reads the dispute rows, not a ticked box. The gate also needs the calculation itself approved, which it names alongside the dispute |
| UC3-6 | `start_review` → `partially_accept`. Raise a waiver: `waiver_type=partial`, `requested_amount` 500, **`approved_amount` 500** (the only figure the line's `waived_amount` reads), reason "Concede band-2 days — examination delay, not the consignee's". Then `submit` (a Commercial Manager case) → approve → `nvocc.charge.waiver.apply` | Review, partial acceptance, waiver submit and approval all pass. **`apply` is refused today by defect D3** (*"Guard expression failed: … approved_amount > 0"*, although the amount is 500). Once D3 is fixed the intended result is `calculated_amount` **980** unchanged, `waived_amount` **500**, `net_amount` **480**: the evidence survives the concession |
| UC3-7 | `nvocc.charge.dispute.resolve` with **no** `resolution`, then with `resolution=waived`, `accepted_amount` 500 and notes | *(Verified live.)* The first call is **refused**, naming the four outcomes (*"waived, adjusted, billed or cancelled (N7-17)"*). The second moves the dispute to `resolved`. **Defect D4:** its stored `is_open` stays true, so the calculation stays `disputed` and the gate still names the dispute |
| UC3-8 | Charges → Free Time Extensions → WAYNE-DND-2610-07 → `nvocc.freetime.extension.announce_approved` | *(Verified live.)* **Refused**: *"Only an approved extension is announced; this one is 'requested'…"* |
| UC3-9 | Set Approved Days to **2** (of 3 asked) → `submit` → approve the Commercial Manager case | *(Verified live.)* `approved`. `requested_days` survives beside `approved_days`, so the approval-cut rate is reportable. **Defect D1:** an approval at 0 days is *also* accepted, so set the days before deciding |
| UC3-10 | `announce_approved`, then `nvocc.charge.recalculate` on the open detention calculation with a reason ("approved free-time extension WAYNE-DND-2610-07") | `announce_approved` succeeds but **re-runs nothing today (defect D2)**, so the explicit recalculation is required. Box 2 is then 10 − (7 + 2) = 1 day → **USD 60**. The USD 180 result moves to `superseded`, linked both ways (verified live). The resolution reads *company_default 7 + extension 2*, not "9": seven and then two more |

#### Defects found by the live verification (carried, not fixed — product code)

| # | Defect | Where |
|---|---|---|
| D1 🔴 | An approval-completion (`on-event`) transition's `guard` is never evaluated: `WorkflowEngine.complete_async_transition` writes the target stage directly. Every such guard is skipped: the free-time extension's `days_granted` (a 0-day extension lands `approved`), the principal's `effective_date_reached` (`agency_masters/data/principal_workflow.xml`, so **a principal can go active before its effective date**) and the CRO amendment's `change_is_described` (`agency_booking/data/cro_amendment_workflow.xml`) | `src/ede/foundation/workflow/services/engine.py` (platform) |
| D2 | `on_extension_approved` re-runs nothing: event handlers get an env with no active organisation, and `nvocc.dnd.calculation` is `company_scope="strict"`, so the search returns 0 rows. Two fix options, neither `sudo()` nor `with_company_test(False)`: (a) in `agency_charges`, carry the extension's `organization_id` in the event payload and run the handler under `env.with_active_organization_id(org)`; or (b) a platform change, needing user approval, where event envs inherit the emitter's organisation | `agency_charges/services/recalculation.py` (option a) or `ede/cli/commands/server.py` `env_factory` (option b) |
| D3 | The waiver `apply` guard `approved_amount > 0` always fails: the guard's subject accessor returns the Monetary value as a string (`'500'`). This also blocks step 6 of the walkthrough at the top of this doc | `data/charge_waiver_workflow.xml` × `foundation.workflow` subject accessor |
| D4 | After `dispute.resolve`, the stored `is_open` does not follow the workflow's status write (`resolved`, `is_open = true`) | `agency_charges/models/dispute.py` |

Full write-up: [Enhancement 01 — Findings](../../../../roadmap/logistics/nvocc-agency/enhancements/01-demo-data-expansion.md#findings-from-the-live-verification--four-product-defects-carried).

### Dependencies and load order

`agency_booking` (the Bluewater booking and containers) · `agency_masters` (Bluewater) ·
`agency_commercial` (`principal_id` on `pricing.rate`) · `logistics_base.commercial_terms` (free
time types) · `sales_crm` (Wayne) · this module's own seeds (`product_charge_dem` / `_det`) and its
existing demo file (the Commercial Manager and D&D Executive role bindings that make the waiver,
dispute and extension approvals assignable). Load with `--with-demo=all`.

### Smoke test — ✅ green and idempotent, 2026-11-22

`ede migrate upgrade -t agencydemo3 --with-demo=all` on a fresh PostgreSQL tenant, run twice, exit
0, zero loader errors. Actual counts match the prediction exactly:

| File | Pass 1 | Pass 2 |
|---|---|---|
| `demo_charges_bluewater_import.xml` | created=6 | created=0 · updated=0 · skipped=6 |
| `demo_charges_exceptions.xml` | created=1 | created=0 · updated=0 · skipped=1 |

All 7 external ids are `is_demo = true`. The extension loads as `requested` with
`requested_days=3` and `approved_days=0`.

**Reload after the walkthrough (agencydemo3):** with the Bluewater tariff approved, the reefer booking confirmed and the DG SI validated through the real commands, a third `--with-demo=all` run exits 0 with zero loader errors, and all three keep their walked state.

Then check that **Pricing → Rates** shows the Bluewater D&D tariff in Draft with two lines and
three slabs, and that **Free Time Extensions** shows WAYNE-DND-2610-07 in `requested`.
