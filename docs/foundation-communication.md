<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Communication Engine — Implementation Docs

**Module:** `foundation.communication` ([src/ede/foundation/communication/](../src/ede/foundation/communication/))
**Roadmap:** [roadmap/foundation/communication/README.md](../roadmap/foundation/communication/README.md)
**Status:** 🟡 In Progress — Phase 1 ✅ Delivered 2026-05-13 (engine + HTTP API + React chatter UI + `<activity>` DSL + `Chatterable` mixin + `res.partner` proof-of-life); Phases 2 & 3 🔴 Not Started. Several Phase-2 building blocks (notifications/email bridges, auto-follow, mention parser) already ship under `services/` but their full wiring + identity centralization (WS-0) lands in Phase 2.
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **polymorphic, record-level communication engine** that gives any EDE record three universal capabilities: a chronological **timeline** (messages, internal notes, outbound emails, completed activities, system notifications), an **activity scheduler** for planned interactions (call, meeting, task, document request) with due dates and outcomes, and a **follower list** of internal users + external contacts subscribed to per-channel updates. Domain modules attach by writing the polymorphic `related_model + related_id` pair on every message / activity / follower row — `foundation.communication` has no FK into any domain package and is consumed by them, never aware of them. Today the four models (`activity.type`, `activity.log`, `communication.message`, `communication.follower`) are declared and migrated; the service layer, HTTP API, React `<Chatter />` component, and bridges to `foundation.notifications` / `foundation.email` / `foundation.storage` are not yet built.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every domain record in the EDE platform needs the same "what happened on this record" timeline, the same activity scheduler, and the same follower list. If chatter logic lived inside each consumer (CRM, logistics, finance, HR, …), every module would re-invent the polymorphic timeline schema, the append-only invariant, the activity scheduler with outcome capture, the follower subscription-flag semantics, the per-follower notification fan-out (and its `note`-suppression rule), the email round-trip (outbound `mail.outbox` link + inbound thread reconstruction), and the React chatter component. That is 6–8 modules each rebuilding 80% of the same code. **Build it once in foundation, mix `Chatterable` into the `DomainModel`, add the four `<activity>` DSL elements to its FormView, and every record in the platform gets the same experience.**
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
Adopting chatter is a three-line recipe; no engine code changes are needed for new consumers.

- **Manifest** — depend on `foundation.communication`. After Phase 2 also depend on `foundation.notifications` (delivery) and `foundation.email` (outbound mail) if you want fan-out + email send.
- **Model** — mix the `Chatterable` abstract base into the `DomainModel` subclass; declare `__ede_auto_follow_fields__` if the model has an owner / assignee field:
  ```python
  from ede.foundation.communication.models.chatterable import Chatterable

  @api.model("domain.X.thing")
  class Thing(DomainModel, Chatterable):
      __ede_auto_follow_fields__ = ("owner_user_id",)  # optional; Phase 2
      # ...domain fields...
  ```
- **FormView** — add the four DSL elements inside an `<activity>` sheet sibling. The React renderer maps each child to its component (`<MessageConversation />`, `<FollowerBar />`, `<ScheduledActivity />`, `<Attachments />` — Phase-3 stub):
  ```xml
  <FormView>
    <header>...</header>
    <sheet>...</sheet>
    <activity>
      <MessageConversation/>
      <Followers/>
      <ScheduledActivity/>
      <Attachments/>
    </activity>
  </FormView>
  ```
- **Domain events as notifications** — when something noteworthy happens on the record (status change, assignment, approval decision, …), call the mixin method or dispatch the command:
  ```python
  thing.post_notification(body="Stage changed to Won by alice@…",
                          system_event="domain.X.thing.stage_changed")
  ```
- **Identity centralization** (Phase 2 WS-0) — every internal `res.user` carries a `partner_id` Reference to a `res.partner`; mention resolution and external-follower identity flow through that single directory.
- **End-user UX** — activity panel rendered below the FormView sheet on any model that opts in. Timeline list with composer (Send message / Log note tabs), "Schedule activity" dialog, follower bar with per-channel toggles, attachments stub.
- **RBAC** — chatter visibility follows record-level read access. If the user can open the record, they can see the chatter; if not, they cannot.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Caller domain]                              foundation.communication
  res.partner / crm.opportunity /              ┌──────────────────────────┐
  ops.shipment / finance.invoice ──┐           │  CommunicationService    │
                                   │           │  ─────────────────────   │
  env.dispatch(Command(            │  related  │  • post_message          │
    "communication.post_message",  ├──model──> │  • post_note             │
    payload={                      │  related  │  • schedule_activity     │
      "related_model": "res.partner", id      │  • complete_activity      │
      "related_id": "<uuid>",      │           │  • subscribe / follow    │
      "message_type": "message",   │           │  • unsubscribe / unfollow │
      "body": "...",               │           │  • list_timeline         │
    }))                            │           │  • list_activities       │
                                   │           └────────────┬─────────────┘
                                                            │
                                ┌───────────────────────────┼──────────────────────────┐
                                ▼                           ▼                          ▼
                  ┌──────────────────────┐    ┌──────────────────────┐    ┌──────────────────────┐
                  │ communication.message│    │   activity.log       │    │ communication.       │
                  │  (append-only,       │    │  (planned → done |   │    │  follower            │
                  │   immutable audit    │    │   cancelled, with    │    │  (internal user OR   │
                  │   trail)             │    │   outcome capture)   │    │   external partner)  │
                  └──────────┬───────────┘    └──────────┬───────────┘    └──────────┬───────────┘
                             │                           │                           │
                             └─────────┬─────────────────┴────────┬──────────────────┘
                                       ▼ (Phase 2 bridges)        ▼
                           ┌────────────────────────┐  ┌────────────────────────┐
                           │ foundation.notifications│  │ foundation.email       │
                           │  (per-follower delivery │  │  (mail.outbox for      │
                           │   per subscription flag)│  │   message_type=email)  │
                           └────────────────────────┘  └────────────────────────┘
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `activity.type` | Controlled master of activity kinds (Phone Call, Email, Meeting, Task, Note, Document, WhatsApp, …). Drives icon, default due-day offset, outcome requirement, email linkage, and `is_standard_locked`. | [models/activity_type.py](../src/ede/foundation/communication/models/activity_type.py) |
| `activity.log` | Polymorphic planned/done activity. Lifecycle `planned → done | cancelled`. Owner, due_at, outcome capture (`outcome_summary`, `outcome_status`), follow-up (`next_action`, `next_action_date`). | [models/activity.py](../src/ede/foundation/communication/models/activity.py) |
| `communication.message` | Polymorphic append-only timeline entry. Five `message_type` values: `message`, `note`, `email`, `activity`, `notification`. Cross-links to `mail.outbox` (`mail_outbox_id`) and `activity.log` (`activity_log_id`). Phase 3 adds `parent_message_id` (threading), `body_format` (`text`/`html`), `attachment_ids` (M2M to `storage.document`), `scheduled_at` + `dispatched_at`. | [models/message.py](../src/ede/foundation/communication/models/message.py) |
| `communication.follower` | Polymorphic subscriber list. `partner_type` ∈ {`internal`, `external`}; external partners stored polymorphically (`partner_model + partner_id`). Four subscription flags: `on_message`, `on_email`, `on_activity`, `on_notification`. Phase 2 adds `source` Enum (`manual` / `auto_owner` / `mention`). | [models/follower.py](../src/ede/foundation/communication/models/follower.py) |
| `communication.message.reaction` | Phase 3 — emoji reaction on a message. Unique on `(message_id, user_id, emoji)`. | _not yet built (Phase 3)_ |
| `communication.escalation.policy` | Phase 3 — activity SLA rule: triggered when `due_at` passed and status still `planned`; effects `notify_owner_manager` / `reassign_to_role` / `escalate_chain`. | _not yet built (Phase 3)_ |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `CommunicationService` | Phase 1 — orchestrate `post_message`, `post_note`, `post_notification`, `schedule_activity`, `complete_activity`, `cancel_activity`, `subscribe`, `unsubscribe`, `list_timeline`, `list_activities`, `list_followers`. Routed through `Command(...)` so lifecycle hooks fire. | _not yet built_ |
| `Chatterable` (abstract mixin) | Phase 1 — `AbstractDomainModel` mixed into any `DomainModel` to make it a chatter consumer. Exposes `post_message` / `post_note` / `subscribe` / `list_timeline` etc., delegating to `CommunicationService.from_env(self.env)`. Carries class-level `__ede_auto_follow_fields__` consumed by the auto-follow hook in Phase 2. | _not yet built_ |
| `ActivityCompletionHook` | Phase 1 — `post.activity.log.update` lifecycle hook: when status flips `planned → done`, auto-creates a `communication.message` of type `activity` linked to the completed `activity.log`. | _not yet built_ |
| `MessageImmutabilityHooks` | Phase 1 — `pre.communication.message.update` and `pre.communication.message.delete` reject all attempts (engine-enforced append-only audit trail). | _not yet built_ |
| `NotificationsBridge` | Phase 2 — handler on `communication.message.posted`. For every non-`note` message, dispatches `NotificationDispatcher.dispatch(...)` per follower whose subscription flag matches the message type; suppresses `note` outright. | _not yet built_ |
| `OutboundEmailBridge` | Phase 2 — `communication.post_email` command handler. Inserts a `message_type='email'` row, creates a `mail.outbox` entry through `foundation.email` (`body_html`/`body_text`, `to_addresses`, `from_address`), stamps `mail_outbox_id` back on the message row, then re-emits `communication.message.posted` for fan-out. | _not yet built_ |
| `AutoFollowHook` (Chatterable) | Phase 2 — `post.{model_key}.create` injected by the `Chatterable` mixin: for each field in `__ede_auto_follow_fields__`, resolves to a `res.user` and dispatches `communication.subscribe` with `source="auto_owner"`. Gated by `COMMUNICATION_AUTO_FOLLOW_OWNER`. | _not yet built_ |
| `MentionParser` | Phase 2 — `post.communication.message.create` hook: parses `@token` from body, resolves against `res.partner.name` / `res.partner.email`, auto-subscribes the partner (internal user form if linked via `res.user.partner_id`, external otherwise) with `source="mention"`, emits `communication.mention` for the dispatcher. Gated by `COMMUNICATION_MENTION_NOTIFY`. | _not yet built_ |
| `IncomingEmailHandler` | Phase 3 — parses inbound mail (`Message-ID` / `In-Reply-To` / `References` headers) to reconstruct the originating record and threads the reply into the timeline as a new `message_type='email'` row. | _not yet built_ |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `communication.post_message` | Phase 1 — caller posts a message on a record. | Inserts a `communication.message` row of type `message` and (Phase 2) fans out per follower subscriptions. |
| `communication.post_note` | Phase 1 — caller posts an internal note on a record. | Inserts a `communication.message` row of type `note`. **Never** notifies (engine-enforced). |
| `communication.post_email` | Phase 2 — caller sends an outbound email tied to a record. | Inserts a `message_type='email'` row, creates a `mail.outbox` entry via `foundation.email`, sets `mail_outbox_id`. |
| `communication.post_notification` | Phase 1 — domain module signals a system event on a record. | Inserts a `message_type='notification'` row (system origin, `author_id` null) and fans out per follower subscriptions. |
| `communication.schedule_activity` | Phase 1 — caller schedules an `activity.log`. | Creates an `activity.log` row with status `planned`, `due_at` derived from `activity.type.default_due_days` if not supplied. |
| `activity.log.complete` | Phase 1 — owner marks the activity done. | Sets `status='done'`, `completed_at`, validates `outcome_summary` if `requires_outcome`, then auto-posts an `activity` message via the lifecycle hook. |
| `activity.log.cancel` | Phase 1 — owner cancels the activity. | Sets `status='cancelled'`. No timeline post. |
| `communication.subscribe` | Phase 1 — caller adds an internal user or external partner as a follower with subscription flags. | Creates a `communication.follower` row. Idempotent on `(related_model, related_id, partner)`. |
| `communication.unsubscribe` | Phase 1 — caller removes (soft-deactivates) a follower. | Sets `active=False` on the matching follower row. |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `communication.message.posted` | After any `communication.message` is inserted (any type). | Phase 2 notifications bridge; analytics dashboards (Phase 3). |
| `communication.activity.completed` | After `activity.log` flips to `done`. | Phase 2 notifications bridge; downstream domain workflows interested in interaction outcomes. |
| `communication.follower.added` | After a new `communication.follower` is created. | Audit log; Phase 2 mention-driven auto-follow consumers. |
| `communication.mention` | Phase 2 — fired after a `@username` token in a message body is resolved to one or more `res.user`s. Carries the message UUID + mentioned user IDs. | Notifications dispatcher (mention notification template). |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `GET /api/chatter/{related_model}/{related_id}` | List the timeline for a record (paginated, optional `message_type` filter, optional date-range filter). | _not yet built (Phase 1)_ |
| `POST /api/chatter/{related_model}/{related_id}/message` | Post a message on a record. | _not yet built (Phase 1)_ |
| `POST /api/chatter/{related_model}/{related_id}/note` | Post an internal note. | _not yet built (Phase 1)_ |
| `POST /api/chatter/{related_model}/{related_id}/email` | Send an outbound email tied to the record. | _not yet built (Phase 2)_ |
| `GET /api/chatter/{related_model}/{related_id}/activities` | List activities (with status filter). | _not yet built (Phase 1)_ |
| `POST /api/chatter/{related_model}/{related_id}/activities` | Schedule an activity. | _not yet built (Phase 1)_ |
| `PATCH /api/chatter/activities/{activity_id}` | Mark done / cancel / edit. | _not yet built (Phase 1)_ |
| `GET /api/chatter/{related_model}/{related_id}/followers` | List followers. | _not yet built (Phase 1)_ |
| `POST /api/chatter/{related_model}/{related_id}/followers` | Add follower. | _not yet built (Phase 1)_ |
| `DELETE /api/chatter/followers/{follower_id}` | Unsubscribe a follower. | _not yet built (Phase 1)_ |
| `PATCH /api/chatter/followers/{follower_id}` | Toggle subscription flags. | _not yet built (Phase 1)_ |
| `POST /api/chatter/messages/{parent_message_id}/reply` | Phase 3 — post a threaded reply to a message. | _not yet built (Phase 3)_ |
| `POST /api/chatter/messages/{message_id}/reactions` | Phase 3 — toggle an emoji reaction on a message. | _not yet built (Phase 3)_ |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `post.activity.log.update` | Phase 1 — when an `activity.log` row flips `status` to `done`, auto-creates a `communication.message` of type `activity` linked back via `activity_log_id`. Phase 2: if `activity_type.links_to_email`, dispatches `communication.post_email` instead of just posting an `activity` message. |
| `pre.activity.log.update` | Phase 1 — when transitioning to `status='done'`, validates `outcome_summary` is non-empty if `activity_type.requires_outcome` is True. Rejects re-completion of already-done activities. |
| `pre.communication.message.update` | Phase 1 — rejects all updates on `communication.message` rows (append-only audit trail). |
| `pre.communication.message.delete` | Phase 1 — rejects all deletes on `communication.message` rows (immutable audit trail). |
| `post.communication.message.create` | Phase 1 — emits the `communication.message.posted` event so Phase 2 bridges (notifications, mention parser) can subscribe. |
| `post.{chatterable_model_key}.create` | Phase 2 — auto-injected on every model that mixes in `Chatterable`. For each field in `__ede_auto_follow_fields__`, dispatches `communication.subscribe` with `source="auto_owner"`. Gated by `COMMUNICATION_AUTO_FOLLOW_OWNER`. |
| `pre.res.user.create` | Phase 2 (WS-0) — if no `partner_id` supplied, creates a `res.partner` from the user's `name` + `email` and stamps the FK. |
| `pre.res.user.update` | Phase 2 (WS-0) — propagates `name` / `email` changes to the linked `res.partner`. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
```
activity.log status:

   planned ──── complete_activity ───▶ done
      │                                 │
      └──── cancel_activity ───▶ cancelled

  done is terminal (no further transitions).
  cancelled is terminal.
  Auto-post of communication.message(type='activity') happens on planned → done only.
```
`communication.message` is append-only — once written, never edited or status-changed.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `communication`
- `ACTIVE_DOMAINS` entry: _not applicable — foundation app_
- Manifest `depends`: `foundation.base`, `foundation.storage` (today). After Phase 2: `foundation.notifications`, `foundation.email`.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `COMMUNICATION_AUTO_FOLLOW_OWNER` | `bool` | `True` | `EDE_COMMUNICATION_AUTO_FOLLOW_OWNER` | Phase 2: auto-subscribe a record's owner / creator at create time. |
| `COMMUNICATION_MENTION_NOTIFY` | `bool` | `True` | `EDE_COMMUNICATION_MENTION_NOTIFY` | Phase 2: dispatch a mention notification when `@username` is matched in a message body. |
| `COMMUNICATION_RICH_TEXT_ENABLED` | `bool` | `True` | `EDE_COMMUNICATION_RICH_TEXT_ENABLED` | Phase 3: allow HTML body; when False, composer is plain-text only. |
| `COMMUNICATION_THREAD_DEPTH_LIMIT` | `int` | `5` | `EDE_COMMUNICATION_THREAD_DEPTH_LIMIT` | Phase 3: max nesting depth for threaded replies. |
| `COMMUNICATION_INBOUND_MAIL_ENABLED` | `bool` | `False` | `EDE_COMMUNICATION_INBOUND_MAIL_ENABLED` | Phase 3: master switch for the inbound mail handler. |
| `COMMUNICATION_SCHEDULED_WORKER_INTERVAL_S` | `int` | `60` | `EDE_COMMUNICATION_SCHEDULED_WORKER_INTERVAL_S` | Phase 3: background scan interval for scheduled-message dispatch. |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `communication.html_allowed_tags` | system | `list[str]` | `["p","br","strong","em","ul","ol","li","a","blockquote","code","pre","h3","h4"]` | Phase 3: bleach allowlist for HTML body sanitization. Tenants can shrink but not extend beyond a hardcoded ceiling. |
| `communication.html_allowed_attrs` | system | `dict[str, list[str]]` | `{"a": ["href","title","rel"]}` | Phase 3: bleach attribute allowlist. |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| Communication Settings (Phase 3) | `views/communication_settings.xml` | `rich_text_enabled`, `thread_depth_limit`, `inbound_mail_enabled` toggles surfaced to system administrators. |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| [data/activity_type_seed.xml](../src/ede/foundation/communication/data/activity_type_seed.xml) | Standard `activity.type` rows (Phone Call, Email, Meeting, Task, Note, Document, WhatsApp, …). Already shipped with the initial models. |
| `data/notification_template_seed.xml` (Phase 2) | `ir.notification.template` rows for `communication.message.posted` and `communication.mention` per transport. |
| `data/escalation_policy_seed.xml` (Phase 3) | Sample `communication.escalation.policy` rows per category demonstrating each `effect_kind`; deactivated by default — admins clone + activate. |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Engine, API, Chatter UI, single-consumer wiring (`res.partner`) | ✅ Delivered 2026-05-13 | [phase-1-implementation.md](../roadmap/foundation/communication/phase-1-implementation.md) |
| Phase 2 | Notifications + Email bridges, cross-domain rollout, mentions, auto-follow | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/communication/phase-2-implementation.md) |
| Phase 3 | Attachments, rich text, threads, mail-in, SLA escalation, scheduled messages | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/communication/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| **Phase 1 — Engine, HTTP API, React Chatter UI, single-consumer wiring** (2026-05-13) | `activity.type`, `activity.log`, `communication.message`, `communication.follower` | [services/communication_service.py](../src/ede/foundation/communication/services/communication_service.py) · [api/chatter_controller.py](../src/ede/foundation/communication/api/chatter_controller.py) · [models/chatterable.py](../src/ede/foundation/communication/models/chatterable.py) · [models/message.py](../src/ede/foundation/communication/models/message.py) (immutability hooks) · [models/activity.py](../src/ede/foundation/communication/models/activity.py) (auto-post hook) · [src/frontend/src/workspace/views/chatter/](../src/frontend/src/workspace/views/chatter/) (7 React components) · [src/ede/core/services/presentation/dsl/parser.py](../src/ede/core/services/presentation/dsl/parser.py) (`_parse_activity_element` + strict unknown-child rejection) · [src/ede/foundation/base/models/partner.py](../src/ede/foundation/base/models/partner.py) (`Partner(Chatterable)`) | [phase-1-implementation.md](../roadmap/foundation/communication/phase-1-implementation.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Phase 2 WS-0 (Identity centralization): `res.user` has no `partner_id` Reference to `res.partner` — mention resolution + external-follower identity prerequisite. Adds nullable column → data-backfill (one partner per existing user) → NOT-NULL alter. | 🔴 Critical (Phase 2 prereq) | [phase-2-implementation.md §WS-0](../roadmap/foundation/communication/phase-2-implementation.md) |
| Auto-follow service exists but not yet wired into the universal `post.{model}.create` hook for `Chatterable` consumers (`__ede_auto_follow_fields__` walk). Phase 2 finishes the wiring. | 🟠 High (Phase 2) | [phase-2-implementation.md §WS-3](../roadmap/foundation/communication/phase-2-implementation.md) |
| Mention parser exists but not wired into `pre.communication.message.create` hook for `@token` resolution against `res.partner.name`/`email`. Depends on WS-0 above. | 🟠 High (Phase 2) | [phase-2-implementation.md §WS-4](../roadmap/foundation/communication/phase-2-implementation.md) |
| Notifications bridge dispatches per-follower but the post-message → follower fan-out is bypass-able from non-`Command` paths; Phase 2 hardens via `communication.message.posted` event subscriber. | 🟠 High (Phase 2) | [phase-2-implementation.md §WS-1](../roadmap/foundation/communication/phase-2-implementation.md) |
| No incoming email handler — replies cannot be threaded back into the record's timeline (no `Message-ID`/`In-Reply-To` parsing). | 🟡 Medium (Phase 3) | [phase-3-implementation.md](../roadmap/foundation/communication/phase-3-implementation.md) |
| No attachment integration with `foundation.storage`. | 🟡 Medium (Phase 3) | [phase-3-implementation.md](../roadmap/foundation/communication/phase-3-implementation.md) |
| No activity SLA / overdue handling — `due_at` collected, nothing escalates. | 🟡 Medium (Phase 3) | [phase-3-implementation.md](../roadmap/foundation/communication/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _Populated as integration learnings emerge once Phase 1 ships._

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Initial migration `c3283c040098_foundation_communication_initial` creates all four tables (`activity_type`, `activity_log`, `communication_message`, `communication_follower`). Already applied in dev environments; no further migration today.
- Phase 1: a follow-up migration may add indexes (`(related_model, related_id, sent_at desc)` on `communication_message`; `(related_model, related_id, status, due_at)` on `activity_log`) once query patterns are confirmed.
- Phase 2 WS-0 (Identity centralization): `res.user.partner_id` Reference to `res.partner`. Migration steps: (1) add nullable column + index; (2) data-migrate — for every existing `res.user` without `partner_id`, insert a `res.partner` with name + email copied and stamp the FK; (3) alter column to NOT NULL.
- Phase 2: additive — `communication.follower.source` Enum (`manual` / `auto_owner` / `mention`) defaulting to `manual` for existing rows.
- Phase 3: additive — `communication.message.attachment_ids` M2M join to `storage.document`; `communication.message.body_format` Enum (`text` / `html`) default `text`; `communication.message.parent_message_id` Reference nullable; `communication.message.scheduled_at` + `dispatched_at` DateTimes nullable; new tables `communication_message_reaction` and `communication_escalation_policy`. No data transformation needed.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Any authenticated user | Read chatter on any record they have read access to (chatter visibility follows record-level read access). |
| Record owner / follower | Post messages, post notes, schedule activities, manage their own subscription flags. |
| System (no user) | Post `message_type='notification'` rows with `author_id=null` for status changes / approval decisions / SLA breaches. |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Notifications](./foundation-notifications.md) — Phase 2 hard-depends on it for per-follower fan-out.
- [Foundation Approval](./foundation-approval.md) — emits decision events that surface as `notification`-type timeline rows.
- [Foundation Storage](./17-storage-module.md) — Phase 3 attachments hang off `communication.message` via `storage.document` M2M.
- [Foundation Email](./11-foundation-apps.md#email) — Phase 2 outbound mail via `mail.outbox`; Phase 3 inbound mail handler reconstructs threads.
- [Foundation Model Naming](./foundation-model-naming.md) — `ir.*` vs `res.*` vs unprefixed engine namespaces.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-13 (Phase 1 ✅ Delivered — 6 WSs shipped; +26 new pytest cases; 1808 pytest + 494 vitest green; `res.partner` extends `Chatterable`; `ChatterController` enforces RBAC via `AuthorizationService.can` since `ede.read_one` skips pre-hooks; DSL `<activity>` parser strict-rejects unknown children). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
