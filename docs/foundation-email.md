<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Email — Implementation Docs

**Module:** `foundation.email` (`src/ede/foundation/email/`)
**Roadmap:** [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md)
**Status:** ✅ Delivered (baseline — pre-roadmap)
**Layer:** Foundation engine — platform substrate

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A platform-level outbound email substrate. It owns templating, the durable queue, and the transport plug-board — and nothing else. Inbound mail, reply threading, marketing campaigns, and chat-style messaging are explicitly out of scope. The module exposes two models (`mail.outbox`, `mail.template`) plus a service layer; transports are registered as `ir.connector` kinds (category `email`) via [`foundation.connectors`](../roadmap/foundation/connectors/README.md) and chosen per-organisation by configuration rather than code.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every domain that sends transactional email needs the same four things: a Jinja2 template engine, a durable outbox queue with retry semantics, pluggable transports configured by an admin rather than coded, and a worker that drains the queue. Building this once in foundation means every consumer — notifications, communication (Phase 2 outbound bridge), approval, password reset, future domain consumers — gets identical plumbing.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX entry points** — "Email" application appears in the app-switcher with two menus: **Outbox** (`mail.outbox` list/form) and **Email Templates** (`mail.template` list/form).
- **Programmatic entry points for other modules** — render a template + queue a send through `EmailRouter`; dispatch `Command("mail.outbox.queue", payload={...})` / `mail.outbox.send_now` / `mail.outbox.cancel` / `mail.outbox.retry`; subscribe to `ede.record.created` on `mail.outbox` if a downstream needs to react to outbound mail.
- **REST endpoints** under `/api/email/*` — `/send`, `/queue`, `/outbox` list, per-row send-now / cancel / retry / delete, `/process-queue`, `/templates` CRUD.
- **Integration boundary** — produces queued outbound mail and successful-delivery records; consumes transport implementations via the `ir.connector` (category `email`) contract from `foundation.connectors`. Has no awareness of any domain model — every consumer talks to `EmailRouter` or dispatches outbox commands without `foundation.email` knowing who they are.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Caller module]                          foundation.email
  notifications / communication /          ┌──────────────────────────┐
  approval / password-reset / ...          │  EmailRouter             │
                                  ─render→ │  ─ render template       │
                                  ─queue→  │  ─ resolve connector      │
                                           │  ─ send / requeue / fail  │
                                           │  ─ process_queue (worker) │
                                           └────────────┬─────────────┘
                                                        │
                              ┌─────────────────────────┼─────────────────────────┐
                              ▼                                                   ▼
                  ┌──────────────────────┐                          ┌──────────────────────┐
                  │  mail.template       │                          │  mail.outbox         │
                  │  (Jinja2 body/subj   │                          │  (durable queue, FSM │
                  │   with named params) │                          │   draft→queued→...)  │
                  └──────────────────────┘                          └──────────┬───────────┘
                                                                               │
                                                                               ▼
                                                          ┌──────────────────────────────┐
                                                          │ foundation.connectors        │
                                                          │  ir.connector (category=email)│
                                                          │  Gmail OAuth2 today;          │
                                                          │  SMTP / SendGrid / Mailgun    │
                                                          │  plug in via same contract    │
                                                          └──────────────────────────────┘
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `mail.template` | Jinja2 body / subject template with named parameters, rendered per send. | [models/template.py](../src/ede/foundation/email/models/template.py) |
| `mail.outbox` | Durable queue row — subject, addresses, body, FSM state, retry count, scheduling timestamps, last-error capture, optional pinned `ir.connector`. | [models/outbox.py](../src/ede/foundation/email/models/outbox.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `EmailRouter` | Resolves the connector for an outbox row, delivers the message via the connector's provider, drains the queue in batches, applies retry semantics. | [services/email_router.py](../src/ede/foundation/email/services/email_router.py) |
| Template renderer | Renders `mail.template` body/subject with Jinja2 against a context payload. | [services/template_renderer.py](../src/ede/foundation/email/services/template_renderer.py) |
| Gmail connector | Concrete `ir.connector` kind shipping OAuth2-based Gmail API v1 delivery. | [connectors/gmail.py](../src/ede/foundation/email/connectors/gmail.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `mail.outbox.queue` | Caller / `/api/email/queue` | Validates required fields, transitions `draft → queued`, optionally pins a `scheduled_at`. |
| `mail.outbox.send_now` | Caller / `/api/email/outbox/{id}/send-now` | Bypasses the queue: validates, resolves the connector, delivers immediately, writes resulting state (`sent` / `failed`). |
| `mail.outbox.cancel` | Caller / `/api/email/outbox/{id}/cancel` | Transitions a non-terminal row to `cancelled`. |
| `mail.outbox.retry` | Caller / `/api/email/outbox/{id}/retry` | Resets a `failed` row back to `queued` for another attempt. |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.record.created` on `mail.outbox` | When a new outbox row is created via `ede.create`. | Any module that wants to react to outbound mail being queued. |
| `ede.record.updated` on `mail.outbox` | On state transitions (queued → sending → sent / failed / cancelled). | Auditors, downstream notification trackers. |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/email/send` | One-shot send (build + queue + send-now in one call). | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `POST /api/email/queue` | Queue a prepared outbox row for the worker to drain. | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `GET /api/email/outbox` | List outbox rows (with filter / pagination). | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `GET /api/email/outbox/{id}` | Fetch one outbox row. | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `POST /api/email/outbox/{id}/send-now` | Trigger immediate delivery. | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `POST /api/email/outbox/{id}/cancel` | Cancel a queued row. | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `POST /api/email/outbox/{id}/retry` | Retry a failed row. | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `DELETE /api/email/outbox/{id}` | Delete an outbox row. | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `POST /api/email/process-queue` | Manually trigger one drain cycle. | [api/email.py](../src/ede/foundation/email/api/email.py) |
| `GET/POST/PUT/DELETE /api/email/templates[/{id}]` | Template CRUD. | [api/email.py](../src/ede/foundation/email/api/email.py) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.mail.outbox.delete` | Guard hook on outbox deletion — enforces invariants before a row is removed. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
`mail.outbox.state`:

```
draft ──queue──> queued ──worker──> sending ──delivery──> sent
  │                │                    │                   │
  │                │                    └──delivery-fail──> failed ──retry──> queued
  │                │
  └──cancel────────┴────────────────────────────────────────> cancelled
```

A `failed` row is permanently failed once `retry_count >= EMAIL_SEND_RETRY_MAX`; until then, manual retry returns it to `queued`.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `email`
- `ACTIVE_DOMAINS` entry: _not applicable — foundation app_
- Manifest `depends`: `foundation.base`, `foundation.connectors`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `EMAIL_QUEUE_BATCH_SIZE` | `int` | `100` | `EDE_EMAIL_QUEUE_BATCH_SIZE` | Max outbox rows the worker pulls per drain cycle. |
| `EMAIL_SEND_RETRY_MAX` | `int` | `3` | `EDE_EMAIL_SEND_RETRY_MAX` | Maximum send attempts before an outbox row is marked permanently failed. |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| [data/ir.rbac.permission.csv](../src/ede/foundation/email/data/ir.rbac.permission.csv) | RBAC permissions for `mail.outbox` and `mail.template`. |
| [data/email_menus.xml](../src/ede/foundation/email/data/email_menus.xml) | Email application root + Outbox / Templates menus, with backing `ir.action` records. |

_No default mail templates ship out of the box — consumers seed their own (password-reset, notification digest, etc.)._
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Baseline | `mail.outbox` + `mail.template` + `EmailRouter` + connector plug-board + REST + UI + RBAC seed | ✅ Delivered (pre-roadmap) | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Jinja2 template rendering | `mail.template` | [models/template.py](../src/ede/foundation/email/models/template.py), [services/template_renderer.py](../src/ede/foundation/email/services/template_renderer.py) | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) |
| Durable outbox queue | `mail.outbox` | [models/outbox.py](../src/ede/foundation/email/models/outbox.py) | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) |
| Queue drain / send worker | `mail.outbox` | [services/email_router.py](../src/ede/foundation/email/services/email_router.py) (`EmailRouter.process_queue`) | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) |
| Transport plug-board (Gmail OAuth2 today; SMTP / SendGrid / Mailgun via same contract) | `ir.connector` (from `foundation.connectors`) | [connectors/gmail.py](../src/ede/foundation/email/connectors/gmail.py) | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) |
| REST API surface | both | [api/email.py](../src/ede/foundation/email/api/email.py) | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) |
| Admin UI (app-switcher app, Outbox + Templates menus) | both | [views/mail_outbox_views.xml](../src/ede/foundation/email/views/mail_outbox_views.xml), [views/mail_template_views.xml](../src/ede/foundation/email/views/mail_template_views.xml), [data/email_menus.xml](../src/ede/foundation/email/data/email_menus.xml) | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| No inbound mail routing / reply threading (no `Message-ID` / `In-Reply-To` parsing). | 🟡 Medium | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) — Known Gaps #1 |
| No bounce + complaint webhook handling. | 🟡 Medium | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) — Known Gaps #2 |
| No platform-level suppression list. | 🟡 Medium | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) — Known Gaps #3 |
| No self-managed DMARC / DKIM / SPF tooling — done at transport, not platform. | 🟢 Low | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) — Known Gaps #4 |
| No preview render for templates (no dry-run against a sample payload). | 🟢 Low | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) — Known Gaps #5 |
| No A/B subject-line testing. | 🟢 Low | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) — Known Gaps #6 |
| Transports beyond Gmail (SMTP / SendGrid / Mailgun) not yet implemented — plug-board is generic. | 🟡 Medium | [roadmap/foundation/email/README.md](../roadmap/foundation/email/README.md) — Known Gaps #7 |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _populated as integration learnings emerge_

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Baseline shipped pre-roadmap; no upgrade-time backfills required for the initial `mail.outbox` + `mail.template` tables.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Seeded via [`data/ir.rbac.permission.csv`](../src/ede/foundation/email/data/ir.rbac.permission.csv) | CRUD permissions on `mail.outbox` and `mail.template` (see seed file for exact role bindings). |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Connectors](../roadmap/foundation/connectors/README.md) — transports plug here via the `ir.connector` kind contract.
- [Foundation Notifications](foundation-notifications.md) — chief consumer; queues per-recipient mails through this module.
- [Foundation Communication](foundation-communication.md) — Phase 2 outbound email bridge writes `mail.outbox` rows for `email`-typed timeline messages.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-14. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
