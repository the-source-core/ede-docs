<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation AI — Implementation Docs

**Module:** `foundation.ai` (`src/ede/foundation/ai/`)
**Roadmap:** [roadmap/foundation/ai/README.md](../roadmap/foundation/ai/README.md)
**Status:** 🟡 In Progress — Phases 1 / 2 / 4 ✅ Delivered (P1 on 2026-05-15, P2 + P4 on 2026-05-17); Phase 3 (Embeddings + Semantic) 🔴 ON HOLD pending `foundation.jobs` Phase 1; [Enhancement 01](../roadmap/foundation/ai/enhancements/01-standalone-app-and-views.md) ✅ Delivered 2026-05-15; [Enhancement 02](../roadmap/foundation/ai/enhancements/02-secrets-singleton-and-builder-method-proxy.md) ✅ Delivered 2026-05-18; [Enhancement 03](../roadmap/foundation/ai/enhancements/03-langchain-provider-adapter.md) Migrate All Providers to LangChain ✅ Delivered 2026-05-19 (5 slices A→E: `FoundationAIAdapter` Layer 5 over any LangChain `BaseChatModel` + Anthropic/OpenAI rewritten as thin builders + new first-class `OllamaProvider` + code-level `ProviderClassRegistry` keyed on `name` (Slice C's `client_class` Char design rejected same-day + dropped via follow-up migration `9a4e2b7d1c80` — net-new providers like Gemini/Bedrock/Cohere register via one-line `register_provider(...)` calls gated behind `try/except ImportError` in `services/provider/__init__.py`; tenants enable them via `[ai-gemini]`/`[ai-ollama]`/`[ai-bedrock]` extras + restart) + SSE wire-shape regression test + native SDK direct deps removed; 143 ai + 160 assistant tests green)
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
| `ai.provider.config` | Per-tenant LLM provider configuration (encrypted API key, default model, max-tokens, temperature). | [src/ede/foundation/ai/models/provider_config.py](../src/ede/foundation/ai/models/provider_config.py) |
| `ai.prompt.template` | Prompt identity (one `key` per tenant). Versioned via `ai.prompt.version`. | [src/ede/foundation/ai/models/prompt_template.py](../src/ede/foundation/ai/models/prompt_template.py) |
| `ai.prompt.version` | Versioned prompt content with system + user templates and declared variables. | [src/ede/foundation/ai/models/prompt_template.py](../src/ede/foundation/ai/models/prompt_template.py) |
| `ai.conversation` | Generic LLM transcript root, owned by a principal user; source enum disc­riminates assistant / mcp / api / copilot. | [src/ede/foundation/ai/models/conversation.py](../src/ede/foundation/ai/models/conversation.py) |
| `ai.message` | One turn of a conversation. Roles: system / user / assistant / tool_call / tool_result. | [src/ede/foundation/ai/models/conversation.py](../src/ede/foundation/ai/models/conversation.py) |
| `ai.usage.log` | Per-call cost / token / latency record with status enum (success / provider_error / bridge_rejected / budget_exceeded / read_only_violation). | [src/ede/foundation/ai/models/usage_log.py](../src/ede/foundation/ai/models/usage_log.py) |
| `ai.skill.pack` | Curated tool-whitelist + prompt + example-questions bundle. | [src/ede/foundation/ai/models/skill_pack.py](../src/ede/foundation/ai/models/skill_pack.py) |
| `ai.safety.rule` | Per-tenant safety rule (PII / injection / output validator). Records exist; behaviour stubbed for P1. | [src/ede/foundation/ai/models/safety_rule.py](../src/ede/foundation/ai/models/safety_rule.py) |
| `ai.tool` | Concrete carrier model — hosts built-in `@api.ai_tool` handlers (Phase 1: `read_schema`). | [src/ede/foundation/ai/tools/read_schema.py](../src/ede/foundation/ai/tools/read_schema.py) |
| `ai.user.quota` | Per-user override of tenant default budget + rate limits. Phase 2. | [src/ede/foundation/ai/models/user_quota.py](../src/ede/foundation/ai/models/user_quota.py) |
| `ai.provider.routing.rule` | Per-tenant rule mapping `(model_key, role) → provider`. Phase 2. | [src/ede/foundation/ai/models/provider_routing.py](../src/ede/foundation/ai/models/provider_routing.py) |
| `ai.write.provenance` | Phase 4 side-table — one row per AI-issued mutation, joining `(model_key, record_uuid)` to `ai_conversation_id` + `created_by_ai` + optional `approval_case_id` + `reverse_command`. Avoids cross-model auto-field migration. | [src/ede/foundation/ai/models/write_provenance.py](../src/ede/foundation/ai/models/write_provenance.py) |
| `ai.provider.config.param` | Extensible key-value child of `ai.provider.config` for provider-specific extras (Bedrock `aws_region`, Azure OpenAI `azure_deployment`, on-prem gateway `mtls_cert`, …). `is_secret=True` flows the value through the same Fernet encryption pipeline as `api_key_encrypted`. | [src/ede/foundation/ai/models/provider_config_param.py](../src/ede/foundation/ai/models/provider_config_param.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `ProviderContract` (Protocol) + `ProviderResponse` + `ProviderError` | Structural Protocol every backend implements. | [src/ede/foundation/ai/services/provider/base.py](../src/ede/foundation/ai/services/provider/base.py) |
| `AnthropicProvider` | Wraps the official `anthropic` SDK; per-model cost math via seeded pricing table. | [src/ede/foundation/ai/services/provider/anthropic.py](../src/ede/foundation/ai/services/provider/anthropic.py) |
| `ProviderRegistry` + `default_registry` | Process-wide name → provider lookup. | [src/ede/foundation/ai/services/provider/registry.py](../src/ede/foundation/ai/services/provider/registry.py) |
| `ToolRegistry` + `ToolSpec` + `validate_read_only` | Scans models for `@api.ai_tool` + `@api.on_command` pairs; boot-time read-only validator. | [src/ede/foundation/ai/services/tool_registry.py](../src/ede/foundation/ai/services/tool_registry.py) |
| `PromptRegistry` + `ResolvedPrompt` | DB-backed `(template_key, tenant_id) → ai.prompt.version` resolver with system-tenant fallback. | [src/ede/foundation/ai/services/prompt_registry.py](../src/ede/foundation/ai/services/prompt_registry.py) |
| `ToolBridge` + `BridgeConfig` + `BridgeResult` | Drives the provider → tool-call → command-bus loop with read-only enforcement and `AI_MAX_TOOL_ITERATIONS` cap. | [src/ede/foundation/ai/services/bridge.py](../src/ede/foundation/ai/services/bridge.py) |
| `CostService` | Per-tenant USD spend rollup over `ai.usage.log`. | [src/ede/foundation/ai/services/cost.py](../src/ede/foundation/ai/services/cost.py) |
| `QuotaService` + `BudgetExceeded` | Per-tenant daily budget enforcement (HTTP 429). | [src/ede/foundation/ai/services/quota.py](../src/ede/foundation/ai/services/quota.py) |
| `SafetyHookPipeline` + `default_pipeline` | In-memory hook chain for `pre/post .ai.invocation/.ai.tool_call`. Stubs in P1; real impls land in P2. | [src/ede/foundation/ai/services/safety.py](../src/ede/foundation/ai/services/safety.py) |
| `AIController` | HTTP surface — `/api/ai/{invoke,conversations,tools,usage,conversations/{id}/confirm,conversations/{id}/undo}`. | [src/ede/foundation/ai/api/ai_routes.py](../src/ede/foundation/ai/api/ai_routes.py) |
| `PIIRedactor` | Phase 2 WS-A11 — regex + field-path PII redaction, strict (round-trip) and lossy modes, idempotent. | [src/ede/foundation/ai/services/safety/pii.py](../src/ede/foundation/ai/services/safety/pii.py) |
| `PromptInjectionDetector` | Phase 2 WS-A12 — 10-signature pattern set + zero-width / RTL-override / base64-smuggling / length-explosion heuristics; threshold-scored. | [src/ede/foundation/ai/services/safety/injection.py](../src/ede/foundation/ai/services/safety/injection.py) |
| `OutputValidatorChain` | Phase 2 WS-A13 — domain-leak detector + field-name hallucination check + schema-conformance validator; warn-or-fail mode. | [src/ede/foundation/ai/services/safety/output_validator.py](../src/ede/foundation/ai/services/safety/output_validator.py) |
| `DataResidencyEnforcer` | Phase 2 WS-A16 — scans outgoing prompts for `__ede_residency_restricted__` model mentions and blocks unallowed providers. | [src/ede/foundation/ai/services/safety/residency.py](../src/ede/foundation/ai/services/safety/residency.py) |
| `ProviderRouter` | Phase 2 WS-A17 — evaluates `ai.provider.routing.rule` rows (model_key + role match) and picks the target provider per call. | [src/ede/foundation/ai/services/provider/router.py](../src/ede/foundation/ai/services/provider/router.py) |
| `OpenAIProvider` | Phase 2 second provider — guarded `openai` SDK import. | [src/ede/foundation/ai/services/provider/openai.py](../src/ede/foundation/ai/services/provider/openai.py) |
| `BudgetAlertService` | Phase 2 WS-A15 — fires `Command("notification.send")` at 50/80/100% of `AI_DAILY_BUDGET_USD`; once-per-threshold-per-day dedup. | [src/ede/foundation/ai/services/budget_alert.py](../src/ede/foundation/ai/services/budget_alert.py) |
| `QuotaService` (extended) | Phase 2 WS-A14 — per-user daily budget override + sliding 60s requests-per-minute + sliding 60-minute tokens-per-tool cap. | [src/ede/foundation/ai/services/quota.py](../src/ede/foundation/ai/services/quota.py) |
| `build_safety_runtime()` | Phase 2 wiring — builds a per-request `SafetyRuntime` bundle from settings + `ai.safety.rule` rows. | [src/ede/foundation/ai/services/safety/wiring.py](../src/ede/foundation/ai/services/safety/wiring.py) |
| `WriteApprovalBridge` | Phase 4 WS-A27 — checks `ir.approval.policy.set` matches; dispatches `ir.approval.case.request` instead of the raw command when an approval rule exists. | [src/ede/foundation/ai/services/write_mode.py](../src/ede/foundation/ai/services/write_mode.py) |
| `WriteBudgetService` | Phase 4 WS-A28 — per-user daily records cap + per-tool daily invocations cap (process-local sliding counters). | [src/ede/foundation/ai/services/write_mode.py](../src/ede/foundation/ai/services/write_mode.py) |
| `UndoWindow` | Phase 4 WS-A29 — looks up the latest un-reverted `ai.write.provenance` row and dispatches its `reverse_command` if within the configured window. | [src/ede/foundation/ai/services/write_mode.py](../src/ede/foundation/ai/services/write_mode.py) |
| `issue_confirmation_token` / `consume_confirmation_token` | Phase 4 WS-A25 — process-memory single-use token registry with TTL; minted by the bridge, consumed by `/confirm`. | [src/ede/foundation/ai/services/write_mode.py](../src/ede/foundation/ai/services/write_mode.py) |
| `SecretsService` | Fernet symmetric-encryption wrapper used by the `pre.ai.provider.config.{create,update}` hooks + by `ai.provider.config.param` (when `is_secret=True`). Idempotent on already-encrypted input; degrades to plaintext mode with a boot WARNING when `AI_PROVIDER_KEY_ENCRYPTION_SECRET` is unset. | [src/ede/foundation/ai/services/secrets.py](../src/ede/foundation/ai/services/secrets.py) |
| `build_provider_from_config()` | DB-driven provider construction — looks up the active `ai.provider.config` row (org-scoped first, then tenant-wide), decrypts `api_key_encrypted` + secret params, assembles the kwargs (filtered against the provider class's signature), returns a fresh provider instance per call. Replaces the previous `default_registry.get(name)` singleton lookup. | [src/ede/foundation/ai/services/provider/builder.py](../src/ede/foundation/ai/services/provider/builder.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ai.tool.read_schema` | LLM tool call via the bridge (tool name `read_schema`) | Reads the EDE registry definition of a model — fields, types, references, default order. Read-only by construction. |
| `ede.create` / `ede.read_one` / `ede.update` / `ede.delete` / `ede.search` / `ede.count` / `ede.read_group` (on every `ai.*` model) | Generic CRUD via `env.dispatch(...)` | Standard CRUD against the conversation / message / usage log / prompt / skill pack / safety rule tables. |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.ai.invoked` | Once per successful bridge invocation (after the LLM returns a non-tool-call final message). | Cost dashboards, downstream observers; future: alerting on per-call thresholds. |
| `ede.record.created` / `ede.record.updated` / `ede.record.deleted` (on every `ai.*` model) | CRUD against transcript / usage / prompt rows. | Generic event consumers. |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/ai/invoke` | Single-turn invocation; body `{user, system?, prompt_template_key?, tools?, model?, max_tokens?, temperature?}`. | [AIController.invoke](../src/ede/foundation/ai/api/ai_routes.py) |
| `POST /api/ai/conversations` | Start a new `ai.conversation` row. | [AIController.create_conversation](../src/ede/foundation/ai/api/ai_routes.py) |
| `POST /api/ai/conversations/{id}/messages` | Append a user message to an existing conversation; runs the bridge with `conversation_id` set. | [AIController.append_message](../src/ede/foundation/ai/api/ai_routes.py) |
| `GET /api/ai/conversations/{id}` | Read full transcript ordered by `sequence`. | [AIController.read_conversation](../src/ede/foundation/ai/api/ai_routes.py) |
| `GET /api/ai/tools` | List tools visible to the calling principal (read-only-filtered). | [AIController.list_tools](../src/ede/foundation/ai/api/ai_routes.py) |
| `GET /api/ai/usage` | Per-tenant daily spend summary. | [AIController.usage](../src/ede/foundation/ai/api/ai_routes.py) |
| `POST /api/ai/conversations/{id}/confirm` | **Phase 4** — body `{confirmation_token}`. Consumes the token minted by the bridge, dispatches the underlying write command (via approval engine if a matching rule exists), and writes an `ai.write.provenance` row. | [AIController.confirm_write](../src/ede/foundation/ai/api/ai_routes.py) |
| `POST /api/ai/conversations/{id}/undo` | **Phase 4** — reverses the most recent un-reverted AI write in this conversation using its declared `reverse_command`, within the configured `AI_WRITE_UNDO_WINDOW_SECONDS`. | [AIController.undo_last_write](../src/ede/foundation/ai/api/ai_routes.py) |

All routes return `403` when `AI_ENABLED=False`; `429` when the daily budget cap, the per-minute rate limit, the per-tool token cap, or the write budget cap is hit; `410 Gone` when a confirmation token has expired or the undo window has elapsed; `202 Accepted` when a write is queued through the approval engine; `503` when the configured provider is not registered, no prompt template resolves, or data-residency rejects the call.
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.ai.invocation` | Phase 1: quota check (real). Phase 2: prompt-injection detector + PII redactor (writes back into the request) + data-residency check + per-minute rate-limit + provider routing. |
| `pre.ai.provider.config.{create,update}` | Phase 2 hardening: Fernet-encrypt the `api_key_encrypted` value before the row is written. Idempotent (no-ops when value is already Fernet ciphertext or empty). |
| `pre.ai.provider.config.param.{create,update}` | Phase 2 hardening: same encryption flow for `ai.provider.config.param.value` rows where `is_secret=True`. |
| `post.ai.invocation` | Phase 2: output validator chain runs (warn-or-fail mode); response un-redacted from PII tokens in strict mode; cost-threshold alerts fired via `BudgetAlertService`. |
| `pre.ai.tool_call` | Defensive read-only check (Phase 1). Phase 4: write-mode allowlist check before minting a confirmation token. |
| `post.ai.tool_call` | Noop (extension point). |

Hooks fire as plain callables registered with `SafetyHookPipeline` rather than via `@api.on_hook` (which is per-model). Custom consumers register via `default_pipeline.register("pre.ai.invocation", fn)`.
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
| `AI_ENABLED` | bool | `False` | `AI_ENABLED` | Master switch per tenant. When False, all `/api/ai/*` routes return 403 and the tool registry is hidden. |
| `AI_DEFAULT_PROVIDER` | str | `"anthropic"` | `AI_DEFAULT_PROVIDER` | Default provider name when `/api/ai/invoke` does not pin one explicitly. |
| `AI_DAILY_BUDGET_USD` | float | `0.0` | `AI_DAILY_BUDGET_USD` | Per-tenant daily spend cap. `0.0` = unlimited. The `pre.ai.invocation` hook returns 429 when hit. |
| `AI_ALLOWED_PROVIDERS` | list[str] | `["anthropic"]` | `AI_ALLOWED_PROVIDERS` | Allow-list of provider names a tenant may configure. Bedrock + OpenAI land in Phase 2. |
| `AI_BYO_KEY_REQUIRED` | bool | `True` | `AI_BYO_KEY_REQUIRED` | When True, every tenant must supply its own provider API key via `ai.provider.config.api_key_encrypted`. |
| `AI_MAX_TOOL_ITERATIONS` | int | `8` | `AI_MAX_TOOL_ITERATIONS` | Maximum tool-call round trips per single LLM invocation; bridge raises `MaxIterationsReached` past this. |
| `AI_PII_REDACTION_ENABLED` | bool | `False` | `AI_PII_REDACTION_ENABLED` | **Phase 2.** Master switch for PII redaction at `pre.ai.invocation`. |
| `AI_PII_REDACTION_MODE` | str | `"strict"` | `AI_PII_REDACTION_MODE` | `strict` (token round-trip; response un-redacted automatically) or `lossy` (tokens flow through to the user). |
| `AI_INJECTION_DETECTION_ENABLED` | bool | `True` | `AI_INJECTION_DETECTION_ENABLED` | Master switch for the prompt-injection detector. On by default. |
| `AI_INJECTION_DETECTION_THRESHOLD` | int | `60` | `AI_INJECTION_DETECTION_THRESHOLD` | Score (0–100) above which the detector raises `PromptInjectionRejected` (HTTP 429). |
| `AI_OUTPUT_VALIDATION_MODE` | str | `"warn"` | `AI_OUTPUT_VALIDATION_MODE` | `warn` (annotate warnings) or `fail` (raise `OutputValidationFailed` → HTTP 429). |
| `AI_DATA_RESIDENCY_STRICT` | bool | `False` | `AI_DATA_RESIDENCY_STRICT` | When True, outgoing prompts mentioning `__ede_residency_restricted__` models may only target providers in the allow-list. |
| `AI_DATA_RESIDENCY_ALLOWED_PROVIDERS` | list[str] | `[]` | `AI_DATA_RESIDENCY_ALLOWED_PROVIDERS` | Providers that may receive residency-restricted data. Empty + strict = fail-closed. |
| `AI_WRITE_TOOLS_ALLOWLIST` | list[str] | `[]` | `AI_WRITE_TOOLS_ALLOWLIST` | **Phase 4.** Tenant allow-list of AI tool names eligible for write-mode. Empty = kill switch (no AI writes accepted). |
| `AI_WRITE_DAILY_CAP_RECORDS_PER_USER` | int | `0` | `AI_WRITE_DAILY_CAP_RECORDS_PER_USER` | Per-user daily cap on AI-issued record mutations. 0 = unlimited. |
| `AI_WRITE_DAILY_CAP_PER_TOOL` | int | `0` | `AI_WRITE_DAILY_CAP_PER_TOOL` | Per-tool daily cap on AI-issued writes across the tenant. 0 = unlimited. |
| `AI_WRITE_CONFIRM_TOKEN_TTL_SECONDS` | int | `300` | `AI_WRITE_CONFIRM_TOKEN_TTL_SECONDS` | TTL for pending-confirmation tokens. Beyond this `/confirm` returns 410 Gone. |
| `AI_WRITE_UNDO_WINDOW_SECONDS` | int | `60` | `AI_WRITE_UNDO_WINDOW_SECONDS` | How long after a confirmed write `/undo` remains valid. Beyond this `/undo` returns 410 Gone. |
| `AI_PROVIDER_KEY_ENCRYPTION_SECRET` | str | `""` | `AI_PROVIDER_KEY_ENCRYPTION_SECRET` | **Phase 2.** Fernet key (base64-encoded 32 bytes) used to encrypt `ai.provider.config.api_key_encrypted` + every `ai.provider.config.param.value` where `is_secret=True`. **Unset = plaintext mode** with a single boot-time WARNING — fine for dev, must be set in production. Generate via `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`. |
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
| _none — AI ships as its own top-level app, not as a Settings panel. The "Configurations" category inside the AI app holds the admin CRUD leaves. See seed-data table below._ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| [data/ai_rbac_roles.xml](../src/ede/foundation/ai/data/ai_rbac_roles.xml) | 2 roles: `ai_user` (under `internal_user`), `ai_admin` (under `ai_user`). |
| [data/ir.rbac.permission.csv](../src/ede/foundation/ai/data/ir.rbac.permission.csv) | 19 RBAC permissions across `ai.conversation`, `ai.message`, `ai.prompt.template`, `ai.prompt.version`, `ai.skill.pack`, `ai.safety.rule`, `ai.usage.log`, `ai.provider.config`. |
| [data/seed_prompts.xml](../src/ede/foundation/ai/data/seed_prompts.xml) | 4 system rows: 2 templates (`ai.system.bridge`, `ai.system.api`) × 2 versions. |
| [data/ai_menus.xml](../src/ede/foundation/ai/data/ai_menus.xml) | 17 records: 8 actions + standalone "AI" app root (`category=application`, sequence 70, lucide `sparkles`) + 2 top-level operational leaves (Conversations, Usage Log) + Configurations category + 5 config leaves (Provider Configurations, Prompt Templates, Prompt Versions, Skill Packs, Safety Rules). |
| [views/ai_provider_config_views.xml](../src/ede/foundation/ai/views/ai_provider_config_views.xml) | 3 views (list / form / search) for `ai.provider.config`. |
| [views/ai_prompt_template_views.xml](../src/ede/foundation/ai/views/ai_prompt_template_views.xml) | 6 views — list/form/search for both `ai.prompt.template` and `ai.prompt.version`. |
| [views/ai_conversation_views.xml](../src/ede/foundation/ai/views/ai_conversation_views.xml) | 5 views — list/form/search for `ai.conversation` + list/form for `ai.message` (embedded inside the conversation form notebook). |
| [views/ai_usage_log_views.xml](../src/ede/foundation/ai/views/ai_usage_log_views.xml) | 3 views (list / read-only form / search) for `ai.usage.log`. |
| [views/ai_skill_pack_views.xml](../src/ede/foundation/ai/views/ai_skill_pack_views.xml) | 3 views (list / form / search) for `ai.skill.pack`. |
| [views/ai_safety_rule_views.xml](../src/ede/foundation/ai/views/ai_safety_rule_views.xml) | 3 views (list / form / search) for `ai.safety.rule`. |
| [data/seed_provider_configs.xml](../src/ede/foundation/ai/data/seed_provider_configs.xml) | Phase 2 — 2 rows: `anthropic` + `openai` `ai.provider.config` (both `active=false` so admin must supply an API key before either picks up traffic). |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Provider Abstraction + Tool/Prompt Registry + Conversations + Cost/Audit (Read-Only Bridge) | ✅ Delivered 2026-05-15 | [phase-1-implementation.md](../roadmap/foundation/ai/phase-1-implementation.md) |
| Phase 2 | Safety Hardening (PII + Injection + Output Validators + Provider Routing + Cost Alerts) | ✅ Delivered 2026-05-17 | [phase-2-implementation.md](../roadmap/foundation/ai/phase-2-implementation.md) |
| Phase 3 | Embeddings + Semantic Primitives | 🔴 ON HOLD (pending `foundation.jobs` Phase 1 — background indexer is the hard prereq) | [phase-3-implementation.md](../roadmap/foundation/ai/phase-3-implementation.md) |
| Phase 4 | Gated Write-Mode (draft → preview → confirm + approval integration + undo) | ✅ Delivered 2026-05-17 | [phase-4-implementation.md](../roadmap/foundation/ai/phase-4-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Provider service + Anthropic backend | — | [services/provider/](../src/ede/foundation/ai/services/provider/) | [Phase 1 WS-A2](../roadmap/foundation/ai/phase-1-implementation.md) |
| `@api.ai_tool` decorator + `ToolRegistry` + boot read-only validator | — | [kernel/ai_decorators.py](../src/ede/core/kernel/ai_decorators.py) · [services/tool_registry.py](../src/ede/foundation/ai/services/tool_registry.py) | [Phase 1 WS-A3](../roadmap/foundation/ai/phase-1-implementation.md) |
| Versioned tenant-overridable prompt registry | `ai.prompt.template`, `ai.prompt.version` | [services/prompt_registry.py](../src/ede/foundation/ai/services/prompt_registry.py) · [data/seed_prompts.xml](../src/ede/foundation/ai/data/seed_prompts.xml) | [Phase 1 WS-A4](../roadmap/foundation/ai/phase-1-implementation.md) |
| Conversation + message primitives | `ai.conversation`, `ai.message` | [models/conversation.py](../src/ede/foundation/ai/models/conversation.py) | [Phase 1 WS-A1](../roadmap/foundation/ai/phase-1-implementation.md) |
| Function-calling bridge with read-only gate | — | [services/bridge.py](../src/ede/foundation/ai/services/bridge.py) | [Phase 1 WS-A5](../roadmap/foundation/ai/phase-1-implementation.md) |
| Cost + quota service (per-tenant daily budget) | `ai.usage.log` | [services/cost.py](../src/ede/foundation/ai/services/cost.py) · [services/quota.py](../src/ede/foundation/ai/services/quota.py) | [Phase 1 WS-A6](../roadmap/foundation/ai/phase-1-implementation.md) |
| Safety hookpoints (stubs in P1; real impls Phase 2) | `ai.safety.rule` | [services/safety.py](../src/ede/foundation/ai/services/safety.py) | [Phase 1 WS-A7](../roadmap/foundation/ai/phase-1-implementation.md) |
| HTTP surface — 6 routes under `/api/ai/*` | — | [api/ai_routes.py](../src/ede/foundation/ai/api/ai_routes.py) | [Phase 1 WS-A8](../roadmap/foundation/ai/phase-1-implementation.md) |
| RBAC seed (2 roles, 19 permissions) | — | [data/ai_rbac_roles.xml](../src/ede/foundation/ai/data/ai_rbac_roles.xml) · [data/ir.rbac.permission.csv](../src/ede/foundation/ai/data/ir.rbac.permission.csv) | [Phase 1 WS-A9](../roadmap/foundation/ai/phase-1-implementation.md) |
| `read_schema` reference adopter (informer tool) | `ai.tool` | [tools/read_schema.py](../src/ede/foundation/ai/tools/read_schema.py) | [Phase 1 WS-A10](../roadmap/foundation/ai/phase-1-implementation.md) |
| Standalone "AI" app + DSL views for every `ai.*` model | — | [data/ai_menus.xml](../src/ede/foundation/ai/data/ai_menus.xml) · [views/](../src/ede/foundation/ai/views/) (6 files, 23 view definitions) | [Enhancement 01](../roadmap/foundation/ai/enhancements/01-standalone-app-and-views.md) |
| **Phase 2** PII redaction (WS-A11) | — | [services/safety/pii.py](../src/ede/foundation/ai/services/safety/pii.py) | [Phase 2 WS-A11](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Phase 2** Prompt-injection detector (WS-A12) | — | [services/safety/injection.py](../src/ede/foundation/ai/services/safety/injection.py) | [Phase 2 WS-A12](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Phase 2** Output-validator chain (WS-A13) | — | [services/safety/output_validator.py](../src/ede/foundation/ai/services/safety/output_validator.py) | [Phase 2 WS-A13](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Phase 2** Per-user quotas + rate limits (WS-A14) | `ai.user.quota` | [services/quota.py](../src/ede/foundation/ai/services/quota.py) · [models/user_quota.py](../src/ede/foundation/ai/models/user_quota.py) | [Phase 2 WS-A14](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Phase 2** Cost dashboards + threshold alerts (WS-A15) | — | [services/budget_alert.py](../src/ede/foundation/ai/services/budget_alert.py) | [Phase 2 WS-A15](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Phase 2** Data-residency enforcement (WS-A16) | — | [services/safety/residency.py](../src/ede/foundation/ai/services/safety/residency.py) | [Phase 2 WS-A16](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Phase 2** Provider routing rules (WS-A17) | `ai.provider.routing.rule` | [services/provider/router.py](../src/ede/foundation/ai/services/provider/router.py) · [models/provider_routing.py](../src/ede/foundation/ai/models/provider_routing.py) | [Phase 2 WS-A17](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Phase 2** OpenAI provider + seed provider configs | — | [services/provider/openai.py](../src/ede/foundation/ai/services/provider/openai.py) · [data/seed_provider_configs.xml](../src/ede/foundation/ai/data/seed_provider_configs.xml) | [Phase 2 WS-A2 (extension)](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Phase 4** Write-tool opt-in path (WS-A24) | — | [kernel/ai_decorators.py](../src/ede/core/kernel/ai_decorators.py) · [services/tool_registry.py](../src/ede/foundation/ai/services/tool_registry.py) | [Phase 4 WS-A24](../roadmap/foundation/ai/phase-4-implementation.md) |
| **Phase 4** Draft → preview → confirm bridge flow (WS-A25) | — | [services/write_mode.py](../src/ede/foundation/ai/services/write_mode.py) · [services/bridge.py](../src/ede/foundation/ai/services/bridge.py) | [Phase 4 WS-A25](../roadmap/foundation/ai/phase-4-implementation.md) |
| **Phase 4** Provenance side-table (WS-A26) | `ai.write.provenance` | [models/write_provenance.py](../src/ede/foundation/ai/models/write_provenance.py) | [Phase 4 WS-A26](../roadmap/foundation/ai/phase-4-implementation.md) |
| **Phase 4** Approval-engine integration (WS-A27) | — | [services/write_mode.py](../src/ede/foundation/ai/services/write_mode.py) (`WriteApprovalBridge`) | [Phase 4 WS-A27](../roadmap/foundation/ai/phase-4-implementation.md) |
| **Phase 4** Write budgets + kill switch (WS-A28) | — | [services/write_mode.py](../src/ede/foundation/ai/services/write_mode.py) (`WriteBudgetService`) | [Phase 4 WS-A28](../roadmap/foundation/ai/phase-4-implementation.md) |
| **Phase 4** Undo window via `reverse_command` (WS-A29) | — | [services/write_mode.py](../src/ede/foundation/ai/services/write_mode.py) (`UndoWindow`) | [Phase 4 WS-A29](../roadmap/foundation/ai/phase-4-implementation.md) |
| **Fernet encryption** for `api_key_encrypted` + `ai.provider.config.param.value` (`is_secret=True`) | — | [services/secrets.py](../src/ede/foundation/ai/services/secrets.py) · [models/provider_config.py](../src/ede/foundation/ai/models/provider_config.py) (pre-hooks) | Phase 2 hardening (post-spec; addresses the plaintext-key gap raised during rollout) |
| **DB-driven provider construction** + extensible per-provider params O2M | `ai.provider.config.param` | [services/provider/builder.py](../src/ede/foundation/ai/services/provider/builder.py) · [models/provider_config_param.py](../src/ede/foundation/ai/models/provider_config_param.py) | Phase 2 hardening (post-spec; replaces the singleton `default_registry.get(name)` lookup with per-tenant DB-derived providers) |
| **Enhancement 02** — lazy + secret-keyed `default_secrets` cache (so `ede.conf`-loaded `AI_PROVIDER_KEY_ENCRYPTION_SECRET` actually reaches the encryption hook) + move `get_extra_params` into `_extract_extra_params(row)` in the provider builder (RecordSet only proxies declared fields; the model method was unreachable and silently swallowed by `_row_to_dto`, producing a misleading "no api_key param row" error even when the row was populated) | — | [services/secrets.py](../src/ede/foundation/ai/services/secrets.py) · [services/provider/builder.py](../src/ede/foundation/ai/services/provider/builder.py) · [models/provider_config.py](../src/ede/foundation/ai/models/provider_config.py) | [Enhancement 02](../roadmap/foundation/ai/enhancements/02-secrets-singleton-and-builder-method-proxy.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Embeddings + semantic search — `pgvector`, `@api.embed_on`, hybrid keyword × vector search. **Phase 3 ON HOLD pending `foundation.jobs` Phase 1** (background indexer is the hard prereq per spec WS-A21). Will resume once jobs ships. | 🔴 | [phase-3-implementation.md](../roadmap/foundation/ai/phase-3-implementation.md) |
| Per-tenant pricing overrides — Phase 1 ships hard-coded seed pricing tables for Anthropic + OpenAI; per-tenant `ai.provider.config.pricing_overrides` deferred. Can now be modeled as an `ai.provider.config.param` row (`key="pricing_overrides"`, `value` = JSON-encoded dict) without a schema change. | 🟢 | [phase-2-implementation.md](../roadmap/foundation/ai/phase-2-implementation.md) |
| **Plaintext-key mode** — when `AI_PROVIDER_KEY_ENCRYPTION_SECRET` is unset, `api_key_encrypted` + secret params are stored as plaintext (loud boot WARNING). Acceptable for dev/personal; production tenants MUST set the secret before storing real keys. *(Until 2026-05-18 an import-order bug also pinned the process to plaintext mode even when the secret WAS configured in `ede.conf`; fixed in Enhancement 02. Existing plaintext rows on disk need to be re-saved through the form — or run a one-shot re-encrypt — to flip to Fernet ciphertext.)* | 🟡 | [Enhancement 02](../roadmap/foundation/ai/enhancements/02-secrets-singleton-and-builder-method-proxy.md) |
| Per-user spend rollup — Phase 2 per-user budget check uses the per-tenant cost rollup as a conservative cap (over-counts when multiple users share a tenant). True per-user rollup deferred until the cost service grows a `principal_user_id` filter. | 🟡 | [phase-2-implementation.md](../roadmap/foundation/ai/phase-2-implementation.md) |
| `ai.write.provenance` is a side-table rather than cross-model auto-fields (deviation from Phase 4 spec). Same join-able query surface; if a future enhancement decides to lift to true auto-fields, this table is the backfill source. | 🟢 | [phase-4-implementation.md](../roadmap/foundation/ai/phase-4-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
_(none yet — populated as integration learnings emerge)_

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- **Phase 1**: initial schema in [`a9e3c1f0d8b4_ai_init.py`](../src/ede/foundation/ai/migrations/versions/a9e3c1f0d8b4_ai_init.py) (`down_revision=f4c8a1e9b6d7`). 9 `op.create_table(...)` blocks (8 model tables + the `ai_tool` carrier) with inline FK + unique constraints, 22 explicit `op.create_index(...)` calls. All constraint names ≤ 53 chars.
- The cyclic FK between `ai_prompt_template.current_version_id` and `ai_prompt_version.template_id` is broken at the DB level by NOT declaring a FK constraint on `current_version_id` (UUID-only). The strong direction `version.template_id → template.record_uuid` is enforced as a real FK with `ondelete=cascade`.
- **Phase 2**: [`b7e2f4a91c08_phase2_quota_routing.py`](../src/ede/foundation/ai/migrations/versions/b7e2f4a91c08_phase2_quota_routing.py) adds `ai_user_quota` + `ai_provider_routing_rule` tables. [`d4e7c91b3f6a_provider_config_org_scope.py`](../src/ede/foundation/ai/migrations/versions/d4e7c91b3f6a_provider_config_org_scope.py) drops `tenant_id` from `ai_provider_config` and adds optional `organization_id` Reference → `res_organization` (wrapped in `op.batch_alter_table(...)` for SQLite parity).
- **Phase 4**: [`c3f8a5d2e911_phase4_write_provenance.py`](../src/ede/foundation/ai/migrations/versions/c3f8a5d2e911_phase4_write_provenance.py) adds `ai_write_provenance` side-table with FKs to `ai_conversation` (cascade), `res_user` (restrict), and `ir_approval_case` (set null). 6 explicit indexes on the high-cardinality join columns.
- **Provider param O2M**: [`e8b1f4c92d5a_provider_config_param.py`](../src/ede/foundation/ai/migrations/versions/e8b1f4c92d5a_provider_config_param.py) adds `ai_provider_config_param` (one row per `(parent, key)`; `value` is `Text` so encrypted Fernet blobs fit; cascade FK on parent delete).
- 6 pre-existing migrations rescued for SQLite parity during Phase 1 — wrapped bare `op.create_foreign_key` / `op.drop_constraint` / `op.alter_column` in `op.batch_alter_table(...)`. Affected: `f153863de80c`, `e7c8a91d2b5f` (logistics.masters); `f51f9b96bab9`, `f27442d33487` (foundation.dataset); `03658b1fa089` (foundation.base); `c58efb0a46d8` (foundation.qa_automation).
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `ai_user` (parent: `internal_user`) | `ai.invoke`, `ai.tools.read`, `ai.conversation.read`, `ai.message.read`, `ai.prompts.read`, `ai.skill_packs.read` |
| `ai_admin` (parent: `ai_user`) | All `ai_user` perms + `ai.prompts.{create,update,delete}`, `ai.prompt_versions.{create,update}`, `ai.skill_packs.{create,update}`, `ai.safety_rules.{read,create,update}`, `ai.usage.read`, `ai.provider.{read,manage}` |
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

*Last sync: 2026-05-19 (Enhancement 03 — Migrate All Providers to LangChain — ✅ Delivered. **Same-day design revision for Slice C**: the initially-shipped per-tenant `ai.provider.config.client_class` Char was rejected on review ("operator should not be typing FQ Python class names into a form; this is a code-shape decision"); replaced with a code-level `ProviderClassRegistry` at `services/provider/class_registry.py` — `register_provider(name, chat_model_class, cost_function=None)` / `resolve_provider(name)` / `registered_names()`. First-class providers (anthropic, openai) register unconditionally at import; Ollama + Gemini / Bedrock / Cohere register inside `try: from <pkg> import …; register_provider(…) except ImportError: pass` blocks in `services/provider/__init__.py`. Tenants enable an optional-extra provider by installing the matching extra + restarting; no DB column to fill. Follow-up migration `9a4e2b7d1c80_drop_client_class_from_provider_config` drops the column (chained off `86c98dafbf5d`; column never reached production — only lived on dev tenants for a few hours). 9 new registry tests + 4 obsolete client_class tests removed = 143 ai + 160 assistant pytest cases green. Shipped across 5 slices: A `FoundationAIAdapter` (Layer 5) + core langchain-core/langchain-anthropic/langchain-openai deps + 14 isolated tests; B Anthropic + OpenAI rewritten as thin builders + new first-class `OllamaProvider` over `langchain_ollama.ChatOllama` + 26 test fixtures refactored from `client=anthropic_sdk_mock` to `chat_model=langchain_mock` + native `anthropic` / `openai` direct deps removed from `pyproject.toml`; C (revised) code-level registry instead of `client_class` column; D `test_sse_wire_shape.py` — byte-identical SSE event-sequence regression; E this sync + developer guide `docs/foundation-ai-providers.md`. Frontend AssistantPanel + `ToolBridge` iteration policy + three-layer read-only enforcement + write-mode confirmation tokens + `ai.write.provenance` + PII redaction + prompt-injection detection + cost via `ai.usage.log` all preserved end-to-end. LangChain agents / `AgentExecutor` / LCEL / LangGraph explicitly out of scope. Prior 2026-05-18 sync: Enhancement 02 ✅ Delivered. Phases 1 / 2 / 4 ✅ Delivered (2026-05-15 / 2026-05-17) · Enhancement 01 ✅ Delivered (2026-05-15) · Phase 3 🔴 ON HOLD pending `foundation.jobs` P1). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
