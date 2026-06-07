<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Assistant — Implementation Docs

**Module:** `foundation.assistant` (`src/ede/foundation/assistant/`)
**Roadmap:** [roadmap/foundation/assistant/README.md](../roadmap/foundation/assistant/README.md)
**Status:** 🟡 In Progress — Phase 1 ✅ Delivered 2026-05-17 end-to-end (all 12 WS shipped, 48 pytest + 26 vitest green, manual smoke gate at [`SMOKE_TEST.md`](../src/ede/foundation/assistant/SMOKE_TEST.md)); [Enhancement 01](../roadmap/foundation/assistant/enhancements/01-org-default-provider.md) Org-Level Default Provider ✅ Delivered 2026-05-18 (consumes `foundation.base` Phase 2 SDK); [Enhancement 02](../roadmap/foundation/assistant/enhancements/02-streaming-and-org-preferences.md) Streaming + Per-Org Preferences + Chat UX ✅ Delivered 2026-05-18; [Enhancement 03](../roadmap/foundation/assistant/enhancements/03-action-sessions-history-and-composer.md) Action-Anchored Session History, Auto-Scope Detection & Composer Redesign ✅ Delivered 2026-05-18 (62 new pytest + 12 new vitest green; Playwright e2e deferred to follow-up); [Phase 2](../roadmap/foundation/assistant/phase-2-implementation.md) Full Read-Only Surface + Action Buttons + Skill Packs ✅ Delivered 2026-05-18 (all 7 WS shipped same-day across 3 commits — 2264 pytest + 556 vitest green; 4 polish slices documented as 🟢 deferred); Phases 3 / 4 🔴 Not Started. Hard prereqs ✅: [`foundation.ai`](foundation-ai.md) Phase 1, [`foundation.security`](foundation-security.md) Phase 5
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
| `assistant.history.page_size` | organization | integer | `25` | Page size for `GET /api/assistant/sessions`. Server caps at 100. (Enhancement 03) |
| `assistant.composer.draft_warn_chars` | organization | integer | `800` | Character threshold at which the composer shows a soft length warning. (Enhancement 03) |
| `assistant.auto_scope.enabled` | organization | boolean | `true` | When `true`, contextual sessions see the union of model-scoped + global tool surfaces (the assistant can answer global questions on a screen without switching to global chat). When `false`, contextual sessions stay strictly model-scoped (Phase-1-style behaviour). (Enhancement 03) |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| Settings → Platform AI → Assistant — section "Assistant" | [src/ede/foundation/assistant/views/assistant_settings.xml](../src/ede/foundation/assistant/views/assistant_settings.xml) | AI service (`assistant.default_provider`), Assistant name (`assistant.alias_name`), Tone (`assistant.tone`), Answer style (`assistant.response_strategy`), Reply language (`assistant.reply_language`) — Enhancements 01 + 02. End-user-friendly help text on every field. |
| Settings → Platform AI → Assistant — section "Conversation history & composer" | [src/ede/foundation/assistant/views/assistant_settings.xml](../src/ede/foundation/assistant/views/assistant_settings.xml) | History page size (`assistant.history.page_size`), Message length warning threshold (`assistant.composer.draft_warn_chars`), Answer global questions on a screen (`assistant.auto_scope.enabled`) — Enhancement 03. |
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
| Phase 2 | Full Read-Only Surface + Action Buttons + Skill Packs | ✅ Delivered 2026-05-18 | [phase-2-implementation.md](../roadmap/foundation/assistant/phase-2-implementation.md) |
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
| Enhancement 03 — Action-anchored session history, auto-scope detection & composer redesign | `assistant.session.organization_id`, `assistant.session.title`, `assistant.session.is_starred`; `assistant.history.page_size`, `assistant.composer.draft_warn_chars`, `assistant.auto_scope.enabled` (ir.config) | [models/session.py](../src/ede/foundation/assistant/models/session.py) (3 new columns), [migrations/versions/e3d5c7a91f4b_enh03_session_lifecycle.py](../src/ede/foundation/assistant/migrations/versions/e3d5c7a91f4b_enh03_session_lifecycle.py) (chained off c7e3a91b5d2f; backfills organization_id ← tenant_id), [services/session_service.py](../src/ede/foundation/assistant/services/session_service.py) (`resume_or_create_session` + multi-row self-heal), [services/turn_service.py](../src/ede/foundation/assistant/services/turn_service.py) (`_CONTEXTUAL_TOOLS` / `_GLOBAL_TOOLS` / `_build_tool_whitelist` / `_derive_intent_scope` / `_read_auto_scope_enabled` + `force_global` parameter on `send_turn` + `send_turn_streaming` + `intent_scope` on response envelope + streaming `complete` event), [api/assistant_routes.py](../src/ede/foundation/assistant/api/assistant_routes.py) (`POST /session` resume opt-in, `POST /archive`, `POST /star`, `POST /rename`, `GET /sessions`), [schemas.py](../src/ede/foundation/assistant/schemas.py) (`RenameSessionRequest`, `intent_scope` on `AssistantMessageResponse`, `force_global` on `SendMessageRequest`), [views/assistant_settings.xml](../src/ede/foundation/assistant/views/assistant_settings.xml) ("Conversation history & composer" section), [AssistantComposer.tsx](../src/frontend/src/workspace/views/assistant/AssistantComposer.tsx), [SlashPalette.tsx](../src/frontend/src/workspace/views/assistant/SlashPalette.tsx), [AssistantHistoryPanel.tsx](../src/frontend/src/workspace/views/assistant/AssistantHistoryPanel.tsx), [AssistantNavigationGuard.tsx](../src/frontend/src/workspace/views/assistant/AssistantNavigationGuard.tsx), [AssistantPanel.tsx](../src/frontend/src/workspace/views/assistant/AssistantPanel.tsx) (composer/history/guard wiring + `intent_scope` badge), [assistantApi.ts](../src/frontend/src/workspace/views/assistant/assistantApi.ts) (`archiveSession` / `starSession` / `renameSession` / `listSessions`), [types.ts](../src/frontend/src/workspace/views/assistant/types.ts) (`RenameSessionRequest`, `AssistantSessionListItem`, `AssistantSessionListResponse`, `intent_scope` on stream / response types, `force_global` on `SendMessageRequest`) | [enhancements/03-action-sessions-history-and-composer.md](../roadmap/foundation/assistant/enhancements/03-action-sessions-history-and-composer.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Full Playwright browser-driven e2e for the session-lifecycle flow (open panel on Leads → switch screen → confirm dialog → resume). Substitute coverage shipped via `test_lifecycle_integration_enh03.py` (13 controller-level cases through a stubbed env) + the manual `SMOKE_TEST.md` flow. | 🟢 Low backlog | [enhancements/03-action-sessions-history-and-composer.md](../roadmap/foundation/assistant/enhancements/03-action-sessions-history-and-composer.md) — Acceptance Criteria deferral note |
| Router-level "Stay" path for `AssistantNavigationGuard` — today the guard offers Keep / Discard, since TanStack Router state is already past the click by the time the panel re-binds. A workspace-shell-level menu-click intercept that can roll back the navigation is tracked for a polish slice. | 🟢 Low backlog | [enhancements/03-action-sessions-history-and-composer.md](../roadmap/foundation/assistant/enhancements/03-action-sessions-history-and-composer.md) |
| **`src/frontend` panel port (WS 5.6 AIAssistantManager)** — the new workspace client has zero AI chat surface; the legacy `src/frontend/src/workspace/views/assistant/` (12 files, 2,484 LOC) carries the working panel against the same `/api/assistant/*` endpoints. This enhancement ports it 1:1 as a WorkspaceShell push-layout slot (sibling of action area, drag-resizable per-user, persisted in localStorage), with `<AssistantTrigger/>` in the top bar (left of UserPreference avatar). History becomes a Radix Popover from the panel header (replaces legacy in-panel sub-panel). Greeting uses the active record's `display_name` for highest-signal context. Slash palette dynamically folds `assistant.skill_pack` entries via `GET /api/assistant/skill-packs` at panel mount. `view_intent.ops` route through `usePlatformCommands` (APPLY_FILTER / SET_GROUPBY / SET_SORT / CLEAR_FILTERS / OPEN_ACTION / OPEN_RECORD / SET_COLUMNS) — replaces legacy window CustomEvents per the WS 5.6 mandate. SSE streaming via `fetch()` + `ReadableStream` (legacy pattern; `EventSource` can't carry custom Authorization headers). Read-only contract preserved verbatim — no Phase 4 write-mode UI in this slice. Two providers: `AssistantPanelProvider` (open/close/width) at the shell, `AssistantBindingProvider` (anchoredModelKey / anchoredActionKey / recordRef / recordDisplayName / viewState / applyOps) at ActionManager. Bulk-fetched skill packs + per-user `auto_apply_proposer_ops` preference cached for 5min via TanStack Query. ~2,000 LOC across 16 new frontend2 files + 7 small shell wire-ups + ~220 LOC of SCSS. 43 new vitest cases planned; zero new pytest (backend unchanged). **Future phase explicitly out of scope — AI Agents**: the v1 read-only safety contract holds (chat body / view-intent ops / click-to-confirm action buttons; no DB mutation). Agents land as `enhancements/05-agent-mode.md` when `foundation.assistant` Phase 4 ships gated write-mode + domain skill packs. | 🔴 Not Started (drafted 2026-06-04) | [enhancements/04-frontend2-panel-port.md](../roadmap/foundation/assistant/enhancements/04-frontend2-panel-port.md) |
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

*Last sync: 2026-06-04 (foundation.assistant Enhancement 04 — **`src/frontend` AssistantPanel Port (WS 5.6 AIAssistantManager) — 🔴 Not Started** authored at [`roadmap/foundation/assistant/enhancements/04-frontend2-panel-port.md`](../roadmap/foundation/assistant/enhancements/04-frontend2-panel-port.md). Closes UI revamp WS 5.6 by porting the legacy `src/frontend/src/workspace/views/assistant/` (12 files, 2,484 LOC) onto `src/frontend`. **Architecturally distinct from Enhancement 08's client-action lifecycle** — the assistant is a persistent WorkspaceShell slot (push-layout sibling of action area), NOT a Radix Dialog or `ClientActionPage` route. Same family as WS 4.6 FormAside chatter; not a client-action adopter. **Locked design** (Q1-Q8 all resolved with user): aligned-legacy push-layout panel · top-bar trigger left of UserPreference avatar · drag-resizable width with localStorage persistence (default 360px, min 280, max 720) · history as Radix Popover from panel header (not in-panel sub-panel) · `react-markdown` for assistant message bodies · read-only contract preserved (no Phase 4 write-mode UI) · greeting personalizes via active record's `display_name` (highest-signal context vs legacy's screen-name template) · slash palette dynamically folds `assistant.skill_pack` registry entries via `GET /api/assistant/skill-packs` at panel mount. **Two-context split**: `AssistantPanelProvider` (open/close/width) at shell level; `AssistantBindingProvider` (anchoredModelKey · anchoredActionKey · recordRef · recordDisplayName · viewState · applyOps) populated by ActionManager. Contextual vs global mode: binding absent → global chat, no proposer tools; binding present → proposer tools active. **`view_intent.ops` → platform commands** per WS 5.6 mandate: each of `apply_filter` / `set_groupby` / `set_sort` / `clear_filters` / `switch_view` / `open_record` / `show_columns` maps to an existing `usePlatformCommands` dispatch (APPLY_FILTER / SET_GROUPBY / SET_SORT / CLEAR_FILTERS / OPEN_ACTION / OPEN_RECORD / SET_COLUMNS); replaces legacy `window.dispatchEvent(new CustomEvent(...))` so assistant-driven changes go through the same middleware (pending-track, error-toast, audit) as user-driven changes. **SSE streaming** via `fetch()` + `ReadableStream` (legacy pattern; `EventSource` doesn't support custom Authorization headers); cancellable via `AbortController` for the nav guard. **Auto-apply gate**: ops apply automatically when `assistant.user_preference.auto_apply_proposer_ops === true` AND contextual mode; otherwise rendered as click-to-confirm chips below the message body. **File inventory**: 12 new frontend2 manager files + `AssistantService` + `IAssistantService` + 2 provider files + 1 SCSS file + 7 small shell wire-ups (WorkspaceShell mounts panel as right sibling; MainMenuManager mounts trigger; ActionManager publishes binding). **Backend changes**: zero — Phases 1+2 + Enhancements 01/02/03 cover everything; wire shapes in `types.ts` mirror `schemas.py` 1:1. **Tests planned**: 43 new vitest cases (trigger toggle/aria · push-layout/resize/persistence · composer Cmd+Enter/slash · history popover · SlashPalette built-ins+skill-pack fold-in · greeting personalization · AssistantService SSE parsing + abort + envelope errors · view_intent op→command mapping per op kind). **Out of scope** (explicit, with future-phase tracker): **AI Agents** — the next major capability where the user asks the assistant to PERFORM operations in the same chat window. Today's v1 contract is strictly read-only (chat text / view-intent ops / click-to-confirm action buttons; no DB mutation). Agents land as `roadmap/foundation/assistant/enhancements/05-agent-mode.md` when `foundation.assistant` Phase 4 (gated write-mode + domain skill packs) ships; they reuse 100% of this enhancement's panel chrome and add a new `agent_step` message kind plus per-step confirmation gate. Also explicitly out of scope: Phase 3 Configurations & Properties Assistant, cross-record activity feed (WS 5.4), multi-language assistant (waits on i18n catalogs), mobile/responsive collapsing (Phase 6.5 polish), voice input. Known Gaps gains the matching 🔴 row. Previous sync: 2026-05-18 (Enhancement 03 ✅ Delivered — Action-Anchored Session History, Auto-Scope Detection & Composer Redesign: schema additions on `assistant.session` + Alembic `e3d5c7a91f4b` + `resume_or_create_session` service helper + 4 new lifecycle routes + per-turn auto-scope routing with `intent_scope` envelope field + `AssistantComposer` / `SlashPalette` / `AssistantHistoryPanel` / `AssistantNavigationGuard` React components + `intent_scope` bubble badge + 3 new org-scoped ir.config keys; 62 new pytest + 12 new vitest green; 2182 pytest + 543 vitest total; Built Capabilities row added; Section 3 ir.config + XML panels updated; Known Gaps updated to track the deferred Playwright e2e + router-level Stay path. Prior same-day syncs: Enhancement 02 ✅ Delivered — real-time SSE streaming + per-org preferences + chat UX overhaul; Enhancement 01 ✅ Delivered earlier same day). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
