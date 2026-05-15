# `foundation.notifications` — Demo Use-Case

**Module:** `ede.foundation.notifications`
**App key:** `foundation.notifications`
**Demo manifest entries** (target): `demo/demo_inbox.xml`
**Status:** 🔴 Not yet authored

---

## Use-case

Notification templates ship as production seeds. Demo's job is to populate the **inbox** state so a tester opening the bell icon sees a meaningful mix of read / unread items — not an empty list.

Driven by the unifying scenario:

- 1 unread "Approval requested" from the demo pricing-rate submission (subject = `pricing.demo_rate_inmum_sgsin_lcl`, recipient = `demo_user_ops_manager`).
- 1 unread "New lead assigned" from the demo CRM pipeline (subject = `sales_crm.demo_lead_globex_001`, recipient = `demo_user_sales_rep`).
- 1 read "Welcome to Acme Forwarding" sent to every demo user 14 days back so the read-history demo isn't empty.

## Records produced

### `demo/demo_inbox.xml`

| External ID | Model | Recipient | Subject | Read |
|---|---|---|---|---|
| `notifications.demo_inbox_rate_pending_ops` | `ir.notification.outbox` | `demo_user_ops_manager` | `pricing.demo_rate_inmum_sgsin_lcl` | false |
| `notifications.demo_inbox_lead_assigned_sales` | `ir.notification.outbox` | `demo_user_sales_rep` | `sales_crm.demo_lead_globex_001` | false |
| `notifications.demo_inbox_welcome_admin` | `ir.notification.outbox` | `demo_user_admin` | _none_ | true |
| `notifications.demo_inbox_welcome_sales` | `ir.notification.outbox` | `demo_user_sales_rep` | _none_ | true |
| `notifications.demo_inbox_welcome_ops` | `ir.notification.outbox` | `demo_user_ops_manager` | _none_ | true |
| `notifications.demo_inbox_welcome_finance` | `ir.notification.outbox` | `demo_user_finance` | _none_ | true |

`sent_at` for unread = `now()`; for read = `now() - 14d` so the inbox sort order is realistic.

## Out of scope

- Notification template definitions — production seeds. Demo uses existing `welcome_user`, `approval.pending`, `crm.lead.assigned` templates.
- Outbound mail delivery — demo runs against an in-process queue; nothing actually leaves the box.
- Push / SMS adapters — Phase 2 of notifications.

## Dependencies

- `foundation.base` demo (recipients)
- `foundation.approval` demo (subjects for the pending case)
- `logistics.pricing` demo, `logistics.sales-crm` demo (subjects for the rate and lead)

Because notifications.demo runs in foundation order (before logistics), the records above use forward refs that resolve when the logistics demo pass runs later in the same session — except `sales_crm.demo_lead_globex_001` and `pricing.demo_rate_inmum_sgsin_lcl` which DON'T exist when this file runs. To work around this, the demo inbox file lives in **the consuming domain modules** (`logistics.sales-crm/demo/demo_pipeline.xml` declares its own `ir.notification.outbox` row). This module's own `demo_inbox.xml` carries ONLY the welcome notifications (no cross-app refs).

## Verification

```
ede migrate upgrade -t demo --with-demo=all
```

Log in as any demo user → bell icon shows the correct unread count and welcome message in read history.

## Authoring order

1. Welcome notifications (6 rows) loaded here.
2. Approval-pending + lead-assigned notifications loaded by the logistics demo files that own the subject records.

---

*Back to [demo-usecase index](../../README.md).*
