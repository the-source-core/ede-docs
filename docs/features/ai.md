# AI Capability Layer

Provider-agnostic LLM substrate: turn any existing read-only command into an LLM-invocable tool with one decorator, run prompts through versioned templates, and let the framework track cost, enforce budgets, and audit every call.

```python
from ede.core import api
from ede.core.kernel.model import DomainModel
from ede.core.types import Command


@api.model("ai.tool")
class AiTool(DomainModel):

    @api.on_command("ai.tool.read_schema")
    @api.ai_tool(
        name="read_schema",
        read_only=True,
        kind="informer",
        description="Return the registry definition of a model.",
        input_schema={
            "type": "object",
            "properties": {"model_key": {"type": "string"}},
            "required": ["model_key"],
        },
    )
    def read_schema(self, cmd: Command):
        return self.env.registry.get_model(cmd.payload["model_key"]).to_dict()
```

```http
POST /api/ai/invoke
Authorization: Bearer <token>

{ "prompt": "What fields are on res.partner?" }
```

The provider picks the `read_schema` tool, the bridge dispatches it through `env.dispatch` under the read-only gate, the result composes into the response, and a `ai.usage.log` row records the tokens and dollar cost. No mutation is possible from this path without the gated write-mode surface below.

---

## What you get

-   **Provider abstraction** — pluggable provider contract following the same pattern as the persistence providers. One provider ships out of the box (Anthropic Messages API); per-tenant routing rules pick which provider answers which call.
-   **`@api.ai_tool(...)`** decorator — promote any `@api.on_command` handler to the LLM tool catalogue. Five parameters: `name`, `read_only`, `kind` (`"informer" | "proposer" | "insight"`), `description`, `input_schema`. A boot-time validator rejects any `read_only=True` tool whose underlying command can mutate state.
-   **Function-calling bridge** — translates LLM tool calls into `env.dispatch(Command(...))`. Honors RBAC, record rules, lifecycle hooks, and the read-only gate.
-   **Versioned prompt registry** — `ai.prompt.template` + `ai.prompt.version` rows; tenant-overridable; admin UI for non-developer authoring.
-   **Conversation primitives** — `ai.conversation` + `ai.message`. Generic; consumed by [Assistant](assistant.md), the future MCP server, and any copilot you write.
-   **Per-call cost + audit** — `ai.usage.log` records tokens in/out, USD spend, latency, conversation, principal, tenant. Per-tenant daily caps enforced as a `pre.ai.invocation` hook.
-   **Safety hooks** — `pre.ai.invocation`, `post.ai.invocation`, `pre.ai.tool_call`, `post.ai.tool_call` hookpoints; PII redaction, injection detection, and output validation ship as default `pre.ai.invocation` implementations (toggle individually).
-   **Per-user quotas** — `ai.user.quota` rows give admins user-grained ceilings on top of the tenant budget.
-   **Skill packs** — `ai.skill.pack` groups a prompt template with a curated tool subset. A consumer module references a skill pack; the catalogue and prompt are decided in data, not code.
-   **Gated write mode** — proposer tools can request a mutation; the user gets a confirmation prompt with a TTL-bounded token; on confirm, the bridge dispatches the mutation and records an `ai.write.provenance` row so every AI-driven change is auditable.
-   **HTTP surface** — `GET /api/ai/tools`, `GET /api/ai/usage`, `POST /api/ai/invoke`, conversation CRUD, message append, plus `POST /api/ai/conversations/{id}/confirm` and `/undo` for the gated write flow.

## How to use it

### Register a tool

Hang the method on the single carrier model `ai.tool` (no new table per tool) and decorate it with both `@api.on_command` and `@api.ai_tool`. The tool registry auto-scans every loaded module at boot.

```python
@api.on_command("ai.tool.count_partners")
@api.ai_tool(
    name="count_partners",
    read_only=True,
    kind="informer",
    description="Count the active partners in the tenant.",
    input_schema={"type": "object", "properties": {}},
)
def count_partners(self, cmd: Command):
    return {"count": self.env.models["res.partner"].search_count([])}
```

The boot validator confirms `count_partners` is `read_only=True`-eligible — `search_count` is in the read-only allow-list. A `pre.ir.tool.registry.bind` hook records the binding so revoking the tool later is one row delete.

### Invoke single-turn

```http
POST /api/ai/invoke
Authorization: Bearer <token>
Content-Type: application/json

{
    "prompt": "How many partners do we have?",
    "skill_pack": "base.read_only_inspector"
}
```

The provider picks a tool (or none), the bridge dispatches, the response composes. A single `ai.usage.log` row lands when the conversation closes.

### Drive a multi-turn conversation

```http
POST /api/ai/conversations
{ "skill_pack": "base.read_only_inspector" }
# → { "id": "<conv_uuid>" }

POST /api/ai/conversations/<conv_uuid>/messages
{ "role": "user", "content": "Show me the partner count again." }
# → { "messages": [...], "usage": {...} }
```

Each message turn runs the provider with the current message list + tool catalogue. The bridge stops after `AI_MAX_TOOL_ITERATIONS` tool calls per turn to bound runaway loops.

### Register a versioned prompt

```xml
<record id="prompt_inspector_v1" model="ai.prompt.template">
    <field name="key">base.inspector</field>
    <field name="name">Schema Inspector</field>
    <field name="description">Answer questions about the platform's registered models.</field>
</record>

<record id="prompt_inspector_v1_body" model="ai.prompt.version">
    <field name="template_id" ref="prompt_inspector_v1"/>
    <field name="version">1</field>
    <field name="body">You are a helpful assistant. Use the available tools to answer questions about the EDE platform schema.</field>
    <field name="is_default">true</field>
</record>
```

Tenants can override a template by inserting a new `ai.prompt.version` row with the same `template_id`; the registry resolves the highest version flagged `is_default` for the active tenant.

### Propose a write under the human-in-loop gate

Mark a tool as a proposer (writes allowed, confirmation required):

```python
@api.on_command("ai.tool.archive_partner")
@api.ai_tool(
    name="archive_partner",
    read_only=False,
    kind="proposer",
    description="Archive a partner record (requires user confirmation).",
    input_schema={
        "type": "object",
        "properties": {"partner_uuid": {"type": "string"}},
        "required": ["partner_uuid"],
    },
)
def archive_partner(self, cmd: Command):
    self.env.models["res.partner"].browse(cmd.payload["partner_uuid"]).write({"active": False})
```

On a proposer tool call, the bridge does *not* dispatch the mutation directly. It returns a confirmation token; the caller posts to `POST /api/ai/conversations/{id}/confirm` with the token within `AI_WRITE_CONFIRM_TOKEN_TTL_SECONDS`. The framework writes an `ai.write.provenance` row tying the resulting mutation to the conversation, the tool, the prompt version, and the confirming principal — so every AI-issued change is fully auditable.

### Read usage and budgets

```http
GET /api/ai/usage?tenant=<tenant_id>&day=2026-05-18
```

Returns the day's `ai.usage.log` rows aggregated by tool and principal. Cap usage with `AI_DAILY_BUDGET_USD`; the `pre.ai.invocation` budget hook returns a `BudgetExceeded` error when the cap is hit.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `AI_ENABLED` | `False` | Master switch — registers the module's models, routes, and command bus integration. |
| `AI_DEFAULT_PROVIDER` | `"anthropic"` | Provider used when a call doesn't pin one. Must be in `AI_ALLOWED_PROVIDERS`. |
| `AI_ALLOWED_PROVIDERS` | `["anthropic"]` | Whitelist of providers the tenant may use. |
| `AI_BYO_KEY_REQUIRED` | `True` | Force tenants to supply their own API keys (encrypted at rest in `ai.provider.config`). |
| `AI_DAILY_BUDGET_USD` | `0.0` | Hard daily spend cap per tenant; `0` means uncapped. |
| `AI_MAX_TOOL_ITERATIONS` | `8` | Maximum tool calls per single invocation; prevents runaway loops. |
| `AI_PII_REDACTION_ENABLED` | `False` | Run the PII redaction hook before sending a prompt to the provider. |
| `AI_PII_REDACTION_MODE` | `"strict"` | `strict` blocks on detected PII; `mask` rewrites it; `warn` only logs. |
| `AI_INJECTION_DETECTION_ENABLED` | `True` | Run the prompt-injection detector on user content. |
| `AI_INJECTION_DETECTION_THRESHOLD` | `60` | Confidence cutoff (0–100); higher = stricter. |
| `AI_OUTPUT_VALIDATION_MODE` | `"warn"` | Validate provider output against the input schema. `warn` / `strict` / `disabled`. |
| `AI_DATA_RESIDENCY_STRICT` | `False` | Refuse calls when the prompt would leave the tenant's allowed residency. |
| `AI_DATA_RESIDENCY_ALLOWED_PROVIDERS` | `[]` | Providers cleared to receive data under strict residency. |
| `AI_WRITE_TOOLS_ALLOWLIST` | `[]` | Proposer tools the tenant may actually use; empty = none. |
| `AI_WRITE_DAILY_CAP_RECORDS_PER_USER` | `0` | Cap on AI-issued mutations per user per day; `0` = unlimited. |
| `AI_WRITE_DAILY_CAP_PER_TOOL` | `0` | Cap on AI-issued mutations per tool per day; `0` = unlimited. |
| `AI_WRITE_CONFIRM_TOKEN_TTL_SECONDS` | `300` | Lifetime of the proposer-tool confirmation token. |

## How it composes with other features

-   [Permissions](security.md) — every AI-issued read still flows through the record-rule engine; an LLM can't see rows the calling principal can't see.
-   [Commands & events](../tutorial/04-commands-and-events.md) — the function-calling bridge is `env.dispatch` with a guard; tools are just commands with a decorator.
-   [Workflow](workflow.md) — workflow approval cases can be the confirmation step for a proposer tool (`ai.write.provenance.approval_case_id`).
-   [Assistant](assistant.md) — the chat UI is a pure consumer of these primitives; it speaks only to the tool registry, prompt registry, and conversation models.

## Reference

-   Models: `src/ede/foundation/ai/models/{conversation,prompt_template,usage_log,write_provenance,user_quota,skill_pack,safety_rule,provider_config,provider_routing}.py`
-   Decorator: `src/ede/core/api.py` (`ai_tool`)
-   Tool registry: `src/ede/foundation/ai/services/tool_registry.py`
-   Function-calling bridge: `src/ede/foundation/ai/services/bridge.py`
-   Cost + budget: `src/ede/foundation/ai/services/{cost,quota,budget_alert}.py`
-   Write-mode gate: `src/ede/foundation/ai/services/write_mode.py`
-   Provider secrets vault: `src/ede/foundation/ai/services/secrets.py`
-   Reference tool: `src/ede/foundation/ai/tools/read_schema.py`
-   HTTP routes: `src/ede/foundation/ai/api/ai_routes.py` (prefix `/api/ai`)
-   Seed prompts & provider configs: `src/ede/foundation/ai/data/seed_prompts.xml`, `src/ede/foundation/ai/data/seed_provider_configs.xml`
