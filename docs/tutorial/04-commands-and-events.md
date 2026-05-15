# 4. Commands & Events

Every mutation in EDE flows through the **command bus**. Reads and writes that look like ORM calls actually dispatch typed `Command` objects under the hood. You hook into this flow by writing handlers — `@api.on_command` for the actual work, `@api.on_event` for downstream reactions, and `@api.on_hook` for synchronous gates around mutations.

```python
from ede.core import api
from ede.core.types import Command
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model("blog.post")
class BlogPost(DomainModel):
    title  = fields.Char(required=True)
    body   = fields.Char(multi_line=True)
    status = fields.Enum(
        selection=[("draft", "Draft"), ("published", "Published")],
        default="draft",
        required=True,
    )

    @api.on_command("blog.post.publish")
    def publish(self, cmd: Command) -> dict:
        if self.status == "published":
            return {"ok": True, "noop": True}
        self.write({"status": "published"})
        self.env.emit("blog.post.published", {"post_uuid": self.id})
        return {"ok": True}

    @api.on_hook("pre.blog.post.publish")
    def require_title(self, cmd: Command) -> None:
        if not (self.title or "").strip():
            raise ValueError("Cannot publish a post with no title.")


@api.on_event("blog.post.published")
def increment_publish_count(event, env):
    # Side-effects that don't block the user — run in the worker.
    metric = env.models["stats.daily_publishes"].search([])
    ...
```

By the end of this chapter you can write a custom command, gate it with a pre-hook, react to its completion with an event handler, and wire it to a button in the form view.

## What you're going to build

Add a `blog.post.publish` command to the `blog.post` model. Hook a button to it in the form view. Add a pre-hook that rejects publishing if the title is blank. Add an event handler that fires after the publish succeeds.

## 1. The four primitives

EDE separates request handling into four roles. Knowing which to reach for is most of the design work:

| Primitive | Decorator | When it fires | What it can do |
|---|---|---|---|
| **Command handler** | `@api.on_command("name")` | When a `Command(name=...)` is dispatched. | Mutates state. Returns a result. The single place business logic lives. |
| **Pre-hook** | `@api.on_hook("pre.<model_key>.<op>")` | Synchronously **before** the command handler. | Validates, gates, or transforms. Raising any exception aborts the command. |
| **Post-hook** | `@api.on_hook("post.<model_key>.<op>")` | Synchronously **after** a successful handler. | Bookkeeping that must happen in the same transaction (audit rows, derived denormalised fields). |
| **Event handler** | `@api.on_event("name")` | Asynchronously, in the worker, after a successful command. | Side-effects that must not block the user — emails, webhooks, search index updates. |

Choose by question: *does the user have to wait for this to succeed?* If yes → hook. If no → event.

## 2. Write a command

Command handlers live as methods on the `DomainModel`. They receive the dispatched `Command` and have access to `self.env` (the request-scoped runtime context):

```python
@api.on_command("blog.post.publish")
def publish(self, cmd: Command) -> dict:
    if self.status == "published":
        return {"ok": True, "noop": True}
    self.write({"status": "published"})
    self.env.emit("blog.post.published", {"post_uuid": self.id})
    return {"ok": True}
```

What's available inside a handler:

-   `self` — a `RecordSet` of the targeted record(s) when the command was dispatched with a `model_id`; an empty `RecordSet` otherwise (e.g. for create commands).
-   `cmd.payload` — the JSON-friendly dict passed when the command was dispatched.
-   `cmd.record` — the live `RecordSet` for `cmd.model_id`. Same as `self` when the handler is on the same model.
-   `self.env` — `Env` with `env.models`, `env.dispatch`, `env.emit`, `env.principal`, `env.tenant_id`.
-   Return value — anything JSON-serialisable. It flows back to whoever dispatched the command (a controller, another handler, the web client).

### Command naming

The name is what the bus dispatches on. The convention is `{model_key}.{verb}`:

| Command name | Meaning |
|---|---|
| `blog.post.publish` | Domain verb on `blog.post`. |
| `blog.post.archive` | Same model, different verb. |
| `ede.create` | The generic CRUD entrypoint — handled by `CrudKernel`, not your code. |

Generic CRUD commands (`ede.create` / `ede.read_one` / `ede.update` / `ede.delete` / `ede.search` / `ede.count` / `ede.read_group`) exist for every registered model automatically. You don't write them — you hook around them.

## 3. Pre-hooks — validate or veto a mutation

A pre-hook fires synchronously **before** the command handler. Raising any exception inside the hook aborts the whole flow — the handler never runs, the database transaction rolls back, and the dispatcher sees the exception.

```python
@api.on_hook("pre.blog.post.publish")
def require_title(self, cmd: Command) -> None:
    if not (self.title or "").strip():
        raise ValueError("Cannot publish a post with no title.")
```

Hooks are methods on the same `DomainModel` as the handler they gate. They receive the same `Command`. `self` is the live record. `self.env` is injected automatically.

### Hook key patterns

The `@api.on_hook(...)` argument is a string that follows one of two patterns:

| Pattern | Example | Fires on |
|---|---|---|
| `pre.<model_key>.<op>` / `post.<model_key>.<op>` | `pre.blog.post.publish` | A specific command on a specific model. |
| `pre.*.<op>` / `post.*.<op>` | `post.*.delete` | The same operation on **every** model. Use for cross-cutting concerns. |

For generic CRUD commands, the framework derives the hook key by stripping `ede.` and tacking the model key on:

| Dispatched command | Hook key |
|---|---|
| `ede.create` on `blog.post` | `pre.blog.post.create` / `post.blog.post.create` |
| `ede.update` on `blog.post` | `pre.blog.post.update` / `post.blog.post.update` |
| `ede.delete` on `blog.post` | `pre.blog.post.delete` / `post.blog.post.delete` |

For domain commands the full name is the hook key — `blog.post.publish` → `pre.blog.post.publish`.

Read-only commands (`ede.read_one` / `ede.search` / `ede.count` / `ede.read_group`) do **not** fire hooks.

### `cmd.record` semantics

Inside a pre-create hook the record doesn't exist yet — but `cmd.record` is still safe to read. The framework wraps the incoming `values` payload in a `__new__` (uncommitted) `RecordSet` so you can validate against the field values without writing to the database:

```python
@api.on_hook("pre.blog.post.create")
def require_slug(self, cmd: Command) -> None:
    # cmd.record is a __new__ RecordSet — reads from the create payload,
    # not from any persisted state.
    if not (cmd.record.slug or "").strip():
        raise ValueError("Slug is required.")
```

For all other operations (`update`, `delete`, domain commands), `cmd.record` is a live DB-backed `RecordSet`.

### Wildcard hooks

`pre.*.<op>` matches the operation on every model. Use them for cross-cutting concerns that don't belong to a specific model:

```python
@api.on_hook("post.*.delete")
def cleanup_external_ids(cmd: Command) -> None:
    """Remove the ir.data.reference row that pointed at this record."""
    env = cmd._env
    env.models["ir.data.reference"].search([
        ("model_key", "=", cmd.model_key),
        ("record_uuid", "=", cmd.model_id),
    ]).delete()
```

Wildcard hooks must be **free functions** (not model methods). Read `env` from `cmd._env` since there's no `self` to inject it onto.

## 4. Events — react asynchronously

Events are how one part of the system tells another part *"this happened"* without being coupled to it. The producer emits; the consumers subscribe. The bus delivers via the event worker, so handlers run **after** the originating command's transaction commits.

Emit from a command handler:

```python
self.env.emit("blog.post.published", {"post_uuid": self.id})
```

Subscribe from a free function — anywhere in your app code, decorated with `@api.on_event`:

```python
from ede.core import api


@api.on_event("blog.post.published")
def notify_subscribers(event, env):
    # event.name == "blog.post.published"
    # event.payload == {"post_uuid": "..."}
    post_uuid = event.payload["post_uuid"]
    subscribers = env.models["blog.subscriber"].search([("active", "=", True)])
    for sub in subscribers:
        env.dispatch(Command(
            name="ede.create",
            model_key="mail.outbox",
            payload={"values": {"to": sub.email, "subject": "New post", "post_uuid": post_uuid}},
        ))
```

Event handler signature: `(event, env)`. `event.name` and `event.payload` are dict-friendly. `env` is a fresh per-event context with its own tenant binding.

### Built-in events

The framework emits these for free — you don't have to call `env.emit` for them:

| Event | When |
|---|---|
| `ede.record.created` | After every successful `ede.create`. |
| `ede.record.updated` | After every successful `ede.update`. |
| `ede.record.deleted` | After every successful `ede.delete`. |
| `<model_key>.field_changed` | When a tracked field changes value — payload includes `field`, `old`, `new`. |
| `web.client.command.done` | After every successful domain command — used to push form-reload notices to connected web clients. |

### Field-tracking events

Subscribe to a specific field's change with `track_fields=[...]` on the event handler:

```python
@api.model("blog.post")
class BlogPost(DomainModel):
    ...

    @api.on_event("blog.post.field_changed", track_fields=["slug"])
    def slug_changed(self, event, env):
        # Fires only when the `slug` field changes — once per write that
        # actually mutates the value.
        env.emit("web.client.reload", {"model_key": "blog.post", "id": self.id})
```

The handler receives `event.payload["field"]`, `event.payload["old"]`, `event.payload["new"]`. Only field names in `track_fields` cause this method to fire — other field changes are ignored.

## 5. The full flow

When a controller (or the web client, or another handler) dispatches a command, here's what happens, in order:

```
env.dispatch(Command(name="blog.post.publish", model_id=post_uuid))
    │
    ├── 1. CommandBus resolves the handler (model + method) from the registry
    │
    ├── 2. cmd._env = env                  (so cmd.record can browse)
    ├── 3. cmd.record loaded               (live record or __new__ for create)
    │
    ├── 4. PRE-HOOKS fire (model-specific then wildcard)
    │       └─ raising any exception aborts the rest
    │
    ├── 5. HANDLER runs
    │       └─ may write(), dispatch(), emit() inside the same transaction
    │
    ├── 6. POST-HOOKS fire
    │
    ├── 7. Transaction commits
    │
    ├── 8. Emitted events flush to the EventQueue
    │       └─ web.client.command.done auto-emitted for domain commands
    │
    └── 9. Return value flows back to dispatcher
```

The worker process drains the EventQueue separately, invoking every matching `@api.on_event` handler in its own fresh `Env` clone.

## 6. Wire the button to the command

You already declared the form-view button in [Chapter 3](03-views-and-web-client.md):

```xml
<button name="blog.post.publish" string="Publish" type="object"
        attrs='{"invisible": [["status", "=", "published"]]}'/>
```

`name="blog.post.publish"` matches the command name on the `@api.on_command` decorator. `type="object"` tells the client to dispatch this against the currently loaded record. The button hides when `status == "published"` so users can't double-publish.

Restart the server, refresh the form, and click **Publish**. The pre-hook validates the title, the handler flips the status, the event handler runs in the worker, and the client receives a `web.client.command.done` push that reloads the form to reflect the new state.

## 7. Dispatching from Python (controllers, scripts, tests)

You'll occasionally need to dispatch outside the web flow — from an HTTP controller, a background job, or a test:

```python
from ede.core.types import Command

result = env.dispatch(Command(
    name="blog.post.publish",
    model_key="blog.post",
    model_id=post.id,            # the record this command targets
    payload={},                  # required, even when empty
))
```

`Command.payload` is **always required** — pass `payload={}` when no payload is needed. `model_id` is the `record_uuid` of the target.

For generic CRUD from Python, prefer the ORM facade — it's the same dispatch under the hood but reads more clearly:

```python
post = env.models["blog.post"].create({"title": "Hello", "slug": "hello", "author_id": user_id})
post.write({"view_count": 1})
post.delete()
```

## 8. Transactions — the `env.transaction()` context manager

When you need a sequence of writes to commit-or-fail together, wrap them in a transaction:

```python
with env.transaction():
    post = env.models["blog.post"].create({...})
    env.dispatch(Command(name="blog.post.publish", model_id=post.id, payload={}))
```

Inside the context manager, every command's auto-commit is suppressed and the whole sequence commits on exit (or rolls back if any exception escapes). Hooks fire normally; events are emitted on the queue but only flush after the outer commit succeeds.

---

## What you just did

-   Added a custom command `blog.post.publish` to the `blog.post` model.
-   Gated it with a **pre-hook** that rejects empty-title posts.
-   Made the form **button** in Chapter 3's view dispatch the command on click.
-   Wrote an **event handler** that runs in the worker after publish succeeds.
-   Learned the full pipeline: dispatch → pre-hook → handler → post-hook → commit → event flush.

[:octicons-arrow-right-24: **Next — Relationships**](05-relationships.md)
