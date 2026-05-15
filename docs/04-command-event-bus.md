# EDE Framework — Command & Event Bus

## Overview

EDE uses two separate buses for intent and fact:

| Bus | Sync/Async | Purpose |
|---|---|---|
| **CommandBus** | Synchronous | Dispatch named intents → model handlers; returns result |
| **EventQueue** | Asynchronous | Enqueue named facts → event handlers (via EventWorker) |

Commands mutate state. Events record that something happened and fan out to listeners.

---

## Commands

### What Is a Command?

A `Command` is an immutable carrier of intent. It has a name and a payload.

```python
from ede.core.types import Command

cmd = Command(
    name="shipment.confirm",    # Required: maps to a registered handler
    payload={"notes": "OK"},    # Required: always pass, even if empty dict
    model_key="logistics.shipment",  # Optional: target model class
    model_id="uuid-1234",            # Optional: target record (populates cmd.record)
)
```

**Important**: `payload` is a required positional argument. Always pass `payload={}` minimum.

### `cmd.record`

When `model_key` and `model_id` are set, `cmd.record` resolves to a `RecordSet` containing
the targeted record:

```python
@api.on_command("shipment.confirm")
def confirm(self, cmd: Command) -> dict:
    record = cmd.record      # → RecordSet(model_key="logistics.shipment", ids=["uuid-1234"])
    record.write({"status": "confirmed"})
```

### Dispatch

Dispatch a command via `env.dispatch()`:

```python
result = env.dispatch(Command(
    name="shipment.confirm",
    payload={"notes": "urgent"},
    model_key="logistics.shipment",
    model_id="uuid-1234",
))
```

---

## CommandBus

`src/ede/core/bus/command_bus.py`

The `CommandBus` is the synchronous dispatcher. It lives on `Env.commands`.

**Dispatch sequence**:
1. `CommandBus.dispatch(cmd, env)` called
2. Looks up `Registry.resolve_command(cmd.name)` → `CommandTarget(model_cls, method_name)`
3. Instantiates `model_cls()` (fresh instance per dispatch)
4. Injects `env` → `model.env = env`
5. Injects `cmd._env = env` (internal use only)
6. Calls `model.<method_name>(cmd)` → returns result

**Error**: Raises `CommandHandlerNotFound` if the command name is not registered.

---

## Registering a Command Handler

Use `@api.on_command(name)` on a `DomainModel` method:

```python
@api.model("logistics.shipment")
class Shipment(DomainModel):

    @api.on_command("shipment.create")
    def create(self, cmd: Command) -> dict:
        record = self.env.models["logistics.shipment"].create(cmd.payload)
        return record.read()[0]

    @api.on_command("shipment.bulk_confirm")
    def bulk_confirm(self, cmd: Command) -> dict:
        ids = cmd.payload["ids"]
        for record in self.env.models["logistics.shipment"].browse(*ids):
            record.write({"status": "confirmed"})
        return {"confirmed": len(ids)}
```

**Rules**:
- Handler method signature: `(self, cmd: Command) -> Any`
- Return value becomes the HTTP response body (serialized to JSON by the adapter)
- One handler per command name across the entire app (duplicates raise `DuplicateHandler`)
- Command name convention: `"{app_key}.{verb}"` e.g. `"logistics.shipment.create"`

---

## Built-in CRUD Commands

`DomainModel` ships with built-in handlers for generic CRUD. These are auto-registered with
the `ede.*` namespace for every concrete model.

| Command | Payload | Description |
|---|---|---|
| `ede.create` | `{field: value, ...}` | Create a record |
| `ede.update` | `{field: value, ...}` (+ `model_id`) | Update a record |
| `ede.delete` | `{}` (+ `model_id`) | Delete a record |
| `ede.read_one` | `{}` (+ `model_id`) | Read single record as dict |
| `ede.search` | `{domain, order, limit, offset}` | Search records |
| `ede.count` | `{domain}` | Count matching records |
| `ede.read_group` | `{domain, group_by, aggregate, ...}` | Group + aggregate |

Do not call `ede.*` commands directly from HTTP controllers — use the ORM layer
(`env.models[key].create(...)`, `record.write(...)`, etc.) instead.

---

## Events

### What Is an Event?

An `Event` records that something happened. It is immutable and timestamped.

```python
from ede.core.bus.types import Event

event = Event.build(
    name="shipment.confirmed",
    payload={"shipment_id": "uuid-1234", "tenant_id": "acme"},
    tenant_id="acme",
    correlation_id="req-abc",
    causation_id="cmd-xyz",
)
# event.event_id    → auto-generated UUID
# event.occurred_at → utc_now()
```

`Event` is a frozen dataclass — all fields immutable after creation.

### Emitting an Event

From inside a model method:
```python
self.emit("shipment.confirmed", {
    "shipment_id": record.id,
    "tenant_id": self.env.tenant_id,
})
```

Or from `Env` directly:
```python
env.emit("shipment.confirmed", {"shipment_id": "uuid-1234"})
```

Both routes call `EventQueue.enqueue(event)`. The queue is non-blocking. Events are
delivered by the `EventWorker` asynchronously.

---

## Registering an Event Handler

Use `@api.on_event(name)` on a **module-level function** (not a method):

```python
from ede.core import api
from ede.core.bus.types import Event
from ede.core.env import Env

@api.on_event("shipment.confirmed")
def on_shipment_confirmed(event: Event, env: Env) -> None:
    payload = event.payload
    # send notification, update analytics, etc.
    user = env.models["res.user"].browse(payload.get("assigned_to"))
    ...
```

**Rules**:
- Function signature: `(event: Event, env: Env) -> None`
- Module-level functions only (not class methods)
- Multiple handlers per event name are allowed (fan-out)
- Tenant context: `event.tenant_id` is set; EventWorker calls
  `set_current_tenant_id(event.tenant_id)` before dispatching

---

## EventQueue Protocol

`src/ede/core/bus/queue.py`

```python
class EventQueue(Protocol):
    provider_key: str

    def enqueue(self, event: Event) -> None: ...
    def dequeue(self, batch_size: int, timeout_seconds: float) -> List[EventDelivery]: ...
    def ack(self, receipt_handle: str) -> None: ...
    def nack(self, receipt_handle: str, *, retry_after_seconds: float) -> None: ...
    def dead_letter(self, receipt_handle: str, *, reason: str) -> None: ...
    def describe(self) -> str: ...
```

### InMemoryEventQueue (dev/test)

`provider_key = "inmemory"`

Thread-safe in-memory implementation. Supports:
- Visible queue, inflight tracking, scheduled retries, dead-letter list
- `nack()` schedules retry after `retry_after_seconds` (released when due)
- `dead_letter()` moves to DLQ (inspectable for tests)

### Kafka (production)

Configured via `DEFAULT_MESSAGE_BROKER_PROVIDER = "kafka"` in settings.
Uses `confluent-kafka` adapter. Same Protocol contract.

---

## EventDelivery & RetryPolicy

`EventDelivery` wraps an `Event` with delivery metadata:

```python
@dataclass(frozen=True)
class EventDelivery:
    event: Event
    receipt_handle: str      # opaque handle for ack/nack
    attempt_number: int      # 1-based; incremented on each retry
```

`RetryPolicy` controls backoff:

```python
@dataclass(frozen=True)
class RetryPolicy:
    max_attempts: int = 5
    base_delay_seconds: float = 0.25
    max_delay_seconds: float = 10.0

    def compute_delay_seconds(self, attempt_number: int) -> float:
        # Exponential backoff: 2^(attempt-1) * base, capped at max
        delay = (2 ** (attempt_number - 1)) * self.base_delay_seconds
        return min(delay, self.max_delay_seconds)
```

Backoff schedule (default policy):
| Attempt | Delay |
|---|---|
| 1 | 0.25s |
| 2 | 0.50s |
| 3 | 1.00s |
| 4 | 2.00s |
| 5 → DLQ | — |

---

## EventWorker

`src/ede/core/bus/worker.py`

The `EventWorker` is a long-running poll loop that dequeues events and dispatches them.
It runs in a separate thread (dev: `ede serve --with-worker`) or process (`ede worker`).

```
EventWorker.run_forever()
  │
  └── loop:
        ├── EventQueue.dequeue(batch_size=10, timeout=0.5s) → [EventDelivery, ...]
        │
        └── for each delivery:
              ├── set_current_tenant_id(event.tenant_id)
              ├── EventDispatcher.dispatch(event, env)
              │     └── Registry.get_event_handlers(event.name)
              │           └── handler(event, env)     (each handler called in order)
              │
              ├── on success → EventQueue.ack(receipt_handle)
              │
              └── on exception:
                    ├── attempt < max → EventQueue.nack(receipt_handle, retry_after=computed_delay)
                    └── attempt >= max → EventQueue.dead_letter(receipt_handle, reason=str(exc))
```

Worker stats available via `worker.health_snapshot()`:
```python
{
    "received": 142,
    "acked": 140,
    "retried": 2,
    "dead_lettered": 0,
}
```

### Graceful Shutdown

```python
worker.request_stop()   # sets stop flag; run_forever() exits after current batch
```

---

## Field Change Tracking

The framework can automatically emit `{model_key}.field_changed` events whenever
a specific field value changes on update. Declare handlers using the existing
`@api.on_event` decorator with the `track_fields` kwarg:

```python
@api.model("res.organization")
class Organization(DomainModel):

    @api.on_event("res.organization.field_changed", track_fields=["code"])
    def handle_code_changed(self, event, env) -> None:
        """Called only when event.payload['field'] == 'code'."""
        env.emit("web.client.reload", {
            "reason": "org_slug_changed",
            "new_slug": event.payload.get("new_value"),
        })
```

### How it works

1. When `register_model()` sees methods with `track_fields=[...]`, it collects all
   declared field names and auto-registers `pre.{key}.update` / `post.{key}.update`
   lifecycle hooks.
2. **Pre-hook** — stashes current DB values of the tracked fields on the command object.
3. **Post-hook** — compares new payload values against the stashed values; for each
   changed field, emits `{model_key}.field_changed` into the EventQueue.
4. **Handler filter** — `track_fields=["code"]` means the handler only fires when
   `event.payload["field"] == "code"`. Omit `track_fields` to receive every
   `field_changed` event for the model.

### Event payload shape

| Key | Type | Description |
|---|---|---|
| `model_key` | `str` | Model key (e.g. `"res.organization"`) |
| `record_id` | `str` | `record_uuid` of the updated record |
| `field` | `str` | Name of the changed field |
| `old_value` | `Any` | Value before the update |
| `new_value` | `Any` | Value after the update |
| `tenant_id` | `str` | Tenant context at the time of the update |

### Notes

- No explicit class-level declaration is needed — tracked fields are auto-derived from
  the `track_fields` argument on every `@api.on_event` method in the class.
- Model-bound event handlers (`@api.on_event` on a `DomainModel` method) are registered
  by `register_model()`. Free-function handlers (at module level) are registered by
  the module loader.
- No `field_changed` event is emitted if the value did not actually change (equality check).
- No event is emitted if the model has no `@api.on_event(track_fields=[...])` methods.
- If no `EventQueue` is configured (e.g. migration context), the post-hook silently skips.

---

## Lifecycle Hooks

Every mutating command dispatched through the `CommandBus` also fires synchronous
**lifecycle hooks** — `pre.*` before the handler and `post.*` after it.

See [14-lifecycle-hooks.md](14-lifecycle-hooks.md) for the full lifecycle hook reference,
including hook key derivation rules, `cmd.record` semantics per hook type, and
practical examples (restriction, validation, audit logging).

---

## Why Two Buses Make the System Robust

### Synchronous commands — fail fast, return immediately

The `CommandBus` is synchronous because mutations need to complete before an HTTP
response is returned. If a command fails (validation error, FK violation, hook veto),
the caller gets an immediate error. There is no ambiguity about whether the operation
succeeded.

This synchrony also makes commands easy to test: call `env.dispatch(cmd)`, check the
return value and the DB state. No async machinery required.

### Asynchronous events — absorb failures, scale independently

The `EventQueue` is asynchronous because side effects should not block the caller.
If a notification email fails because an SMTP server is down, the shipment was still
confirmed successfully. The event will be retried automatically when the SMTP server
recovers.

This asymmetry is deliberate. Events model facts — things that *have already happened*
and cannot be undone. Reactions to facts (notifications, analytics, integrations) are
inherently optional and independently reliable.

In production, swapping `"inmemory"` for `"kafka"` requires one config change. All
event handlers, retry logic, and dead-letter handling remain identical. The `EventQueue`
Protocol is the firewall between your business logic and the broker vendor.

### Field change tracking — declarative reactivity

Without field change tracking, you would either instrument every `write()` call that
touches a specific field, or run periodic diff jobs. Both approaches are fragile and
hard to maintain. The `track_fields` mechanism lets you declare *on the model itself*
which fields matter, and the framework takes care of before/after comparison, equality
checks, and event emission. The update command stays clean.

### Composing all three for a reactive system

See [15-command-event-guide.md](15-command-event-guide.md) for the full architectural
rationale, decision guide, concrete examples, and anti-patterns.

---

## Summary: Command vs Event vs Hook

| | Command | Event | Hook |
|---|---|---|---|
| Direction | Request → Response (sync) | One-way (async) | Inline, synchronous |
| Return value | Yes (any dict/list) | None | None (raise to veto) |
| Dispatch | `env.dispatch(cmd)` | `env.emit(name, payload)` | Implicit — fired by CommandBus |
| Handler signature | `(self, cmd) -> Any` (model method) | `(event, env) -> None` (free function) | `(self, cmd) -> None` (model method) |
| Fan-out | No (one handler per name) | Yes (multiple handlers per event name) | Yes (multiple hooks per key) |
| Retry | No | Yes (exponential backoff, configurable DLQ) | No |
| Failure mode | Raises exception in request | Retried, then dead-lettered | Raises exception in request |
| Convention | `shipment.create` (verb) | `shipment.created` (past tense) | `pre.shipment.create` / `post.shipment.create` |
