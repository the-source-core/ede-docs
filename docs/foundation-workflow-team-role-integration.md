# Workflow — Team-Role Integration

> Guard helpers and transition actions that let a workflow reason about **team-functional
> roles** (MANAGER, REVIEWER, ACCOUNT_MANAGER, …) instead of hard-coding user lookups in
> command handlers. Shipped by `foundation.workflow` Enhancement 01. Pairs with the
> `<workflow>` DSL guide ([foundation-workflow-dsl.md](./foundation-workflow-dsl.md)) and the
> engine overview ([foundation-workflow.md](./foundation-workflow.md)).

Team-functional roles come from the team substrate (`foundation.base` Enhancement 06): a
team (`res.team`) has role-holders (`res.team.role.assignment` rows linking a team, an
`ir.team.role`, and a `res.user`, ordered by `sequence`). Both the workflow engine and the
approval engine resolve those holders through one shared `TeamRoleService`, so the two
engines never drift on resolution semantics.

This enhancement adds two capabilities to the workflow engine:

1. **Three guard helpers** so a transition's guard can ask team-role questions.
2. **Two transition actions** — `notify_team_role` and `assign_team_role`.

---

## 1. Guard helpers

The safe-AST guard evaluator now permits an explicit, whitelisted set of function calls.
Three helpers are exposed to every workflow guard expression:

| Helper | Returns | Meaning |
|---|---|---|
| `team_has_role(team, role_code, user)` | bool | Does `user` hold `role_code` on `team`? |
| `team_role_user(team, role_code)` | user uuid or `None` | The team's **primary** (lowest-sequence) holder of `role_code`. |
| `team_role_users(team, role_code, walk_up=False)` | list of user uuids | **All** holders of `role_code`, ordered by sequence. |

Operands are the values guard authors already have in scope:

- a **team** is normally `subject.<team_field>` — e.g. `subject.team_id` (a record uuid string).
- a **user** is normally `user_id` — a convenience key in the guard context holding the
  current principal's user uuid as a bare string. (`principal` is still available as the
  full dict.)

Because the helpers return uuid strings, comparisons read naturally:

```xml
<!-- Only the team's primary MANAGER may activate -->
<transition code="activate" from="draft" to="active" on-command="x.activate">
    <guard expr="team_role_user(subject.team_id, 'MANAGER') == user_id"/>
</transition>

<!-- Any REVIEWER on the team (or an ancestor team) may submit for review -->
<transition code="submit" from="draft" to="review" on-command="x.submit">
    <guard expr="user_id in team_role_users(subject.team_id, 'REVIEWER', walk_up=True)"/>
</transition>

<!-- Predicate form -->
<transition code="approve" from="review" to="approved" on-command="x.approve">
    <guard expr="team_has_role(subject.team_id, 'PRICING_APPROVER', user_id)"/>
</transition>
```

**Zero-assignee semantics are intentional.** If a team has no holder of the role,
`team_role_user` returns `None` and `team_role_users` returns `[]`. So
`team_role_user(...) == user_id` is `False` and the transition is blocked — if there's no
MANAGER, no one is the MANAGER. No escalation here; escalation is an approval-engine concept.

> The evaluator only permits these three names, and only as direct `name(...)` calls — never
> `obj.method(...)`. Existing literal-only guards (no calls) are completely unaffected.

---

## 2. Transition actions

Actions run **after** the stage change is applied (same as `dispatch_command` / `emit_event`).
The engine reads the record's team from a `team_field` config key (default `team_id`).

### `notify_team_role`

Resolve the team's role-holder(s) and send them a notification via `foundation.notifications`.

```xml
<transition code="submit" from="draft" to="under_review" on-command="x.submit">
    <action type="notify_team_role"
            team-role="REVIEWER"
            mode="all"
            walk-up="false"
            event-key="quote.submitted_for_review"/>
</transition>
```

| Attribute | Required | Default | Meaning |
|---|---|---|---|
| `team-role` | ✅ | — | `ir.team.role.code` to resolve. |
| `event-key` | ✅ | — | Notification template event key. |
| `mode` | | `all` | `primary` (one user) or `all` (every holder). |
| `walk-up` | | `false` | Walk up `team.parent_id` when the direct lookup is empty. |
| `team-field` | | `team_id` | FK field on the bound model that points at `res.team`. |

The engine resolves the holders once and dispatches a single `notification.send` with
`recipient_spec={"user_ids": [...]}`; the notifications engine handles per-user transport
fan-out (email / web / in-app).

### `assign_team_role`

Stamp the team's **primary** role-holder into a record field — e.g. set the new opportunity's
owner to the team's ACCOUNT_MANAGER on qualification.

```xml
<transition code="qualify" from="lead" to="opportunity" on-command="x.qualify">
    <action type="assign_team_role"
            team-role="ACCOUNT_MANAGER"
            target-field="owner_user_id"
            mode="primary"
            overwrite="false"/>
</transition>
```

| Attribute | Required | Default | Meaning |
|---|---|---|---|
| `team-role` | ✅ | — | `ir.team.role.code` to resolve. |
| `target-field` | ✅ | — | Reference field on the record to stamp with the user uuid. |
| `mode` | | `primary` | Must be `primary` — `all` is rejected at boot (a field holds one user). |
| `overwrite` | | `false` | When `false`, an already-set `target-field` is left untouched. |
| `team-field` | | `team_id` | FK field on the bound model that points at `res.team`. |

The write is routed through the command bus (`ede.update`), so the target model's pre/post
hooks fire normally.

---

## 3. Zero-assignee handling

When a `notify_team_role` / `assign_team_role` action resolves to **no users**, behaviour is
governed by the `ir.config` key `workflow.team_role.lookup_failure_severity`
(Settings → Workflows → Team-Role Integration):

| Value | Effect |
|---|---|
| `warn` (default) | Log a `team_role_lookup_failed` audit row; the transition **completes**. |
| `error` | Log the row and raise `TeamRoleLookupFailed`, surfacing the failure to the caller. |
| `silent` | Continue with no log. |

> **`error` does not roll the stage back.** Actions run after the stage change is applied, so
> `error` surfaces the failure loudly (fails the dispatch) but the stage has already moved. For
> a true pre-transition veto, use a **guard** (e.g. `team_role_user(subject.team_id, 'X') != None`)
> rather than relying on the action.

---

## 4. Audit trail

Every team-role action appends to the append-only `ir.workflow.event.log`:

| `event_type` | When | Payload highlights |
|---|---|---|
| `transition_action_executed` | Action ran (incl. an `assign` skipped by `overwrite=false`) | `action_type`, `team_role`, `team_id`, resolved/assigned user ids |
| `team_role_lookup_failed` | Zero assignees, severity `warn`/`error` | `action_type`, `team_role`, `team_id`, `walk_up`, `reason` |

---

## 5. Workflow vs approval team-roles

Both engines consume the same `TeamRoleService`, but solve different problems:

- **Workflow** team-roles are **declarative routing on transitions** — guards gate a move; the
  two actions notify or stamp synchronously as the transition fires.
- **Approval** team-roles drive **case-driven routing + escalation** — a TEAM_ROLE assignment on
  an approval step, with sequence-aware escalation strategies and SLA.

Share the lookup service; pick the engine that matches the lifecycle you're modelling.

---

## 6. Consumer adoption checklist

1. Ensure the bound model has a `res.team` reference (default field name `team_id`, else set
   `team-field` on the action).
2. Seed your `ir.team.role` codes and `res.team.role.assignment` rows (consumers own these — the
   foundation ships none).
3. Author guards / actions in the model's `<workflow>` XML using the attributes above.
4. Decide the zero-assignee severity for your tenant in Settings → Workflows.
5. For `assign_team_role`, point `target-field` at a `res.user` Reference field.

---

*Engine reference: [foundation-workflow.md](./foundation-workflow.md) · DSL grammar:
[foundation-workflow-dsl.md](./foundation-workflow-dsl.md) · Roadmap:
[enhancements/01-team-role-integration.md](../roadmap/foundation/workflow/enhancements/01-team-role-integration.md).*
