# Email — Outbound Queue

Queue-based outbound email with retry, templates, and a transport plug-board. Switch between SMTP, SendGrid, Mailgun, and Gmail OAuth2 by configuring a connector — your application code stays the same.

```python
env.models["mail.outbox"].create({
    "to_emails": ["customer@example.com"],
    "subject": "Welcome to Acme",
    "body_html": "<p>Hi {{ partner.name }} — thanks for signing up.</p>",
    "template_id": welcome_template.id,
    "context": {"partner": partner},
})
```

The router picks up the message, renders the template against the context, picks the active transport via the [Connectors](connectors.md) framework, and delivers — with retry on transient failures.

---

## What you get

-   **`mail.outbox`** — outbound message queue with status (`queued`, `sending`, `sent`, `failed`).
-   **`mail.template`** — reusable templates with Jinja2-style variable interpolation.
-   **`EmailRouter`** — picks the active transport per organization and drains the queue.
-   **Transport contracts** — SMTP, SendGrid, Mailgun, Gmail OAuth2 are connector kinds.
-   **HTTP API** — `/api/email/*` for queue management.
-   **Email app** in the app-switcher — admin view of the queue, templates, and transports.
-   **Retry policy** — exponential backoff with configurable max attempts.

## How to use it

### Send a one-off message

```python
env.models["mail.outbox"].create({
    "to_emails": ["alice@example.com"],
    "subject": "Your invoice is ready",
    "body_html": "<p>Invoice #INV-0001 attached.</p>",
})
```

The record lands in `queued`; `EmailRouter` drains it on the next worker tick.

### Send via a template

Define the template (XML or Settings UI):

```xml
<record id="tpl_welcome" model="mail.template">
    <field name="name">Welcome Email</field>
    <field name="subject">Welcome to {{ org.name }}</field>
    <field name="body_html">
        <p>Hi {{ partner.name }},</p>
        <p>Thanks for joining {{ org.name }}. Your account is ready.</p>
    </field>
</record>
```

Send it:

```python
env.models["mail.outbox"].create({
    "to_emails": [partner.email],
    "template_id": env.models["mail.template"].browse("tpl_welcome").id,
    "context": {"partner": partner, "org": env.active_organization},
})
```

### Configure a transport

Create a connector of the right kind (one of `smtp`, `sendgrid`, `mailgun`, `gmail_oauth2`):

```xml
<record id="conn_sendgrid" model="ir.connector">
    <field name="name">Acme SendGrid</field>
    <field name="category">email</field>
    <field name="kind">sendgrid</field>
    <field name="organization_id" ref="res.organization.acme"/>
    <field name="is_default">true</field>
    <field name="config" eval="{'api_key': '$ENV:SENDGRID_API_KEY'}"/>
</record>
```

Mark it `is_default=True` and every email this organization sends goes through SendGrid.

### Inspect failures

```python
failed = env.models["mail.outbox"].search([
    ("status", "=", "failed"),
    ("organization_id", "=", env.active_organization_id),
])
for msg in failed:
    print(msg.last_error, msg.attempts)
```

The "Email" app in the web client also surfaces a failed-messages dashboard.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `EMAIL_QUEUE_BATCH_SIZE` | `25` | Messages drained per worker tick. |
| `EMAIL_MAX_ATTEMPTS` | `5` | Retries before status flips to `failed`. |
| `EMAIL_RETRY_BACKOFF_SECONDS` | `60` | Base backoff between retries (exponential). |

## How it composes with other features

-   **[Connectors](connectors.md)** — transports register here.
-   **[Notifications](notifications.md)** — channels can route to email via this engine.

## Reference

| Concept | Where it lives |
|---|---|
| `mail.outbox`, `mail.template` | `src/ede/foundation/email/models/` |
| `EmailRouter` | `src/ede/foundation/email/services/email_router.py` |
| Transport implementations | `src/ede/foundation/email/transports/` |
| HTTP API | `src/ede/foundation/email/api/` |
