# Notifications

Push a notification to one user, a role, or a custom audience. Each user controls their own opt-ins per channel; the engine fans out only to the channels they accept.

```python
env.dispatch(Command("notification.send", payload={
    "to_user_id": user.id,
    "subject": "Your PO is approved",
    "body": "PO-2026-0042 was approved by Finance.",
    "link": f"/wc#/action/purchase.order/{po.id}",
}))
```

Inbox + browser push delivery happen immediately for online users; offline users see it on next login.

---

## What you get

-   **`ir.notification`** — persisted notification rows per recipient.
-   **`ir.notification.setting`** — per-user per-channel opt-in / opt-out.
-   **Channels** — `inbox` (always on), `web_push` (browser SSE), `email` (via [Email](email.md)).
-   **Audience helpers** — `to_user_id`, `to_role_id`, `to_organization_id`, `to_followers_of(model, record_uuid)`.
-   **`notification.send`** command — primary entrypoint.
-   **Inbox UI** — bell icon in the web client header; per-notification read / unread / link-out.

## How to use it

### Send to one user

```python
env.dispatch(Command("notification.send", payload={
    "to_user_id": user.id,
    "subject": "Welcome to Acme",
    "body": "Your account is ready.",
}))
```

### Send to everyone with a role

```python
env.dispatch(Command("notification.send", payload={
    "to_role_id": finance_manager_role.id,
    "subject": "Month-end close starts tomorrow",
    "body": "Lock invoicing by EOD.",
}))
```

### Listen to record events and notify followers

```python
@api.on_event("blog.post.published")
def notify_followers(event, env):
    env.dispatch(Command("notification.send", payload={
        "to_followers_of": ("blog.post", event.payload["record_uuid"]),
        "subject": "New post published",
        "body": event.payload["title"],
    }))
```

### Let users tune their channels

The Settings → Notifications page exposes a matrix: rows = notification kinds, columns = channels. Defaults are inherited from the role; users override per-row.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `NOTIFICATION_WEB_PUSH_TIMEOUT_SECONDS` | `60` | SSE keepalive interval. |
| `NOTIFICATION_EMAIL_DIGEST_HOUR` | `8` | Hour of day (local org timezone) to send digest emails. |

## Reference

-   Source: `src/ede/foundation/notifications/`
-   Related: [Email](email.md), [Communication (Chatter)](communication.md) (auto-posts notifications on activity).
