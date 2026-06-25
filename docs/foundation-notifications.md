<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Notifications Engine — Implementation Docs

**Module:** `foundation.notifications` (Phase 1 ✅ Delivered — module extracted from `foundation.base`, `foundation.presentation`, `foundation.email`)
**Roadmap:** [roadmap/foundation/notifications/README.md](../roadmap/foundation/notifications/README.md)
**Status:** ✅ Phase 1 Delivered (Phase 2 / Phase 3 not started)
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **multi-transport, recipient-aware, template-driven notification engine** that any EDE module emits to without knowing which channels each recipient has enabled. The engine resolves recipients (by user, role, group, branch, or dynamic query), looks up templates per `event_key` and locale, reads per-user channel preferences, persists an `ir.notification` record per recipient, and fans out to every enabled transport (email, web push, in-app, SMS, mobile push) in one call. Today this engine **does not exist as a standalone module** — its responsibilities are fragmented across `foundation.base` (email-only `NotificationService`, org-level `ir.notification.setting`), `foundation.email` (templates + outbox), and `foundation.presentation` (SSE web push). Phase 1 extracts and unifies these into `foundation.notifications`.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Without a real engine, every consumer module (approval, gateway, RMS, CRM, HR…) has to re-invent the same 80% of code: resolve its own recipients (loop users by role), hardcode template names, manually emit to email **and** web push if it wants both, and build its own per-user preference UI to respect "I don't want push for this." That's 5–6 modules each duplicating the same plumbing. **Promote it once.** Concretely, `foundation.approval` cannot ship a usable notification bar without this engine first — the current `NotificationService` is a thin email-only wrapper, not an engine. Promoting notifications to a first-class foundation module unblocks approval and every other downstream consumer.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Manifest** — depend on `foundation.notifications` in your app's `__manifest__.py`.
- **Templates** — register `ir.notification.template` data fixtures keyed by `event_key` (e.g. `approval.task.assigned`) with subject + body templates per transport.
- **Emit** — at the moment of business event, dispatch a single command:
  ```python
  env.dispatch(Command("notification.send", payload={
      "event_key": "approval.task.assigned",
      # A recipient is a res.partner (Enhancement 01). If you hold a user id,
      # convert: partner_recipient_spec(env, assignee_id) -> {"partner_ids": [...]}.
      "recipient_spec": {"partner_id": assignee_partner_id},
      "template_vars": {...},
      "source_model": "ir.approval.case",
      "source_id": case_uuid,
      "action_url": f"/approvals/case/{case_uuid}",
  }))
  ```
- **Action-from-notification** — include `action_url` in the payload; bell click navigates there. Phase 3 adds signed deep-links for cross-channel "Approve from email"-style flows.
- **That's it.** Engine handles recipient resolution, template lookup, preference filtering, persistence, multi-transport delivery, and dedup. No per-domain RBAC needed — preference management is engine-owned.
- **End-user UX** — bell icon in workspace header (`UserNotification.tsx` — currently empty shell, hydrates from `GET /api/notifications` once Phase 1 ships); in-app toast via `NotificationDialog.tsx`; email via `mail.outbox`.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Caller]  env.dispatch(Command(
            "notification.send",
            payload={
              "event_key": "approval.task.assigned",
              "recipient_spec": {"role": "pricing_manager", "branch_id": "..."},
              "template_vars": {"subject_name": "Acme Buy Rate", ...},
              "source_model": "ir.approval.case",
              "source_id": "<uuid>",
            },
          ))
                  │
                  ▼
        ┌──────────────────────────────┐
        │  NotificationDispatcher      │
        │  ─────────────────────────   │
        │  1. Resolve recipients       │
        │     (role/group/branch/dynamic) │
        │  2. Look up template         │
        │     (per event_key + locale) │
        │  3. Read user preferences    │
        │     (channels per event)     │
        │  4. Persist ir.notification  │
        │     record (one per recipient)│
        │  5. Fan out to transports    │
        └──────────────────────────────┘
                  │
        ┌─────────┼─────────┬──────────┬──────────┐
        ▼         ▼         ▼          ▼          ▼
     [email]  [web push]  [in-app]  [SMS]     [mobile push]
                                              (Phase 3)
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.notification` | One record per recipient: subject, body, level, transports_sent, source link, action_url, read_at_utc, dismissed_at_utc, correlation_id | [src/ede/foundation/notifications/models/notification.py](../src/ede/foundation/notifications/models/notification.py) |
| `ir.notification.template` | Central template registry keyed by (event_key, transport, locale_code) — Jinja2 subject + body + default level | [src/ede/foundation/notifications/models/notification_template.py](../src/ede/foundation/notifications/models/notification_template.py) |
| `ir.notification.setting` | Org-level enable/disable per event_type + allowed channels (kept in `foundation.base` for compat with `rbac_seed.xml`) | [src/ede/foundation/base/models/notification_setting.py](../src/ede/foundation/base/models/notification_setting.py) |
| `ir.notification.preference` *(planned, Phase 2)* | Per-user channel routing, snooze, mute, quiet hours | _Phase 2_ |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `NotificationDispatcher` | Recipient resolution → template render → org-pref check → `ir.notification` persistence → multi-transport fan-out | [src/ede/foundation/notifications/services/dispatcher.py](../src/ede/foundation/notifications/services/dispatcher.py) |
| `RecipientResolver` | Turns recipient_spec (`partner_id` / `partner_ids` / `role` / `role+branch_id`) into deduplicated **res.partner** UUIDs (Enhancement 01 — partner-centric recipients). `user_id`/`user_ids`/`group_id`/`dynamic_query` are no longer supported | [src/ede/foundation/notifications/services/recipient_resolver.py](../src/ede/foundation/notifications/services/recipient_resolver.py) |
| `EmailTransport` | Drops a queued `mail.outbox` row for the recipient — delivery via `EmailRouter.process_queue` | [src/ede/foundation/notifications/services/transports/email_transport.py](../src/ede/foundation/notifications/services/transports/email_transport.py) |
| `WebPushTransport` | Emits `web.client.notification` event → SSE handler in `foundation.presentation` broadcasts to recipient's open browser tabs | [src/ede/foundation/notifications/services/transports/web_push_transport.py](../src/ede/foundation/notifications/services/transports/web_push_transport.py) |
| `InAppTransport` | No-op symbolic transport — the `ir.notification` record itself is the delivery (bell reads via `GET /api/notifications`) | [src/ede/foundation/notifications/services/transports/in_app_transport.py](../src/ede/foundation/notifications/services/transports/in_app_transport.py) |
| `NotificationService` *(compat shim)* | Translates legacy `NotificationService(env).send(event_type=…, to_user_id=…, …)` into a `Command("notification.send", ...)` dispatch | [src/ede/foundation/base/services/notification_service.py](../src/ede/foundation/base/services/notification_service.py) |
| `mail.template` engine | Jinja2 template rendering (reused for non-email channels) | [src/ede/foundation/email/](../src/ede/foundation/email/) |
| `mail.outbox` | Reliable retry-aware email delivery queue (used as the email transport's queue layer) | [src/ede/foundation/email/](../src/ede/foundation/email/) |
| `web.client.notification` SSE handler | Broadcasts to connected browser sessions for the WebPushTransport | [src/ede/foundation/presentation/api/web_push.py](../src/ede/foundation/presentation/api/web_push.py), [src/ede/foundation/presentation/events/web_client_events.py](../src/ede/foundation/presentation/events/web_client_events.py) |
| `WebPushService.ts` + `WebPushEventRegistry.ts` | Frontend SSE client + typed pub/sub — bell subscribes for live updates | [src/frontend/src/services/push/](../src/frontend/src/services/push/) |
| `UserNotification.tsx` | Bell UI — hydrates from `/api/notifications`, subscribes to SSE for live updates, mark-read + dismiss handlers | [src/frontend/src/workspace/components/header/UserNotification.tsx](../src/frontend/src/workspace/components/header/UserNotification.tsx) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `notification.send` | Any consumer module at the moment of a business event | Resolves recipients, looks up template, applies preferences, persists `ir.notification` per recipient, fans out to all enabled transports. Routed to `Notification.handle_notification_send` |
| `ir.notification.mark_read` | Bell click + `POST /api/notifications/{id}/mark-read` | Sets `read_at_utc = now()` |
| `ir.notification.dismiss` | Bell × button + `DELETE /api/notifications/{id}` | Sets `dismissed_at_utc = now()` |
| `ir.notification.setting.update` | Settings UI / admin API | Updates org-level `is_enabled` + `channels` for an event_type |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `web.client.notification` | Dispatcher pushes to recipient's connected browser session(s) | `WebPushEventRegistry` on the frontend → bell badge + toast |
| _Dedicated lifecycle events (planned)_ | On notification create / read / dismiss | Frontend bell sync, audit log, analytics |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `GET /api/notifications` | List current user's notifications (filters: `unread_only`, `include_dismissed`, `limit`, `offset`) | [`NotificationController.list_notifications`](../src/ede/foundation/notifications/api/notification_routes.py) |
| `GET /api/notifications/unread-count` | Unread+undismissed count for the badge | `NotificationController.unread_count` |
| `POST /api/notifications/{id}/mark-read` | Set `read_at_utc = now()` (own records only) | `NotificationController.mark_read` |
| `POST /api/notifications/mark-all-read` | Bulk mark every unread notification for current user as read | `NotificationController.mark_all_read` |
| `DELETE /api/notifications/{id}` | Soft-dismiss (sets `dismissed_at_utc = now()`) | `NotificationController.dismiss` |
| `GET /api/notifications/by-source?model=…&id=…` | All notifications for a (source_model, source_id) pair scoped to current user | `NotificationController.by_source` |
| `POST /api/notifications/send` | Manual dispatch — admin/debug use; production code should `Command("notification.send", ...)` directly | `NotificationController.send` |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
Once `ir.notification` ships in Phase 1, each notification record will follow a simple state model:

```
[unread] ──mark-read──▶ [read] ──dismiss──▶ [dismissed]
   └────────────────dismiss──────────────────▶ [dismissed]
```

No transitions back from `dismissed`. `mark-all-read` is a bulk transition over all `unread` rows for the current user.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> ⚠ **Roadmap predates the Configuration Introduced discipline (added 2026-05-08) AND the engine itself does not yet exist as a standalone module.** Backfill needed in [roadmap/foundation/notifications/README.md](../roadmap/foundation/notifications/README.md) and per-phase files via the `roadmap-driven-delivery` skill. Until then, this section is empty. Likely Phase 1 introductions: per-user `ir.notification.preference` keys, default channel routing in `ir.config`, template registry seed data.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `notifications` (loaded last so deps `base`, `email`, `presentation` register first)
- `ACTIVE_DOMAINS` entry: n/a (foundation app)
- Manifest `depends`: `foundation.base`, `foundation.email`, `foundation.presentation`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
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
| `data/ir.rbac.permission.csv` | RBAC keys: `ir.notification.read`/`update`/`delete` (internal user) and `ir.notification.template.read`/`create`/`update`/`delete` (system admin) |
| `data/notification_templates.xml` | Default `ir.notification.template` rows for `approval.task.assigned`, `approval.task.escalated`, `approval.case.decided` across email + web + in_app transports (locale `en`) |
| `data/notification_menus.xml` | `Settings → Notifications → Templates` and `Settings → Notifications → Delivery Log` menu entries (action_id wired to the new models) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Extract & Unify (unblocks foundation.approval) | ✅ Delivered | [phase-1-implementation.md](../roadmap/foundation/notifications/phase-1-implementation.md) |
| Phase 2 | Preferences & Quality of Life | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/notifications/phase-2-implementation.md) |
| Phase 3 | Mobile, Localization, Advanced | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/notifications/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Module skeleton + activation (WS-N1) | n/a | [`__manifest__.py`](../src/ede/foundation/notifications/__manifest__.py) · [`settings.py`](../src/ede/foundation/settings.py) | [phase-1 WS-N1](../roadmap/foundation/notifications/phase-1-implementation.md) |
| Persistent notification + recipient resolver (WS-N2) | `ir.notification`, `ir.notification.template` | [`models/notification.py`](../src/ede/foundation/notifications/models/notification.py) · [`models/notification_template.py`](../src/ede/foundation/notifications/models/notification_template.py) · [`services/recipient_resolver.py`](../src/ede/foundation/notifications/services/recipient_resolver.py) | [phase-1 WS-N2](../roadmap/foundation/notifications/phase-1-implementation.md) |
| Multi-transport dispatcher (WS-N3) | `ir.notification` (handler) | [`services/dispatcher.py`](../src/ede/foundation/notifications/services/dispatcher.py) · [`services/transports/`](../src/ede/foundation/notifications/services/transports/) | [phase-1 WS-N3](../roadmap/foundation/notifications/phase-1-implementation.md) |
| HTTP API + bell hydration (WS-N4) | `ir.notification` | [`api/notification_routes.py`](../src/ede/foundation/notifications/api/notification_routes.py) · [`UserNotification.tsx`](../src/frontend/src/workspace/components/header/UserNotification.tsx) | [phase-1 WS-N4](../roadmap/foundation/notifications/phase-1-implementation.md) |
| Compat shim for legacy callers | `ir.notification` | [`base/services/notification_service.py`](../src/ede/foundation/base/services/notification_service.py) | phase-1 §"Compat shim" |

> Phase 1 ✅ Delivered. Backend + frontend shipped; full test suite (1155 Python + 343 frontend) green; 23 dispatcher / resolver unit tests under [`src/tests/notifications/`](../src/tests/notifications/); bell hydration + click-through to `action_url` verified in a running browser session.
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| **Org-level prefs only** — engine reads `ir.notification.setting` (org + event_type); per-user channel routing is Phase 2 | 🟠 High (Phase 2) | [README.md gap #6](../roadmap/foundation/notifications/README.md) |
| **No deduplication / rollup** — if 5 events fire in 10 seconds for the same user, 5 bell rows; rollup is Phase 2 (`correlation_id` field reserved) | 🟡 Medium (Phase 2) | [README.md gap #8](../roadmap/foundation/notifications/README.md) |
| **No locale handling** — templates use `locale_code='en'` only; locale selection lands in Phase 3 | 🟡 Medium (Phase 3) | [README.md gap #9](../roadmap/foundation/notifications/README.md) |
| **No quiet hours / digest** — Phase 2 deliverables | 🟡 Medium (Phase 2) | [README.md gap #10](../roadmap/foundation/notifications/README.md) |
| **No mobile push / SMS** — Phase 3 deliverables | 🟢 Phase 3 | [Phase 3 plan](../roadmap/foundation/notifications/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- **Don't add new email-only logic to `NotificationService`** — channel-fan-out belongs in the planned `NotificationDispatcher`. Every email-only branch added today is debt that has to be unwound when Phase 1 lands.
- **Don't emit `web.client.notification` directly from a domain** — once the dispatcher exists, every notification must go through it so persistence, preferences, and dedup happen consistently. Today, callers that want both bell and email have to emit twice; Phase 1 fixes this.
- **Don't hardcode template name derivation** (e.g. `event_type.replace(".", "_")` like `approval_service.py` does today) — register an `ir.notification.template` fixture keyed by `event_key` once that model lands. Hardcoding template names is a Phase-1 migration trap.
- **Don't build per-domain notification preference UIs** — preference management is engine-owned (`ir.notification.preference` lands in Phase 2). Per-domain RBAC for notifications is an anti-pattern.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- **Phase 1 schema additions** — `ir.notification`, `ir.notification.template`, plus consolidation of `ir.notification.setting` into the new `foundation.notifications` namespace. New tables; no destructive migrations on existing data.
- **Compat shim during transition** — consumers using `NotificationService.send()` directly today (e.g. approval) will continue to work via a thin wrapper that translates legacy single-recipient email calls into `Command("notification.send", ...)` dispatches. The shim is removed once all in-tree consumers cut over.
- **Template fixture migration** — domains with hardcoded template names (e.g. `approval` deriving `approval_task_assigned` from `approval.task.assigned`) need a one-time data migration to register the corresponding `ir.notification.template` fixtures keyed by `event_key`.
- **Frontend cutover** — `UserNotification.tsx` will hydrate from `GET /api/notifications` instead of the current hard-coded empty array. No breaking change for consumers; the bell simply starts showing data.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `rbac.role_internal_user` | `ir.notification.read` (own only via controller scoping), `ir.notification.update` (mark-read), `ir.notification.delete` (dismiss), `ir.notification.template.read` |
| `rbac.role_system_admin` | `ir.notification.template.create` / `update` / `delete` |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Approval](foundation-approval.md) — first major consumer; Phase 1 of this engine is a blocking dependency for approval's notification bar
- [Foundation Email](../src/ede/foundation/email/) — owns SMTP plumbing, `mail.template`, and `mail.outbox` (reused as the email transport)
- [Foundation Presentation](../src/ede/foundation/presentation/) — owns the `web.client.notification` SSE plumbing reused as the web-push transport
- [Foundation Model Naming](foundation-model-naming.md) — `ir.*` vs `res.*` conventions for the new models
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-09 (Phase 1 ✅ Delivered — browser bell walkthrough complete). To refresh, invoke the syncing-roadmap-to-docs skill.*
