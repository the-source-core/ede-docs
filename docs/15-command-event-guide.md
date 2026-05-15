# EDE Framework — Command & Event Usage Guide

A practical guide for choosing between `@api.on_command`, `@api.on_event`, and
`@api.on_event(track_fields=...)`, and for emitting `web.client.*` push events.

---

## Why These Three Primitives Exist

Most frameworks give you one abstraction for all behavior — a method call, a signal,
a hook. EDE deliberately provides **three distinct primitives** that map cleanly to
three different concerns:

| Primitive | What it answers | When it runs | Who depends on the result |
|-----------|----------------|--------------|--------------------------|
| `@api.on_command` | *Do this, give me back a result* | Synchronous, inline | The caller (HTTP response) |
| `@api.on_event` | *Something happened — react* | Asynchronous, retried | Nobody — fire and forget |
| `@api.on_hook` | *Before/after this mutation, enforce a rule* | Synchronous, inline | The mutation itself |

This separation is what lets the architecture survive growth:

- **Adding a new side effect** (send email when shipment is confirmed) never touches the
  command that confirms the shipment. Add a new `@api.on_event("shipment.confirmed")`
  handler — done.
- **Adding a new validation** (block delete when active invoices exist) never touches
  the delete command. Add a `@api.on_hook("pre.logistics.shipment.delete")` — done.
- **Replacing the HTTP framework** never touches domain commands or event handlers.
  The command bus and event queue are infrastructure contracts, not FastAPI internals.
- **Scaling to Kafka** requires only changing `DEFAULT_MESSAGE_BROKER_PROVIDER = "kafka"`.
  Event handlers remain untouched.

---

## How the Three Primitives Form a Reactive Loop

Every meaningful state change in EDE follows this chain:

```
HTTP Request
  └─► Controller          (@api.route)
        └─► env.dispatch(Command(...))
              └─► @api.on_command handler
                    └─► ORM write (RecordSet.write / create / delete)
                          │
                          ├─► Synchronous hooks fire (pre/post)
                          │     └─► @api.on_hook methods
                          │           (validation, derivation, field stashing)
                          │
                          └─► Async events emitted (self.emit / env.emit)
                                └─► EventWorker delivers to
                                      └─► @api.on_event handlers
                                            (notifications, integrations, browser push)
```

This means:
1. **Validation is synchronous** — hooks block the mutation and surface the error to the caller.
2. **Side effects are asynchronous** — events decouple the mutation from its consequences;
   the HTTP response returns before the notification is sent.
3. **Neither hooks nor events know about HTTP** — both receive a plain `Command` or `Event`;
   no `Request` objects, no FastAPI types.

---

## The Full Reactive Chain: A Concrete Example

Consider `res.organization.code` (the URL slug used by the frontend router).
When an admin changes it:

```
PATCH /api/res/organization/{id}   body: {"code": "acme-new"}
  │
  └─► Controller → env.dispatch(Command("ede.update",
                                         model_key="res.organization", model_id="..."))
        │
        └─► CommandBus:
              ├─► pre.res.organization.update hook (auto)
              │     field tracking: stashes old code="acme" on cmd
              │
              ├─► CrudKernel.handle_update → DB write (code → "acme-new")
              │
              └─► post.res.organization.update hook (auto)
                    field tracking: code changed → emit field_changed
                      │
                      └─► EventWorker → Organization.handle_code_changed
                            (track_fields=["code"] — only fires for code)
                            └─► self.emit("web.client.reload", {new_slug: "acme-new"})
                                  │
                                  └─► EventWorker → handle_web_client_reload
                                        └─► WebPushRegistry.broadcast(tenant_id)
                                              └─► SSE → browser navigates to new URL
```

**Zero coupling between layers.** The DB write doesn't know about the browser. The field
tracking hook doesn't know about SSE. The SSE handler doesn't know about org slugs. Each
step reacts to a well-defined fact.

---

## Quick Decision Guide

```
Does the caller need a return value?
  Yes → @api.on_command

Does it react to something that already happened?
  Yes →
    Is it triggered by a specific field changing on a model?
      Yes → @api.on_event(track_fields=[...]) on a DomainModel method
      No  → @api.on_event on a free function

Does it need to intercept a mutation synchronously (validate, guard, derive)?
  Yes → @api.on_hook  (see 14-lifecycle-hooks.md)
```

---

## 1. `@api.on_command` — Intent with a Result

### When to use

Use a command handler when:

- The operation **mutates state** (write, create, delete, transition).
- The caller **needs a result** (HTTP response body, values, status).
- The operation **requires business validation** before persisting (e.g. resolve a FK, check eligibility).
- The operation **coordinates multiple writes** atomically.

Do **not** use a command to react to something that already happened — that is what events are for.

### Definition

```python
from ede.core import api
from ede.core.kernel.model import DomainModel
from ede.core.types import Command


@api.model("logistics.shipment")
class Shipment(DomainModel):

    @api.on_command("logistics.shipment.confirm")
    def handle_confirm(self, cmd: Command) -> dict:
        """
        Confirm a shipment — sets status to 'confirmed'.

        Payload
        -------
        notes (str, optional): internal notes to store on confirmation.
        """
        notes = (cmd.payload or {}).get("notes", "")
        cmd.record.write({"status": "confirmed", "notes": notes})

        # Emit an event — downstream handlers decide what to do next
        self.emit("logistics.shipment.confirmed", {
            "shipment_id": cmd.model_id,
            "tenant_id": self.env.tenant_id,
        })

        return {"success": True, "record": cmd.record}
```

### Dispatch

From a controller or another handler:

```python
result = env.dispatch(Command(
    name="logistics.shipment.confirm",
    payload={"notes": "verified by warehouse"},
    model_key="logistics.shipment",
    model_id="<record_uuid>",
))
```

### Rules

| Rule | Detail |
|------|--------|
| Signature | `(self, cmd: Command) -> Any` |
| Return value | Serialised to JSON by the HTTP adapter; returned to caller |
| Name convention | `{model_key}.{verb}` — present tense, e.g. `logistics.shipment.confirm` |
| One handler per name | Duplicate registration raises `DuplicateHandler` |
| `payload` is required | Always pass `payload={}` minimum — it has no default |
| `cmd.record` | Populated when `model_key` + `model_id` are set on the Command |
| Basic CRUD | Already covered by `ede.create / ede.update / ede.delete`; do not re-implement |

### Why commands make the system future-ready

A command is the **stable public interface** of a domain capability. The HTTP layer
translates `POST /api/logistics/shipments/{id}/confirm` → `Command("logistics.shipment.confirm")`.
If you later add a CLI, a Celery task, or an admin panel, they all dispatch the same command.
The business logic lives once. Nothing else needs to change.

---

## 2. `@api.on_event` — Reaction to a Fact

### When to use

Use a free-function event handler when:

- You want to **react** to something that already happened (a past-tense fact).
- The reaction is **fire-and-forget** — the original caller does not need the result.
- You need **fan-out**: multiple independent handlers listening to the same event.
- The logic spans a **different bounded context** (e.g. notification, analytics, cache
  invalidation, external integration).

Do **not** use an event handler for operations that must complete before the HTTP response
is returned — that is what commands (or hooks) are for.

### Definition

Free-function handlers live at **module level** (not inside a class):

```python
from ede.core import api
from ede.core.bus.types import Event
from ede.core.env import Env


@api.on_event("logistics.shipment.confirmed")
def notify_assigned_user(event: Event, env: Env) -> None:
    """Send an in-app notification when a shipment is confirmed."""
    user_id = event.payload.get("assigned_to")
    if not user_id:
        return
    env.models["notifications.message"].create({
        "user_id": user_id,
        "body": f"Shipment {event.payload['shipment_ref']} has been confirmed.",
    })


@api.on_event("logistics.shipment.confirmed")
def update_analytics(event: Event, env: Env) -> None:
    """Second listener — fan-out with zero coupling to the first handler."""
    env.models["analytics.shipment_stat"].create({
        "event": "confirmed",
        "tenant_id": event.tenant_id,
    })
```

### Emitting the event

From inside a model method or command handler:

```python
self.emit("logistics.shipment.confirmed", {
    "shipment_id": cmd.model_id,
    "shipment_ref": cmd.record.ref,
    "assigned_to": cmd.record.assigned_user_id,
    "tenant_id": self.env.tenant_id,
})
```

Or directly via `env`:

```python
env.emit("logistics.shipment.confirmed", {...})
```

### Rules

| Rule | Detail |
|------|--------|
| Signature | `(event: Event, env: Env) -> None` |
| Placement | Module-level functions only (not class methods) |
| Fan-out | Multiple handlers per event name are allowed |
| Delivery | Async — delivered by `EventWorker`; the emitting request does not wait |
| Retries | Automatic exponential back-off (5 attempts, then dead-letter) |
| Name convention | `{model_key}.{past_tense}`, e.g. `logistics.shipment.confirmed` |
| Tenant context | `event.tenant_id` is set; `EventWorker` restores tenant context before calling |

### Why events make the system future-ready

Events are the **escape valve from coupling**. When you add a new integration (Slack
notification, external webhook, BI pipeline), you add a new `@api.on_event` handler.
Nothing in the domain model changes. The emitting code never knows who is listening.
This is how you grow a system without touching working code.

Retries with exponential back-off mean temporary failures (network blip, DB contention)
heal themselves — no compensating workflows or cron jobs needed for this class of failure.

---

## 3. `@api.on_event(track_fields=[...])` — Field Change Tracking

### When to use

Use the `track_fields` variant when:

- You want to **react to a specific field changing** on a model (e.g. a slug, a status,
  a category FK).
- The reaction is **data-driven**: only fire when the value actually changed.
- You want this logic **co-located with the model** that owns the field.

Do **not** use `track_fields` for general event reactions — use a plain `@api.on_event`
free function instead.

### Definition

The handler is a **method on the owning `DomainModel`** class:

```python
@api.model("res.organization")
class Organization(DomainModel):

    @api.on_event("res.organization.field_changed", track_fields=["code"])
    def handle_code_changed(self, event, env) -> None:
        """
        Fires only when the 'code' field value actually changes.
        event.payload contains: model_key, record_id, field, old_value, new_value, tenant_id.
        """
        self.emit("web.client.reload", {
            "reason": "org_slug_changed",
            "org_id": event.payload.get("record_id"),
            "new_slug": event.payload.get("new_value").lower(),
            "tenant_id": event.tenant_id,
        })
```

### How it works internally

1. `register_model()` inspects all methods with `track_fields=[...]` and collects all
   declared field names across the class.
2. Two lifecycle hooks are auto-registered: `pre.{model_key}.update` and
   `post.{model_key}.update`.
3. **Pre-hook** — stashes current DB values of the tracked fields on the command object
   before the update runs.
4. **Post-hook** — compares the new values in the write payload against the stashed values.
   For each field whose value changed, emits `{model_key}.field_changed` into the EventQueue.
5. The `track_fields=["code"]` filter ensures the handler only receives deliveries where
   `event.payload["field"] == "code"`. Omit `track_fields` to receive every
   `field_changed` event for the model.

### Event payload shape

| Key | Type | Description |
|-----|------|-------------|
| `model_key` | `str` | e.g. `"res.organization"` |
| `record_id` | `str` | `record_uuid` of the updated record |
| `field` | `str` | Name of the field that changed |
| `old_value` | `Any` | Value before the update |
| `new_value` | `Any` | Value after the update |
| `tenant_id` | `str` | Tenant context at the time of update |

### Notes

- No `field_changed` event is emitted if the value did not change (equality check).
- No explicit class-level declaration is needed — tracked fields are auto-derived from
  the `track_fields` arguments on the methods.
- If no `EventQueue` is configured (e.g. migration context), the post-hook silently skips.

### Why field tracking makes the system future-ready

Field tracking is the **bridge between persistence and reactivity**. Instead of
instrumenting every `write()` call that touches a particular field, you declare once —
on the model that owns the field — what matters. Any number of downstream reactions can
subscribe. The update command stays clean; the model stays self-contained.

This also prevents the anti-pattern of embedding cross-cutting concerns (cache busting,
push notifications, audit trails) directly in command handlers.

---

## 4. `web.client.*` Events — Browser Push

`web.client.*` events are special domain events that cross the server–browser boundary.
When emitted, they are picked up by the presentation layer's free-function handlers which
broadcast them to all connected browser tabs for the tenant via SSE (Server-Sent Events).

### Available events

| Event name | Effect on connected browser tabs |
|------------|----------------------------------|
| `web.client.reload` | Tab reloads the workspace (re-bootstraps apps/menus/session) |
| `web.client.logout` | Tab clears the session and redirects to the login page |

### How to emit

From inside a model method (typically inside an `@api.on_event` or `@api.on_command`
handler):

```python
# Reload all tabs for the current tenant
self.emit("web.client.reload", {
    "reason": "org_slug_changed",
    "tenant_id": self.env.tenant_id,
})

# Force logout of all tabs for the current tenant
self.emit("web.client.logout", {
    "reason": "session_revoked",
    "tenant_id": self.env.tenant_id,
})
```

Or directly via `env`:

```python
env.emit("web.client.reload", {"reason": "config_changed", "tenant_id": env.tenant_id})
env.emit("web.client.logout", {"reason": "admin_forced_logout", "tenant_id": env.tenant_id})
```

### How it works

```
env.emit("web.client.reload", payload)
  │
  └─ EventQueue.enqueue(event)
       │
       └─ EventWorker delivers to:
            presentation/events/web_client_events.py
              handle_web_client_reload(event, env)
                └─ WebPushRegistry.broadcast(tenant_id, {type: "web.client.reload", ...})
                     │
                     └─ SSE stream → all connected tabs for the tenant
```

The free-function handler in
[`src/ede/foundation/presentation/events/web_client_events.py`](../src/ede/foundation/presentation/events/web_client_events.py)
bridges the domain event into the SSE broadcast.

### When to use

| Scenario | Event |
|----------|-------|
| A URL slug / org code changes | `web.client.reload` |
| Admin updates a global config that affects the UI | `web.client.reload` |
| Session revoked server-side (timeout, admin action) | `web.client.logout` |
| User's role/permissions changed and must re-authenticate | `web.client.logout` |

### Important: always include `tenant_id`

The `WebPushRegistry` uses `tenant_id` to scope the broadcast. Always include it:

```python
self.emit("web.client.reload", {
    "reason": "...",
    "tenant_id": self.env.tenant_id,   # required
})
```

### Why web push completes the reactive loop

Without `web.client.*`, the reactive chain stops at the backend. Field changes, session
revocations, and config updates happen silently — users need a manual refresh. With these
events, the same event-driven pattern that connects domain models to each other now
extends to the browser. The domain model emits a fact; the browser reacts. No polling.
No WebSocket server. No coupling between the domain model and the SSE infrastructure.

---

## 5. How All Three Work Together

The three primitives are designed to compose into a coherent reactive ecosystem:

```
@api.on_command     → mutates state, returns result, optionally emits events
@api.on_event       → reacts to events, emits more events (fan-out, pipeline)
@api.on_hook        → guards and derives synchronously before/after a command
track_fields=[...]  → auto-emits field_changed events; connects persistence to @api.on_event
web.client.*        → bridges the event bus to the browser via SSE
```

A production feature often uses all of them:

**Feature**: org code rename → all connected browsers navigate to the new URL.

```
1. ede.update Command               → writes new code to DB
2. pre/post hooks (auto-registered) → detect code field changed → emit field_changed
3. handle_code_changed (Event)      → reacts → emit web.client.reload
4. handle_web_client_reload (Event) → broadcasts to SSE
5. Browser tab receives reload      → navigates to new URL
```

Each step is independently testable. Each step can be changed or extended without
touching the others. That is the design goal.

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it breaks things | Correct approach |
|--------------|----------------------|------------------|
| Business logic in a controller | Logic trapped in HTTP layer; can't reuse from CLI or tests | Move to `@api.on_command` |
| Cross-model side effects in a command handler | Command becomes responsible for unrelated concerns | Emit an event; handle in `@api.on_event` |
| Calling `env.dispatch(Command(...))` from an event handler | Commands are sync; event handlers are fire-and-forget — mixing breaks the mental model | Use ORM layer directly in event handlers, or emit another event |
| Putting `web.client.*` emit directly in a command handler | Couples domain command to the presentation layer | Emit a domain event from the command; let an `@api.on_event` handler emit `web.client.*` |
| Using `@api.on_event` with `track_fields` as a free function | `track_fields` only works on DomainModel methods (registration via `register_model`) | Place as a DomainModel method |

---

## Summary Table

| Decorator | Where defined | Sync/Async | Returns | Use when |
|-----------|--------------|------------|---------|----------|
| `@api.on_command("x.verb")` | DomainModel method | Sync | Yes | Mutate state; need result |
| `@api.on_event("x.happened")` | Free function | Async | No | React to a fact; fan-out; cross-context |
| `@api.on_event("x.field_changed", track_fields=[...])` | DomainModel method | Async | No | React when a specific field value changes |
| `env.emit("web.client.reload", {...})` | Anywhere | Async | No | Push reload instruction to all browser tabs |
| `env.emit("web.client.logout", {...})` | Anywhere | Async | No | Force logout of all browser tabs |

---

## See Also

- [04-command-event-bus.md](04-command-event-bus.md) — full reference: Command, Event,
  CommandBus, EventQueue, EventWorker, retries, field change tracking internals.
- [14-lifecycle-hooks.md](14-lifecycle-hooks.md) — `@api.on_hook` reference for
  synchronous pre/post interception.
- [07-http-layer.md](07-http-layer.md) — Web Push (SSE) endpoint and `WebPushRegistry`.
- [01-overview.md](01-overview.md) — five-layer architecture and how the buses fit in.
