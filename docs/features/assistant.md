# Assistant (In-App AI Chat)

A chat side panel bound to the user's currently open list, kanban, or form. The user types in natural language; the AI proposes filter / sort / groupby operations that auto-apply to the active screen with a ✨ marker showing the AI authored them. Read-only by contract — the assistant never mutates a record.

```http
POST /api/assistant/session
Authorization: Bearer <token>
Content-Type: application/json

{
    "active_action": {
        "model": "res.partner",
        "view_type": "list",
        "domain": [["active", "=", true]],
        "groupby": null,
        "sort": null
    }
}
# → { "session_id": "<uuid>", "panel_state": {...} }
```

```http
POST /api/assistant/session/<uuid>/message
{ "content": "Show partners in Germany sorted by name" }
# → {
#     "chat": "Filtering to partners in Germany, sorted alphabetically.",
#     "view_intent": { "ops": [
#         {"kind": "APPLY_FILTER_DOMAIN", "domain": [["country_id.code", "=", "DE"]]},
#         {"kind": "SET_SORT", "field": "name", "direction": "asc"}
#     ]},
#     "actions": []
# }
```

The frontend `AssistantPanel` dispatches each op into the same `ActionContext` reducer the SearchPanel uses — the resulting chips and sort indicator render identically whether the human or the AI typed it. One click on a chip removes it.

---

## What you get

-   **`AssistantPanel`** React side-panel — slides in from the right; bound to the active `ActionContext`. Configurable default width via `ASSISTANT_PANEL_DEFAULT_WIDTH_PX`.
-   **`assistant.session`** — one conversation row per user/screen pair. Holds the active `ai.conversation_id`, the captured screen context, the auto-apply preference, and the close state.
-   **`assistant.view_intent.log`** — every op the AI proposes lands here with an applied / dismissed flag. Audit trail for what the AI did to a user's screen.
-   **`assistant.skill.pack`** — domain modules ship rows to enrich the assistant's tool surface for their models. The Phase 1 row is a stub; the surface is wired so future packs slot in without a code change.
-   **Six tools out of the box** — three proposers (`propose_filter`, `propose_sort`, `propose_groupby`), one shared informer (`read_schema` — borrowed from [AI](ai.md)), one local informer (`explain_current_view`), one insight (`run_read_group`).
-   **View-intent response schema** — every turn returns `{ chat, view_intent: { ops[] }, actions[] }`. The frontend reducer interprets `ops` (filter / sort / groupby / view-type switch / open record / clear); `actions[]` are click-to-confirm buttons (navigate, export, etc.).
-   **Five HTTP endpoints** under `/api/assistant/` — session open, message append, intent-applied confirmation, session close, session read.
-   **Hard read-only contract** — the underlying tools register through [`@api.ai_tool(read_only=True)`](ai.md); the boot validator rejects any proposer tool whose command can mutate state. The assistant cannot dispatch a write.
-   **Polymorphic over every model the user can see** — the proposer tools operate generically against the registry and view DSL, so a new module gets a functional assistant on day one without registering anything.

## How to use it

### Enable the feature

```env
ASSISTANT_ENABLED=true
ASSISTANT_DEFAULT_AUTO_APPLY=true
```

Then ensure the underlying `foundation.ai` module is enabled (`AI_ENABLED=true`) and at least one provider config has a working API key — the assistant is a pure consumer of that layer.

### Open a session

A session captures the screen context — model, view type, current filters, current groupby, current sort. Open one when the user clicks the chat icon:

```http
POST /api/assistant/session
{
    "active_action": {
        "model": "res.organization",
        "view_type": "kanban",
        "domain": [],
        "groupby": "country_id",
        "sort": null
    }
}
```

The response includes the new `session_id` plus a `panel_state` describing what the user should see — typically a greeting and the most recent N messages if the session is being reopened.

### Send a message

```http
POST /api/assistant/session/<session_id>/message
{ "content": "Group instead by status, and only show active ones." }
```

A typical response:

```json
{
    "chat": "Switching groupby to status; filtering to active records.",
    "view_intent": {
        "ops": [
            {"kind": "SET_GROUPBY", "field": "status"},
            {"kind": "APPLY_FILTER_DOMAIN", "domain": [["active", "=", true]]}
        ]
    },
    "actions": []
}
```

The frontend dispatches the ops into `ActionContext`; chips render with the ✨ provenance marker.

### Confirm an op was applied

The frontend pings the backend after the reducer commits the ops so the audit log knows which proposals were applied vs. ignored:

```http
POST /api/assistant/session/<session_id>/intent-applied
{ "ops_applied": [0, 1], "ops_dismissed": [] }
```

The handler writes one `assistant.view_intent.log` row per op with the final applied / dismissed state.

### Close a session

```http
POST /api/assistant/session/<session_id>/close
```

Marks the session closed and frees the underlying `ai.conversation` for archival.

## The six view-intent op kinds

A proposer tool returns one or more ops; the frontend reducer recognises these:

| Kind | Payload | Effect |
|---|---|---|
| `APPLY_FILTER_DOMAIN` | `{domain: [...]}` | Merges into the active filter chip set. |
| `SET_GROUPBY` | `{field: "..."}` | Replaces the active groupby pill. |
| `SET_SORT` | `{field, direction}` | Sets the active sort indicator. |
| `SWITCH_VIEW_TYPE` | `{view_type: "list|kanban|form"}` | Switches the rendered view. |
| `OPEN_RECORD` | `{record_uuid: "..."}` | Navigates into a record's form view. |
| `CLEAR_FILTERS` | `{}` | Clears all active chips. |

A turn can return up to `ASSISTANT_MAX_VIEW_INTENT_OPS_PER_TURN` ops (default 6) — beyond that the reducer rejects the response.

## The three tool kinds

The assistant inherits the [AI tool-kind contract](ai.md):

| Kind | Output | Used by |
|---|---|---|
| **proposer** | View-intent ops auto-applied to screen | `propose_filter`, `propose_sort`, `propose_groupby` |
| **informer** | Chat text from metadata (no DB read) | `read_schema`, `explain_current_view` |
| **insight** | Chat text from a real read-only query | `run_read_group` |

Every tool ships with `read_only=True`. The boot validator rejects any proposer whose underlying command isn't on the read-only allow-list — write-mode is opt-in at a different surface entirely.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `ASSISTANT_ENABLED` | `False` | Register the module's models, routes, and frontend panel. |
| `ASSISTANT_DEFAULT_AUTO_APPLY` | `True` | Auto-apply view-intent ops as the AI proposes them. When `False`, ops show as click-to-confirm buttons instead. |
| `ASSISTANT_PANEL_DEFAULT_WIDTH_PX` | `420` | Initial width of the side-panel; user can drag to resize. |
| `ASSISTANT_MAX_VIEW_INTENT_OPS_PER_TURN` | `6` | Cap on ops per AI response; prevents runaway responses. |

Requires `foundation.ai` settings to be configured — at minimum `AI_ENABLED=true`, a provider in `AI_ALLOWED_PROVIDERS`, and an `ai.provider.config` row with credentials.

## How it composes with other features

-   [AI](ai.md) — the assistant is a pure consumer; every tool, prompt, and conversation lives in `foundation.ai`.
-   [Permissions](security.md) — every insight tool's read passes through the record-rule engine; the AI sees only what the calling principal can see.
-   [Form views](../tutorial/03-views-and-web-client.md) — the underlying `ActionContext` reducer is the same one the SearchPanel uses; chips render identically.
-   The [Model & View Extension SDK](base.md) extends `res.organization` with an "AI Assistant" preferences section via `<extend ref="...">` — see `src/ede/foundation/assistant/views/res_organization_ai_extension.xml`.

## Reference

-   Models: `src/ede/foundation/assistant/models/{session,view_intent_log,skill_pack,organization_extension}.py`
-   Turn service: `src/ede/foundation/assistant/services/turn_service.py`
-   Built-in tools: `src/ede/foundation/assistant/tools/assistant_tools.py`
-   HTTP routes: `src/ede/foundation/assistant/api/assistant_routes.py` (prefix `/api/assistant`)
-   Response schema: `src/ede/foundation/assistant/schemas.py`
-   Frontend panel: `src/frontend/src/managers/AssistantPanel.tsx` + `src/frontend/src/react/providers/AssistantPanelProvider.tsx` + `src/frontend/src/core/services/AssistantService.ts`
-   Org form extension: `src/ede/foundation/assistant/views/res_organization_ai_extension.xml`
-   Smoke-test gate: `src/ede/foundation/assistant/SMOKE_TEST.md`
