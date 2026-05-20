# Foundation AI — Provider Configuration Guide

**Module:** `foundation.ai` (`src/ede/foundation/ai/`)
**Roadmap:** [Enhancement 03 — Migrate All Providers to LangChain](../roadmap/foundation/ai/enhancements/03-langchain-provider-adapter.md)
**Status:** Reflects the platform after Enhancement 03 ✅ Delivered 2026-05-19.

> Every LLM in EDE goes through one canonical adapter — `FoundationAIAdapter` (Layer 5). The adapter wraps any LangChain `BaseChatModel` over the platform's `ProviderContract` port (Layer 4); per-provider Layer 3 builder modules construct the right ChatModel for each provider name. No EDE source code imports a native provider SDK directly — every chat call, every stream chunk, every tool-use round-trip flows through LangChain.

---

## 1. Why LangChain is the Only Path

Before this enhancement, `AnthropicProvider` and `OpenAIProvider` were hand-rolled wrappers — each carried its own chat-message conversion, function-calling glue, streaming event mapper, and cost extractor. Adding a third provider meant writing a third copy. LangChain already maintains those concerns upstream, kept in lockstep with each provider's SDK; we own one adapter and inherit the rest.

The migration is **transparent** to everything downstream of `ProviderContract`:

- `ToolBridge` iteration loop, read-only enforcement (3 layers), `pre.ai.tool_call` hook chain — unchanged.
- Write-mode confirmation tokens + `WriteApprovalBridge` + `ai.write.provenance` audit table — unchanged.
- PII redaction, prompt-injection detection, output validators (Phase 2 safety pipeline) — unchanged.
- Cost tracking via `ai.usage.log` — unchanged (LangChain's `usage_metadata` covers `input_tokens` + `output_tokens` uniformly; provider-specific extras like Anthropic prompt-caching read from `response.response_metadata`).
- SSE event sequence the React `AssistantPanel.sendMessageStream` consumes — byte-identical. Pinned by [`src/tests/foundation/ai/test_sse_wire_shape.py`](../src/tests/foundation/ai/test_sse_wire_shape.py).

**Out of scope** (explicit): LangChain agents, `AgentExecutor`, LCEL chains, LangGraph. Those would replace `ToolBridge`'s iteration loop and force re-implementing the read-only validator + write-mode confirmation tokens inside LangChain abstractions. The provider-only migration delivers "one adapter, every provider" with a much smaller surface change.

## 2. Architecture — Three Layers

```
Layer 3 (Domain)               ai.provider.config row + per-provider builder
                               function. Knows: name, default model, pricing
                               table, secrets. Resolves credentials per
                               (tenant, organization, name).

Layer 4 (Platform Capability)  ProviderContract Protocol — invoke() + stream().
                               The port the bridge consumes. Provider-agnostic.

Layer 5 (Infrastructure)       FoundationAIAdapter — single adapter over any
                               LangChain ChatModel. Doesn't know which provider
                               it's wrapping; doesn't see secrets.
```

Switching the **adapter** (e.g. LangChain → LiteLLM in some future world) only touches one file (`foundation_ai_adapter.py`). Switching the **provider** (Anthropic → OpenAI → Gemini) flips one column — the `ai.provider.config.name` — and the code-level class registry resolves it to the matching LangChain ChatModel class at the next request. No FQ class name in any DB column.

```
Layer 3 thin builder (per provider)
        ├── anthropic.py    → langchain_anthropic.ChatAnthropic + cost table
        ├── openai.py       → langchain_openai.ChatOpenAI    + cost table
        ├── ollama.py       → langchain_ollama.ChatOllama    + zero cost
        └── builder.py      → resolves `name` via class_registry, builds ChatModel
                                  │
                                  ▼
                          FoundationAIAdapter (Layer 5)
                                  │
                                  ▼
                          ToolBridge → CommandBus → @api.ai_tool handlers
```

## 3. First-Class Providers (Out of the Box)

These ship in the core install — no extras to enable. Pick one by setting `ai.provider.config.name` to the listed key.

| Provider | `name` | Default Model | Required Params | Cost Table | Source |
|---|---|---|---|---|---|
| Anthropic | `anthropic` | `claude-sonnet-4-6` | `api_key` (secret param) | Sonnet-4.6 / Opus-4.7 / Haiku-4.5 per-1K tokens | [`anthropic.py`](../src/ede/foundation/ai/services/provider/anthropic.py) |
| OpenAI | `openai` | `gpt-4o` | `api_key` (secret param) | GPT-4o / 4o-mini / 4-turbo / 3.5 per-1K tokens | [`openai.py`](../src/ede/foundation/ai/services/provider/openai.py) |
| Ollama (local) | `ollama` | `llama3.2` | _(none — runs on `endpoint_url`, default `http://localhost:11434`)_ | Zero (local inference has no per-token billing) | [`ollama.py`](../src/ede/foundation/ai/services/provider/ollama.py) |

### Wiring an Anthropic row

Existing `ai.provider.config` rows from before Enhancement 03 work unchanged. For a new tenant, create the row through **AI → Configurations → Provider Configurations** in the webclient (or via XML seed for demo data):

```xml
<record id="prov_anthropic" model="ai.provider.config">
    <field name="name">anthropic</field>
    <field name="display_name">Anthropic (Production)</field>
    <field name="default_model">claude-sonnet-4-6</field>
    <field name="max_tokens_default">4096</field>
    <field name="temperature_default">0.0</field>
    <field name="active" eval="True"/>
    <!-- api_key arrives via the param O2M (key='api_key', is_secret=True). -->
</record>
```

Then open the row in the form view, switch to the **Parameters** tab, and add a row: `key=api_key`, `value=<your Anthropic key>`, `is_secret=true`. The secret flows through the `pre.ai.provider.config.param.create` hook and lands encrypted at rest under the tenant's Fernet key.

### Wiring an OpenAI row

Identical shape with `name=openai` + `default_model=gpt-4o`.

### Wiring Ollama

Local inference — no API key needed. Point at the host running `ollama serve`:

```xml
<record id="prov_ollama" model="ai.provider.config">
    <field name="name">ollama</field>
    <field name="display_name">Ollama (Local llama3.2)</field>
    <field name="default_model">llama3.2</field>
    <field name="endpoint_url">http://localhost:11434</field>
    <field name="active" eval="True"/>
</record>
```

Requires `pip install -e ".[ai-ollama]"` to pull `langchain-ollama`. The builder constructs `ChatOllama(model=..., base_url=...)` and hands it to `FoundationAIAdapter`. Cost rolls up to `$0` in `ai.usage.log` — fine, because local inference is amortised compute the tenant already owns.

## 4. Net-New Providers via the Class Registry

There is **no per-tenant field** that tells the platform which LangChain class to instantiate. Provider name → ChatModel class is a **code-level decision** maintained in [`class_registry.py`](../src/ede/foundation/ai/services/provider/class_registry.py); the operator just sets `ai.provider.config.name` to a known key and the registry resolves the rest.

The registry is populated by **module-import side effects**:

- First-class providers (Anthropic, OpenAI) register unconditionally — their langchain wrappers are core deps.
- Optional-extra providers (Ollama, Gemini, Bedrock, Cohere) register inside a `try: … except ImportError: pass` block in [`services/provider/__init__.py`](../src/ede/foundation/ai/services/provider/__init__.py); installing the matching extra makes the import succeed at next boot, which flips the registration on.

### Tenant workflow — enabling Gemini

```bash
pip install -e ".[ai-gemini]"   # pulls langchain-google-genai
# restart the server so the registration side effect fires at import time
```

Then in the AI app → Provider Configurations → New:

```xml
<record id="prov_gemini" model="ai.provider.config">
    <field name="name">gemini</field>
    <field name="display_name">Google Gemini Pro</field>
    <field name="default_model">gemini-1.5-pro</field>
    <field name="active" eval="True"/>
</record>
```

Add a parameter row `key=api_key`, `is_secret=true`, paste the Google API key. The next assistant turn that routes to `gemini` resolves `(ChatGoogleGenerativeAI, None)` from the registry, instantiates `ChatGoogleGenerativeAI(model="gemini-1.5-pro", api_key=...)`, and streams chat + tool-use through `FoundationAIAdapter`. No code change. No FQ class name in any form.

### Tenant workflow — enabling AWS Bedrock

```bash
pip install -e ".[ai-bedrock]"
```

```xml
<record id="prov_bedrock" model="ai.provider.config">
    <field name="name">bedrock</field>
    <field name="display_name">AWS Bedrock — Claude on Bedrock</field>
    <field name="default_model">anthropic.claude-3-5-sonnet-20241022-v2:0</field>
    <field name="active" eval="True"/>
</record>
```

Bedrock takes additional params (`aws_region`, `aws_access_key_id`, `aws_secret_access_key`, optionally `aws_role_arn`). Add each as a parameter row; secret params are encrypted at rest. The builder filters the assembled kwargs against `ChatBedrock.__init__`'s signature so unrelated params on the same row are silently dropped instead of raising `TypeError`.

### Tenant workflow — enabling Cohere

```bash
pip install langchain-cohere
```

```xml
<record id="prov_cohere" model="ai.provider.config">
    <field name="name">cohere</field>
    <field name="display_name">Cohere Command R+</field>
    <field name="default_model">command-r-plus</field>
    <field name="active" eval="True"/>
</record>
```

Same pattern — `api_key` param row, and Cohere streams through `FoundationAIAdapter` unchanged.

### Onboarding a brand-new provider (developer task, not operator)

If LangChain ships a wrapper for a provider we don't yet register (or if a customisation module ships its own ChatModel class), adding it is one line of code:

```python
# services/provider/__init__.py
try:
    from langchain_mistralai import ChatMistralAI
    register_provider("mistral", chat_model_class=ChatMistralAI, cost_function=None)
except ImportError:
    pass
```

Or, from a customisation module loaded after `foundation.ai`:

```python
from ede.foundation.ai.services.provider import register_provider
from acme_customisation.chat import ChatAcme
register_provider("acme", chat_model_class=ChatAcme, cost_function=_acme_cost)
```

The registry rejects duplicate names with a different class (`ValueError`), so misconfigurations surface at boot rather than at the first call.

## 5. Cost Tracking — How It Works Across Providers

`FoundationAIAdapter` extracts token counts from LangChain's standardized `usage_metadata` dict (`input_tokens` / `output_tokens` / `total_tokens`). Per-provider per-1K-token rates come from the thin builder's `_compute_cost(model, in, out)` callback:

- **Anthropic** ships a pricing table for `claude-sonnet-4-6`, `claude-opus-4-7`, `claude-haiku-4-5` (see [`anthropic.py`](../src/ede/foundation/ai/services/provider/anthropic.py)).
- **OpenAI** ships a table for `gpt-4o`, `gpt-4o-mini`, `gpt-4-turbo`, `gpt-3.5-turbo` (see [`openai.py`](../src/ede/foundation/ai/services/provider/openai.py)).
- **Ollama** uses the FoundationAIAdapter's zero-cost default — local inference has no per-token billing.
- **Optional-extra providers (Gemini, Bedrock, Cohere, etc.)** register with `cost_function=None` and inherit the FoundationAIAdapter's zero-cost default; per-1K-token pricing for those will be plumbed by a future enhancement (per-tenant `pricing_overrides`). Operators see `cost_usd=0` rows in `ai.usage.log` until that lands; token counts are still recorded accurately.

Unknown models inside known providers fall back to the **lowest-tier rate** in the table — a conservative under-counting that flags untracked models for operator review without dropping the audit row.

Anthropic's prompt-caching beta surfaces `cache_read_input_tokens` / `cache_creation_input_tokens` in `response.response_metadata`. Those don't flow through LangChain's `usage_metadata`, so they're not currently rolled into cost; the raw response is captured in `ai.message.raw` for forensic re-computation when needed.

## 6. Streaming Wire-Shape Contract

`FoundationAIAdapter.stream(...)` yields the exact event shape the bridge + AssistantPanel consume — independent of which LangChain ChatModel is wrapping:

```
{"type": "text_delta", "text": str}              # zero or more per iteration
{"type": "done", "text": str | None,             # exactly one, last
                 "tool_calls": list[{id,name,input}],
                 "tokens_in": int, "tokens_out": int,
                 "cost_usd": Decimal,
                 "latency_ms": int,
                 "raw": {...}}
```

`ToolBridge.invoke_streaming(...)` flattens that into the bridge-level sequence:

```
{"type": "text_delta", "text": str}              # zero or more
{"type": "final", ...}                           # exactly one, last
```

And `send_turn_streaming(...)` (the assistant's SSE producer) bookends with `started` / `complete`:

```
{"type": "started"}                              # first
{"type": "text_delta", "text": str}              # zero or more
{"type": "complete", "message_id": str, "body": str,
                     "view_intent": {...} | None, "actions": [...],
                     "usage": {...}}             # last
```

**The shape is byte-identical across providers.** If a future LangChain change drifts the chunk shape (`AIMessageChunk.tool_call_chunks` ordering, `usage_metadata` keys, …), [`test_sse_wire_shape.py`](../src/tests/foundation/ai/test_sse_wire_shape.py) is the regression gate that catches it. AssistantPanel does not see this migration at all.

## 7. Streaming-Mode Tool Calls — How They Reassemble

LangChain streams tool calls as `AIMessageChunk.tool_call_chunks` — partial JSON-arg fragments accumulated across chunks, keyed by `index` for parallel-tool-call disambiguation. The adapter:

1. Accumulates fragments per `index` until end-of-stream.
2. If the provider ALSO populates `chunk.tool_calls` with fully parsed args on the final chunk (some providers do), prefers the parsed form as authoritative.
3. Otherwise JSON-parses the accumulated buffer + maps LangChain's `args` → bridge's `input` field rename.
4. Emits the assembled tool calls in the single `done` event.

Edge case the adapter explicitly protects against: LangChain 0.3 sometimes auto-populates `chunk.tool_calls` mid-stream with **empty** `args` dicts before the JSON-arg buffer has fully arrived. The adapter only treats `tool_calls` as authoritative when ALL entries have non-empty parsed args; otherwise it falls back to the accumulated `tool_call_chunks` buffer. See [`foundation_ai_adapter.py`](../src/ede/foundation/ai/services/provider/foundation_ai_adapter.py).

## 8. Provider-Specific Capabilities — What's Accessible

| Feature | Anthropic | OpenAI | Ollama | Optional-extra (Gemini/Bedrock/Cohere/…) |
|---|---|---|---|---|
| Streaming text + tool calls | ✅ | ✅ (real streaming, no faux-stream) | ✅ | ✅ if LangChain wrapper supports it |
| Tool binding (`bind_tools`) | ✅ | ✅ | ✅ (per model — tool-use support varies) | Provider-dependent |
| `usage_metadata` (token counts) | ✅ | ✅ | ✅ | Provider-dependent |
| Anthropic prompt-caching beta | Captured in `ai.message.raw`; not in cost yet | n/a | n/a | n/a |
| OpenAI structured outputs / JSON mode | Not currently surfaced through LangChain's normalized layer; raw kwargs flow through `extra_params` | | | |

## 9. Operational Notes

- **Deployment** — existing tenants need a `pip install -e .` (or container rebuild) on the next deploy to pick up the new `langchain-*` deps. Existing `ai.provider.config` rows work unchanged — the builder resolves the ChatModel class by `name` through the in-process registry; no DB-level change is required.
- **LangChain version policy** — pinned to `langchain-core>=0.3,<0.4` and matching `langchain-anthropic>=0.3` / `langchain-openai>=0.3`. A follow-up enhancement will retest under the next major.
- **Provider routing rules** — `ai.provider.routing.rule` (Phase 2) keeps working. Rules pick which `ai.provider.config` row to use; the row is now always LangChain-backed.
- **Per-organization scoping** — the builder honours the standard precedence: org-scoped row first (matched by `principal.active_organization_id`), then the tenant-wide fallback (`organization_id IS NULL`).

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ProviderUnavailable: Provider 'gemini' is not registered. Known providers: ...` | Optional extra not installed → the registration side effect in `__init__.py`'s `try/except ImportError` block didn't fire | `pip install -e ".[ai-gemini]"` (or `[ai-bedrock]` / `[ai-ollama]`) **and restart the server** so the import-time side effect runs. |
| `ProviderUnavailable: Provider 'X' has no api_key param row` | First-class provider row missing the `api_key` param | Open the row → Parameters tab → add `key=api_key`, `is_secret=true`. |
| Streaming "ended without a done event" | A registered ChatModel doesn't implement `.stream(...)` per LangChain contract | Confirm the wrapper supports streaming; otherwise fall back to non-streaming `invoke()` path (set the consumer to `bridge.invoke(...)` instead of `bridge.invoke_streaming(...)`). |
| Tool calls arrive with empty `input` dicts | LangChain 0.3 mid-stream auto-population of `tool_calls` with partial args | Already handled — adapter falls back to accumulated `tool_call_chunks` when `tool_calls` entries have empty args. If still seeing this, the provider wrapper's streaming chunks may not include `tool_call_chunks` at all; report the wrapper version. |
| `cost_usd=0` rows for a new provider that should be priced | Pricing table only ships for Anthropic / OpenAI first-class builders | Add an `ai.provider.config.param` row with `key=pricing_overrides`, `value=<JSON>` (deferred to a follow-up enhancement; the column is reserved). |

## 11. Related Files

- [`foundation_ai_adapter.py`](../src/ede/foundation/ai/services/provider/foundation_ai_adapter.py) — Layer 5 adapter (the canonical implementation).
- [`anthropic.py`](../src/ede/foundation/ai/services/provider/anthropic.py) / [`openai.py`](../src/ede/foundation/ai/services/provider/openai.py) / [`ollama.py`](../src/ede/foundation/ai/services/provider/ollama.py) — Layer 3 thin builders.
- [`class_registry.py`](../src/ede/foundation/ai/services/provider/class_registry.py) — `name → (ChatModel class, cost fn)` registry. `register_provider`, `resolve_provider`, `registered_names`.
- [`builder.py`](../src/ede/foundation/ai/services/provider/builder.py) — DB-row → registry lookup → ChatModel construction → `FoundationAIAdapter` wrapping. Org-scoping resolver + kwarg filtering live here.
- [`provider_config.py`](../src/ede/foundation/ai/models/provider_config.py) — `ai.provider.config` model (operator-tunable knobs only).
- [`test_sse_wire_shape.py`](../src/tests/foundation/ai/test_sse_wire_shape.py) — byte-identical SSE event-sequence regression test.
- [`foundation-ai.md`](./foundation-ai.md) — module overview / status / configuration introduced.
- [Enhancement 03 roadmap](../roadmap/foundation/ai/enhancements/03-langchain-provider-adapter.md) — the spec this guide is implemented from.
