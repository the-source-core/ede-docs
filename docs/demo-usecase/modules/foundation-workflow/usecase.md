# `foundation.workflow` — Demo Use-Case

**Module:** `ede.foundation.workflow`
**App key:** `foundation.workflow`
**Demo manifest entries** (target): _none directly_ — workflow demo state lives in domain demo files
**Status:** 🔴 Not authored (intentional — see below)

---

## Use-case

Workflow definitions (`ir.workflow.definition`, stages, transitions, guards) are **production** seeds shipped by each consumer module's `data/*.xml` — e.g. `crm.inquiry.lifecycle`, `pricing.rate.lifecycle`. They are not demo concerns.

The demo state we want is records *sitting in different stages* of those workflows, so a tester opening a list sees a realistic distribution (not everything `Draft`). That state is set by domain modules in **their** demo files, using `<field name="status">` to start the record at a non-initial stage (the workflow engine accepts seeded stage values on create).

Documenting it here so future maintainers don't reflexively create `foundation.workflow/demo/*` files that duplicate domain demo state.

## What the consumer-side demo files cover

| Workflow definition | Where demo records live | Stages exercised by demo |
|---|---|---|
| `crm.inquiry.lifecycle` | `logistics.sales-crm/demo/demo_pipeline.xml` | `new`, `qualified`, `converted` |
| `crm.lead.lifecycle` | same | `discovery`, `qualified` |
| `crm.opportunity.lifecycle` | same | `negotiation` (one), `won` (one), `lost` (one for the lost-reason guard) |
| `pricing.rate.lifecycle` | `logistics.pricing/demo/demo_rates.xml` | `draft`, `pending_approval`, `active` |

## Records this module's demo file produces

_None._ If a phase enhancement to `foundation.workflow` itself introduces a model that ships records (e.g. a workflow-template gallery), this section moves out of "none" and a `demo/*.xml` is added.

## Out of scope

- Stage definitions themselves — production seeds.
- Guard rules — production seeds.
- Manual transition logs / audit history — generated at runtime, not seeded.

## Verification

The workflow demo is verified through its consumers — listing demo CRM leads should show a mix of stages; the demo pricing rate should show in approval flow correctly.

---

*Back to [demo-usecase index](../../README.md).*
