# Phase 4C — AI-Assisted View Customization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a user add a custom field by asking the in-app assistant ("add a Risk Band selection to this form"); a confirm-gated card then performs the real writes (`ir.model.property.definition` + `ir.application.view` extension) through the command bus with RBAC, and the field renders at the placed position.

**Architecture:** A read-only **proposer tool** (`propose_custom_field` on the `assistant.tool` carrier) resolves the host `ir.model` uuid + the acting principal's organization server-side and returns a **confirm-gated `ActionButton`** (the same `actions[]` channel `propose_navigate` already uses — the turn_service comment at `services/turn_service.py:84-86` explicitly reserves this channel for Phase 4 confirm buttons). The assistant itself never mutates (BR-AI-01, auto-enforced at boot by `validate_read_only` + re-checked in the bridge). The frontend renders a **`CustomFieldConfirmCard`**; on confirm it dispatches two `ede.create`s (definition + extension) via `runtime.services.action.createRecord`, supplying the extension's `parent_id` from a new render-plan field **`view_parent_id`** (the 4B-side enabler), then invalidates `["action-load"]` so the field re-composes in place.

**Tech Stack:** Python 3.10 / SQLAlchemy / pytest (backend); React 19 / TanStack Query / Vitest / Bun (frontend); XML DSL views; `@api.ai_tool` decorator surface.

**Design decision flagged for review (transport channel):** The roadmap §6 illustrates the proposal as a view-intent op (`{"op": "propose_custom_field", ...}`). This plan instead routes it through the existing **confirm-gated `ActionButton`** channel rather than the **auto-apply `view_intent.ops[]`** channel, because the op must NOT auto-apply (it must wait for explicit confirm) and `ActionButton`/`propose_navigate` is the established confirm-gated path. Same observable behavior; the ✨ marker is rendered on the card. If you'd rather it ride the `view_intent.ops[]` array with a "do-not-auto-apply" flag, say so before Task 2.

---

## File Structure

**Backend (`src/ede/foundation/`)**
- `presentation/models/presentations.py` — MODIFY: thread `view_parent_id` into each composed render plan (the 4B enabler).
- `assistant/schemas.py` — MODIFY: add `"propose_custom_field"` to `ActionButtonKind`.
- `assistant/tools/assistant_tools.py` — MODIFY: add the `propose_custom_field` proposer tool to the `assistant.tool` carrier.
- `assistant/services/turn_service.py` — MODIFY: register the tool in `_ACTION_BUTTON_TOOL_NAMES`, the allowed-kinds tuple, and `_CONTEXTUAL_TOOLS`.
- `assistant/data/seed_customization_pack.xml` — CREATE: the customization skill pack (4 rows) + prompt template/version.
- `assistant/__manifest__.py` — MODIFY: declare the new seed file.

**Frontend (`src/frontend/src/`)**
- `managers/assistant/types.ts` — MODIFY: extend `ActionButtonKind`; add the proposal payload type.
- `react/providers/AssistantBindingProvider.tsx` — MODIFY: expose `viewKey` + `viewParentId`.
- `managers/ActionManager.tsx` — MODIFY: source `viewKey`/`viewParentId` from the loaded render plan into the binding.
- `managers/assistant/CustomFieldConfirmCard.tsx` — CREATE: the confirm card.
- `managers/assistant/AssistantMessageLine.tsx` — MODIFY: render the card for `kind === "propose_custom_field"`.
- `managers/assistant/AssistantPanel.tsx` — MODIFY: the confirm handler (two creates + invalidate).

**Tests**
- `src/tests/assistant/test_customization_proposals.py` — CREATE (BR-AI-01).
- `src/tests/foundation/base/test_application_view_xpath_guard.py` — CREATE (4C-named xpath guard).
- `src/tests/assistant/test_customization_write_path.py` — CREATE (write path + RBAC denial, BR-AI-02).
- `src/frontend/src/managers/assistant/CustomFieldConfirmCard.test.tsx` — CREATE (vitest).
- `src/tests/e2e/usecases/customization/test_customization_property.py` — MODIFY (extend e2e).

**No migration:** all three target models (`ir.model.property.definition`, `ir.model.property.selection`, `ir.application.view`) already exist; `view_parent_id` is a response field, not a DB column. RBAC rows already exist (`ir.rbac.permission.csv:54-61`).

---

## Task 0: Roadmap honesty — record the `view_parent_id` enabler

**Files:**
- Modify: `roadmap/foundation/customization/phase-4/03-ai-driven-view-customization.md`
- Modify: `roadmap/foundation/customization/phase-4/02-*.md` (the feature-02 file; find exact name)

- [ ] **Step 1: Note the enabler on the 4C feature file**

Under the "Confirm-gated write path" section, add a bullet recording the new dependency:

```markdown
- [ ] **4B enabler — `view_parent_id` in the render plan:** `presentation.load_action` exposes, per composed view, the `record_uuid` of its primary `ir.application.view` row as `view_parent_id` (mirrors the existing `view_id`). The confirm card uses it as the extension row's `parent_id`. Falls back to `null` (confirm disabled) for file-only views with no DB row.
```

- [ ] **Step 2: Mirror the enabler note on feature 02** (the `ir.application.view` store), since the field is produced there.

```markdown
> 4C enabler: the composed render plan now also surfaces `view_parent_id` (the primary row `record_uuid`) so a client can author an extension against the open view.
```

- [ ] **Step 3: Commit**

```bash
git add roadmap/foundation/customization/phase-4/
git commit -m "[ADD] roadmap.customization: 4C view_parent_id enabler note (feature 02 + 4C)"
```

---

## Task 1: Backend enabler — `view_parent_id` in the render plan

**Files:**
- Modify: `src/ede/foundation/presentation/models/presentations.py` (`load_action`, ~line 690-790; add a helper)
- Modify: `src/ede/core/services/presentation/view_composer.py` (expose primary uuid)
- Test: `src/tests/foundation/presentation/test_load_action_view_parent_id.py`

- [ ] **Step 1: Write the failing test**

```python
# src/tests/foundation/presentation/test_load_action_view_parent_id.py
"""load_action surfaces view_parent_id (primary ir.application.view uuid) per view."""
from ede.core.types import Command


def test_load_action_includes_view_parent_id(seeded_env_with_db_views):
    """Each composed view in load_action carries view_parent_id = its primary row uuid."""
    env = seeded_env_with_db_views
    # The fixture seeds a primary ir.application.view for res.organization's form.
    primary = env.models["ir.application.view"].search(
        [("view_key", "=", "res_organization_form_view"), ("mode", "=", "primary")],
        limit=1,
    ).read(["record_uuid"])[0]

    result = env.models["foundation.presentation"].load_action(
        Command(name="presentation.load_action", payload={"action_key": "organizations"})
    )
    form_plan = result["views"]["form"]
    assert form_plan["view_parent_id"] == primary["record_uuid"]


def test_view_parent_id_null_for_file_only_view(seeded_env_file_views_only):
    """A view with no DB primary row falls back to view_parent_id = None."""
    env = seeded_env_file_views_only
    result = env.models["foundation.presentation"].load_action(
        Command(name="presentation.load_action", payload={"action_key": "organizations"})
    )
    assert result["views"]["form"]["view_parent_id"] is None
```

> If the named fixtures don't exist, reuse the fixtures the existing `src/tests/foundation/presentation/` and `src/tests/foundation/base/test_application_view.py` tests use to seed a DB-backed primary view; adapt the action_key/view_key to whatever those fixtures seed. The assertion (`view_parent_id == primary uuid`, and `None` for file-only) is the contract.

- [ ] **Step 2: Run it to confirm failure**

Run: `./run_tests.sh 2>&1 | tee /tmp/ede-tests-4c.log | grep -E 'view_parent_id|PASS|FAIL'`
Expected: FAIL — `KeyError: 'view_parent_id'`.

- [ ] **Step 3: Add a primary-uuid lookup on the ViewComposer**

In `src/ede/core/services/presentation/view_composer.py`, add a public method next to `compose_xml` (it already reads the primary row via `_read_primary`, which returns `record_uuid`):

```python
    def primary_view_uuid(self, *, view_key: str) -> Optional[str]:
        """Return the primary ir.application.view row's record_uuid for view_key.

        None when no DB-backed primary row exists (file-only view) — the caller
        degrades gracefully (the AI customization confirm is disabled client-side).
        """
        if not has_model("ir.application.view"):
            return None
        primary = self._read_primary(view_key=(view_key or "").strip())
        if not primary:
            return None
        return (primary.get("record_uuid") or None)
```

> Verify `_read_primary` already reads `record_uuid`; the existing call at `view_composer.py:313` does `.read(_PRIMARY_READ_FIELDS + ["record_uuid"])`. If `_read_primary` (the cached one) does NOT include it, add `"record_uuid"` to `_PRIMARY_READ_FIELDS`.

- [ ] **Step 4: Inject `view_parent_id` in `load_action`**

In `src/ede/foundation/presentation/models/presentations.py`, in `load_action`, immediately before `views[view_type] = render_plan` (line ~777):

```python
                render_plan["view_parent_id"] = ViewComposer(env=self.env).primary_view_uuid(
                    view_key=view_id
                )
                views[view_type] = render_plan
```

Ensure `ViewComposer` is imported at module top (it is used by `_compose_view_xml`; confirm `from ede.core.services.presentation.view_composer import ViewComposer` exists — if `_compose_view_xml` instantiates it inline, hoist the import to the top per CLAUDE.md "imports at module top").

- [ ] **Step 5: Run the test to confirm pass**

Run: `./run_tests.sh 2>&1 | tee /tmp/ede-tests-4c.log | grep -E 'view_parent_id|FAIL'`
Expected: the two new tests PASS, no new failures.

- [ ] **Step 6: Commit**

```bash
git add src/ede/core/services/presentation/view_composer.py src/ede/foundation/presentation/models/presentations.py src/tests/foundation/presentation/test_load_action_view_parent_id.py
git commit -m "[IMP] foundation.presentation: surface view_parent_id (primary ir.application.view uuid) in load_action (4C enabler)"
```

---

## Task 2: Backend — `propose_custom_field` ActionButtonKind

**Files:**
- Modify: `src/ede/foundation/assistant/schemas.py:38-43`

- [ ] **Step 1: Extend `ActionButtonKind`**

```python
ActionButtonKind = Literal[
    "navigate",
    "apply_to_current_view",
    "open_record",
    "export",
    "propose_custom_field",   # Phase 4C — confirm-gated custom-field proposal
]
```

- [ ] **Step 2: Commit** (committed together with Task 3-4 wiring; no test in isolation).

---

## Task 3: Backend — the `propose_custom_field` proposer tool

**Files:**
- Modify: `src/ede/foundation/assistant/tools/assistant_tools.py` (add method to `AssistantTool`, `@api.model("assistant.tool")` at line 173)
- Test: `src/tests/assistant/test_customization_proposals.py`

- [ ] **Step 1: Write the failing test (BR-AI-01 — well-formed op + zero mutating dispatches)**

```python
# src/tests/assistant/test_customization_proposals.py
"""Phase 4C — the customization proposer tool returns a confirm-gated ActionButton
payload and performs ZERO mutating dispatches (BR-AI-01)."""
import pytest

from ede.core.types import Command


def _call(env, payload):
    return env.dispatch(
        Command(name="assistant.tool.propose_custom_field", payload=payload, model_key="assistant.tool")
    )


def test_propose_custom_field_returns_actionbutton_payload(assistant_env):
    """A selection-field proposal returns kind=propose_custom_field with resolved model_id."""
    env = assistant_env  # principal has an active_organization_id; res.organization is a known model
    result = _call(env, {
        "model_key": "res.organization",
        "field_key": "risk_band",
        "field_label": "Risk Band",
        "field_type": "selection",
        "options": [{"key": "low", "label": "Low"}, {"key": "high", "label": "High"}],
        "required": False,
        "xpath": "//sheet",
        "position": "inside",
        "scope": "organization",
    })
    assert result["kind"] == "propose_custom_field"
    assert result["label"]
    p = result["payload"]
    assert p["field"]["key"] == "risk_band"
    assert p["field"]["type"] == "selection"
    assert len(p["field"]["options"]) == 2
    assert p["model_key"] == "res.organization"
    # model_id resolved server-side to the ir.model row uuid (a 36-char uuid string).
    assert isinstance(p["model_id"], str) and len(p["model_id"]) >= 32
    assert p["placement"] == {"xpath": "//sheet", "position": "inside"}
    assert p["scope"] == "organization"
    # org-scoped proposals carry the resolved organization_id (BR-AI-04).
    assert p["organization_id"]


def test_propose_global_field_has_null_org(assistant_env):
    result = _call(assistant_env, {
        "model_key": "res.organization",
        "field_key": "ext_ref",
        "field_label": "External Ref",
        "field_type": "char",
        "xpath": "//sheet",
        "position": "inside",
        "scope": "global",
    })
    assert result["payload"]["scope"] == "global"
    assert result["payload"]["organization_id"] is None


def test_propose_rejects_duplicate_key(assistant_env_with_existing_property):
    """Proposing a key that already exists on the model raises ValueError (no dup)."""
    env = assistant_env_with_existing_property  # seeds ir.model.property.definition key='risk_band' on res.organization
    with pytest.raises(ValueError, match="already exists"):
        _call(env, {
            "model_key": "res.organization",
            "field_key": "risk_band",
            "field_label": "Risk Band",
            "field_type": "char",
            "xpath": "//sheet",
            "position": "inside",
            "scope": "global",
        })


def test_proposal_turn_does_no_mutating_dispatch(assistant_env, dispatch_spy):
    """BR-AI-01: not a single create/update/delete is dispatched during the proposal."""
    _call(assistant_env, {
        "model_key": "res.organization",
        "field_key": "k1",
        "field_label": "K1",
        "field_type": "char",
        "xpath": "//sheet",
        "position": "inside",
        "scope": "global",
    })
    mutating = [c for c in dispatch_spy.commands if c.name in ("ede.create", "ede.update", "ede.delete")]
    assert mutating == []
```

> `dispatch_spy` records dispatched commands. If no such fixture exists, implement it inline: wrap `env.dispatch` with a recorder, or assert via a monkeypatch on the command bus. The key assertion is that no `ede.create/update/delete` fires. `assistant_env` / `assistant_env_with_existing_property` should reuse the assistant test fixtures in `src/tests/assistant/conftest.py`; if a principal-with-org fixture is missing, build one from the existing org-seeding fixtures.

- [ ] **Step 2: Run to confirm failure**

Run: `./run_tests.sh 2>&1 | tee /tmp/ede-tests-4c.log | grep -E 'propose_custom_field|test_customization_proposals|FAIL'`
Expected: FAIL — command `assistant.tool.propose_custom_field` not registered.

- [ ] **Step 3: Implement the proposer tool**

In `src/ede/foundation/assistant/tools/assistant_tools.py`, add to the `AssistantTool` class. First add the input schema constant near the other `_PROPOSE_*_SCHEMA` constants:

```python
_PROPOSE_CUSTOM_FIELD_SCHEMA = {
    "type": "object",
    "properties": {
        "model_key": {"type": "string", "description": "Host model the field attaches to (the current screen's model)."},
        "field_key": {"type": "string", "description": "lower_snake_case slug for the new field, e.g. 'risk_band'."},
        "field_label": {"type": "string", "description": "Human label, e.g. 'Risk Band'."},
        "field_type": {
            "type": "string",
            "enum": ["char", "integer", "decimal", "boolean", "date", "datetime", "selection"],
            "description": "Property type. Use 'selection' for a fixed option list.",
        },
        "options": {
            "type": "array",
            "description": "Required when field_type='selection': the option list.",
            "items": {
                "type": "object",
                "properties": {"key": {"type": "string"}, "label": {"type": "string"}},
                "required": ["key", "label"],
            },
        },
        "required": {"type": "boolean", "description": "Whether the host record must have a value.", "default": False},
        "help": {"type": "string", "description": "Optional help text."},
        "xpath": {"type": "string", "description": "Placement target in the open view, e.g. \"//sheet\" or \"//page[@name='other']\"."},
        "position": {"type": "string", "enum": ["inside", "after", "before"], "default": "inside"},
        "scope": {"type": "string", "enum": ["organization", "global"], "default": "organization",
                  "description": "'organization' restricts the field to the user's org; 'global' applies tenant-wide."},
    },
    "required": ["model_key", "field_key", "field_label", "field_type", "xpath"],
}
```

Then the handler (read-only — validates + shapes, never dispatches):

```python
    @api.ai_tool(
        name="propose_custom_field",
        read_only=True,
        kind="proposer",
        description=(
            "Propose adding a new custom field to the model the user is currently looking at. "
            "Gather the field's label, type (char/integer/decimal/boolean/date/datetime/selection), "
            "and — for selection — its options, plus where it should sit (xpath of a container in the "
            "open view, e.g. //sheet). Returns a confirm card; the field is only created after the user "
            "confirms. Use scope='organization' unless the user explicitly wants it for the whole company."
        ),
        input_schema=_PROPOSE_CUSTOM_FIELD_SCHEMA,
    )
    @api.read_only_command
    @api.on_command("assistant.tool.propose_custom_field")
    def propose_custom_field(self, cmd: Command) -> dict:
        payload = cmd.payload or {}
        model_key = str(payload.get("model_key") or "").strip()
        field_key = str(payload.get("field_key") or "").strip()
        field_label = str(payload.get("field_label") or "").strip()
        field_type = str(payload.get("field_type") or "").strip()
        scope = str(payload.get("scope") or "organization").strip()
        position = str(payload.get("position") or "inside").strip()
        xpath = str(payload.get("xpath") or "").strip()

        _require_known_model(self.env.registry, model_key)
        if not field_key or not field_label or not field_type:
            raise ValueError("propose_custom_field requires field_key, field_label and field_type.")
        if not xpath:
            raise ValueError("propose_custom_field requires an xpath placement target.")
        if field_type == "selection" and not (payload.get("options") or []):
            raise ValueError("propose_custom_field: field_type='selection' requires a non-empty 'options' list.")

        # Resolve the host ir.model row uuid (direct read — never via the bus).
        ir_model_rows = self.env.models["ir.model"].search(
            [("model_key", "=", model_key)], limit=1
        ).read(["record_uuid"])
        if not ir_model_rows:
            raise ValueError(f"propose_custom_field: model {model_key!r} has no ir.model row.")
        model_id = ir_model_rows[0]["record_uuid"]

        # Reject a key that already exists on this model (no duplicate property).
        existing = self.env.models["ir.model.property.definition"].search(
            [("model_id", "=", model_id), ("key", "=", field_key)], limit=1
        )
        if existing:
            raise ValueError(
                f"propose_custom_field: a property with key {field_key!r} already exists on {model_key!r}."
            )

        # BR-AI-04: org-scope resolves organization_id from the acting principal.
        organization_id = None
        if scope == "organization":
            organization_id = (self.env.principal or {}).get("active_organization_id")
            if not organization_id:
                raise ValueError(
                    "propose_custom_field: scope='organization' but the principal has no active organization."
                )

        field = {
            "key": field_key,
            "label": field_label,
            "type": field_type,
            "required": bool(payload.get("required") or False),
            "help": str(payload.get("help") or "") or None,
        }
        if field_type == "selection":
            field["options"] = [
                {"key": str(o.get("key")), "label": str(o.get("label"))}
                for o in (payload.get("options") or [])
            ]

        return {
            "kind": "propose_custom_field",
            "label": f"Add “{field_label}” field",
            "payload": {
                "model_key": model_key,
                "model_id": model_id,
                "field": field,
                "placement": {"xpath": xpath, "position": position},
                "scope": scope,
                "organization_id": organization_id,
            },
        }
```

> `_require_known_model` already exists in this module (used by `propose_filter`). Confirm its import/definition; reuse it.

- [ ] **Step 4: Run the test to confirm pass**

Run: `./run_tests.sh 2>&1 | tee /tmp/ede-tests-4c.log | grep -E 'test_customization_proposals|FAIL'`
Expected: all four PASS.

- [ ] **Step 5: Commit**

```bash
git add src/ede/foundation/assistant/schemas.py src/ede/foundation/assistant/tools/assistant_tools.py src/tests/assistant/test_customization_proposals.py
git commit -m "[ADD] foundation.assistant: propose_custom_field proposer tool (4C, read-only, BR-AI-01/04)"
```

---

## Task 4: Backend — wire the tool into the turn composer

**Files:**
- Modify: `src/ede/foundation/assistant/services/turn_service.py:86`, `:120-135`, `:972`

- [ ] **Step 1: Register the tool name as an action-button producer**

```python
_ACTION_BUTTON_TOOL_NAMES = frozenset({"propose_navigate", "propose_custom_field"})
```

- [ ] **Step 2: Accept the new kind in the composer**

At `turn_service.py:972`, extend the allowed-kinds guard:

```python
            if kind in ("navigate", "apply_to_current_view", "open_record", "export", "propose_custom_field") and label:
```

- [ ] **Step 3: Expose the tool to contextual sessions**

Add `"propose_custom_field"` to the `_CONTEXTUAL_TOOLS` tuple (it stays out of `_GLOBAL_TOOLS` — customization needs an anchored screen):

```python
_CONTEXTUAL_TOOLS = (
    "propose_filter",
    "propose_groupby",
    "propose_sort",
    "propose_view_switch",
    "propose_open_record",
    "propose_columns",
    "propose_clear",
    "propose_custom_field",   # Phase 4C
    "explain_current_view",
    "read_schema",
    "run_read_group",
    "propose_navigate",
)
```

- [ ] **Step 4: Run the proposal test through a full turn** (if a turn-level fixture exists, assert the action surfaces in `response.actions[]`). Otherwise rely on Task 3 unit coverage + the e2e in Task 11.

Run: `./run_tests.sh 2>&1 | tee /tmp/ede-tests-4c.log | grep -E 'turn|assistant|FAIL'`
Expected: no regressions.

- [ ] **Step 5: Commit**

```bash
git add src/ede/foundation/assistant/services/turn_service.py
git commit -m "[IMP] foundation.assistant: route propose_custom_field through actions[] (confirm-gated, 4C)"
```

---

## Task 5: Backend — seed the customization skill pack

**Files:**
- Create: `src/ede/foundation/assistant/data/seed_customization_pack.xml`
- Modify: `src/ede/foundation/assistant/__manifest__.py` (`data` list, ~line 42-57)

- [ ] **Step 1: Author the pack (4 rows in lockstep, mirroring `seed_skill_packs.xml`)**

```xml
<?xml version="1.0" encoding="utf-8"?>
<ede>
    <data>
        <record id="assistant.prompt_template_customization" model="ai.prompt.template">
            <field name="key">assistant.skill.customization</field>
            <field name="name">View Customization — System Prompt</field>
            <field name="description">Proposer pack. Turns "add a field" asks into a confirm-gated custom-field proposal.</field>
            <field name="tenant_id">system</field>
            <field name="active">true</field>
        </record>

        <record id="assistant.prompt_version_customization_v1" model="ai.prompt.version">
            <field name="template_id" ref="assistant.prompt_template_customization"/>
            <field name="version">1</field>
            <field name="system_prompt">You help the user add a custom field to the model they are currently viewing. Collect: the field label, its type (char/integer/decimal/boolean/date/datetime, or selection with options), whether it is required, and roughly where it should appear. Default scope to 'organization' unless the user clearly wants it company-wide. When you have enough, call propose_custom_field with an xpath placement (use "//sheet" for a form's main body unless the user names a section). Do NOT create anything yourself — the user confirms the proposal card. Reply in one short sentence describing what you proposed.</field>
        </record>

        <record id="assistant.ai_skill_pack_customization" model="ai.skill.pack">
            <field name="key">assistant.customization</field>
            <field name="name">View Customization</field>
            <field name="description">Add custom fields to the current screen by asking.</field>
            <field name="tool_names">["propose_custom_field", "read_schema", "explain_current_view"]</field>
            <field name="prompt_template_id" ref="assistant.prompt_template_customization"/>
            <field name="example_questions">["Add a Risk Band selection field to this form", "Create a text field called Internal Notes"]</field>
            <field name="tenant_id">system</field>
            <field name="active">true</field>
        </record>

        <record id="assistant.skill_pack_customization" model="assistant.skill.pack">
            <field name="key">assistant.customization</field>
            <field name="name">View Customization</field>
            <field name="description">Add custom fields to the current screen by asking.</field>
            <field name="ai_skill_pack_id" ref="assistant.ai_skill_pack_customization"/>
            <field name="launch_mode">contextual</field>
            <field name="anchored_model_keys">[]</field>
            <field name="tenant_id">system</field>
            <field name="active">true</field>
        </record>
    </data>
</ede>
```

> Cross-check the exact field names against `seed_skill_packs.xml` (e.g. `example_questions` vs `example_prompts`, `tool_names` shape). Match that file verbatim — it is the authority on the pack record shape.

- [ ] **Step 2: Declare it in the manifest**

In `src/ede/foundation/assistant/__manifest__.py`, append to `data`:

```python
        "data/seed_customization_pack.xml",
```

- [ ] **Step 3: Smoke-load on a throwaway tenant**

Run: `ede migrate upgrade -t <dev-tenant> 2>&1 | tail -20`
Expected: the 4 rows load without error; no "unknown field" complaints.

- [ ] **Step 4: Commit**

```bash
git add src/ede/foundation/assistant/data/seed_customization_pack.xml src/ede/foundation/assistant/__manifest__.py
git commit -m "[ADD] foundation.assistant: seed View Customization skill pack (4C)"
```

---

## Task 6: Backend — 4C-named xpath guard tests

**Files:**
- Create: `src/tests/foundation/base/test_application_view_xpath_guard.py`

> The guard itself shipped in 4B (`application_view.py:316-344` create-side dry-run; `view_composer.py:269-283` render-side skip). This task is the 4C-named coverage the roadmap calls for — no production change expected.

- [ ] **Step 1: Write the tests**

```python
# src/tests/foundation/base/test_application_view_xpath_guard.py
"""Phase 4C — xpath safety on ir.application.view extension rows.

Create-side: an unresolvable xpath is rejected before persist (BR-AV-04).
Render-side: an extension whose xpath no longer resolves is skipped with a
warning, never crashing composition.
"""
import logging

import pytest

from ede.core.types import Command


def test_unresolvable_xpath_rejected_at_create(env_with_primary_view):
    """An extension targeting a non-existent node fails the create dry-run."""
    env = env_with_primary_view
    parent = env.models["ir.application.view"].search(
        [("mode", "=", "primary")], limit=1
    ).read(["record_uuid", "view_type"])[0]
    with pytest.raises(ValueError, match="BR-AV-04|dry-run"):
        env.dispatch(Command(name="ede.create", payload={"values": {
            "view_key": "ext_bad_xpath",
            "view_type": parent["view_type"],
            "mode": "extension",
            "owner": "user",
            "parent_id": parent["record_uuid"],
            "scope": "global",
            "arch": "<xpath expr=\"//nonexistent_node\" position=\"inside\"><property name=\"properties:x\"/></xpath>",
        }}}, model_key="ir.application.view"))


def test_drifted_xpath_skipped_at_compose(env_with_primary_view, caplog):
    """When a valid-at-create extension later targets a drifted base, compose skips it."""
    # Implementation: create a valid extension, then mutate the parent arch so the
    # xpath no longer resolves (sync-guard write), then compose and assert the view
    # still renders (no exception) and a warning was logged.
    env = env_with_primary_view
    # ... seed valid extension, drift parent arch via view_sync_writes(), compose ...
    with caplog.at_level(logging.WARNING):
        xml = env.models["foundation.presentation"]._compose_view_xml(
            view_registry=env.view_registry, view_id="<the_primary_view_key>"
        )
    assert xml  # composition succeeded despite the drifted extension
    assert any("skipping extension" in r.message for r in caplog.records)
```

> Fill the drift step concretely using `view_sync_writes()` (imported from `ede.foundation.base.models.application_view`) to update the parent `arch` past the ownership guard. Model the create/seed on the existing `src/tests/foundation/base/test_application_view.py` fixtures (it already has xpath-validation tests ~line 170-195 — reuse its `env_with_primary_view`-style fixture; rename to whatever that file uses).

- [ ] **Step 2: Run**

Run: `./run_tests.sh 2>&1 | tee /tmp/ede-tests-4c.log | grep -E 'xpath_guard|FAIL'`
Expected: PASS (production already implements the guard).

- [ ] **Step 3: Commit**

```bash
git add src/tests/foundation/base/test_application_view_xpath_guard.py
git commit -m "[ADD] tests.foundation.base: 4C-named xpath guard coverage (BR-AV-04 create + render)"
```

---

## Task 7: Backend — write-path + RBAC denial test (BR-AI-02)

**Files:**
- Create: `src/tests/assistant/test_customization_write_path.py`

> This proves the two `ede.create`s the frontend will dispatch succeed for an admin and are blocked for a non-admin. RBAC rows already exist: `ir.model.property.definition.create` and `ir.application.view.create` are gated to `rbac.role_system_admin` (`ir.rbac.permission.csv:55,59`); an `internal_user` principal has only read.

- [ ] **Step 1: Write the tests**

```python
# src/tests/assistant/test_customization_write_path.py
"""Phase 4C — the confirm write path: definition + extension creates, gated by RBAC."""
import pytest

from ede.core.types import Command


def _create_definition(env, model_id):
    return env.dispatch(Command(name="ede.create", payload={"values": {
        "model_id": model_id,
        "key": "risk_band",
        "label": "Risk Band",
        "property_type": "selection",
    }}}, model_key="ir.model.property.definition"))


def _create_extension(env, parent_uuid, view_type):
    return env.dispatch(Command(name="ede.create", payload={"values": {
        "view_key": "ext_risk_band_user",
        "view_type": view_type,
        "mode": "extension",
        "owner": "user",
        "parent_id": parent_uuid,
        "scope": "global",
        "arch": "<xpath expr=\"//sheet\" position=\"inside\"><property name=\"properties:risk_band\"/></xpath>",
    }}}, model_key="ir.application.view"))


def test_admin_can_create_definition_and_extension(admin_env_with_primary_view):
    env = admin_env_with_primary_view
    model_id = env.models["ir.model"].search(
        [("model_key", "=", "res.organization")], limit=1
    ).read(["record_uuid"])[0]["record_uuid"]
    definition = _create_definition(env, model_id)
    assert definition is not None
    # add the selection options
    env.dispatch(Command(name="ede.create", payload={"values": {
        "definition_id": definition.record_uuid, "key": "low", "label": "Low",
    }}}, model_key="ir.model.property.selection"))
    primary = env.models["ir.application.view"].search(
        [("view_key", "=", "res_organization_form_view"), ("mode", "=", "primary")], limit=1
    ).read(["record_uuid", "view_type"])[0]
    ext = _create_extension(env, primary["record_uuid"], primary["view_type"])
    assert ext is not None


def test_non_admin_blocked_by_rbac(internal_user_env_with_primary_view):
    """BR-AI-02: a principal without create permission cannot complete the writes."""
    env = internal_user_env_with_primary_view
    model_id = env.models["ir.model"].search(
        [("model_key", "=", "res.organization")], limit=1
    ).read(["record_uuid"])[0]["record_uuid"]
    with pytest.raises(Exception):  # PermissionError / authorization error from the RBAC layer
        _create_definition(env, model_id)
```

> Use the project's existing RBAC-aware env fixtures (search `src/tests/` for how `using-rbac-permission-registry` tests build an `internal_user` vs `system_admin` principal env). Tighten the `pytest.raises(Exception)` to the framework's actual authorization error class once you see it in the first run. Do NOT relax any RBAC rule to make this pass — the denial IS the assertion.

- [ ] **Step 2: Run**

Run: `./run_tests.sh 2>&1 | tee /tmp/ede-tests-4c.log | grep -E 'write_path|FAIL'`
Expected: both PASS (admin succeeds, internal_user denied).

- [ ] **Step 3: Commit**

```bash
git add src/tests/assistant/test_customization_write_path.py
git commit -m "[ADD] tests.assistant: 4C write-path + RBAC denial (BR-AI-02)"
```

---

## Task 8: Frontend — types + binding exposure

**Files:**
- Modify: `src/frontend/src/managers/assistant/types.ts:14` (and add proposal payload type)
- Modify: `src/frontend/src/react/providers/AssistantBindingProvider.tsx:17-48`
- Modify: `src/frontend/src/managers/ActionManager.tsx` (where the binding value is assembled)

- [ ] **Step 1: Extend the TS contract**

In `types.ts`:

```typescript
export type ActionButtonKind =
    | "navigate"
    | "apply_to_current_view"
    | "open_record"
    | "export"
    | "propose_custom_field";

export interface CustomFieldProposal {
    model_key: string;
    model_id: string;
    field: {
        key: string;
        label: string;
        type: "char" | "integer" | "decimal" | "boolean" | "date" | "datetime" | "selection";
        required: boolean;
        help: string | null;
        options?: { key: string; label: string }[];
    };
    placement: { xpath: string; position: "inside" | "after" | "before" };
    scope: "organization" | "global";
    organization_id: string | null;
}
```

- [ ] **Step 2: Expose `viewKey` + `viewParentId` on the binding**

In `AssistantBindingProvider.tsx`, add to `AssistantBindingContextValue`:

```typescript
    viewKey: string | null;          // the open view's view_id (XML id)
    viewParentId: string | null;     // primary ir.application.view uuid (null for file-only views)
```

- [ ] **Step 3: Source them in ActionManager**

Where ActionManager builds the binding value (the same place it sets `anchoredModelKey`/`viewType`), read from the loaded `ActionLoad` render plan for the active `viewType`:

```typescript
    const activePlan =
        viewType === "form" ? actionLoad?.formPlan :
        viewType === "list" ? actionLoad?.listPlan :
        viewType === "kanban" ? actionLoad?.kanbanPlan : null;
    const viewKey = (activePlan as { view_id?: string } | null)?.view_id ?? null;
    const viewParentId = (activePlan as { view_parent_id?: string } | null)?.view_parent_id ?? null;
```

Add `view_id` and `view_parent_id` to the relevant render-plan TS contracts in `src/frontend/src/core/contracts/domain.ts` (the `RenderPlan` base already allows index access, but type them explicitly on `FormRenderPlan`/`ListRenderPlan`/`KanbanPlan` for safety).

- [ ] **Step 4: Verify build**

Run: `cd src/frontend && bun run build 2>&1 | tail -20`
Expected: clean (no TS errors).

- [ ] **Step 5: Commit**

```bash
git add src/frontend/src/managers/assistant/types.ts src/frontend/src/react/providers/AssistantBindingProvider.tsx src/frontend/src/managers/ActionManager.tsx src/frontend/src/core/contracts/domain.ts
git commit -m "[IMP] frontend.assistant: expose viewKey + viewParentId on binding; propose_custom_field types (4C)"
```

---

## Task 9: Frontend — the confirm card

**Files:**
- Create: `src/frontend/src/managers/assistant/CustomFieldConfirmCard.tsx`
- Modify: `src/frontend/src/managers/assistant/AssistantMessageLine.tsx:117-127`

- [ ] **Step 1: Build the card (Tailwind utilities + theme tokens only — no inline styles/custom CSS)**

```tsx
// src/frontend/src/managers/assistant/CustomFieldConfirmCard.tsx
import { useState } from "react";
import { Sparkles } from "lucide-react";

import type { ActionButton, CustomFieldProposal } from "./types";

interface Props {
    action: ActionButton;
    disabled?: boolean;
    onConfirm: (proposal: CustomFieldProposal) => Promise<void>;
}

export function CustomFieldConfirmCard({ action, disabled, onConfirm }: Props) {
    const proposal = action.payload as unknown as CustomFieldProposal;
    const [busy, setBusy] = useState(false);
    const [done, setDone] = useState(false);

    const handleConfirm = async () => {
        setBusy(true);
        try {
            await onConfirm(proposal);
            setDone(true);
        } finally {
            setBusy(false);
        }
    };

    const f = proposal.field;
    return (
        <div
            className="assistant-confirm-card rounded-md border border-border bg-surface p-3 text-sm"
            data-testid="custom-field-confirm-card"
        >
            <div className="flex items-center gap-2 text-primary">
                <Sparkles className="h-4 w-4" aria-hidden />
                <span className="font-medium">{action.label}</span>
            </div>
            <dl className="mt-2 grid grid-cols-[auto_1fr] gap-x-3 gap-y-1 text-muted-foreground">
                <dt>Field</dt><dd className="text-foreground">{f.label} ({f.type})</dd>
                {f.type === "selection" && f.options ? (
                    <>
                        <dt>Options</dt>
                        <dd className="text-foreground">{f.options.map((o) => o.label).join(", ")}</dd>
                    </>
                ) : null}
                <dt>Placement</dt><dd className="text-foreground">{proposal.placement.position} {proposal.placement.xpath}</dd>
                <dt>Scope</dt><dd className="text-foreground">{proposal.scope}</dd>
            </dl>
            <div className="mt-3 flex justify-end gap-2">
                {done ? (
                    <span className="text-success" data-testid="custom-field-confirm-done">Added.</span>
                ) : (
                    <button
                        type="button"
                        className="rounded-md bg-primary px-3 py-1 text-primary-foreground disabled:opacity-50"
                        data-testid="custom-field-confirm-btn"
                        disabled={disabled || busy}
                        onClick={handleConfirm}
                    >
                        {busy ? "Adding…" : "Add field"}
                    </button>
                )}
            </div>
        </div>
    );
}
```

> Confirm the exact theme token class names against `src/frontend/src/theme/` (`bg-surface`, `text-muted-foreground`, `border-border`, `text-success`, `bg-primary`/`text-primary-foreground`). Swap to whatever the theme actually defines — do NOT introduce new raw values.

- [ ] **Step 2: Render it in the message line**

In `AssistantMessageLine.tsx`, replace the thin-chip map so `propose_custom_field` renders the card:

```tsx
                    {line.actions.map((action, i) =>
                        action.kind === "propose_custom_field" ? (
                            <CustomFieldConfirmCard
                                key={`ccf-${i}`}
                                action={action}
                                disabled={!onCustomFieldConfirm}
                                onConfirm={onCustomFieldConfirm ?? (async () => {})}
                            />
                        ) : (
                            <AssistantActionButton
                                key={`${action.kind}-${i}`}
                                action={action}
                                onClick={onActionClick ?? (() => {})}
                            />
                        ),
                    )}
```

Add `onCustomFieldConfirm?: (p: CustomFieldProposal) => Promise<void>;` to the message-line `Props` and import the card + type.

- [ ] **Step 3: Verify build**

Run: `cd src/frontend && bun run build 2>&1 | tail -20`
Expected: clean.

- [ ] **Step 4: Commit**

```bash
git add src/frontend/src/managers/assistant/CustomFieldConfirmCard.tsx src/frontend/src/managers/assistant/AssistantMessageLine.tsx
git commit -m "[ADD] frontend.assistant: CustomFieldConfirmCard + message-line render branch (4C)"
```

---

## Task 10: Frontend — the confirm handler (two creates + invalidate) + vitest

**Files:**
- Modify: `src/frontend/src/managers/assistant/AssistantPanel.tsx`
- Test: `src/frontend/src/managers/assistant/CustomFieldConfirmCard.test.tsx`

- [ ] **Step 1: Write the failing vitest**

```tsx
// src/frontend/src/managers/assistant/CustomFieldConfirmCard.test.tsx
import { fireEvent, render, screen, waitFor } from "@testing-library/react";
import { describe, expect, it, vi } from "vitest";

import { CustomFieldConfirmCard } from "./CustomFieldConfirmCard";
import type { ActionButton } from "./types";

const action: ActionButton = {
    kind: "propose_custom_field",
    label: "Add “Risk Band” field",
    payload: {
        model_key: "res.organization",
        model_id: "uuid-model",
        field: { key: "risk_band", label: "Risk Band", type: "selection", required: false, help: null,
                 options: [{ key: "low", label: "Low" }, { key: "high", label: "High" }] },
        placement: { xpath: "//sheet", position: "inside" },
        scope: "organization",
        organization_id: "uuid-org",
    },
} as ActionButton;

describe("CustomFieldConfirmCard", () => {
    it("renders the proposal summary and the confirm button", () => {
        render(<CustomFieldConfirmCard action={action} onConfirm={vi.fn().mockResolvedValue(undefined)} />);
        expect(screen.getByTestId("custom-field-confirm-card")).toBeInTheDocument();
        expect(screen.getByText(/Risk Band \(selection\)/)).toBeInTheDocument();
        expect(screen.getByText(/Low, High/)).toBeInTheDocument();
    });

    it("calls onConfirm with the proposal and shows Added on success", async () => {
        const onConfirm = vi.fn().mockResolvedValue(undefined);
        render(<CustomFieldConfirmCard action={action} onConfirm={onConfirm} />);
        fireEvent.click(screen.getByTestId("custom-field-confirm-btn"));
        await waitFor(() => expect(onConfirm).toHaveBeenCalledWith(action.payload));
        await waitFor(() => expect(screen.getByTestId("custom-field-confirm-done")).toBeInTheDocument());
    });
});
```

- [ ] **Step 2: Run to confirm failure**

Run: `cd src/frontend && bun run test CustomFieldConfirmCard 2>&1 | tail -20`
Expected: FAIL (component wiring not complete / import errors) — or PASS for render if Task 9 done; the handler test drives Step 3.

- [ ] **Step 3: Implement the confirm handler in AssistantPanel**

Add a `useCallback` that performs the two creates then invalidates, and pass it to the message-line render (thread `onCustomFieldConfirm` through to `AssistantMessageLine`):

```tsx
    const runtime = useRuntime();
    const queryClient = useQueryClient();
    const binding = useAssistantBinding();

    const handleCustomFieldConfirm = useCallback(
        async (p: CustomFieldProposal): Promise<void> => {
            if (!binding.viewParentId) {
                throw new Error("Cannot add a field: this view is not customizable (no DB-backed parent).");
            }
            // 1) property definition
            const def = await runtime.services.action.createRecord("ir.model.property.definition", {
                model_id: p.model_id,
                key: p.field.key,
                label: p.field.label,
                property_type: p.field.type,
                required: p.field.required,
                help: p.field.help,
            });
            // 2) selection options (selection type only)
            if (p.field.type === "selection" && p.field.options) {
                let seq = 10;
                for (const opt of p.field.options) {
                    await runtime.services.action.createRecord("ir.model.property.selection", {
                        definition_id: (def as { record_uuid: string }).record_uuid,
                        key: opt.key,
                        label: opt.label,
                        sequence: seq,
                    });
                    seq += 10;
                }
            }
            // 3) the ir.application.view extension placing the property
            const arch =
                `<xpath expr="${p.placement.xpath}" position="${p.placement.position}">` +
                `<property name="properties:${p.field.key}"/></xpath>`;
            await runtime.services.action.createRecord("ir.application.view", {
                view_key: `ext_${p.model_key.replace(/\./g, "_")}_${p.field.key}_user`,
                view_type: binding.viewType,
                mode: "extension",
                owner: "user",
                parent_id: binding.viewParentId,
                scope: p.scope,
                organization_id: p.scope === "organization" ? p.organization_id : null,
                arch,
            });
            // 4) re-compose: the new property re-renders in place
            queryClient.invalidateQueries({ queryKey: ["action-load"] });
            queryClient.invalidateQueries({ queryKey: ["record"] });
        },
        [runtime, queryClient, binding.viewParentId, binding.viewType],
    );
```

> Errors (incl. RBAC 403) propagate — the command-bus middleware in `react/runtime/buildRuntime.ts:151` shows the destructive toast automatically. The card's `finally` re-enables the button. Confirm `useAssistantBinding`/`useRuntime`/`useQueryClient` import paths against the existing AssistantPanel imports. Confirm the actual query keys (`["action-load"]`, `["record"]`) against `usePlatformCommands.ts:206-218`.

- [ ] **Step 4: Run vitest to pass**

Run: `cd src/frontend && bun run test CustomFieldConfirmCard 2>&1 | tail -20`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/frontend/src/managers/assistant/AssistantPanel.tsx src/frontend/src/managers/assistant/CustomFieldConfirmCard.test.tsx
git commit -m "[ADD] frontend.assistant: confirm handler (definition + extension creates + re-compose) + vitest (4C)"
```

---

## Task 11: E2E — propose → confirm → render → save

**Files:**
- Modify: `src/tests/e2e/usecases/customization/test_customization_property.py` (find the exact 4B path via `authoring-e2e-tests` skill)

- [ ] **Step 1: Extend the existing 4B property e2e**

Add a flow: open the customization-enabled demo form (the 4B Vendor Code rate form), open the assistant panel, ask "add a selection field …", assert the confirm card appears, click confirm, assert the new field renders at the placement, set a value, save, reload, assert the value persisted.

> Author this with the `authoring-e2e-tests` skill (it governs the recorder/replay harness and the demo-tenant setup). Reuse the 4B demo tenant + the rate form already wired for custom properties.

- [ ] **Step 2: Run the e2e** per the `authoring-e2e-tests` skill's invocation (not part of `./run_tests.sh` unit flow).

- [ ] **Step 3: Commit**

```bash
git add src/tests/e2e/usecases/customization/test_customization_property.py
git commit -m "[ADD] e2e.customization: AI propose → confirm → render → save (4C)"
```

---

## Task 12: Full verification + status flip

**Files:**
- Modify (status, 4 sites): `roadmap/foundation/customization/phase-4/03-ai-driven-view-customization.md`, `phase-4/README.md`, `roadmap/foundation/customization/README.md`, `roadmap/roadmap-tracker.md`

- [ ] **Step 1: Full backend suite** (use the `running-tests` skill)

Run: `./run_tests.sh 2>&1 | tee /tmp/ede-tests-4c.log | tail -30`
Expected: ≥ 3779 pass (the checkpoint baseline) + the new 4C tests, 0 failures.

- [ ] **Step 2: Full frontend gate** (use `verifying-frontend-build`)

Run: `cd src/frontend && bun run build && bun run test 2>&1 | tail -30`
Expected: clean build; ≥ 560 vitest + the new card test pass.

- [ ] **Step 3: Demo-data gate** (use `preparing-demo-data`)

The skill pack ships as `data/` system seed (always loaded), like the filter-builder pack — not a `demo/` file. The customizable host (Vendor Code rate form) ships with 4B demo. Confirm no new `demo/` file is required; document the rationale in the feature file's verification section.

- [ ] **Step 4: Flip status to ✅ across all four sites in one diff** (only after Steps 1-2 green AND the flow is reachable in the webclient). Recompute the tracker Status Roll-up + bump "Last refreshed" to today.

- [ ] **Step 5: Sync docs** (use `syncing-roadmap-to-docs`) — mirror the ✅ flip into `docs/`.

- [ ] **Step 6: Commit** (status + docs together)

```bash
git add roadmap/ docs/
git commit -m "[IMP] roadmap.customization: Phase 4C ✅ Delivered — AI-assisted view customization"
```

- [ ] **Step 7: Ask the user about logging to PROGRESS.md** (per CLAUDE.md — do NOT log automatically). The checkpoint already lists 4A/4B/code-widget as unlogged; bundle per the `logging-progress-sprint` skill if they say yes.

---

## Self-Review

**Spec coverage (roadmap §Targets / §Business Rules / §Code Checklist):**
- Customization skill pack + context exposure → Task 5 (pack) + Task 3 (model_key/view_key/existing-schema awareness in the tool).
- `propose_custom_field` proposer op shape → Task 3 (returns the field/placement/scope; channel = ActionButton, see flagged decision).
- Confirm card → Task 9; two `ede.create`s on confirm → Task 10; re-compose → Task 10 Step 3.
- `scope=organization` resolves org from principal (BR-AI-04) → Task 3.
- xpath safety (BR-AV-04) → already shipped 4B; 4C-named tests → Task 6.
- BR-AI-01 (zero mutating dispatch) → Task 3 Step 1 `test_proposal_turn_does_no_mutating_dispatch`.
- BR-AI-02 (RBAC) → Task 7.
- BR-AI-03 (owner=user, mode=extension) → enforced by `application_view.py` hooks; exercised in Task 7 (`owner:"user", mode:"extension"`).
- The checkpoint "gotcha" (view_key/parent uuid not exposed) → Task 1 (`view_parent_id`) + Task 8 (binding).
- All five test artifacts in the Code Checklist → Tasks 3, 6, 7, 10, 11.

**Type consistency:** `propose_custom_field` is the tool name, the ActionButtonKind, and the payload `kind` — consistent across Tasks 2/3/4/8/9. The payload shape (`{model_key, model_id, field{key,label,type,required,help,options?}, placement{xpath,position}, scope, organization_id}`) is identical in the backend tool return (Task 3), the TS `CustomFieldProposal` (Task 8), and the confirm handler reads (Task 10). `view_parent_id` (response field) / `viewParentId` (binding) naming is intentional (snake wire, camel TS).

**Placeholder scan:** Fixture names (`assistant_env`, `env_with_primary_view`, `admin_env_with_primary_view`, etc.) are the one soft spot — each is annotated with how to source it from existing conftests. The drift-step body in Task 6 Step 1 and the e2e body in Task 11 are deferred to their governing skills (`authoring-e2e-tests`) by design rather than invented blind.

**Open risk to resolve at execution:** confirm `seed_customization_pack.xml` field names against `seed_skill_packs.xml` (Task 5 Step 1) and theme token class names against `src/frontend/src/theme/` (Task 9 Step 1) before committing those tasks.
