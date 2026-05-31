# Team-Role Routing — `foundation.approval` Enhancement 01

**Module:** `foundation.approval`
**Roadmap:** [enhancements/01-team-role-routing.md](../roadmap/foundation/approval/enhancements/01-team-role-routing.md)
**Depends on:** `foundation.base` Enhancement 06 (Team Substrate — `res.team`, `ir.team.role`, `res.team.role.assignment`, `TeamRoleService`)

> This is a developer guide for the `TEAM_ROLE` assignment kind. It explains when to use it versus `ROLE` / `USER`, how the subject-model binding works, the resolution and escalation semantics, the zero-assignee → immediate-escalation contract, sizing `max_escalations`, migrating an existing `ROLE` flow, and troubleshooting stalls.

---

## When to use `TEAM_ROLE` vs `ROLE` vs `USER`

Every approval flow step declares an `approver_type`. All three kinds coexist permanently — pick the one that fits the step.

| Kind | `approver_ref` holds | Resolves to | Use when |
|---|---|---|---|
| `USER` | a `res.user` UUID | exactly that person | a named individual always signs off (rare; brittle) |
| `ROLE` | an `ir.rbac.role` code | *any* user holding that RBAC role (branch-scoped, then global) | anyone with the permission may act |
| `TEAM_ROLE` | an `ir.team.role` code | the holder(s) of that **team-functional role on the record's own team** | the correct approver depends on *which team owns the record* |

The motivating case: a pricing approval submitted by Alice (Mumbai West Sales) should go to **Mumbai West's** pricing approver — not to any of the eight `PRICING_APPROVER`s across the country. `ROLE` routes to "anyone with the role"; `TEAM_ROLE` routes to "the right person for *this* record's team".

---

## Binding a flow template to a subject model

A `TEAM_ROLE` step needs to know how to find the team from the record under approval. That binding lives on the **flow template** (`ir.approval.flow.template`), set once and shared by all its steps:

| Field | Meaning | Example |
|---|---|---|
| `subject_model_key` | the model key the flow applies to | `crm.quote` |
| `subject_team_field` | the Reference-to-`res.team` field on that model carrying the team | `sales_team_id` |

These are plain strings (model key / field name), not Reference columns — the framework has no `ir.model` / `ir.model.field` registry to foreign-key into. They are **nullable**: an all-`USER` / all-`ROLE` template leaves both blank and behaves exactly as before.

**Save-time validation.** When any step on a template is `TEAM_ROLE`, the engine vetoes the step's save unless:

1. the template sets **both** `subject_model_key` and `subject_team_field`;
2. `subject_model_key` names a registered model;
3. `subject_team_field` is a `Reference` field on that model whose target is `res.team`;
4. the step's `approver_ref` matches an existing `ir.team.role.code`.

It also blocks *unbinding* a template (clearing either field) while it still carries `TEAM_ROLE` steps. `USER` / `ROLE` steps skip all of this — full backward compatibility.

---

## Resolution semantics (arm time)

When a `TEAM_ROLE` step is armed, the engine reads `subject_record.<subject_team_field>` to get the team, then calls the shared `TeamRoleService.resolve(team, role_code, mode, walk_up)`.

`resolution_strategy` (per-step, nullable) controls who gets the task:

| Strategy | Behaviour |
|---|---|
| `PRIMARY_ONLY` *(default)* | only the lowest-`sequence` holder on the team (the primary) gets the task |
| `ALL_PARALLEL` | every holder gets a task simultaneously; first to act wins |

`sequence` on `res.team.role.assignment` orders holders: `sequence=1` is the primary, `2` the backup, and so on.

---

## Escalation semantics (SLA breach)

When a `TEAM_ROLE` task breaches its `sla_hours`, the SLA worker drives escalation according to `escalation_strategy` (per-step, nullable):

| Strategy | Behaviour on breach |
|---|---|
| `NONE` | the task expires; the case sits (legacy behaviour) |
| `NEXT_IN_SEQUENCE` | reassign to the next backup on the **same team** by sequence (`escalation_depth`-th holder) |
| `WALK_UP_HIERARCHY` | climb `team.parent_id`: each breach starts one level higher and resolves there (or above, skipping empty ancestors) |

On each escalation the engine:

- creates the new task(s) with `escalation_depth = previous + 1`;
- **supersedes** the breached task — marks it `EXPIRED` and points `superseded_by` at the replacement (audit-preserving; never a hard delete);
- writes an `escalated` audit row (`from_user_ids`, `to_user_ids`, `depth`, `team_id`, `escalation_strategy`);
- notifies the new assignee(s).

When no assignee remains, or the cap is hit, the task expires and a `stalled` row is written with a `reason` (`no_more_assignees` / `escalation_cap` / `no_team`).

### The zero-assignee → immediate-escalation contract

If, at **case-create** time, the record's team has no holder of the named role, the engine does **not** make the requester wait `sla_hours` for a phantom breach. With `escalation_strategy = WALK_UP_HIERARCHY`, the initial lookup walks up `team.parent_id` immediately (`walk_up=True`) and assigns the first ancestor that has a holder. Only when *no* ancestor has one does the case stall at create with a clear "No approvers could be resolved" error (rolled back by the workflow bridge so the requester lands back at the pre-submit stage). With `NEXT_IN_SEQUENCE` (no walk), a team with zero holders stalls at create — there is nothing to escalate to on the same team.

### Sizing `max_escalations`

`max_escalations` (per-step, nullable, default **3**) caps the number of escalation walks before the case stalls with `reason=escalation_cap`. It prevents an infinite walk-up loop on broken team data (e.g. a `parent_id` cycle or a chain with no role-holder anywhere). Set it to the realistic depth of your team hierarchy plus one — a three-tier org (Local → Region → Country) rarely needs more than 3.

---

## Tenant defaults (`ir.config`)

The three per-step columns are nullable. When a step leaves one unset, the engine falls back to the tenant default, then to the hardcoded default:

| `ir.config` key | Scope | Default | Applies to |
|---|---|---|---|
| `approval.team_role.default_resolution` | system | `primary_only` | `resolution_strategy` |
| `approval.team_role.default_escalation` | system | `next_in_sequence` | `escalation_strategy` |
| `approval.team_role.default_max_escalations` | system | `3` | `max_escalations` |

Edit them under **Settings → Approvals → Team-Role Routing Defaults** (`views/approval_team_role_settings.xml`). Effective value = explicit step value → `ir.config` value → hardcoded default (`PRIMARY_ONLY` / `NONE` / `3`). Step Enum values are uppercase; `ir.config` values are lowercase — the engine normalizes.

---

## Migrating an existing `ROLE`-based flow to `TEAM_ROLE`

Incremental and reversible — both kinds coexist, so migrate one step at a time:

1. On the flow template, set `subject_model_key` and `subject_team_field` to point at the model + its team field.
2. Ensure the team-functional role exists as an `ir.team.role` (code matching what you'll put in `approver_ref`) and that the relevant teams have `res.team.role.assignment` rows.
3. Change the step's `approver_type` from `ROLE` to `TEAM_ROLE` and set `approver_ref` to the team-role code.
4. Optionally set `resolution_strategy` / `escalation_strategy` / `max_escalations`, or leave them blank to inherit the tenant defaults.

There is **no forced migration**: the existing `pricing.rate` `ROLE` flow keeps working untouched until its owners choose to adopt team routing.

---

## Audit trail

`ir.approval.event.log` gains three event types (within the existing `payload` JSON — no schema change):

| `event_type` | When | Key payload fields |
|---|---|---|
| `step_assigned` | a `TEAM_ROLE` step is armed | `step_def_id`, `team_id`, `team_role`, `resolution_mode`, `assigned_user_ids` |
| `escalated` | on each SLA-breach reassignment | `from_user_ids`, `to_user_ids`, `depth`, `team_id`, `escalation_strategy` |
| `stalled` | cap exhausted / no assignees | `reason` (`escalation_cap` \| `no_more_assignees` \| `no_team`), `depth` |

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Case-create fails with "No approvers could be resolved" | the record's team (and, for walk-up, its ancestors) has no holder of the role; or the subject record's team field is empty |
| Save rejected: "TEAM_ROLE steps require the flow template to set subject_model_key AND subject_team_field" | bind the template before adding a `TEAM_ROLE` step |
| Save rejected: "must be a Reference to res.team" | `subject_team_field` points at a non-Reference field, or a Reference to the wrong model |
| Save rejected: "team-role code … does not match any ir.team.role.code" | seed the `ir.team.role` row first, or fix the `approver_ref` typo |
| Case stalls with `reason=escalation_cap` | the role-holder chain is deeper than `max_escalations`, or the hierarchy has no holder anywhere — raise the cap or assign a holder higher up |
| Escalation reassigns to the same person each tick (walk-up) | the only role-holder is on an ancestor with no further parents; expected once the top is reached, then it stalls at the cap |

---

*Engine: [`services/team_role_routing.py`](../src/ede/foundation/approval/services/team_role_routing.py) · wired into [`services/approval_service.py`](../src/ede/foundation/approval/services/approval_service.py) (`_create_step_tasks`, `_escalate_team_role`) · validators on [`models/flow_template.py`](../src/ede/foundation/approval/models/flow_template.py) · tests in [`src/tests/foundation/approval/test_team_role_routing.py`](../src/tests/foundation/approval/test_team_role_routing.py).*
