<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation MCP — Implementation Docs

**Module:** `foundation.mcp` (`src/ede/foundation/mcp/`)
**Roadmap:** [roadmap/foundation/mcp/README.md](../roadmap/foundation/mcp/README.md)
**Status:** 🔴 Not Started — roadmap drafted 2026-05-15
**Layer:** Foundation engine — *protocol-exposure consumer of [`foundation.ai`](foundation-ai.md)*

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A Model Context Protocol server that publishes the EDE platform's AI surface (tools, resources, prompts) over [MCP](https://modelcontextprotocol.io), a vendor-neutral standard for connecting LLMs to tools and data. External MCP clients (Claude Desktop, Claude Code, IDE plugins, third-party agents) authenticate as an EDE user and consume the same curated, read-only tool surface the in-app assistant uses — with the same record-rule gating and the same cost / audit pipeline.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every AI-driven feature otherwise needs its own protocol adapter. Because [`foundation.ai`](foundation-ai.md)'s tool registry is already self-describing, MCP can publish it directly — zero per-tool MCP code, zero per-feature protocol glue. Annotating any command with `@api.ai_tool` makes it MCP-callable the moment `foundation.mcp` is active.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX entry points:** _none_ in-app. Users generate MCP tokens from Settings → AI → MCP and configure their external MCP client.
- **Programmatic entry points:** standard MCP wire over HTTP-SSE (Phase 1) or stdio (Phase 3). Clients discover capabilities via the protocol handshake.
- **Integration boundary:** produces an MCP tool/resource/prompt surface. Consumes [`foundation.ai`](foundation-ai.md) registries + the existing JWT auth pipeline + [`foundation.security`](foundation-security.md) Phase 5 record-rule gating.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
External clients (Claude Desktop / IDE / agent) connect over HTTP-SSE (or stdio in Phase 3). Authenticated via the existing JWT pipeline. The capability publisher reads `foundation.ai`'s tool, prompt, and conversation registries, applies per-principal RBAC filters + `read_only=True` filter, and returns the MCP catalogue. Tool calls dispatch through the same `foundation.ai` function-calling bridge — same hook chain, same audit, same `ai.usage.log` cost row.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none yet_ — planned in Phase 1: `mcp.client`, `mcp.session` | | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ — planned in Phase 1: `MCPHttpSSEController`, `MCPCapabilityService`, `MCPAuthBridge`, `MCPToolDispatcher`. Phase 3 adds `MCPStdioLauncher`. | | |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none yet_ — planned in Phase 1: `mcp.session.open`, `mcp.session.close`, `mcp.token.issue`, `mcp.token.revoke` | | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none yet_ — planned in Phase 1: `mcp.session.opened`, `mcp.session.closed`, `mcp.tool.invoked` (mirrors `ede.ai.invoked` with mcp source) | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none yet_ — planned in Phase 1: `POST /mcp/sse` (transport), `GET /mcp/capabilities` (diagnostics) | | |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none yet_ — Phase 1 reuses `foundation.ai`'s `pre.ai.tool_call` / `post.ai.tool_call` chain; no MCP-specific hooks. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_ — sessions are short-lived; no persistent state machine.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry: `mcp`
- Manifest `depends`: `["base", "auth", "ai", "security"]`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none yet_ — planned in Phase 1: `MCP_ENABLED` (bool, False), `MCP_HTTP_SSE_PATH` (str, `/mcp/sse`), `MCP_ALLOWED_CLIENTS` (list, empty = all), `MCP_TOKEN_DEFAULT_TTL_MINUTES` (int, 60) | | | | |
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
| _none yet_ — planned in Phase 1: Settings → AI → MCP for enable flag + token issuance UI | | |
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
| Phase 1 | HTTP-SSE Server + Auto-Tool Publishing + Auth Bridge (Read-Only) | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/mcp/phase-1-implementation.md) |
| Phase 2 | Resources + Prompts | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/mcp/phase-2-implementation.md) |
| Phase 3 | stdio Transport (Single-User Local) | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/mcp/phase-3-implementation.md) |
| Phase 4 | Gated Write-Mode + Approval Integration | 🔴 Not Started | [phase-4-implementation.md](../roadmap/foundation/mcp/phase-4-implementation.md) |
| Phase 5 | Capability Negotiation & Tool Composition Metadata | 🔴 Not Started | [phase-5-implementation.md](../roadmap/foundation/mcp/phase-5-implementation.md) |
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
| _none yet_ — module roadmap drafted 2026-05-15; phase-1 implementation not started | 🔴 | [roadmap/foundation/mcp/README.md](../roadmap/foundation/mcp/README.md) |
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
| _none yet_ — planned in Phase 1: `mcp.user` (issue own tokens, open sessions), `mcp.admin` (manage allowed clients, see all sessions) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.ai`](foundation-ai.md) — hard prerequisite
- [`foundation.assistant`](foundation-assistant.md) — sibling consumer (in-app chat panel)
- [`foundation.security`](foundation-security.md) — Phase 5 record-rule engine gates MCP-issued reads
- [`foundation.auth`](foundation-auth.md) — JWT pipeline reused for MCP authentication
- [Model Context Protocol specification](https://modelcontextprotocol.io)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-15. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
