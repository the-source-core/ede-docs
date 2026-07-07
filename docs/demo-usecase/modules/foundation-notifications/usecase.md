# `foundation.notifications` — Demo Use-Case

**Module:** `ede.foundation.notifications`
**App key:** `foundation.notifications`
**Demo manifest entries:** `demo/demo_preferences.xml`
**Status:** ✅ Delivered (2026-07-03) — Phase 2 preferences slice

---

## Use-case

Notification **templates** ship as production seeds and the **inbox** (persisted
`ir.notification` rows) is produced at runtime by real events, so it is not
seeded. What Phase 2 (Preferences & Quality of Life) adds — and what benefits
from a demo fixture — is the **per-user control surface**: a tester opening
Settings → Notifications → My Preferences should land on a screen that already
shows a realistic configuration rather than system defaults.

The demo is **self-contained on the always-seeded `base.admin_user`** so it loads
without depending on the (not-yet-started) foundation demo-user rollout
([`platform/04-demo-data-foundation-rollout.md`](../../../../roadmap/platform/04-demo-data-foundation-rollout.md)).
When that rollout ships demo users, per-persona preference rows can be added here.

It illustrates the three Phase 2 preference primitives:

- **Quiet hours + email mode** — the admin has a 22:00–07:00 Asia/Kolkata quiet
  window (intrusive channels defer to 07:00; web/in-app always fire) with
  realtime email.
- **Muting a channel for an event** — approval-assignment emails are turned off
  (the admin still gets the bell + web push for them).
- **A severity floor** — web push only fires for `warning`+ notifications across
  all events, so low-priority info toasts stay quiet.

## Records produced

### `demo/demo_preferences.xml`

| External ID | Model | Notes |
|---|---|---|
| `notifications.demo_user_setting_admin` | `ir.notification.user.setting` | admin — quiet hours 22:00→07:00 Asia/Kolkata, `email_mode=realtime` |
| `notifications.demo_pref_admin_approval_email_muted` | `ir.notification.preference` | admin — `approval.task.assigned` / `email` disabled |
| `notifications.demo_pref_admin_web_warning_floor` | `ir.notification.preference` | admin — all-events / `web` enabled, `severity_floor=warning` |

## Out of scope

- **Inbox rows** (`ir.notification`) — produced at runtime by real events; seeding
  them needs demo users + demo source records (approval cases, CRM leads) that
  belong to the foundation demo rollout, not this module.
- **Delivery log / deferred queue / digest queue** (`ir.notification.delivery` /
  `.queue` / `.digest.queue`) — transient worker state; they reference a live
  `ir.notification` FK and are best demonstrated live, not as fixtures.
- **Per-persona preferences** — deferred until `base` ships demo users.

## Dependencies

- `foundation.base` production seed — `base.admin_user` (always present; no demo
  dependency).

## Verification

`ede migrate upgrade -t <tenant> --with-demo=foundation.notifications` — expected
**3 created** on first run, **3 updated** on re-run (idempotent). Then open
Settings → Notifications → My Preferences as the admin and confirm the quiet-hours
window, the muted approval email, and the web severity floor are pre-filled.

## Authoring order

Single file; the two `ir.notification.preference` rows and the one
`ir.notification.user.setting` row are independent (each only `ref`s the seeded
`base.admin_user`), so intra-file order is not load-bearing.

---

*Back to [demo-usecase index](../../README.md).*
