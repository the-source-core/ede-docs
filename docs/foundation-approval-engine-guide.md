# Foundation Approval — Consumer Integration Guide

This guide is for engineers integrating the approval engine into a domain
module (e.g. `logistics.pricing`, `logistics.sales-crm`). It covers when to
use it, the integration steps, the rule-context schema, the lifecycle event
contract, the subject-lock helper, and the RBAC + notification namespaces.

**Audience:** domain engineers wiring approvals into a new domain. Not for
end users.

---

## When to use foundation.approval

Use the engine when **all** of these hold:

1. The decision needs to be **routed to one or more reviewers** based on
   business rules (margin band, value threshold, role, geography).
2. Reviewers need a **standard inbox**, comment-based decisions, and an
   immutable audit trail.
3. The originating record needs to be **locked** while the case is pending,
   then **react** when the case resolves.

Do **not** use it when:

- A simple boolean field on the record (`needs_review`) is enough.
- The "approver" is system-only (use a hook + worker instead).
- You need free-form workflow with arbitrary state transitions — the engine
  is intentionally constrained to request → decide → close.

---

## 9-step integration checklist

For every new domain integrating approvals:

1. **Decide on the subject model.** Whatever record is being approved
   (`pricing.rate`, `crm.quote`). It needs a `record_uuid` and a
   user-meaningful name (used in notifications).
2. **Register a `policy.set` for your domain.** Domain string is your
   namespace, e.g. `domain="pricing"`.
3. **Author one or more `flow.template` records** with their `step.def`
   children. Choose `SERIAL` vs `PARALLEL` per step and `ALL` vs `ANY`
   join. Each step picks `approver_type` (`USER` or `ROLE`) plus an
   `approver_ref`.
4. **Author rules** that select a template by evaluating
   `trigger_expr` (an expression in the AST language — see §Rule Context).
5. **Subscribe to lifecycle events** in your domain (`approval.case.approved`,
   `approval.case.rejected`, etc. — see §Event Contract). Transition the
   subject record's status accordingly.
6. **Register a subject lock** so the subject record cannot be edited while
   pending — see §Subject-Lock Helper.
7. **Add an "approval" RBAC namespace** for your domain. The engine ships
   `ir.approval.case.read/create/execute/recall`; domain code declares its
   own `{your_domain}.{approve,recall,...}` if needed.
8. **Author notification templates** (subject + body) for any domain-specific
   events you emit. Engine-level events already have templates (see
   `data/notification_templates.xml`).
9. **Submit a case** by dispatching `ir.approval.case.request` whenever the
   subject crosses the request threshold (e.g. on save, on submit).

---

## Rule Context Schema

The trigger expression of `ir.approval.rule` is evaluated through a safe AST
evaluator. It is fed a context dict with these top-level keys:

| Namespace    | Type            | Contents |
|--------------|-----------------|----------|
| `subject.*`  | DottedDict      | Snapshot of the subject record fields (read via `env.models[resource_model].browse(resource_id).read()`). |
| `requester.*`| DottedDict      | `user_id`, `email`, `roles` (list of role codes), `branch_id`. |
| `org.*`      | DottedDict      | `tenant_id`, `org_id`, `org_name`, `id`. |
| `now.*`      | DottedDict      | `date` (ISO date), `time` (HH:MM:SS), `weekday` (lowercase), `iso` (full ISO timestamp). |
| `input.*`    | DottedDict      | Caller-supplied `input_data` from `request_case`'s payload. |
| `payload`    | DottedDict (alias) | Backward-compat alias for `input`. |

Both **subscript** and **attribute** access work:

```python
subject.amount > 1000               # attribute style
subject['region'] == 'EU'           # subscript style
'sales_executive' in requester.roles
now.weekday in ['saturday', 'sunday']
input.is_priority and org.id == 'main'
```

**Allowed AST nodes:** comparisons, boolean operators, arithmetic, subscript,
attribute access on namespace dicts, list/tuple literals.

**Disallowed:** function calls, imports, subscript with non-constant slices on
unknown objects, attribute access traversal off-context. Any disallowed node
raises `ValueError` and the rule is treated as non-matching.

---

## Lifecycle Event Contract

Every state transition emits exactly one event. Subscribe via
`@api.on_event(...)`. **All payloads share a base schema:**

```python
{
  "case_id": str,            # ir.approval.case.record_uuid
  "domain": str,             # domain string from the case
  "resource_model": str,     # subject model key
  "resource_id": str,        # subject record_uuid
  # … plus event-specific fields below
}
```

| Event | Additional fields |
|---|---|
| `approval.case.requested` | `requester_id`, `requested_at` |
| `approval.case.approved`  | `decided_by`, `decided_at`, `comment` |
| `approval.case.rejected`  | `decided_by`, `decided_at`, `comment` |
| `approval.case.returned`  | `decided_by`, `decided_at`, `comment` |
| `approval.case.cancelled` | `cancelled_by`, `cancelled_at`, `reason` |
| `approval.case.recalled`  | `decided_by`, `decided_at`, `comment` (= recall reason) |

**Example consumer:**

```python
from ede.core import api

@api.on_event("approval.case.approved")
def on_pricing_rate_approved(event, env):
    payload = event.payload
    if payload.get("resource_model") != "pricing.rate":
        return
    rate_proxy = env.models["pricing.rate"]
    rate_proxy.browse(payload["resource_id"]).write({"status": "approved"})
```

---

## Subject-Lock Helper

While a case is `PENDING` or `APPROVED`, you usually want to lock the subject
record. Use `make_subject_lock_hooks` from
`ede.foundation.approval.services.subject_lock`:

```python
# src/domains/logistics/pricing/_approval_locks.py
from ede.foundation.approval.services.subject_lock import make_subject_lock_hooks

# Assign to module-level names so the loader's free-function scan picks them up.
pricing_rate_pre_update_lock, pricing_rate_pre_delete_lock = make_subject_lock_hooks(
    model_key="pricing.rate",
    locked_fields=["customer_id", "carrier_id", "valid_from", "valid_to", "amount"],
    locked_states=("PENDING", "APPROVED"),
    error_message=(
        "This rate is under approval and cannot be edited. "
        "Recall the approval first if you need to change it."
    ),
)
```

The helper installs `pre.{model_key}.update` and `pre.{model_key}.delete`
hooks. Each hook looks up the latest `ir.approval.case` for the subject and
vetoes the operation if its state is in `locked_states`.

For tests, programmatic registration is available:

```python
from ede.foundation.approval.services.subject_lock import register_subject_lock_on
register_subject_lock_on(
    env.registry,
    model_key="pricing.rate",
    locked_fields=["amount"],
)
```

---

## RBAC Naming Convention

Approval-related permissions follow the pattern `{resource}.{action}`:

| Permission code | Resource | Action | Used by |
|---|---|---|---|
| `ir.approval.case.read` | `ir.approval.case` | `read` | List/get cases |
| `ir.approval.case.read.own` | `ir.approval.case` | `read` (own) | Submitter view |
| `ir.approval.case.create` | `ir.approval.case` | `create` | Submit a case |
| `ir.approval.case.execute` | `ir.approval.case` | `execute` | Decide / delegate |
| `ir.approval.case.recall` | `ir.approval.case` | `recall` | Recall an approved case |
| `ir.approval.task.read.assigned` | `ir.approval.task` | `read` (own) | Inbox |
| `ir.approval.task.execute` | `ir.approval.task` | `execute` | Decide own task |

When your domain needs domain-specific approval permissions, follow the same
pattern — e.g. `pricing.rate.recall` if recalls require domain-level RBAC
beyond the engine's check.

---

## Notification Template Namespace

The engine seeds 24 templates (8 events × 3 transports — email, web, in_app)
in `src/ede/foundation/approval/data/notification_templates.xml`. Variables
available in any approval template:

- `{{ case_id }}`, `{{ domain }}`, `{{ subject_name }}`, `{{ subject_url }}`
- `{{ requester_id }}`, `{{ resource_model }}`, `{{ resource_id }}`
- `{{ decision_comment }}` (approved/rejected/returned/recalled)
- `{{ decided_by }}` (decision events)
- `{{ cancelled_by }}`, `{{ reason }}` (cancellation)
- `{{ due_at }}` (task assigned, SLA breach)
- `{{ action_url }}` — engine-supplied deep link

To override a template per-domain, register a higher-priority template with
the same `(event_key, transport, locale_code)` triple in your own
manifest's `data` list — the dispatcher selects the most-recently-loaded.

---

## Common Pitfalls

1. **Forgetting to lock the subject.** Domain users will edit records
   mid-approval and your event subscribers will see stale data. Always pair
   the lifecycle subscriber with `make_subject_lock_hooks`.
2. **Spoofing principal in service calls.** All approval service methods
   read the actor from `env.principal`. Never accept user_id parameters from
   HTTP query strings — let the auth middleware do its job.
3. **Mismatched event names.** Pre–Phase 1 code subscribed to
   `approval.case.decided`. That event is gone — split into `approved`,
   `rejected`, `returned`. Update old subscribers.
4. **Circular delegation.** The engine enforces `max_delegation_depth` on
   the `delegation.policy` — but if you set it too high, two users can
   bounce a task indefinitely. Cap at 2–3 in production.
5. **Rule expressions referencing missing namespaces.** If a rule uses
   `subject.foo` but the subject record doesn't have a `foo` field, the
   AST evaluator returns `False` and falls through to the next rule —
   silent. Test rules with realistic subject data before promoting them.
6. **Loading order.** `foundation.approval` depends on
   `foundation.notifications`. If you change `ACTIVE_MODULES`, ensure
   notifications stays before approval.

---

*Back to* [Approval Roadmap](../roadmap/foundation/approval/README.md)
