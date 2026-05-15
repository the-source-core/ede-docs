# Workflow Engine

Declarative state machines for any record. Define states + allowed transitions in XML; the engine enforces them, fires events on transitions, and surfaces a status-bar widget in the form view.

```xml
<workflow id="wf_purchase_order" model="purchase.order">
    <state name="draft"     label="Draft"     initial="true"/>
    <state name="submitted" label="Submitted"/>
    <state name="approved"  label="Approved"/>
    <state name="received"  label="Received"  terminal="true"/>
    <state name="cancelled" label="Cancelled" terminal="true"/>

    <transition from="draft"     to="submitted" command="submit"/>
    <transition from="submitted" to="approved"  command="approve"
                guard="$principal.has_role('finance_manager')"/>
    <transition from="approved"  to="received"  command="receive"/>
    <transition from="draft"     to="cancelled" command="cancel"/>
</workflow>
```

The PO model picks up the workflow automatically — its `state` field becomes the workflow state, and writes to it must come via the registered commands.

---

## What you get

-   **`workflow`** XML DSL element — declare states + transitions as data.
-   **State guards** — domain-filter or `$principal.*` expressions that block transitions.
-   **`on_enter` / `on_exit` hooks** — fire commands or events on state changes.
-   **Statusbar widget** — auto-rendered in the form view; only allowed transitions show as buttons.
-   **`workflow.transition.fired`** event — emitted after every successful transition; subscribe with `@api.on_event`.
-   **Audit** — every transition writes to `workflow.history` on the record.

## How to use it

### Listen to transitions

```python
@api.on_event("workflow.transition.fired")
def on_transition(event, env):
    if event.payload["model"] == "purchase.order" \
       and event.payload["to_state"] == "approved":
        env.dispatch(Command("purchase.order.create_grn", payload={
            "po_uuid": event.payload["record_uuid"],
        }))
```

### Guard a transition with a domain filter

```xml
<transition from="draft" to="submitted" command="submit"
            guard="[('total', '>', 0)]"/>
```

The engine evaluates the guard against the record; if false, the transition is rejected with `WorkflowGuardError`.

### Run a side-effect on entering a state

```xml
<state name="approved" label="Approved">
    <on_enter command="purchase.order.lock_pricing"/>
</state>
```

`on_enter` and `on_exit` accept a command name; the engine dispatches it as the record transitions through.

### Trigger from a button in the form view

```xml
<form model="purchase.order">
    <header>
        <button name="submit"  string="Submit"
                attrs="{'invisible': [('state','!=','draft')]}"/>
        <button name="approve" string="Approve"
                attrs="{'invisible': [('state','!=','submitted')]}"/>
        <field name="state" widget="statusbar"/>
    </header>
    …
</form>
```

The button's `name` matches a workflow `transition.command`. The engine runs the transition; the form re-renders.

## Reference

-   Source: `src/ede/foundation/workflow/`
-   DSL grammar: see the workflow DSL repo doc (will be ported to this manual).
-   Related: [Approval Workflows](approval.md) (orthogonal — workflows are state machines, approvals are reviewer chains).
