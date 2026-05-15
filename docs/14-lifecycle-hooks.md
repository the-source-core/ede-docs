# EDE Framework — Lifecycle Hooks

## Overview

Every mutating command dispatched through the `CommandBus` fires two synchronous hook points:

```
pre.{model_key}.{operation}   →  fires BEFORE the command handler executes
post.{model_key}.{operation}  →  fires AFTER a successful handler execution
```

Hooks receive the same `Command` object as the command handler. `cmd.record` is always
a `RecordSet` — an uncommitted (`__new__`) one for create operations, a live DB record
for everything else.

**Use hooks for:**
- Record restriction / validation (raise to veto the operation)
- Derived field synchronization
- Audit logging (post-hooks)
- Workflow guards (check state machine transitions)

**Hooks are NOT for:**
- Long-running work — use `@api.on_event` (async, retried)
- Cross-model notifications — emit an event and subscribe
- Per-request context injection — use `env.with_principal()`

### Why synchronous hooks are the right tool for data integrity

The alternative to hooks is putting guards inside every command handler that could
trigger the operation. This works for commands you own, but it breaks down for the
generic `ede.delete` command, which is shared across all models. You can't modify
`ede.delete` to add model-specific guards without violating the framework's command
ownership rules.

Lifecycle hooks solve this cleanly: `pre.res.organization.delete` fires for every
`ede.delete` command with `model_key="res.organization"`, regardless of where the
command was dispatched from. A controller, a CLI, another command handler — all get
the same protection.

**Pre-hooks are the data integrity layer. Post-hooks are the audit/reaction layer.**
Pre-hooks veto operations before they happen. Post-hooks react after they succeed.
Neither affects the command handler's implementation. Both can be added to an existing
model without changing any existing code.

This is especially powerful for cross-cutting concerns: add a post-hook to emit an
audit event on every update to every `res.*` model, and the audit trail covers the
entire foundation without touching any of the command handlers.

---

## The Full Execution Flow

```
env.dispatch(Command(name="ede.delete", model_key="res.organization", model_id="uuid-..."))
                │
                ▼
         CommandBus.dispatch()
                │
                ├── 1. resolve_command("ede.delete") → CommandTarget(CrudKernel, "handle_delete")
                │
                ├── 2. instantiate CrudKernel(), inject env
                │
                ├── 3. derive hook key
                │       cmd.name starts with "ede." → strip prefix
                │       "ede.delete" + model_key "res.organization" → "res.organization.delete"
                │
                ├── 4. fire pre-hooks: env.run_hooks("pre.res.organization.delete", cmd)
                │       └── Organization.restrict_delete(cmd)   ← RAISES ValueError
                │
                │   (exception propagates — steps 5 and 6 never execute)
                │
                ├── 5. [skipped] call handler → CrudKernel.handle_delete(cmd)
                │
                └── 6. [skipped] fire post-hooks: env.run_hooks("post.res.organization.delete", cmd)
```

For a successful operation:

```
env.dispatch(Command(name="ede.create", model_key="hook_test.item",
                     payload={"values": {"name": "Anchor"}}))
                │
                ▼
         CommandBus.dispatch()
                │
                ├── 1. resolve_command("ede.create") → CommandTarget(CrudKernel, "handle_create")
                │
                ├── 2. instantiate CrudKernel(), inject env
                │
                ├── 3. hook key → "hook_test.item.create"
                │
                ├── 4. inject __new__ RecordSet into cmd._record_override
                │       (so cmd.record works in pre-hook before the DB insert)
                │
                ├── 5. fire pre-hooks: env.run_hooks("pre.hook_test.item.create", cmd)
                │       └── cmd.record.name == "Anchor"  (reads from in-memory values)
                │
                ├── 6. call handler → CrudKernel.handle_create(cmd) → inserts row → returns record
                │
                ├── 7. clear _record_override (so post-hook sees the real saved record)
                │
                └── 8. fire post-hooks: env.run_hooks("post.hook_test.item.create", cmd)
                        └── cmd.record.name == "Anchor"  (reads from DB via browse)
```

---

## Hook Key Derivation

The hook key determines which registered hooks fire for a given command.

| Command name | `model_key` | Derived hook key |
|---|---|---|
| `ede.create` | `res.organization` | `res.organization.create` |
| `ede.update` | `res.user` | `res.user.update` |
| `ede.delete` | `res.organization` | `res.organization.delete` |
| `res.organization.deactivate` | `res.organization` | `res.organization.deactivate` |
| `res.user.register` | `res.user` | `res.user.register` |
| `hook_test.item.activate` | `hook_test.item` | `hook_test.item.activate` |

**Rule:**
- CRUD commands (`ede.*`): strip the `ede.` prefix, use `model_key` as qualifier
  → `{model_key}.{operation}` e.g. `res.organization.delete`
- Domain commands (everything else): command name IS the full key
  → `{cmd.name}` e.g. `res.organization.deactivate`

**Pre/post keys** wrap the derived key:
- Pre-hook: `pre.{derived_key}`
- Post-hook: `post.{derived_key}`

**No hook key is derived when:**
- `cmd.model_key` is `None` (no model scope → hooks silently skip)

**Read-only commands never fire hooks** (no `pre.*` / `post.*`):
- `ede.read_one`
- `ede.search`
- `ede.count`
- `ede.read_group`

---

## Defining a Hook

Use `@api.on_hook(key)` on a **`DomainModel` method** (inside the class body only —
not on free functions):

```python
from ede.core import api
from ede.core.kernel.model import DomainModel
from ede.core.types import Command


@api.model("res.organization")
class Organization(DomainModel):

    @api.on_hook("pre.res.organization.delete")
    def restrict_delete(self, cmd: Command) -> None:
        """Block hard-deletion. Raise to veto the delete."""
        raise ValueError(
            "Organization records cannot be deleted. "
            "Use 'res.organization.deactivate' instead."
        )
```

**Method signature:** `(self, cmd: Command) -> None`

The return value is ignored. Raise any exception to veto the operation (pre-hooks only).

**Rules:**
- Must be inside a `DomainModel` subclass decorated with `@api.model(key)`
- `self.env` is injected automatically (same as command handlers)
- Multiple hooks with the same key are allowed — all run in registration order
- Hook registration happens automatically when `register_model()` is called at boot;
  no extra wiring required

---

## `cmd.record` Inside a Hook

`cmd.record` is always a `RecordSet`. Its source depends on the hook type:

| Hook type | `cmd.record` source | Notes |
|---|---|---|
| `pre.*.create` | `RecordSet.new()` — in-memory, not in DB | `cmd.record.name` reads from payload values |
| `pre.*.update` | Live DB browse (`browse(model_id)`) | Reflects values before the write |
| `pre.*.delete` | Live DB browse (`browse(model_id)`) | Record still exists; read fields here |
| `post.*.create` | Live DB browse (`browse(model_id)`) | Record committed; has real `record_uuid` |
| `post.*.update` | Live DB browse (`browse(model_id)`) | Reflects values after the write |
| `post.*.delete` | Live DB browse (`browse(model_id)`) | Record deleted; DB query returns empty |
| `pre.*` / `post.*` (domain cmd) | Live DB browse (`browse(model_id)`) | Requires `model_id` on the Command |

### Pre-create: `RecordSet.new()`

Before the record is inserted, `cmd.record` returns an in-memory `RecordSet` populated
with the values from `cmd.payload["values"]`. Field reads work the same way:

```python
@api.on_hook("pre.logistics.shipment.create")
def validate_on_create(self, cmd: Command) -> None:
    rs = cmd.record        # RecordSet.__new__ — not in DB yet
    if rs.origin == rs.destination:
        raise ValueError("Origin and destination cannot be the same.")
```

`rs._is_new` is `True`; `rs._ids` is empty; reads come from `rs._new_values`.

### Pre-delete: read before it is gone

Inside a `pre.*.delete` hook the record still exists in the database. Read all needed
field values inside the hook — after the handler executes the row is gone:

```python
@api.on_hook("pre.logistics.shipment.delete")
def guard_delete(self, cmd: Command) -> None:
    rs = cmd.record                    # live DB record
    if rs.status not in ("draft", "cancelled"):
        raise ValueError(f"Cannot delete a shipment in status '{rs.status}'.")
```

---

## Practical Examples

### 1. Restrict deletion

```python
@api.model("res.organization")
class Organization(DomainModel):

    @api.on_hook("pre.res.organization.delete")
    def restrict_delete(self, cmd: Command) -> None:
        raise ValueError(
            "Organization records cannot be deleted. "
            "Use 'res.organization.deactivate' instead."
        )
```

### 2. Conditional validation on create

```python
@api.model("logistics.shipment")
class Shipment(DomainModel):

    @api.on_hook("pre.logistics.shipment.create")
    def validate_route(self, cmd: Command) -> None:
        rs = cmd.record
        if rs.origin == rs.destination:
            raise ValueError("Origin and destination cannot be the same.")
```

### 3. Deny deactivation when active users exist

```python
@api.model("res.organization")
class Organization(DomainModel):

    @api.on_hook("pre.res.organization.deactivate")
    def check_no_active_users(self, cmd: Command) -> None:
        users = self.env.models["res.user"].search(
            [("organization_id", "=", cmd.model_id), ("is_active", "=", True)]
        )
        if users:
            raise ValueError(
                f"Cannot deactivate organization: {len(users)} active user(s) still linked."
            )
```

### 4. Audit log on update (post-hook)

```python
@api.model("res.user")
class User(DomainModel):

    @api.on_hook("post.res.user.update")
    def audit_update(self, cmd: Command) -> None:
        # Post-hook fires only after a successful write.
        # Emit an async event so the audit log is written without blocking the request.
        self.emit("res.user.updated", {
            "record_id": cmd.model_id,
            "updated_by": (cmd._env.principal or {}).get("sub"),
        })
```

### 5. Multiple hooks on the same key

All hooks run in registration order. The first one to raise wins (remaining hooks skip):

```python
@api.model("logistics.shipment")
class Shipment(DomainModel):

    @api.on_hook("pre.logistics.shipment.delete")
    def check_status(self, cmd: Command) -> None:
        if cmd.record.status not in ("draft", "cancelled"):
            raise ValueError("Only draft or cancelled shipments may be deleted.")

    @api.on_hook("pre.logistics.shipment.delete")
    def check_no_documents(self, cmd: Command) -> None:
        if cmd.record.has_documents:
            raise ValueError("Remove attached documents before deleting.")
```

---

## How Hooks Are Registered

At boot, `Registry.register_model(model_cls)` scans every method in the class body.
Methods decorated with `@api.on_hook` carry a `__ede_hook__` attribute. The registry
wraps each method in a thin closure that:

1. Instantiates the model class
2. Injects `cmd._env` as `self.env`
3. Calls the original method

This mirrors exactly how `CommandBus` calls command handlers — uniform instantiation,
no shared state between calls.

```
registry._hooks: Dict[str, List[Callable]] = {
    "pre.res.organization.delete": [<wrapper for Organization.restrict_delete>],
    "pre.logistics.shipment.create": [<wrapper for Shipment.validate_route>, ...],
    ...
}
```

`env.run_hooks(hook_key, cmd)` iterates the list and calls each wrapper in order.

---

## Comparison: Command vs Event vs Hook

| | Command | Event | Hook |
|---|---|---|---|
| **Trigger** | Explicit `env.dispatch(cmd)` | `env.emit(name, payload)` / `self.emit(...)` | Implicit — fired by CommandBus for every mutating command |
| **Execution** | Synchronous | Asynchronous (EventWorker) | Synchronous, inline |
| **Return value** | Yes (dict / list) | None | None (raise to veto) |
| **Fan-out** | No (one handler per name) | Yes (multiple handlers per event) | Yes (multiple hooks per key) |
| **Veto the operation** | N/A | No | Yes — raise any exception in a pre-hook |
| **Handler location** | `DomainModel` method | Module-level free function | `DomainModel` method |
| **`cmd.record` available** | Yes | No | Yes |
| **Retry on failure** | No | Yes (exponential backoff + DLQ) | No |
| **Failure mode** | Raises in request | Retried, then dead-lettered | Raises in request (same as command) |
| **Convention** | `shipment.create` (verb) | `shipment.created` (past tense) | `pre.shipment.create` / `post.shipment.create` |

---

## Command & Event Bus — Recap

### CommandBus (synchronous)

`src/ede/core/bus/command_bus.py`

```
env.dispatch(cmd)
    │
    └── CommandBus.dispatch(cmd, env)
            │
            ├── 1. Registry.resolve_command(cmd.name) → CommandTarget
            ├── 2. Instantiate model_cls(), inject env
            ├── 3. Inject cmd._env = env
            ├── 4. Derive hook key (_derive_hook_key)
            ├── 5. (if create) Inject __new__ RecordSet into cmd._record_override
            ├── 6. env.run_hooks("pre.{hook_key}", cmd)     ← pre-hooks
            ├── 7. model.<method_name>(cmd)                 ← handler
            ├── 8. (if create) Clear cmd._record_override
            └── 9. env.run_hooks("post.{hook_key}", cmd)    ← post-hooks
```

### EventQueue / EventWorker (asynchronous)

`src/ede/core/bus/queue.py` / `src/ede/core/bus/worker.py`

```
self.emit("shipment.confirmed", {...})
    │
    └── EventQueue.enqueue(Event)        ← non-blocking, returns immediately
                    │
                    ▼           (separate thread / process)
             EventWorker.run_forever()
                    │
                    └── dequeue(batch=10, timeout=0.5s)
                            └── for each delivery:
                                    ├── handler(event, env)
                                    ├── success → ack()
                                    └── failure → nack() [retry] or dead_letter()
```

See [04-command-event-bus.md](04-command-event-bus.md) for full details on EventQueue,
EventWorker, retry policies, and the InMemory / Kafka providers.

---

## Testing Hooks

Use `build_test_runtime` from `tests._common` and register hooks directly via
`registry.register_hook()` when you need isolated tests without a full model:

```python
from tests._common import build_test_runtime
from ede.core.registry import Registry
from ede.core.types import Command


def test_pre_hook_blocks_delete():
    reg = Registry()
    # ... register your model ...

    calls = []

    def guard(cmd: Command) -> None:
        raise ValueError("blocked")

    reg.register_hook("pre.mymodel.item.delete", guard)
    runtime = build_test_runtime(reg)

    with pytest.raises(ValueError, match="blocked"):
        runtime.env.dispatch(Command(
            name="ede.delete",
            payload={},
            model_key="mymodel.item",
            model_id="some-uuid",
        ))
```

For `@api.on_hook` methods declared inside a model class, hooks are auto-registered when
`registry.register_model(ModelCls)` is called — no manual wiring needed.

---

## Auto-Registered Field Tracking Hooks

When a model has `@api.on_event` methods with a `track_fields=[...]` argument,
`register_model()` automatically registers additional `pre` and `post` update hooks
— **no `@api.on_hook` decoration required**.

```python
@api.model("res.organization")
class Organization(DomainModel):

    @api.on_event("res.organization.field_changed", track_fields=["code"])
    def handle_code_changed(self, event, env) -> None:
        ...  # fires only when event.payload["field"] == "code"
```

This declaration causes `register_model()` to register:

| Hook key | What it does |
|---|---|
| `pre.res.organization.update` | Stashes old values of `["code"]` on the command object |
| `post.res.organization.update` | Compares new vs old; emits `res.organization.field_changed` for each changed field |

The tracked field list is **derived from the union of all `track_fields` lists** across
all `@api.on_event` methods on the class. An explicit `__ede_track_fields__ = [...]`
class attribute is also supported and merged, but not required.

See [04-command-event-bus.md — Field Change Tracking](04-command-event-bus.md#field-change-tracking)
for the full event payload shape and handler filter semantics.

Full test suite: `src/tests/bus/test_lifecycle_hooks.py` (37 tests covering registry,
`env.run_hooks`, CRUD hooks with SQLite, domain command hooks, read-only exclusions,
and `RecordSet.new()`).
