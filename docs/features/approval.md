# Approval Workflows

Route any record through a multi-step approval chain. Configure per company, per model, per amount threshold — the engine handles routing, escalations, and approver lookup.

```python
# Mark a model as approval-able
@api.model("blog.post", approval=True)
class BlogPost(DomainModel):
    title = fields.Char(required=True)
    body = fields.Text()
```

Once the model opts in, the web client renders an "Submit for approval" button on the form view; backend hooks block writes by anyone other than the current approver until the chain completes.

---

## What you get

-   **`approval.rule`** — define an approval chain: which model, which trigger, which approvers, optional amount thresholds.
-   **`approval.request`** — a live approval-in-progress for one record.
-   **`approval.step`** — one step in the chain (sequential or parallel).
-   **`ApprovalService`** — methods: `submit()`, `approve()`, `reject()`, `cancel()`, `delegate()`.
-   **HTTP endpoints** — `POST /api/approvals/{id}/{action}` for the four lifecycle verbs.
-   **Web-client widgets** — approval-bar in form view, "My Pending Approvals" inbox, audit-trail panel.

## How to use it

### Mark a model for approval

```python
@api.model("expense.report", approval=True)
class ExpenseReport(DomainModel):
    amount = fields.Decimal(precision=12, scale=2, required=True)
    submitter_id = fields.Reference("res.user", required=True)
```

Setting `approval=True` makes the engine listen for the model's `ede.create` and `ede.update` events and consult the configured approval rules.

### Define an approval rule

Approval rules are data, not code. Author them as XML in your app's `data/`:

```xml
<record id="rule_expense_over_5000" model="approval.rule">
    <field name="name">Expenses over $5,000</field>
    <field name="model_key">expense.report</field>
    <field name="trigger">on_submit</field>
    <field name="condition">amount &gt; 5000</field>
    <field name="step_ids" eval="[
        ('approval.step.line_manager',),
        ('approval.step.finance_head',),
    ]"/>
</record>
```

### Submit programmatically

```python
from ede.foundation.approval.services.approval_service import ApprovalService

req = ApprovalService.from_env(env).submit(
    record=expense,
    rule=env.models["approval.rule"].browse("rule_expense_over_5000"),
)
```

### Act on a pending approval

```python
ApprovalService.from_env(env).approve(req, note="Looks good")
# or
ApprovalService.from_env(env).reject(req, reason="Missing receipts")
```

## How it composes with other features

-   **[Communication (Chatter)](communication.md)** — every approval action posts to the record's chatter timeline.
-   **[Notifications](notifications.md)** — pending approvers get inbox + email pings.
-   **[Record Rules](record-rules.md)** — only the current approver and the original submitter can read records mid-approval.

## Reference

| Concept | Where it lives |
|---|---|
| Approval models | `src/ede/foundation/approval/models/` |
| `ApprovalService` | `src/ede/foundation/approval/services/approval_service.py` |
| HTTP API | `src/ede/foundation/approval/api/` |
