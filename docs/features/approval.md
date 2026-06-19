# Approval Workflows

Route any record through a multi-step approval chain — configured entirely as data. Policy sets bind to a domain (and optionally a model); rules decide which flow template fires; the engine spawns tasks, escalates on SLA breach, and keeps an immutable decision ledger.

```python
from ede.foundation.approval.services.approval_service import ApprovalService

# Submit a record for approval — the engine matches a rule and spawns the flow.
result = ApprovalService(env).request_case(
    case_id=None,
    payload={
        "subject": "Quote #QT-2026-001",
        "domain": "blog.post",
        "resource_model": "blog.post",
        "resource_id": post.id,
        "input_data": {"view_count": 15000},
    },
)
case_id = result["case_id"]
```

Approval-ability is **not** declared on the model. A record is routed into approval purely by the policy sets and rules that match its `domain` at request time.

---

## What you get

-   **`ir.approval.policy.set`** — groups rules for one `domain`, scoped GLOBAL / ORG / BRANCH, optionally bound to a model via `model_id`.
-   **`ir.approval.rule`** — a `trigger_expr` (safe-AST expression over the request context) that, when it matches, fires a `template_id` flow — or marks the case auto-approved.
-   **`ir.approval.flow.template`** + **`ir.approval.flow.step.def`** — the reusable flow and its ordered steps (serial / parallel, ALL / ANY join, USER / ROLE / TEAM_ROLE approver routing, per-step SLA + escalation policy).
-   **`ir.approval.case`** + **`ir.approval.task`** — a live case and the tasks assigned within each step.
-   **`ir.approval.decision`** + **`ir.approval.event.log`** — append-only decision and event ledgers for audit.
-   **`ir.approval.escalation.policy`** + **`ir.approval.delegation.policy`** — SLA-breach behaviour and delegation rules.
-   **`ApprovalService`** — `request_case`, `decide_case`, `delegate_case`, `escalate_case`, `cancel_case`, `recall_case`.
-   **HTTP API** — `/api/approval/cases`, `/cases/{id}/decide`, `/cases/{id}/delegate`, `/cases/{id}/cancel`, `/cases/{id}/recall`, `/inbox`, plus admin `/policy-sets` and `/flow-templates`.
-   **Subject lock** — `register_subject_lock_on(...)` installs hooks that block edits to the underlying record while its case is pending.

## How to use it

### Define a policy set, rule, and flow

Approval configuration is data — author it as XML in your app's `data/`. A policy set scopes a `domain`; a rule decides when to fire; the flow template holds the steps.

```xml
<record id="policy_post_publish" model="ir.approval.policy.set">
    <field name="name">Post Publishing Approvals</field>
    <field name="domain">blog.post</field>
    <field name="scope_type">GLOBAL</field>
    <field name="is_active">true</field>
    <field name="priority">10</field>
</record>

<record id="flow_editorial_review" model="ir.approval.flow.template">
    <field name="name">Editorial Review</field>
    <field name="is_active">true</field>
</record>

<record id="flow_step_editor" model="ir.approval.flow.step.def">
    <field name="template_id" ref="flow_editorial_review"/>
    <field name="name">Editor Review</field>
    <field name="sequence">10</field>
    <field name="step_type">SERIAL</field>
    <field name="join_rule">ALL</field>
    <field name="approver_type">ROLE</field>
    <field name="approver_ref">editor</field>
    <field name="sla_hours">24</field>
</record>

<record id="rule_high_traffic" model="ir.approval.rule">
    <field name="policy_set_id" ref="policy_post_publish"/>
    <field name="name">High-traffic posts need editorial review</field>
    <field name="trigger_expr">subject.view_count > 10000</field>
    <field name="template_id" ref="flow_editorial_review"/>
    <field name="auto_approve">false</field>
    <field name="is_active">true</field>
</record>
```

A rule with an empty `trigger_expr` always matches (catch-all). A rule with `auto_approve` true sends the case straight to APPROVED with no tasks — the `approval.case.approved` event still fires, so downstream listeners treat it like a human approval.

### Submit a record for approval

The actor is always taken from `env.principal` — never passed as an argument. Set the principal, then call `request_case`:

```python
service = ApprovalService(env.with_principal({"user_id": requester_id}))
result = service.request_case(
    case_id=None,
    payload={
        "subject": "Post: Launch announcement",
        "domain": "blog.post",
        "resource_model": "blog.post",
        "resource_id": post.id,
        "input_data": {"view_count": 15000},
    },
)
case_id = result["case_id"]
```

`input_data` and the resolved subject record are what `trigger_expr` evaluates against (`subject.*`, `requester.*`, `input.*`).

### Act on a pending task

```python
service = ApprovalService(env.with_principal({"user_id": editor_id}))

service.decide_case(case_id, decision="APPROVE", comment="Looks good.")
# or
service.decide_case(case_id, decision="REJECT", comment="Needs sources.")
# or
service.decide_case(case_id, decision="RETURN", comment="Send back for rework.")
```

Other lifecycle moves: `delegate_case(case_id, delegate_to_user_id, comment)`, `cancel_case(case_id, reason)` (DRAFT / PENDING only), and `recall_case(case_id, reason)` (an APPROVED case — requires the `approval.recall` permission).

### Lock the record while it's under review

```python
from ede.foundation.approval.services.subject_lock import register_subject_lock_on

register_subject_lock_on("blog.post")
```

This installs pre-hooks that veto edits and deletes on a `blog.post` while a related case is PENDING or APPROVED.

## Configuration

These `ir.config` keys set defaults for TEAM_ROLE steps when the step def leaves them unset:

| Key | Default | What it controls |
|---|---|---|
| `approval.team_role.default_resolution` | `PRIMARY_ONLY` | How a team role resolves to approvers (`PRIMARY_ONLY` / `ALL_PARALLEL`). |
| `approval.team_role.default_escalation` | `NONE` | Escalation strategy (`NONE` / `NEXT_IN_SEQUENCE` / `WALK_UP_HIERARCHY`). |
| `approval.team_role.default_max_escalations` | `3` | Escalation depth cap. |

## How it composes with other features

-   **[Notifications](notifications.md)** — task assignment, delegation, and SLA-breach events dispatch through `notification.send`.
-   **[Workflow Engine](workflow.md)** — a workflow transition with a `request_approval` action submits a case and resumes on the `approval.case.approved` event.
-   **[Record Rules](record-rules.md)** — scope who can read cases mid-approval.

## Reference

| Concept | Where it lives |
|---|---|
| Approval models | `src/ede/foundation/approval/models/` |
| `ApprovalService` | `src/ede/foundation/approval/services/approval_service.py` |
| Subject lock | `src/ede/foundation/approval/services/subject_lock.py` |
| HTTP API | `src/ede/foundation/approval/api/approval_routes.py` |
