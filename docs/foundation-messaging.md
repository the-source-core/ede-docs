<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Messaging Channel Engine — Implementation Docs

**Module:** `foundation.messaging` (`src/ede/foundation/messaging/`)
**Roadmap:** [roadmap/foundation/messaging/](../roadmap/foundation/messaging/README.md)
**Status:** 🔴 Not Started — drafted 2026-05-26
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A provider-agnostic, connector-driven messaging substrate that owns the inbound webhook surface, outbound send queue, channel-aware threading model, and external-party identity resolution for every messaging app the platform integrates with. It does not know what Telegram, WhatsApp, or Messenger *are* — it only knows that a channel is one configured `ir.connector` + one `MessagingProvider` plug-in that speaks a normalised protocol. Mirrors `foundation.email` in shape (same connector plug-board + provider contract + outbox semantics) but for **stateful threaded conversations** rather than one-shot transactional mail.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every consumer that wants to "talk to a customer on WhatsApp" otherwise re-invents the same six concerns: webhook signature verification, payload normalisation, provider client + rate-limiting, external-user-to-`res.partner` mapping, outbound queue, and chatter mirror. Six consumers × the same six concerns = thirty-six ad-hoc implementations that drift in policy. The platform owns it once; consumers subscribe to one event (`messaging.inbound_received`) and dispatch one command (`messaging.send`) and get all six for free.

Email's `mail.outbox` shape was considered and rejected — messaging adds threading, session windows (WhatsApp 24h rule), channel-specific media APIs, `(channel_kind, external_id)` identity resolution, mandatory webhook ingress, and channel-specific receipt semantics. Bolting these onto email's transactional-send shape would corrupt both.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX entry points** — Settings → Integrations → Channels (admin creates / edits channels); Settings → Integrations → Threads (ops debug; read-only); inbound + outbound messages appear in the linked record's chatter (`res.partner`, `pricing.rate.request`, etc.) via the chatter mirror.
- **Programmatic entry points for other modules** —
  - Subscribe to `messaging.inbound_received` event via `@api.on_event` to react to inbound messages.
  - Dispatch `Command("messaging.send", payload={"thread_id": ..., "body": ..., "media_document_ids": [...]?})` to send outbound.
  - Call `MessagingService.link_thread_to_record(thread_uuid, related_model, related_id)` when the conversation becomes "about" a record.
  - Implement a new `MessagingProvider` subclass + register via `connector_registry` to add a new channel kind.
- **Integration boundary** — produces `messaging.inbound_received` events; consumes nothing from the consumer side (it owns the transport). Each inbound + outbound mirrors to chatter via `foundation.communication`.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[External party]              [Channel connector]            [Platform]
─────────────────             ──────────────────             ──────────
Telegram customer  ──webhook──►  TelegramProvider       ──►  messaging.inbound_received
                                  (verify + parse)                    │
                                                            ┌─────────┴─────────┐
                                                            ▼                   ▼
                                                  identity resolution     chatter mirror
                                                  (telegram_user_id →     (communication.
                                                   messaging.identity →   message row of
                                                   res.partner)           type="message")
                                                            │
                                                            ▼
                                                  consumer module
                                                  (foundation.converse,
                                                   or a custom handler)
                                                            │
                                                            ▼
                                              env.dispatch(Command(
                                                "messaging.send",
                                                payload={thread_id, body}))
                                                            │
                                                            ▼
                                                  TelegramProvider.send
                                                            │
                                                            ▼
                                                  outbound messaging.message
                                                  + chatter mirror
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `messaging.channel` | One configured channel instance binding an `ir.connector` to an `res.organization` (channel kind Enum, display handle, auto-create-partner policy, webhook secret) | `src/ede/foundation/messaging/models/channel.py` (planned) |
| `messaging.thread` | One conversation with one external party on one channel; polymorphic `(related_model, related_id)` link to the consumer's record once promoted | `src/ede/foundation/messaging/models/thread.py` (planned) |
| `messaging.message` | One atomic inbound or outbound message with direction, body, media M2M, delivery status, provenance to chatter | `src/ede/foundation/messaging/models/message.py` (planned) |
| `messaging.identity` | `(channel_kind, external_id) → res.partner` resolver; auto-created subject to the channel's policy | `src/ede/foundation/messaging/models/identity.py` (planned) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `MessagingProvider` (abstract) | Contract every channel-kind plug-in implements: `verify_webhook`, `parse_inbound`, `send`, `download_media` | `src/ede/foundation/messaging/connectors/base.py` (planned) |
| `TelegramProvider` | First concrete provider; HMAC via `X-Telegram-Bot-Api-Secret-Token`; `sendMessage` / `sendPhoto` / `sendDocument` | `src/ede/foundation/messaging/connectors/telegram.py` (planned) |
| `MessagingService` | Orchestrator: `handle_inbound`, `send`, `resolve_partner`, `link_thread_to_record`, `list_threads_for_partner` | `src/ede/foundation/messaging/services/messaging_service.py` (planned) |
| `IdentityResolver` | `(channel_kind, external_id) → res.partner` lookup + auto-create per channel policy | `src/ede/foundation/messaging/services/identity_resolver.py` (planned) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `messaging.send` | Dispatched by any consumer module (e.g. `foundation.converse`, or the chatter composer when an internal user replies on a `res.partner` chatter that has a linked messaging thread) | Inserts an outbound `messaging.message` row; resolves the provider via `channel.connector_id`; calls `provider.send`; updates delivery status; mirrors to chatter |
| `messaging.thread.link` | Consumer module declares "this conversation is now about record X" | Sets `related_model` + `related_id` on the thread; triggers retro-mirror of all prior messages to the newly linked record's chatter |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `messaging.inbound_received` | After an inbound message is parsed, identity-resolved, thread-upserted, and committed | `foundation.converse` (dialog orchestrator); custom routing handlers in consumer modules |
| `messaging.outbound_failed` | When an outbound message exhausts `MESSAGING_OUTBOUND_MAX_ATTEMPTS` with status still `pending` | Ops alerting; custom escalation handlers |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/messaging/webhook/{channel_uuid}/{secret}` | Inbound webhook receiver; HMAC + URL-secret authenticated | `MessagingWebhookController` (planned) |
| (admin endpoints reuse generic CRUD via the existing `/api/<model>/*` shape for `messaging.channel` / `messaging.thread` / `messaging.message`) | Channel CRUD + thread / message browsing | (planned) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.messaging.send` | Last chance to moderate / throttle / veto an outbound message (consumer modules may register PII redaction, rate-limiting, content-policy hooks here) |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
`messaging.message.delivery_status`:
```
pending  →  sent  →  delivered  →  read
   │
   └────► failed  (after MESSAGING_OUTBOUND_MAX_ATTEMPTS)
```
`messaging.thread.status`: `open` | `closed` (consumer-controlled).
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` (in `src/ede/foundation/settings.py`): add `"messaging"` after `"communication"`.
- `ACTIVE_DOMAINS`: n/a (foundation engine).
- Manifest `depends`: `["base", "connectors", "communication", "storage"]`.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `MESSAGING_ENABLED` | `bool` | `True` | `EDE_MESSAGING_ENABLED` | Hard kill-switch for the entire engine (webhook ingress + outbound dispatch) |
| `MESSAGING_WEBHOOK_BASE_URL` | `str` | `""` | `EDE_MESSAGING_WEBHOOK_BASE_URL` | Public-facing base URL the platform uses when registering its inbound webhook with a provider; empty in dev |
| `MESSAGING_OUTBOUND_MAX_ATTEMPTS` | `int` | `5` | `EDE_MESSAGING_OUTBOUND_MAX_ATTEMPTS` | Maximum send attempts before an outbound `messaging.message` is marked permanently failed |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `messaging.default_auto_create_partner_policy` | org | Enum (`off` / `on` / `prompt_internal`) | `prompt_internal` | Default value for `messaging.channel.auto_create_partner` when an admin creates a new channel |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| Settings → Integrations → Messaging | `src/ede/foundation/messaging/data/messaging_settings.xml` (planned) | `messaging.default_auto_create_partner_policy` |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `data/ir.rbac.permission.csv` (planned) | 5 permissions: `messaging.channel.read`, `messaging.channel.manage`, `messaging.thread.read`, `messaging.message.read`, `messaging.send` |
| `data/messaging_menus.xml` (planned) | Settings → Integrations → Channels menu leaf + Threads ops-debug leaf |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Telegram Bidirectional + Chatter Bridge | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/messaging/phase-1-implementation.md) |
| Phase 2 (planned) | WhatsApp Cloud + Templates + Media | 🔴 Not Started | (drafted in README) |
| Phase 3 (planned) | More Channels + Operator Console | 🔴 Not Started | (drafted in README) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Entire module is 🔴 Not Started — Phase 1 (Telegram) drafted but not shipped | 🔴 | [phase-1-implementation.md](../roadmap/foundation/messaging/phase-1-implementation.md) |
| WhatsApp Cloud + template-message library + 24h-window enforcement | 🔴 (Phase 2) | [README](../roadmap/foundation/messaging/README.md) |
| Messenger / SMS / Webchat providers | 🔴 (Phase 3) | [README](../roadmap/foundation/messaging/README.md) |
| Operator-facing live chat console | 🔴 (Phase 3) | [README](../roadmap/foundation/messaging/README.md) |
| Cross-channel identity merge (one partner across Telegram + WhatsApp + SMS) | 🔴 (Phase 3) | [README](../roadmap/foundation/messaging/README.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _Populated as the first paying-customer integration ships._

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1 ships a single Alembic revision creating the four `messaging.*` tables. No backfills.
- Adding `"messaging"` to `ACTIVE_MODULES` must come **after** `"communication"` so the chatter bridge resolves at boot.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `internal_user` | `messaging.thread.read`, `messaging.message.read`, `messaging.send` (chatter composer dispatches this) |
| `system_admin` | `messaging.channel.manage` |
| `foundation.converse` system principal (Phase 2 of converse) | `messaging.send` |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.converse`](foundation-converse.md) — first user-facing consumer (turns inbound messages into AI-driven dialogs)
- [`foundation.connectors`](foundation-connectors.md) — channel credentials plug here via the connector kind contract
- [`foundation.communication`](foundation-communication.md) — every inbound/outbound mirrors as a chatter row
- [`foundation.email`](foundation-email.md) — sibling transport substrate (different lifecycle, same connector pattern)
- [`foundation.storage`](foundation-storage.md) — inbound media land as `storage.document` rows
- Use case driver: [`uc.freight-quote-via-telegram`](../roadmap/usecases/freight-quote-via-telegram.md)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-26. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
