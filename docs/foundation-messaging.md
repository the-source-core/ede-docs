<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Messaging Channel Engine — Implementation Docs

**Module:** `foundation.messaging` (`src/ede/foundation/messaging/`)
**Roadmap:** [roadmap/foundation/messaging/](../roadmap/foundation/messaging/README.md)
**Status:** 🟡 In Progress — Phase 1 ✅ Delivered 2026-07-15. Phases 2–3 planned.
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
| `messaging.channel` | One configured channel instance binding an `ir.connector` to an `res.organization` (channel kind Enum, display handle, auto-create-partner policy, per-channel `outbound_max_attempts`, webhook secret) | `src/ede/foundation/messaging/models/channel.py` |
| `messaging.thread` | One conversation with one external party on one channel; polymorphic `(related_model, related_id)` link to the consumer's record once promoted | `src/ede/foundation/messaging/models/thread.py` |
| `messaging.message` | One atomic inbound or outbound message with direction, body, media M2M, delivery status, provenance to chatter | `src/ede/foundation/messaging/models/message.py` |
| `messaging.identity` | `(channel_kind, external_id) → res.partner` resolver; auto-created subject to the channel's policy | `src/ede/foundation/messaging/models/identity.py` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `MessagingProvider` (abstract) | Contract every channel-kind plug-in implements: `verify_webhook`, `parse_inbound`, `send`, `download_media` | `src/ede/foundation/messaging/connectors/base.py` |
| `TelegramBotProvider` | First concrete provider; per-channel URL secret; `sendMessage` / `sendPhoto` / `sendDocument`; registered as `telegram_bot` | `src/ede/foundation/messaging/connectors/telegram.py` |
| `MessagingService` | Orchestrator: `handle_inbound`, `send`, `resolve_partner`, `link_thread_to_record`, `list_threads_for_partner` | `src/ede/foundation/messaging/services/messaging_service.py` |
| `IdentityResolver` | `(channel_kind, external_id) → res.partner` lookup + auto-create per channel policy | `src/ede/foundation/messaging/services/identity_resolver.py` |
| `MessagingRouter` | Resolves an `ir.connector` → live `MessagingProvider` (mirrors `EmailRouter`) | `src/ede/foundation/messaging/services/messaging_router.py` |
| Reverse chatter bridge | Relays a chatter reply on a linked record to the external party (loop-safe) | `src/ede/foundation/messaging/services/chatter_bridge.py` |
| Outbound retry sweep | `@api.scheduled_job` re-dispatching `pending` outbound messages; per-channel ceiling fails exhausted ones | `src/ede/foundation/messaging/services/retry_worker.py` |
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
| `messaging.outbound_failed` | When an outbound message fails permanently (provider 4xx) or exhausts the channel's `outbound_max_attempts` | Ops alerting; custom escalation handlers |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/messaging/webhook/{channel_uuid}/{secret}` | Public inbound webhook receiver; per-channel URL-secret + provider signature authenticated | `MessagingWebhookController` |
| `POST /api/messaging/thread/{thread_id}/send` | Internal user sends an outbound reply on a thread | `MessagingController` |
| `POST /api/messaging/thread/{thread_id}/link` | Link a thread to a business record | `MessagingController` |
| `GET /api/messaging/partner/{partner_id}/threads` | List a partner's threads | `MessagingController` |
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
   └────► failed  (provider 4xx, or after the channel's outbound_max_attempts)
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
_None._ The module deliberately adds **no** keys to `settings.py` — only genuine
deployment infra belongs there, and messaging has none. The webhook URL is
derived from the controller path (`/api/messaging/webhook/{channel_uuid}/{secret}`)
plus the request origin and shown for copy-paste on the channel form; every
operational knob is a per-channel field the admin configures.
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
_None._ Policy is per-channel, not global: `messaging.channel.auto_create_partner`
(`off` / `on` / `prompt_internal`, default `on`) and `messaging.channel.outbound_max_attempts`
(default `5`) are configured on each channel record.
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
_None._ Configuration lives on the `messaging.channel` form (Settings → Messaging → Channels).
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `data/ir.rbac.permission.csv` | 7 permissions: `messaging.channel` read/create/update/delete, `messaging.thread.read`, `messaging.message.read`, `messaging.message.execute` (send) |
| `data/messaging_menus.xml` | Settings → Messaging → Channels leaf + Threads ops-debug leaf |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Telegram Bidirectional + Chatter Bridge | ✅ Delivered 2026-07-15 | [phase-1-implementation.md](../roadmap/foundation/messaging/phase-1-implementation.md) |
| Phase 2 (planned) | WhatsApp Cloud + Templates + Media | 🔴 Not Started | (drafted in README) |
| Phase 3 (planned) | More Channels + Operator Console | 🔴 Not Started | (drafted in README) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Provider-agnostic engine + Telegram connector | `messaging.channel`, `messaging.thread`, `messaging.message`, `messaging.identity` | `connectors/telegram.py`, `services/messaging_service.py` | [phase-1-implementation.md](../roadmap/foundation/messaging/phase-1-implementation.md) |
| Inbound webhook → identity → thread → chatter mirror + `messaging.inbound_received` | `messaging.*` | `api/webhook_controller.py`, `services/identity_resolver.py` | phase-1 |
| Outbound `messaging.send` + per-channel retry ceiling + reverse chatter bridge | `messaging.message` | `services/messaging_service.py`, `services/retry_worker.py`, `services/chatter_bridge.py` | phase-1 |
| `StubMessagingProvider` for consumer tests | — | `src/tests/foundation/messaging/helpers/stub_provider.py` | phase-1 |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Outbound media egress to the provider deferred (inbound media fully lands as `storage.document`; outbound message rows record media but Phase 1 transmits text) | 🟡 (Phase 1 seam) | [phase-1-implementation.md](../roadmap/foundation/messaging/phase-1-implementation.md) |
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
| `internal_user` | `messaging.channel.read`, `messaging.thread.read`, `messaging.message.read`, `messaging.message.execute` (send — chatter composer dispatches this) |
| `system_admin` | `messaging.channel` create / update / delete |
| `foundation.converse` system principal (Phase 2 of converse) | `messaging.message.execute` (send) |
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

*Last sync: 2026-07-15. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
