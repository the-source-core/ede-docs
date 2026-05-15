# Communication (Chatter)

A per-record message thread, activity timeline, and mention system that you embed on any model with a single mixin. Users post updates, schedule follow-ups, mention each other, and see the full audit history right next to the record.

```python
from ede.core.kernel.model import DomainModel
from ede.foundation.communication.mixins import Chatterable


@api.model("blog.post")
class BlogPost(DomainModel, Chatterable):
    title = fields.Char(required=True)
    body = fields.Text()
```

Inheriting `Chatterable` is the entire integration. The form view DSL renders the chatter panel automatically:

```xml
<chatter/>
```

---

## What you get

-   **`Chatterable` mixin** — opt-in on any `DomainModel`.
-   **`communication.message`** — one immutable message in a record's thread.
-   **`communication.activity`** — scheduled / completed activities on a record.
-   **`CommunicationService`** — 11 methods, all routed through `Command`: `post()`, `log_note()`, `schedule_activity()`, `complete_activity()`, `mention_user()`, etc.
-   **HTTP API** — `/api/chatter/*` (10 endpoints, RBAC-gated).
-   **7 React components** — `<ChatterPanel/>`, `<MessageList/>`, `<ActivityList/>`, `<MentionInput/>`, `<TimelineEntry/>` and friends.
-   **Activity → timeline auto-post** — completing an activity automatically posts to the message thread.
-   **Immutability** — messages cannot be edited after creation (lifecycle hook blocks `ede.update`).

## How to use it

### Post a message

```python
from ede.foundation.communication.services.communication_service import CommunicationService

svc = CommunicationService.from_env(env)
svc.post(
    record=post,
    body="Reviewed and ready to publish.",
    author_id=current_user.id,
)
```

### Schedule an activity

```python
svc.schedule_activity(
    record=post,
    activity_type="review",
    due_date=date.today() + timedelta(days=2),
    assignee_id=editor.id,
    note="Final copy pass before publish.",
)
```

### Complete an activity

```python
activity = post.activity_ids.filtered(lambda a: a.activity_type == "review")
svc.complete_activity(activity, note="Approved")
# → posts "Marked review as Done — Approved" to the chatter
```

### Read the thread

```python
messages = post.message_ids
# → ordered descending by created_at_utc
```

### Render in a form view

```xml
<form>
    <sheet>
        <field name="title"/>
        <field name="body"/>
    </sheet>
    <chatter/>
</form>
```

The `<chatter/>` element pulls the latest 50 messages, the open activities, and the mention input — all from the React web client.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `CHATTER_PAGE_SIZE` | `50` | Messages per page in the panel. |
| `CHATTER_ATTACHMENTS_ENABLED` | `True` | Whether the chatter input accepts file uploads (uses [Storage](storage.md)). |

## How it composes with other features

-   **[Storage](storage.md)** — attachments on messages.
-   **[Notifications](notifications.md)** — mentions push to the mentioned user's inbox.
-   **[Approval Workflows](approval.md)** — every approval transition posts a chatter line.

## Reference

| Concept | Where it lives |
|---|---|
| `Chatterable` mixin | `src/ede/foundation/communication/mixins.py` |
| `CommunicationService` | `src/ede/foundation/communication/services/communication_service.py` |
| HTTP API | `src/ede/foundation/communication/api/` |
| React components | `src/frontend/src/features/chatter/` |
