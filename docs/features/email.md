# Email — Outbound Queue

Queue-based outbound email with retry, templates, and a connector-driven transport plug-board. Your application code creates a queued message; the router resolves the organization's active email connector and delivers it.

```python
outbox = env.models["mail.outbox"].create({
    "to_addresses": ["customer@example.com"],
    "subject": "Welcome to Acme",
    "body_html": "<p>Hi there — thanks for signing up.</p>",
    "organization_id": env.active_organization_id,
})
env.dispatch(Command("mail.outbox.queue", payload={}, record=outbox))
```

The message lands in the queue; `EmailRouter` drains it on the next worker tick, picks the org's default email connector, and delivers — retrying on transient failures.

---

## What you get

-   **`mail.outbox`** — outbound message queue. State enum: `draft`, `queued`, `sending`, `sent`, `failed`, `cancelled`. Carries `to_addresses` / `cc_addresses` / `bcc_addresses` (JSON arrays), `subject`, `body_html`, `body_text`, `organization_id`, `connector_id` (optional pin), `error_message`, `retry_count`, `provider_message_id`.
-   **`mail.template`** — reusable templates (`subject`, `body_html`, `body_text`) with Jinja2 variable interpolation.
-   **`TemplateRenderer`** — `render(template_record, context)` produces the rendered subject + bodies from a template and a context dict.
-   **`EmailRouter`** — `process_queue(env)` drains the queue; `send(message, org_id, env)` delivers one message; resolves the transport via the [Connectors](connectors.md) framework (pinned `connector_id`, else the org's default `email` connector).
-   **Gmail OAuth2 transport** — the shipped email connector (`provider_type="gmail"`). The transport is a connector, so additional providers register the same way.
-   **Outbox commands** — `mail.outbox.queue`, `mail.outbox.send_now`, `mail.outbox.cancel`, `mail.outbox.retry`.
-   **HTTP API** — `/api/email/send`, `/queue`, `/outbox`, `/outbox/{id}/send-now`, `/cancel`, `/retry`, `/process-queue`, and template CRUD + `/preview`.

## How to use it

### Send a one-off message

```python
outbox = env.models["mail.outbox"].create({
    "to_addresses": ["alice@example.com"],
    "subject": "Your invoice is ready",
    "body_html": "<p>Invoice #INV-0001 attached.</p>",
    "organization_id": env.active_organization_id,
})
env.dispatch(Command("mail.outbox.queue", payload={}, record=outbox))
```

`organization_id` is what lets the router resolve which transport to use. To deliver immediately instead of waiting for the worker, dispatch `mail.outbox.send_now`.

### Render and send a template

Templates store the content; `TemplateRenderer` interpolates a context, and you queue the rendered result:

```xml
<record id="tpl_welcome" model="mail.template">
    <field name="name">Welcome Email</field>
    <field name="subject">Welcome to {{ org.name }}</field>
    <field name="body_html"><![CDATA[<p>Hi {{ partner.name }}, your account is ready.</p>]]></field>
</record>
```

```python
from ede.foundation.email.services.template_renderer import TemplateRenderer

tpl = env.models["mail.template"].browse(tpl_uuid).read()[0]
rendered = TemplateRenderer.render(tpl, {"partner": partner, "org": env.active_organization})

outbox = env.models["mail.outbox"].create({
    "to_addresses": [partner.email],
    "subject": rendered["subject"],
    "body_html": rendered["body_html"],
    "organization_id": env.active_organization_id,
})
env.dispatch(Command("mail.outbox.queue", payload={}, record=outbox))
```

### Configure the transport

Email transports are connectors with `category="email"`. Create a Gmail connector and mark it the org default:

```xml
<record id="conn_gmail" model="ir.connector">
    <field name="name">Acme Gmail</field>
    <field name="category">email</field>
    <field name="provider_type">gmail</field>
    <field name="organization_id" ref="res.organization.acme"/>
    <field name="is_default">true</field>
    <field name="config" eval="{
        'client_id': '$ENV:GMAIL_CLIENT_ID',
        'client_secret': '$ENV:GMAIL_CLIENT_SECRET',
        'refresh_token': '$ENV:GMAIL_REFRESH_TOKEN',
        'sender_email': 'noreply@acme.example',
    }"/>
</record>
```

With `is_default=True`, every email this organization sends goes through this connector unless a message pins a different one via `connector_id`.

### Inspect failures

```python
failed = env.models["mail.outbox"].search([
    ("state", "=", "failed"),
    ("organization_id", "=", env.active_organization_id),
])
for msg in failed:
    print(msg.error_message, msg.retry_count)
```

The "Email" app in the web client also surfaces a failed-messages view.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `EMAIL_QUEUE_BATCH_SIZE` | `100` | Messages drained per `process_queue` call. |
| `EMAIL_SEND_RETRY_MAX` | `3` | Send attempts before the state flips to `failed`. |

## How it composes with other features

-   **[Connectors](connectors.md)** — email transports register here as `category="email"` connectors.
-   **[Notifications](notifications.md)** — the email channel routes through this engine.

## Reference

| Concept | Where it lives |
|---|---|
| `mail.outbox`, `mail.template` | `src/ede/foundation/email/models/` |
| `EmailRouter` | `src/ede/foundation/email/services/email_router.py` |
| `TemplateRenderer` | `src/ede/foundation/email/services/template_renderer.py` |
| Gmail transport | `src/ede/foundation/email/connectors/gmail.py` |
| HTTP API | `src/ede/foundation/email/api/email.py` (prefix `/api/email`) |
