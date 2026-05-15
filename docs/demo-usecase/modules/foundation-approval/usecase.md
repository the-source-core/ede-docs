# `foundation.approval` — Demo Use-Case

**Module:** `ede.foundation.approval`
**App key:** `foundation.approval`
**Demo manifest entries** (target): `demo/demo_approval_rules.xml`, `demo/demo_approval_case_sample.xml`
**Status:** 🔴 Not yet authored

---

## Use-case

Demonstrate the **two-step approval flow for pricing rates** end-to-end on a demo tenant, using the unifying-scenario actors:

- Sales rep submits a Mumbai → Singapore rate (loaded by `logistics.pricing` demo).
- Ops manager approves at step 1.
- Finance approver approves at step 2 (for high-value rates).
- A second submitted rate is shown in *pending* state so the inbox demo isn't empty.

## Records produced

### `demo/demo_approval_rules.xml`

| External ID | Model | Notes |
|---|---|---|
| `approval.demo_rule_pricing_rate` | `ir.approval.rule` | Subject: `pricing.rate`; trigger on submit; two-step flow (ops_manager → finance_approver) |
| `approval.demo_flow_pricing_rate` | `ir.approval.flow.def` + steps | Step 1: `ref(base.demo_user_ops_manager)`; Step 2: `ref(base.demo_user_finance)` when `subject.total_amount > 5000 USD` |

### `demo/demo_approval_case_sample.xml`

| External ID | Model | State |
|---|---|---|
| `approval.demo_case_rate_inmum_sgsin_lcl` | `ir.approval.case` | linked to `ref(pricing.demo_rate_inmum_sgsin_lcl)` — step 1 already approved by ops_manager, step 2 pending finance |

A second case (`approval.demo_case_rate_sgsin_inmum_lcl`) loaded in fully-approved state so users can see both `pending` and `approved` cases in the inbox.

## Out of scope

- New role definitions — uses existing `pricing.ops_manager` / `pricing.finance_approver` roles seeded in `foundation.base` production data.
- Notification templates — those are production seeds; demo just produces case events that re-use existing templates (handled by `foundation.notifications` demo).

## Dependencies

- `foundation.base` demo (`base.demo_user_*` users)
- `logistics.pricing` demo (`pricing.demo_rate_*` subjects to attach cases to)

## Verification

```
ede migrate upgrade -t demo --with-demo=all
```

Log in as `demo_user_ops_manager` → see pending case in Approvals inbox; advance to step 2. Log in as `demo_user_finance` → second-step approval. End state: case `approved`, rate flips to `Active`.

## Authoring order

1. Pricing demo must already be loaded (case `subject_ref` resolves a real `pricing.rate` UUID).
2. `demo_approval_rules.xml` — defines rule + flow.
3. `demo_approval_case_sample.xml` — creates the cases with `step_state` already advanced.

---

*Back to [demo-usecase index](../../README.md).*
