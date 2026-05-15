# 4. Commands & Events

*Tutorial chapter — coming soon.*

This chapter will cover writing a custom command (`@api.on_command("blog.post.publish")`), emitting an event after the command succeeds, handling it with `@api.on_event`, and using lifecycle hooks (`@api.on_hook("pre.blog.post.create")`) to enforce invariants.

In the meantime:

-   [Command & Event Bus](../04-command-event-bus.md) — `Command`, `CommandBus`, `Event`, `EventQueue`, retries, field change tracking.
-   [Command & Event Usage Guide](../15-command-event-guide.md) — when to use `on_command` vs. `on_event` vs. `track_fields` vs. `web.client.*` push events.
-   [Lifecycle Hooks](../14-lifecycle-hooks.md) — `pre.*` / `post.*` hooks, `cmd.record` semantics, field tracking hooks.

[:octicons-arrow-right-24: Continue — Relationships](05-relationships.md)
