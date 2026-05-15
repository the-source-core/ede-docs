<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Assistant — Implementation Docs

**Module:** `foundation.assistant` (`src/ede/foundation/assistant/`)
**Roadmap:** [roadmap/foundation/assistant/README.md](../roadmap/foundation/assistant/README.md)
**Status:** 🔴 Not Started — roadmap drafted 2026-05-15
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
| _none yet_ — planned in Phase 1: `assistant.session`, `assistant.view_intent.log`, `assistant.skill.pack` | | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ — planned in Phase 1: `AssistantService` (turn orchestrator), `ViewIntentComposer`, `AssistantPanel.tsx` (React), `assistantApi.ts` (frontend service) | | |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none yet_ — planned in Phase 1: `assistant.session.start`, `assistant.session.send_message`, `assistant.session.close` | | |
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
| _none yet_ — planned in Phase 1: `POST /api/assistant/session`, `POST /api/assistant/session/{id}/message`, `GET /api/assistant/session/{id}` | | |
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
| _none yet_ — planned in Phase 1: `ASSISTANT_ENABLED` (bool, False), `ASSISTANT_DEFAULT_AUTO_APPLY` (bool, True), `ASSISTANT_PANEL_DEFAULT_WIDTH_PX` (int, 420) | | | | |
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
| _none yet_ — planned in Phase 2: Settings → AI → Assistant for skill-pack enable matrix | | |
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
| Phase 1 | Chat Panel + View-Intent Protocol + First Proposer / Informer / Insight Tools | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/assistant/phase-1-implementation.md) |
| Phase 2 | Full Read-Only Surface + Action Buttons + Skill Packs | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/assistant/phase-2-implementation.md) |
| Phase 3 | Configurations & Properties Assistant (Read-Only) | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/assistant/phase-3-implementation.md) |
| Phase 4 | Gated Write-Mode + Domain Skill Packs | 🔴 Not Started | [phase-4-implementation.md](../roadmap/foundation/assistant/phase-4-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| _none yet_ — module roadmap drafted 2026-05-15; phase-1 implementation not started | 🔴 | [roadmap/foundation/assistant/README.md](../roadmap/foundation/assistant/README.md) |
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
| _none yet_ — planned in Phase 1: `assistant.user` (start/use sessions), `assistant.admin` (manage skill packs + global prompt overrides) |
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

*Last sync: 2026-05-15. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
