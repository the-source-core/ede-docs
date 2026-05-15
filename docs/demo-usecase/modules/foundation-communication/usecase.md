# `foundation.communication` — Demo Use-Case

**Module:** `ede.foundation.communication`
**App key:** `foundation.communication`
**Demo manifest entries** (target): `demo/demo_chatter_messages.xml`, `demo/demo_followers.xml`
**Status:** 🔴 Not yet authored

---

## Use-case

Light up the chatter UI on demo records the scenario already produced — so a tester opening a lead or rate sees realistic activity, not an empty timeline:

- Each demo lead / opportunity has 2–3 prior chatter notes from `demo_user_sales_rep` (handoff comments, price discussion, customer email summary).
- Each demo rate has one note from `demo_user_ops_manager` explaining the submission rationale.
- Followers are seeded so notifications fan out to the right inbox in the `foundation.notifications` demo.

## Records produced

### `demo/demo_chatter_messages.xml`

Per scenario record, e.g.:

| External ID | Model | Subject ref | Body summary |
|---|---|---|---|
| `communication.demo_msg_lead_globex_001_intro` | `ir.chatter.message` | `ref(sales_crm.demo_lead_globex_001)` | "Initial discovery call — Globex looking at 40 LCL/yr Mumbai→Singapore" |
| `communication.demo_msg_lead_globex_001_quote` | `ir.chatter.message` | (same) | "Quote prepared based on demo rate; awaiting customer confirmation" |
| `communication.demo_msg_rate_inmum_sgsin_submit` | `ir.chatter.message` | `ref(pricing.demo_rate_inmum_sgsin_lcl)` | "Submitted for approval — Q3 rate refresh" |

### `demo/demo_followers.xml`

| External ID | Model | Notes |
|---|---|---|
| `communication.demo_follower_lead_globex_sales` | `ir.chatter.follower` | follower=`ref(base.demo_user_sales_rep)`, subject=demo lead |
| `communication.demo_follower_rate_inmum_ops` | `ir.chatter.follower` | follower=`ref(base.demo_user_ops_manager)`, subject=demo rate |

## Out of scope

- New `ir.chatter.*` schema — chatter models ship in production data.
- Live-typing simulation or scheduled message backfill — demo records are static.
- Cross-tenant message threading.

## Dependencies

- `foundation.base` demo (users)
- `logistics.sales-crm` demo (leads / opportunities to attach messages to)
- `logistics.pricing` demo (rates)

## Verification

```
ede migrate upgrade -t demo --with-demo=all
```

Open any demo lead → chatter panel shows 2–3 messages. Open the demo rate → one ops note. The right followers fire downstream notifications.

## Authoring order

Communication's demo XML must run **after** domain modules have produced the subject records they attach to (per `sorted_app_specs`, foundation runs first — so this module's demo loads before logistics.sales-crm / logistics.pricing demo). To work around this we declare the cross-app chatter messages in the *consuming* module's demo file instead:

- `logistics.sales-crm/demo/demo_pipeline.xml` includes chatter messages on its leads.
- `logistics.pricing/demo/demo_rates.xml` includes one chatter note on its rate.

This module's own `demo/*.xml` then carries only the **followers** + any chatter messages on subjects that ALSO live in foundation.base (e.g. partner-level notes).

---

*Back to [demo-usecase index](../../README.md).*
