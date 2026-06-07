# Communication (Chatter)

Embed a per-record message thread, activity timeline, follower list, and attachments on any record with a single mixin and one DSL block. Users post updates, schedule follow-ups, mention each other, and see the audit history right next to the record.

```python
from ede.core import api, fields
from ede.core.kernel.model import DomainModel
from ede.foundation.communication.models.chatterable import Chatterable


@api.model("blog.post")
class Post(DomainModel, Chatterable):
    title = fields.Char(required=True)
    body = fields.Text()
```

Inheriting `Chatterable` is the entire backend integration. Add an `<activity>` block to the form view and the React web client renders the chatter panel automatically:

```xml
<FormView model="blog.post">
    <sheet>
        <field name="title"/>
        <field name="body"/>
    </sheet>
    <activity>
        <MessageConversation/>
        <Followers/>
        <ScheduledActivity/>
        <Attachments/>
    </activity>
</FormView>
```

---

## What you get

-   **`Chatterable` mixin** — opt-in on any `DomainModel`. Adds message, follower, and activity relations.
-   **`communication.message`** — one immutable message entry on a record's thread.
-   **`communication.follower`** — subscription rows for users who want updates on a record.
-   **`activity.log`** — a scheduled or completed activity (task, call, review) on a record.
-   **`activity.type`** — admin-managed catalog of activity kinds and default delays.
-   **`CommunicationService`** — six routed methods: `post_message`, `post_note`, `post_notification`, `schedule_activity`, `complete_activity`, `subscribe`.
-   **`<activity>` DSL block** — render any combination of `<MessageConversation/>`, `<Followers/>`, `<ScheduledActivity/>`, `<Attachments/>` inside a FormView.
-   **HTTP API** — 11 endpoints under `/api/chatter/{model}/{id}/...` for thread reads, post / note / email, activities, and followers. All RBAC-gated.
-   **Auto-follow on ownership change** — toggle `COMMUNICATION_AUTO_FOLLOW_OWNER` to make assignment automatically subscribe the owner.
-   **Mention parser** — `@username` in a message body resolves to a `res.user`, fires the notifications bridge, and links the user as a follower.
-   **Email bridge** — incoming and outgoing email is normalised into the same `communication.message` stream.

## How to use it

### Make a model chatter-enabled

```python
from ede.foundation.communication.models.chatterable import Chatterable

@api.model("blog.post")
class Post(DomainModel, Chatterable):
    title = fields.Char(required=True)
    body = fields.Text()
```

Generate and apply a migration — the mixin pulls in the relations the chatter needs.

### Post a message

```python
from ede.foundation.communication.services.communication_service import (
    CommunicationService,
)

svc = CommunicationService.from_env(env)
svc.post_message(
    related_model="blog.post",
    related_id=post.record_uuid,
    body="Reviewed and ready to publish.",
    author_id=current_user.record_uuid,
)
```

Use `post_note` for an internal-only note (filtered out of email bridges) and `post_notification` for a system-generated event message (typically posted by a workflow or hook, not by a user).

### Schedule an activity

```python
svc.schedule_activity(
    related_model="blog.post",
    related_id=post.record_uuid,
    activity_type_id=env.ref("communication.activity_type_review").record_uuid,
    due_date=date.today() + timedelta(days=2),
    assignee_id=editor.record_uuid,
    note="Final copy pass before publish.",
)
```

`activity.type` rows are seeded by `data/activity_type_seed.xml`; reference them by external ID with [`env.ref()`](customization.md).

### Complete an activity

```python
svc.complete_activity(
    activity_id=activity.record_uuid,
    summary="Approved",
)
# → posts an automated message to the thread; closes the activity row
```

### Subscribe a user as a follower

```python
svc.subscribe(
    related_model="blog.post",
    related_id=post.record_uuid,
    user_id=reviewer.record_uuid,
)
```

If `COMMUNICATION_AUTO_FOLLOW_OWNER=True`, ownership changes call `subscribe` automatically.

### Read the thread over HTTP

```http
GET /api/chatter/blog.post/{post_uuid}
Authorization: Bearer <token>
```

Returns messages, open activities, and the follower list in one payload. The 11-endpoint surface covers reads, writes, edits, and deletes for messages, notes, email, activities, and followers — see `src/ede/foundation/communication/api/chatter_controller.py` for the full route list.

## The four `<activity>` components

`<activity>` accepts exactly these children (in any order; the parser rejects unknown tags at load time):

| Component | What it renders |
|---|---|
| `<MessageConversation/>` | Threaded message timeline with mention input. |
| `<Followers/>` | Follower list with add/remove controls. |
| `<ScheduledActivity/>` | Open and completed activities with deadline indicators. |
| `<Attachments/>` | Files uploaded to the record (uses [Storage](storage.md)). |

Use any subset — a read-only record might only show `<MessageConversation/>` and `<Attachments/>`, for example.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `COMMUNICATION_AUTO_FOLLOW_OWNER` | `True` | Subscribe the owner automatically when an ownership field changes. |
| `COMMUNICATION_MENTION_NOTIFY` | `True` | Fire a notification (via the notifications bridge) when a user is `@mentioned`. |

## How it composes with other features

-   [Storage](storage.md) — `<Attachments/>` uploads route through the storage engine.
-   [Notifications](notifications.md) — mentions and new-message events publish to the notifications bridge.
-   [Email](email.md) — the email bridge normalises inbound and outbound mail into `communication.message` rows.
-   [Permissions](security.md) — the 11 HTTP routes are RBAC-gated; record rules on the parent model decide who can read a thread.

## Reference

-   Mixin: `src/ede/foundation/communication/models/chatterable.py`
-   Models: `src/ede/foundation/communication/models/{message,follower,activity,activity_type}.py`
-   Service: `src/ede/foundation/communication/services/communication_service.py`
-   Bridges & helpers: `src/ede/foundation/communication/services/{notifications_bridge,email_bridge,auto_follow,mention_parser}.py`
-   HTTP controller: `src/ede/foundation/communication/api/chatter_controller.py`
-   DSL parser (`<activity>` element): `src/ede/core/services/presentation/dsl/parser.py`
-   React components: planned in `src/frontend/src/managers/FormAside.tsx` (chatter / activity / followers / files tabs — tracked under [Communication Enhancement 01](../../roadmap/foundation/communication/enhancements/01-form-chatter-frontend2.md) — 🔴 post-legacy-cutover)
-   Activity-type seed: `src/ede/foundation/communication/data/activity_type_seed.xml`
