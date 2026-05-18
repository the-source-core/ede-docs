<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Assistant — Implementation Docs

**Module:** `foundation.assistant` (`src/ede/foundation/assistant/`)
**Roadmap:** [roadmap/foundation/assistant/README.md](../roadmap/foundation/assistant/README.md)
**Status:** 🟡 In Progress — Phase 1 ✅ Delivered 2026-05-17 end-to-end (all 12 WS shipped, 48 pytest + 26 vitest green, manual smoke gate at [`SMOKE_TEST.md`](../src/ede/foundation/assistant/SMOKE_TEST.md)); [Enhancement 01](../roadmap/foundation/assistant/enhancements/01-org-default-provider.md) Org-Level Default Provider ✅ Delivered 2026-05-18 (consumes `foundation.base` Phase 2 SDK); Phases 2 / 3 / 4 🔴 Not Started. Hard prereqs ✅: [`foundation.ai`](foundation-ai.md) Phase 1, [`foundation.security`](foundation-security.md) Phase 5
**Layer:** Foundation engine — *use-case consumer of [`foundation.ai`](foundation-ai.md)*

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A chat companion that reads the user's current screen context and turns natural-language questions into either (a) view-state changes applied in real time, (b) a chat answer, or (c) click-to-confirm action buttons — never a database mutation. Phase 1's hard contract is **read-only**.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Operators need a faster path between "I have a question about my data" and "I'm looking at the answer in the UI I already know". The assistant binds AI to the existing webclient's `ActionContext` reducer so AI-proposed changes (filter / sort / groupby / open record) are indistinguishable from the user typing them — and identically reversible. Safety is structural: AI emits view-intent ops or chat text, never SQL or commands; record-rule engine still gates every read.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX entry points:** chat icon visible on any action (contextual mode) + global chat launcher in the app shell (no anchored screen).
- **Programmatic entry points for consumer modules:** ship an `assistant.skill.pack` row to bundle tools + a prompt template + example questions for a use case. Decorate richer informer commands with `@api.ai_tool(kind="informer", description=...)`.
- **Integration boundary:** produces view-intent ops + chat text + action-button payloads. Consumes [`foundation.ai`](foundation-ai.md)'s tool registry, prompt registry, conversation primitives.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
The chat panel is a side companion to the active `ActionContext`. AI-issued view-intent ops dispatch into the **same reducer** the SearchPanel uses, so AI-applied filter chips/sort/groupby render identically to human-typed ones (small ✨ marker distinguishes provenance). Three tool kinds — proposer (auto-applied to screen), informer (chat-only metadata), insight (chat-only read-side queries) — registered with [`foundation.ai`](foundation-ai.md) under `read_only=True`.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `assistant.session` | Per-user chat session, optional anchor on action | [session.py](../src/ede/foundation/assistant/models/session.py) |
| `assistant.view_intent.log` | Audit row per AI-emitted view-intent op | [view_intent_log.py](../src/ede/foundation/assistant/models/view_intent_log.py) |
| `assistant.skill.pack` | Assistant-specific bundle (P1 stub, P2 populated) | [skill_pack.py](../src/ede/foundation/assistant/models/skill_pack.py) |
| `assistant.tool` | Carrier that hosts `@api.ai_tool` methods | [assistant_tools.py](../src/ede/foundation/assistant/tools/assistant_tools.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `send_turn` | One-turn orchestrator: ToolBridge → ViewIntentOps composer | [services/turn_service.py](../src/ede/foundation/assistant/services/turn_service.py) |
| `AssistantController` | 5 HTTP routes for session lifecycle | [api/assistant_routes.py](../src/ede/foundation/assistant/api/assistant_routes.py) |
| `AssistantTool` | Carrier hosting 5 Phase 1 tool methods | [tools/assistant_tools.py](../src/ede/foundation/assistant/tools/assistant_tools.py) |
| `assistantApi.ts` | Frontend fetch wrappers (panel + reducer ops still deferred) | [src/frontend/.../assistant/assistantApi.ts](../src/frontend/src/workspace/views/assistant/assistantApi.ts) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `assistant.tool.propose_filter` | Bridge tool-call (proposer) | Returns `{op:"apply_filter", payload:{domain}}` for ActionContext |
| `assistant.tool.propose_groupby` | Bridge tool-call (proposer) | Returns `{op:"set_groupby", payload:{field}}` |
| `assistant.tool.propose_sort` | Bridge tool-call (proposer) | Returns `{op:"set_sort", payload:{field, direction}}` |
| `assistant.tool.explain_current_view` | Bridge tool-call (informer) | Returns chat-friendly view summary text |
| `assistant.tool.run_read_group` | Bridge tool-call (insight) | Dispatches `ede.read_group`; record-rule engine gates the read |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none yet_ — planned in Phase 1: `assistant.session.started`, `assistant.message.posted`, `assistant.view_intent.applied`, `assistant.action.confirmed` | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| POST `/api/assistant/session` | Start a session (contextual or global) | `AssistantController.start_session` |
| GET `/api/assistant/session/{id}` | Read full transcript | `AssistantController.read_session` |
| POST `/api/assistant/session/{id}/message` | Send a user message → `AssistantMessageResponse` (one-shot, non-streaming) | `AssistantController.send_message` |
| POST `/api/assistant/session/{id}/message/stream` | Stream a turn over SSE — emits `started` / `text_delta` / `complete` / `error` events as `data: {json}\n\n` frames (Enhancement 02) | `AssistantController.send_message_stream` |
| POST `/api/assistant/session/{id}/intent-applied` | Client callback after reducer dispatch (best-effort) | `AssistantController.intent_applied` |
| POST `/api/assistant/session/{id}/close` | Mark session closed | `AssistantController.close_session` |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none yet_ — Phase 1 may register `pre.assistant.session.send_message` for rate-limit / quota enforcement |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_ — sessions are conversational; no persistent state machine.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry: `assistant`
- Manifest `depends`: `["base", "auth", "ai", "presentation", "security"]`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `ASSISTANT_ENABLED` | bool | `False` | `ASSISTANT_ENABLED` | Master switch — when False, every `/api/assistant/*` route returns 403. |
| `ASSISTANT_DEFAULT_AUTO_APPLY` | bool | `True` | `ASSISTANT_DEFAULT_AUTO_APPLY` | Tenant default for auto-applying proposer-tool ops vs surfacing as click-to-confirm buttons. |
| `ASSISTANT_PANEL_DEFAULT_WIDTH_PX` | int | `420` | `ASSISTANT_PANEL_DEFAULT_WIDTH_PX` | Default AssistantPanel side-slide width (user resize persisted in localStorage). |
| `ASSISTANT_MAX_VIEW_INTENT_OPS_PER_TURN` | int | `6` | `ASSISTANT_MAX_VIEW_INTENT_OPS_PER_TURN` | Cap on view-intent ops emitted per turn — defends against runaway proposer output. |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `assistant.default_provider` | organization | many2one(`ai.provider.config`) | _AI_DEFAULT_PROVIDER fallback_ | Active org's pinned AI provider — `related_field` bound to `res.organization.default_ai_assistant_provider_id` so the Settings panel and the org form view share one column (Enhancement 01). |
| `assistant.alias_name` | organization | char | `Lucy` | Self-introduction name. Splices into the system prompt; if the user asks the assistant's name, the LLM uses this value. (Enhancement 02) |
| `assistant.tone` | organization | selection (`friendly` / `professional` / `casual` / `concise`) | `friendly` | Voice / tone hint blended into the system prompt. (Enhancement 02) |
| `assistant.response_strategy` | organization | selection (`concise` / `balanced` / `detailed` / `step_by_step`) | `balanced` | Answer length / structure hint blended into the system prompt. (Enhancement 02) |
| `assistant.reply_language` | organization | selection (`en` / `es` / `fr` / `de` / `pt` / `hi` / `zh` / `ja` / `vi` / `auto`) | `auto` | Pins the reply language regardless of the user's input language; `auto` preserves the legacy match-user-language behaviour. (Enhancement 02) |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| Settings → Platform AI → Assistant | [src/ede/foundation/assistant/views/assistant_settings.xml](../src/ede/foundation/assistant/views/assistant_settings.xml) | AI service (`assistant.default_provider`), Assistant name (`assistant.alias_name`), Tone (`assistant.tone`), Answer style (`assistant.response_strategy`), Reply language (`assistant.reply_language`) — Enhancements 01 + 02. End-user-friendly help text on every field. |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| _none yet_ — planned in Phase 2: seeded skill packs (registry-explorer, filter-builder, reporting-copilot) | |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Chat Panel + View-Intent Protocol + First Proposer / Informer / Insight Tools | ✅ Delivered 2026-05-17 — all 12 WS shipped end-to-end | [phase-1-implementation.md](../roadmap/foundation/assistant/phase-1-implementation.md) |
| Phase 2 | Full Read-Only Surface + Action Buttons + Skill Packs | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/assistant/phase-2-implementation.md) |
| Phase 3 | Configurations & Properties Assistant (Read-Only) | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/assistant/phase-3-implementation.md) |
| Phase 4 | Gated Write-Mode + Domain Skill Packs | 🔴 Not Started | [phase-4-implementation.md](../roadmap/foundation/assistant/phase-4-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Phase 1 (all 12 WS) — chat panel + view-intent protocol + first proposer / informer / insight tools | `assistant.session`, `assistant.view_intent.log`, `assistant.skill.pack`, `assistant.tool` | [models/](../src/ede/foundation/assistant/models/), [tools/assistant_tools.py](../src/ede/foundation/assistant/tools/assistant_tools.py), [api/assistant_routes.py](../src/ede/foundation/assistant/api/assistant_routes.py), [services/turn_service.py](../src/ede/foundation/assistant/services/turn_service.py), [AssistantPanel.tsx](../src/frontend/src/workspace/views/assistant/AssistantPanel.tsx), [WorkspaceActionController.tsx](../src/frontend/src/workspace/components/action/WorkspaceActionController.tsx) (`applyAssistantOps`), [ControlPanelSearchView.tsx](../src/frontend/src/workspace/components/search/ControlPanelSearchView.tsx) (✨ marker), [SMOKE_TEST.md](../src/ede/foundation/assistant/SMOKE_TEST.md) | [phase-1-implementation.md](../roadmap/foundation/assistant/phase-1-implementation.md) |
| Enhancement 01 — Org-level default assistant provider | `res.organization.default_ai_assistant_provider_id` (added via `@api.extend_model`) | [models/organization_extension.py](../src/ede/foundation/assistant/models/organization_extension.py), [services/turn_service.py](../src/ede/foundation/assistant/services/turn_service.py) (`_resolve_provider_name`), [views/res_organization_ai_extension.xml](../src/ede/foundation/assistant/views/res_organization_ai_extension.xml), [views/assistant_settings.xml](../src/ede/foundation/assistant/views/assistant_settings.xml), [migrations/.../c7e3a91b5d2f_org_default_provider.py](../src/ede/foundation/assistant/migrations/versions/c7e3a91b5d2f_org_default_provider.py) | [enhancements/01-org-default-provider.md](../roadmap/foundation/assistant/enhancements/01-org-default-provider.md) |
| Enhancement 02 — Real-time streaming + per-org preferences + chat UX polish | `assistant.alias_name`, `assistant.tone`, `assistant.response_strategy`, `assistant.reply_language` (ir.config) | [services/provider/base.py](../src/ede/foundation/ai/services/provider/base.py) (`stream()` Protocol + `StreamEvent` + `fallback_stream_from_invoke`), [services/provider/anthropic.py](../src/ede/foundation/ai/services/provider/anthropic.py) (`stream()` via `client.messages.stream()`), [services/provider/openai.py](../src/ede/foundation/ai/services/provider/openai.py) (faux-stream wrapper), [services/bridge.py](../src/ede/foundation/ai/services/bridge.py) (`invoke_streaming` + extracted `_run_iteration_tool_calls`), [services/turn_service.py](../src/ede/foundation/assistant/services/turn_service.py) (`send_turn_streaming` + `_build_system_prompt` + `_read_ir_config_value`), [api/assistant_routes.py](../src/ede/foundation/assistant/api/assistant_routes.py) (`send_message_stream` SSE route + `session.conversation_id.id` deref), [views/assistant_settings.xml](../src/ede/foundation/assistant/views/assistant_settings.xml), [src/ede/core/services/settings/service.py](../src/ede/core/services/settings/service.py) (`_settings_org_id` helper), [AssistantPanel.tsx](../src/frontend/src/workspace/views/assistant/AssistantPanel.tsx), [assistantApi.ts](../src/frontend/src/workspace/views/assistant/assistantApi.ts) (`sendMessageStream`), [types.ts](../src/frontend/src/workspace/views/assistant/types.ts) (`AssistantStreamEvent`) | [enhancements/02-streaming-and-org-preferences.md](../roadmap/foundation/assistant/enhancements/02-streaming-and-org-preferences.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| _none — Phase 1 ✅ end-to-end; first gap row will land on Phase 2 kickoff_ | | |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
_(none yet — populated as integration learnings emerge)_

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
_none yet — schema lands in Phase 1._
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `assistant_user` | `assistant.session.{start,read,message,close}`, `assistant.view_intent.log.read`, `assistant.skill.pack.read` — own sessions only |
| `assistant_admin` | inherits `assistant_user` + `assistant.skill.pack.{manage,update,delete}` |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.ai`](foundation-ai.md) — hard prerequisite
- [`foundation.mcp`](foundation-mcp.md) — sibling protocol-exposure module
- [`foundation.security`](foundation-security.md) — Phase 5 record-rule engine gates insight tool reads
- [`foundation.customization`](foundation-customization.md) — Phase 3 custom-field advisor reads schema from here
- [`foundation.workflow`](foundation-workflow.md) / [`foundation.approval`](foundation-approval.md) — Phase 3 explainer tools read these configs
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-18 (Enhancement 02 ✅ Delivered — real-time SSE streaming end-to-end + per-org assistant preferences `alias_name` / `tone` / `response_strategy` / `reply_language` + chat UX overhaul + `session.conversation_id.id` deref fix + `SettingsService._settings_org_id` uses JWT `active_organization_id`; 519 vitest + 87 pytest green. Enhancement 01 ✅ Delivered earlier same day). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
