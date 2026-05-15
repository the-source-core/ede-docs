<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation AI — Implementation Docs

**Module:** `foundation.ai` (`src/ede/foundation/ai/`)
**Roadmap:** [roadmap/foundation/ai/README.md](../roadmap/foundation/ai/README.md)
**Status:** 🔴 Not Started — roadmap drafted 2026-05-15
**Layer:** Foundation engine — *capability service*

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A provider-agnostic, config-driven AI capability layer that turns EDE's existing command bus, registry, hooks, event stream, and view DSL into a first-class LLM-accessible surface — without forcing every consumer to re-implement provider clients, prompt management, safety pipelines, or cost tracking. Consumer modules ([`foundation.assistant`](foundation-assistant.md), [`foundation.mcp`](foundation-mcp.md), future copilot / search / agentic) build their UX or protocol on top.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every AI-driven feature otherwise re-invents six concerns: provider client + retry, prompt versioning, tool exposure + safety filtering, conversation transcript, cost tracking, audit. Six consumers × six concerns = thirty-six ad-hoc implementations that drift in policy. The platform owns it once; consumers register one `@api.ai_tool` decorator (per command) and one `ai.skill.pack` row (per use case) and get all six for free.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- End-user UX entry points: _none_ in this module — UX surfaces ship in [`foundation.assistant`](foundation-assistant.md). Settings → AI provides admin-only configuration (prompts, tools, skill packs, budgets, usage logs).
- Programmatic entry points: `POST /api/ai/invoke` (single-turn) and `POST /api/ai/conversations/{id}/messages` (multi-turn) for consumer modules; `@api.ai_tool(...)` decorator to register a command as AI-invocable; `env.ai_invoke(...)` for in-process consumers.
- Integration boundary — produces: a curated tool surface, prompt registry, conversation transcripts, cost log, event stream (`ede.ai.invoked`). Consumes: existing command bus, registry, view DSL, principal/tenant, lifecycle hooks, [`foundation.security`](foundation-security.md) Phase 5 record-rule gating, [`foundation.jobs`](foundation-jobs.md) Phase 1 for Phase 3 embedding indexer.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
@api.ai_tool(read_only=True, ...)   ──►   ai.tool.registry (auto-scanned on boot)
on existing command handler
                                          │
LLM message arrives  ──►  ai.invoke  ──►  provider.run(prompt, tools)
                                          │
                                          ▼
                                    tool-call from LLM
                                          │
                                          ▼
                                    read-only gate ──►  env.dispatch(Command)
                                          │
                                          ▼
                                    response composed → ai.message
                                          │
                                          ▼
                                    cost + audit → ai.usage.log
                                    event stream → ede.ai.invoked
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none yet_ — planned in Phase 1: `ai.provider.config`, `ai.prompt.template`, `ai.prompt.version`, `ai.conversation`, `ai.message`, `ai.usage.log`, `ai.skill.pack`, `ai.safety.rule` | | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ — planned in Phase 1: `ProviderService` (pluggable backends), `ToolBridge` (LLM tool-call → `env.dispatch`), `PromptRegistry`, `CostService`, `SafetyHookPipeline` | | |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none yet_ — planned in Phase 1: `ede.ai.invoked`, `ede.ai.tool_called`, `ede.ai.budget_breach` | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none yet_ — planned in Phase 1: `POST /api/ai/invoke`, `POST /api/ai/conversations`, `POST /api/ai/conversations/{id}/messages`, `GET /api/ai/tools`, `GET /api/ai/usage` | | |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none yet_ — planned in Phase 1 (hookpoints, stubs only): `pre.ai.invocation`, `post.ai.invocation`, `pre.ai.tool_call`, `post.ai.tool_call`. Real PII / injection / output-validator implementations land in Phase 2. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_ — invocation is a single-turn request/response; no persistent state machine on `ai.conversation`.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `ai`
- Manifest `depends`: `["base", "auth", "security"]`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none yet_ — planned in Phase 1: `AI_ENABLED` (bool, default False), `AI_DEFAULT_PROVIDER` (str, default `anthropic`), `AI_DAILY_BUDGET_USD` (float, default 0.0 = unlimited), `AI_ALLOWED_PROVIDERS` (list, default `["anthropic"]`), `AI_BYO_KEY_REQUIRED` (bool, default True) | | | | |
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
| _none yet_ — planned in Phase 1: Settings → AI panel for provider config, budget, allowed tools | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Provider Abstraction + Tool/Prompt Registry + Conversations + Cost/Audit (Read-Only Bridge) | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/ai/phase-1-implementation.md) |
| Phase 2 | Safety Hardening (PII + Injection + Output Validators) | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/ai/phase-2-implementation.md) |
| Phase 3 | Embeddings + Semantic Primitives | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/ai/phase-3-implementation.md) |
| Phase 4 | Gated Write-Mode | 🔴 Not Started | [phase-4-implementation.md](../roadmap/foundation/ai/phase-4-implementation.md) |
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
| _none yet_ — module roadmap drafted 2026-05-15; phase-1 implementation not started | 🔴 | [roadmap/foundation/ai/README.md](../roadmap/foundation/ai/README.md) |
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
| _none yet_ — planned in Phase 1: `ai.user` (invoke ai.invoke + read own conversations), `ai.admin` (manage prompts / tools / skill packs / safety rules / usage), `ai.system_admin` (provider config) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.assistant`](foundation-assistant.md) — first user-facing consumer (chat companion to actions)
- [`foundation.mcp`](foundation-mcp.md) — protocol exposure of the same registries to external AI clients
- [`foundation.security`](foundation-security.md) — Phase 5 record-rule engine gates every AI-issued read
- [`foundation.jobs`](foundation-jobs.md) — Phase 1 substrate for Phase 3 embedding indexer
- [Foundation Model Naming](foundation-model-naming.md) — `ir.*` vs `res.*` vs `ai.*` conventions
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-15. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
