<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Notifications Engine — Implementation Docs

**Module:** `foundation.notifications` (Phase 1 ✅ Delivered · Phase 2 ✅ Delivered)
**Roadmap:** [roadmap/foundation/notifications/README.md](../roadmap/foundation/notifications/README.md)
**Status:** ✅ Phase 1 + Phase 2 Delivered (Phase 3 not started)
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
| `ir.notification.preference` *(Phase 2)* | Per-user, per-event, per-transport routing rule — `enabled` + `severity_floor`; exact-event overrides `event_key=''` default | [src/ede/foundation/notifications/models/preference.py](../src/ede/foundation/notifications/models/preference.py) |
| `ir.notification.user.setting` *(Phase 2)* | Per-user globals — quiet hours (`HH:MM` + IANA tz) + `email_mode` (realtime / digest_daily / digest_weekly / off) | [src/ede/foundation/notifications/models/preference.py](../src/ede/foundation/notifications/models/preference.py) |
| `ir.notification.queue` *(Phase 2)* | Deferred single-channel delivery held for quiet hours; released by the worker at `deferred_until_utc` | [src/ede/foundation/notifications/models/queue.py](../src/ede/foundation/notifications/models/queue.py) |
| `ir.notification.digest.queue` *(Phase 2)* | Pending email-digest item; batched into one summary email per user at their digest hour | [src/ede/foundation/notifications/models/queue.py](../src/ede/foundation/notifications/models/queue.py) |
| `ir.notification.delivery` *(Phase 2)* | Per-transport delivery-attempt log (queued / sent / failed / bounced / suppressed + error + external_id) | [src/ede/foundation/notifications/models/delivery.py](../src/ede/foundation/notifications/models/delivery.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `NotificationDispatcher` | Recipient resolution → dedup (Phase 2) → template render → org + per-user pref check → `ir.notification` persistence → per-channel routing (send / defer / digest / suppress) + delivery-log writes | [src/ede/foundation/notifications/services/dispatcher.py](../src/ede/foundation/notifications/services/dispatcher.py) |
| `PreferenceResolver` *(Phase 2)* | Resolves a user's effective channels for an event: most-specific matrix rule (exact-event > default) with `enabled` + `severity_floor`; also exposes quiet-hours + email-mode globals | [src/ede/foundation/notifications/services/preference_resolver.py](../src/ede/foundation/notifications/services/preference_resolver.py) |
| `quiet_until` *(Phase 2)* | Pure quiet-hours window math (`HH:MM` in an IANA tz, wrap-past-midnight, next-end release instant in UTC) | [src/ede/foundation/notifications/services/quiet_hours.py](../src/ede/foundation/notifications/services/quiet_hours.py) |
| `release_deferred_tick` / `digest_tick` *(Phase 2)* | `@api.scheduled_job` workers — release quiet-hours-held deliveries (every minute) and send per-user email digests (hourly) | [src/ede/foundation/notifications/services/workers.py](../src/ede/foundation/notifications/services/workers.py) |
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
| `ir.notification.preference.set` *(Phase 2)* | My Preferences UI / `POST /api/notifications/preferences` | Upserts one routing rule for the acting user (unique on user+event+transport) |
| `ir.notification.user.setting.set` *(Phase 2)* | My Preferences UI / `POST /api/notifications/settings` | Upserts the acting user's quiet-hours + email-mode globals |
| `notification.recall` *(Phase 2)* | Consumer when an event is undone (e.g. approval case cancelled) | Dismisses a `correlation_id` group (or source) + cancels its still-pending queue/digest rows |
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
| `GET /api/notifications/preferences` *(Phase 2)* | Current user's routing matrix + global settings (for My Preferences UI) | `NotificationController.get_preferences` |
| `POST /api/notifications/preferences` *(Phase 2)* | Upsert one routing rule for the current user | `NotificationController.set_preference` |
| `POST /api/notifications/settings` *(Phase 2)* | Upsert current user's quiet-hours + email-mode | `NotificationController.set_settings` |
| `GET /api/notifications/admin/analytics` *(Phase 2)* | Delivery counts grouped by (event_key, status) for the analytics dashboard — RBAC-gated | `NotificationController.admin_analytics` |
| `POST /api/notifications/admin/replay/{id}` *(Phase 2)* | Re-send an existing notification across its sent channels — RBAC-gated | `NotificationController.admin_replay` |
| `POST /api/notifications/admin/test` *(Phase 2)* | Render subject/body templates against vars without persisting — RBAC-gated | `NotificationController.admin_test` |
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

> Captures every knob this module adds. Phase 2 introduced three `FoundationSettings` keys (dedup window + digest schedule) and the per-user preference models; per-user routing is stored as records (`ir.notification.preference` / `ir.notification.user.setting`), not config.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `notifications` (ordered after `jobs` so the scheduled-job workers register)
- `ACTIVE_DOMAINS` entry: n/a (foundation app)
- Manifest `depends`: `foundation.base`, `foundation.email`, `foundation.presentation`, `foundation.jobs` (Phase 2 — for the release + digest workers)
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `NOTIFICATION_DEDUP_WINDOW_SECONDS` | int | `60` | same | Window in which a repeat (event_key, recipient, source_id) dedups |
| `NOTIFICATION_DIGEST_DAILY_HOUR` | int | `9` | same | User-local hour the daily digest email is sent |
| `NOTIFICATION_DIGEST_WEEKLY_DOW` | int | `0` | same | Weekday (0=Mon) the weekly digest is sent |
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
| `data/notification_menus.xml` | `Settings → Notifications` menu — My Preferences + Templates + User Settings + **Delivery Analytics** (client-action dashboard) + Delivery Log + Deferred Queue (Phase 2 adds 11 more RBAC perms for preference/user-setting/delivery/queue/digest) |
| `demo/demo_preferences.xml` *(Phase 2 demo)* | Admin preference config on `base.admin_user` — quiet hours + muted approval email + web severity floor (loads via `--with-demo=foundation.notifications`) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Extract & Unify (unblocks foundation.approval) | ✅ Delivered | [phase-1-implementation.md](../roadmap/foundation/notifications/phase-1-implementation.md) |
| Phase 2 | Preferences & Quality of Life | ✅ Delivered (2026-07-07) | [phase-2-implementation.md](../roadmap/foundation/notifications/phase-2-implementation.md) |
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
| **Per-user preferences (WS-NP1)** | `ir.notification.preference`, `ir.notification.user.setting` | [`models/preference.py`](../src/ede/foundation/notifications/models/preference.py) · [`services/preference_resolver.py`](../src/ede/foundation/notifications/services/preference_resolver.py) · [`frontend/client-actions/NotificationPreferences.tsx`](../src/ede/foundation/notifications/frontend/client-actions/NotificationPreferences.tsx) | [phase-2 WS-NP1](../roadmap/foundation/notifications/phase-2-implementation.md) |
| **Quiet hours + digest + dedup (WS-NP1.3/NP2.2/NP2.3)** | `ir.notification.queue`, `ir.notification.digest.queue` | [`services/quiet_hours.py`](../src/ede/foundation/notifications/services/quiet_hours.py) · [`services/workers.py`](../src/ede/foundation/notifications/services/workers.py) · [`services/dispatcher.py`](../src/ede/foundation/notifications/services/dispatcher.py) | [phase-2 WS-NP2](../roadmap/foundation/notifications/phase-2-implementation.md) |
| **Correlation rollup + recall (WS-NP2.1)** | `ir.notification` (`correlation_id`) | [`models/notification.py`](../src/ede/foundation/notifications/models/notification.py) · [`UserNotification.tsx`](../src/frontend/src/managers/UserNotification.tsx) | [phase-2 WS-NP2.1](../roadmap/foundation/notifications/phase-2-implementation.md) |
| **Delivery log + admin analytics/replay/test (WS-NP3)** | `ir.notification.delivery` | [`models/delivery.py`](../src/ede/foundation/notifications/models/delivery.py) · [`api/notification_routes.py`](../src/ede/foundation/notifications/api/notification_routes.py) · [`frontend/client-actions/NotificationAnalytics.tsx`](../src/ede/foundation/notifications/frontend/client-actions/NotificationAnalytics.tsx) | [phase-2 WS-NP3](../roadmap/foundation/notifications/phase-2-implementation.md) |

> Phase 1 + Phase 2 ✅ Delivered. Phase 2 adds 5 models, 2 scheduled workers, migration `d95cbdb8319a` (applies exit-0 from scratch), the My Preferences + Delivery Analytics client actions, and the bell correlation rollup. 51 new pytest under [`src/tests/notifications/test_phase2.py`](../src/tests/notifications/test_phase2.py) + full backend suite exit 0; frontend tsc/vite/626 vitest green; `--with-demo` idempotent. Live browser walkthrough deferred (backend/frontend-confirmed reachability, per the repo's e2e-infra precedent).
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| **No locale handling** — templates use `locale_code='en'` only; locale selection lands in Phase 3 | 🟡 Medium (Phase 3) | [README.md gap #9](../roadmap/foundation/notifications/README.md) |
| **No mobile push / SMS** — Phase 3 deliverables | 🟢 Phase 3 | [Phase 3 plan](../roadmap/foundation/notifications/phase-3-implementation.md) |
| **Analytics dashboard shows delivery/success only** — open/click/opt-out rates need extra instrumentation (future) | 🟢 Low | [phase-2 NP3.2](../roadmap/foundation/notifications/phase-2-implementation.md) |
| **Live browser walkthrough deferred** — backend + frontend verified (tests/build green); interactive e2e per repo e2e-infra precedent | 🟢 Low | [phase-2 verification](../roadmap/foundation/notifications/phase-2-implementation.md) |
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
- **Phase 2 schema additions** — migration `d95cbdb8319a` adds 5 tables (`ir.notification.preference`, `ir.notification.user.setting`, `ir.notification.queue`, `ir.notification.digest.queue`, `ir.notification.delivery`); additive, no destructive ops; applies exit-0 from scratch on a fresh tenant. Generated via a postgres-safe, created-table-scoped drift stripper added to `src/ede/cli/commands/migrate.py` (the sqlite autogen reference is blocked repo-wide by a pre-existing non-sqlite `foundation.communication` self-FK migration; the postgres reference otherwise leaks inherent Enum/index/audit-FK drift on the existing notification tables).
- **Module load order** — `foundation.jobs` added to `depends`; `ACTIVE_MODULES` reordered so `jobs` loads before `notifications` (the scheduled-job workers register on boot).
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `rbac.role_internal_user` | `ir.notification.read`/`update`/`delete` (own, via controller scoping), `ir.notification.template.read`, and **(Phase 2)** own-CRUD on `ir.notification.preference` + `ir.notification.user.setting` |
| `rbac.role_system_admin` | `ir.notification.template.create`/`update`/`delete`, and **(Phase 2)** read on `ir.notification.delivery` / `ir.notification.queue` / `ir.notification.digest.queue` (gates the admin analytics/replay/test surface) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Approval](foundation-approval.md) — first major consumer; Phase 1 of this engine is a blocking dependency for approval's notification bar
- [Foundation Email](../src/ede/foundation/email/) — owns SMTP plumbing, `mail.template`, and `mail.outbox` (reused as the email transport)
- [Foundation Presentation](../src/ede/foundation/presentation/) — owns the `web.client.notification` SSE plumbing reused as the web-push transport
- [Foundation Model Naming](foundation-model-naming.md) — `ir.*` vs `res.*` conventions for the new models
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-07-07 (Phase 2 ✅ Delivered — preferences, quiet hours, dedup, digest, delivery log, admin analytics dashboard). To refresh, invoke the syncing-roadmap-to-docs skill.*
