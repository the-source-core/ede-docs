# Workflow Engine

Declarative state machines for any record. Mark a field `workflow=True`, declare the stages and transitions in XML, and the engine owns that field — every change goes through a guarded, audited transition and surfaces as a status bar in the form view.

```xml
<workflow code="blog.post.lifecycle" model="blog.post" field="status"
          name="Post Lifecycle">
    <stage code="DRAFT"     label="Draft"     initial="true"  sequence="10"/>
    <stage code="REVIEW"    label="In Review" sequence="20"/>
    <stage code="PUBLISHED" label="Published" terminal="true" sequence="30"/>

    <transition code="submit"  from="DRAFT"  to="REVIEW"     on-command="blog.post.submit"/>
    <transition code="publish" from="REVIEW" to="PUBLISHED"  on-command="blog.post.publish"
                guard-expr="subject.body != ''"/>
</workflow>
```

The `status` field is now workflow-controlled. Direct ORM writes to it are rejected — the only way to change it is `env.workflow.transition(record, "submit")`.

---

## What you get

-   **`<workflow>` XML DSL** — declare a workflow with `code`, `model`, `field`, `name`; nest `<stage>`, `<transition>`, and an optional `<guards>` block.
-   **`workflow=True` field binding** — mark any field workflow-controlled; the engine forbids direct writes and validates at boot that a matching `ir.workflow.definition` exists.
-   **Guards** — reusable `<guard>` expressions (referenced by `guard="<code>"`) or inline `guard-expr="<expr>"`; both use the formula DSL with `subject.*` and `requester.*` paths.
-   **Transition actions** — `request_approval`, `dispatch_command`, `emit_event`, `notify_team_role`, `assign_team_role`.
-   **Lifecycle events** — `workflow.stage.entered`, `workflow.stage.exited`, `workflow.transition.pending`, `workflow.transition.cancelled`; subscribe with `@api.on_event`.
-   **Audit ledger** — every transition writes an append-only row to `ir.workflow.event.log`.
-   **Status bar** — `<statusbar field="..."/>` in a form view renders the current stage plus guard-evaluated transition buttons.
-   **HTTP API** — `/api/workflow/transition`, `/available`, `/cancel-pending`, `/instance`, `/graph`.

## How to use it

### Bind a field to a workflow

Declare the field `workflow=True`, then ship the `<workflow>` definition in your app's data. The engine refuses to boot a model whose `workflow=True` field has no definition.

```python
@api.model("blog.post")
class Post(DomainModel):
    title = fields.Char(required=True)
    body = fields.Text()
    status = fields.Enum(
        selection=[("DRAFT", "Draft"), ("REVIEW", "In Review"), ("PUBLISHED", "Published")],
        default="DRAFT",
        workflow=True,
    )
```

### Run a transition

```python
post = env.models["blog.post"].browse(post_uuid)
result = env.workflow.transition(post, "submit")
# result.code, result.from_stage, result.to_stage, result.pending
```

If the transition's guard fails, the engine raises and the field is left unchanged.

### Guard a transition

Inline, for a one-off check:

```xml
<transition code="publish" from="REVIEW" to="PUBLISHED" on-command="blog.post.publish"
            guard-expr="subject.body != '' and subject.title != ''"/>
```

Or declare a reusable guard once and reference it by code:

```xml
<workflow code="blog.post.lifecycle" model="blog.post" field="status" name="Post Lifecycle">
    <guards>
        <guard code="has_content" label="Body is non-empty"
               expression="subject.body != ''"/>
    </guards>
    <transition code="publish" from="REVIEW" to="PUBLISHED"
                on-command="blog.post.publish" guard="has_content"/>
</workflow>
```

### Run an action on a transition

Each transition may carry one `<action>`. To request approval (and resume when it clears) or dispatch a follow-on command:

```xml
<transition code="publish" from="REVIEW" to="PUBLISHED" on-command="blog.post.publish">
    <action type="request_approval" policy-set="blog.post.publish"
            success-event="approval.case.approved"
            failure-event="approval.case.rejected"/>
</transition>
```

When `success-event` is set the stage advances on approval; a rejection aborts the pending transition. Other action types: `dispatch_command` (fire a command), `emit_event` (publish a bus event), and the team-role actions `notify_team_role` / `assign_team_role`.

### React to transitions

```python
@api.on_event("workflow.stage.entered")
def on_stage_entered(event, env):
    payload = event.payload or {}
    if payload.get("record_model") == "blog.post" and payload.get("stage") == "PUBLISHED":
        env.dispatch(Command("blog.post.announce", payload={
            "post_uuid": payload["record_uuid"],
        }))
```

The stage events carry `record_model`, `record_uuid`, and `stage`. There is no single "transition fired" event — listen to `workflow.stage.entered` / `workflow.stage.exited` for the stage change, or `workflow.transition.pending` / `workflow.transition.cancelled` for async transitions awaiting approval.

### Render the status bar

```xml
<form model="blog.post">
    <header>
        <statusbar field="status"/>
    </header>
    <sheet>
        <field name="title"/>
        <field name="body"/>
    </sheet>
</form>
```

The status bar shows the current stage and renders a button for each transition whose guard passes for the current user. Clicking a button calls `POST /api/workflow/transition`.

## Reference

| Concept | Where it lives |
|---|---|
| Workflow models (`ir.workflow.definition` / `.stage` / `.transition` / `.instance`) | `src/ede/foundation/workflow/models/` |
| Audit ledger `ir.workflow.event.log` | `src/ede/foundation/workflow/models/` |
| HTTP API | `src/ede/foundation/workflow/` (prefix `/api/workflow`) |
| DSL grammar | `src/ede/core/services/presentation/dsl/` |

-   Related: [Approval Workflows](approval.md) — workflows are state machines; approvals are reviewer chains. A transition action bridges the two.
