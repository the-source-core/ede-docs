# `foundation.agent` — Demo Use-Case

**Module:** `ede.foundation.agent`
**App key:** `foundation.agent`
**Demo manifest entries** (target): `demo/demo_agent.xml`, `demo/demo_automation.xml`
**Status:** ✅ Delivered 2026-06-18 — Phase 1 agents + Phase 2 AI Automations. Smoke: `--with-demo=foundation.agent` → **8 created** (1 agent + 3 automations + 4 actions), re-run idempotent (0 created, 8 updated).

---

## Use-case

In the **Acme Forwarding Ltd. (Mumbai HQ)** tenant, the AI layer ships two things a demo
viewer can explore under the standalone **AI** app:

1. **Agents** (`demo/demo_agent.xml`) — the built-in deterministic *Customization Field
   Builder* (seeded as data) plus a read-only *Schema Explorer* showing the user-craftable
   agent pattern.
2. **AI Automations** (`demo/demo_automation.xml`) — three saved **Trigger → AI Agent →
   Actions** pipelines that illustrate each in-scope trigger style + the confirm-gate, all
   self-contained (no cross-module record refs — foundation cannot ref domain rows):
   - **Greet new leads** — `record_event` on `crm.lead` *create* → posts a chatter note +
     sends a notification (auto posture). The everyday "react to a record" pattern.
   - **Summarise on request** — `manual` trigger with the *Schema Explorer* AI agent →
     sends the agent's `{{ai.summary}}` as a notification. Shows the AI step feeding actions.
   - **Daily ops digest** — `schedule` (daily 08:00), **confirm** posture → sends a digest
     notification only after a human approves the run. Shows scheduling + the confirm-gate.

The automations are illustrative: their triggers reference model keys as strings, so they
display fully even though they only *fire* when the referenced models exist and the worker
is running.

## Records produced

### `demo/demo_agent.xml` (Phase 1 — pre-existing)

| External ID | Model | Notes |
|---|---|---|
| `agent.demo_schema_explorer` | `agent.definition` | read-only Schema Explorer (read_schema + find_field capabilities) |

### `demo/demo_automation.xml` (Phase 2 — new)

| External ID | Model | Notes |
|---|---|---|
| `agent.demo_automation_greet_leads` | `agent.automation` | `record_event` crm.lead/create, posture auto, active |
| `agent.demo_automation_greet_leads_act1` | `agent.automation.action` | `post_message` on the new lead |
| `agent.demo_automation_greet_leads_act2` | `agent.automation.action` | `send_notification` to the sales team |
| `agent.demo_automation_summarise` | `agent.automation` | `manual`, AI agent = Schema Explorer, posture auto |
| `agent.demo_automation_summarise_act1` | `agent.automation.action` | `send_notification` with `{{ai.summary}}` |
| `agent.demo_automation_digest` | `agent.automation` | `schedule` daily 08:00, posture **confirm**, active |
| `agent.demo_automation_digest_act1` | `agent.automation.action` | `send_notification` daily digest |

## Out of scope

- No cross-module refs (no demo `crm.lead` row is referenced — foundation runs before domains; the trigger matches the `crm.lead` *model key* as a string).
- `owner_id` left unset (no `base.demo_user_*` records ship yet) — runs fall back to the caller principal.
- `webhook` / `incoming_email` triggers (tracked as enhancements, out of Phase 2).

## Dependencies

- `foundation.agent` Phase 1 data (`agent.builtin_*`, capabilities) loads first; `agent.demo_schema_explorer` is referenced by the *Summarise on request* automation (same-module, same demo pass).

## Verification

`ede migrate upgrade -t <tenant> --with-demo=foundation.agent` — expect the existing Schema
Explorer + **3 automations and their 4 action rows** created; re-run is idempotent (`updated`,
not `created`). In the UI: AI → AI Automations lists the three pipelines (one Active/confirm).

## Authoring order

`demo/demo_agent.xml` before `demo/demo_automation.xml` (the Summarise automation refs the
Schema Explorer agent). Within `demo_automation.xml`, each `agent.automation` precedes its
`agent.automation.action` rows (the actions `ref=` the parent's id).

---

*Back to [demo-usecase index](../../README.md).*
