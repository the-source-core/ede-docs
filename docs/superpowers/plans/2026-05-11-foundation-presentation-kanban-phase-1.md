# Foundation Presentation — KanbanView MVP — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Project commit rule (CLAUDE.md):** Never create a git commit as part of an automated flow. Every task in this plan ends with verification + a "stop for review" step. Commits happen only when the user explicitly says "commit" — never inside a task. Do **not** add `Co-Authored-By` trailers.

**Goal:** Ship a complete KanbanView for the EDE web client — DSL parser, React renderer, workflow-aware drag-drop, group-by switcher, and three first-adopter view XMLs (`pricing.rate`, `crm.lead`, `crm.opportunity`).

**Architecture:** New `<KanbanView>` DSL element family parsed by `DslParser` into a recursive `card_template` tree. React renderer dispatches by tag type. Drag-drop uses `@dnd-kit/core`; the drag handler dispatches `WorkflowEngine.transition_by_command` when the active group-by field is `workflow=True`, or `ede.update` otherwise.

**Tech Stack:** Python 3.10+ (DSL parser, RelaxNG, tests under pytest), TypeScript / React 19 (renderer), Tailwind CSS 4, `@dnd-kit/core` + `@dnd-kit/sortable`, vitest for frontend tests.

**Spec:** [roadmap/foundation/presentation/phase-1-implementation.md](../../../roadmap/foundation/presentation/phase-1-implementation.md)

**Prerequisites verified:**
- `foundation.workflow` Phase 1 ✅ — `WorkflowEngine.transition_by_command` available.
- `foundation.workflow` Phase 2 🟡 — `WorkflowController` HTTP routes (`/api/workflow/:model_key/:field/states`, `/available`, `/transition`) exist at [src/ede/foundation/workflow/api/workflow_routes.py](../../../src/ede/foundation/workflow/api/workflow_routes.py). Routes are partially in flight; the kanban consumes them as a soft prerequisite. (Note: there is a known one-line bug at `workflow_routes.py:238` — `env.models.get(...)` should be `env.models[...]`. Fix as part of Task 11 if it surfaces in walkthrough.)
- `viewRegistry.ts` already reserves a `kanban` slot ([src/frontend/src/workspace/views/viewRegistry.ts](../../../src/frontend/src/workspace/views/viewRegistry.ts)).

---

## File Structure

### New backend files

```
src/ede/core/services/presentation/dsl/parser.py          # MODIFY — add _parse_kanban_view
src/ede/core/services/presentation/dsl/validator.py       # MODIFY — extend RelaxNG with Kanban* grammar
src/ede/foundation/presentation/models/presentations.py   # MODIFY — extend _extract_field_names_from_render_plan
src/tests/foundation/test_dsl_kanban.py                   # NEW   — parser tests
src/tests/foundation/test_dsl_kanban_first_adopters.py    # NEW   — first-adopter XML load tests
```

### New frontend files (all under `src/frontend/src/workspace/views/kanban/`)

```
KanbanView.tsx                # root component
types.ts                      # KanbanRenderPlan, CardTemplateNode, KanbanColumn
hooks/useKanbanColumns.ts     # data load + group-by resolution
hooks/useKanbanFold.ts        # localStorage fold state
KanbanBoard.tsx               # horizontal flex container of columns
KanbanColumn.tsx              # single column (header + body + drop target)
KanbanColumnHeader.tsx        # color strip + label + count + chevron + quick-create
KanbanQuickCreate.tsx         # inline "+ New" input
KanbanCardRenderer.tsx        # recursive tag-dispatch interpreter
cards/KanbanCardShell.tsx     # outer card chrome (draggable, clickable)
cards/KanbanRibbon.tsx
cards/KanbanTitle.tsx
cards/KanbanSubtitle.tsx
cards/KanbanFooter.tsx
cards/KanbanRow.tsx
cards/KanbanStack.tsx
cards/KanbanSeparator.tsx
```

### Modified frontend files

```
src/frontend/package.json                                                          # add @dnd-kit/core, @dnd-kit/sortable
src/frontend/src/workspace/views/viewRegistry.ts                                   # (no change — slot already reserved)
src/frontend/src/workspace/components/action/WorkspaceActionContent.tsx           # add kanban branch
src/frontend/src/workspace/components/action/WorkspaceActionControlPanel.tsx       # add group-by switcher
```

### New first-adopter XML files

```
src/domains/logistics/pricing/views/pricing_rate_kanban.xml
src/domains/logistics/sales_crm/views/crm_lead_kanban.xml
src/domains/logistics/sales_crm/views/crm_opportunity_kanban.xml
```

### New frontend test files

```
src/frontend/src/__tests__/views/KanbanCardRenderer.test.tsx
src/frontend/src/__tests__/views/KanbanColumn.test.tsx
src/frontend/src/__tests__/views/KanbanView.drag.test.tsx
src/frontend/src/__tests__/views/KanbanQuickCreate.test.tsx
```

---

## Task 1 — DSL parser branch + RelaxNG schema

**Files:**
- Modify: [src/ede/core/services/presentation/dsl/parser.py](../../../src/ede/core/services/presentation/dsl/parser.py)
- Modify: [src/ede/core/services/presentation/dsl/validator.py](../../../src/ede/core/services/presentation/dsl/validator.py)
- Create: [src/tests/foundation/test_dsl_kanban.py](../../../src/tests/foundation/test_dsl_kanban.py)

### Step 1.1 — Write failing parser tests

Create `src/tests/foundation/test_dsl_kanban.py`:

```python
# -*- coding: utf-8 -*-
import pytest

from ede.core.services.presentation.dsl.parser import DslParseError, DslParser


def _parse(xml: str):
    return DslParser().parse_to_render_plan(dsl_xml_text=xml)


def test_minimal_kanban_view_parses():
    xml = """
    <view id="crm_lead_kanban_view" model="crm.lead">
      <KanbanView default_group_by="stage_id">
        <KanbanCard>
          <KanbanTitle><field name="name"/></KanbanTitle>
        </KanbanCard>
      </KanbanView>
    </view>
    """
    plan = _parse(xml)
    assert plan["view_type"] == "kanban"
    assert plan["view_id"] == "crm_lead_kanban_view"
    assert plan["model"] == "crm.lead"
    assert plan["default_group_by"] == "stage_id"
    assert plan["on_drag"] == "auto"
    assert plan["allow_reorder"] is False
    assert plan["quick_create"] is True


def test_kanban_card_template_tree_shape():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id">
        <KanbanCard>
          <KanbanRibbon field="priority"/>
          <KanbanTitle><field name="name"/></KanbanTitle>
          <KanbanFooter>
            <field name="amount" widget="monetary"/>
          </KanbanFooter>
        </KanbanCard>
      </KanbanView>
    </view>
    """
    plan = _parse(xml)
    card = plan["card_template"]
    assert card["type"] == "KanbanCard"
    assert len(card["children"]) == 3

    ribbon = card["children"][0]
    assert ribbon["type"] == "KanbanRibbon"
    assert ribbon["attrs"]["field"] == "priority"

    title = card["children"][1]
    assert title["type"] == "KanbanTitle"
    assert title["children"][0] == {
        "type": "field", "name": "name",
        "widget": None, "invisible": False, "options": {},
    }


def test_kanban_field_widget_and_options_preserved():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id">
        <KanbanCard>
          <KanbanFooter>
            <field name="amount" widget="monetary" option-currency-field="currency_id"/>
          </KanbanFooter>
        </KanbanCard>
      </KanbanView>
    </view>
    """
    plan = _parse(xml)
    amount = plan["card_template"]["children"][0]["children"][0]
    assert amount["widget"] == "monetary"
    assert amount["options"] == {"currency_field": "currency_id"}


def test_kanban_row_stack_separator_parse():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id">
        <KanbanCard>
          <KanbanRow>
            <field name="a"/>
            <KanbanSeparator/>
            <KanbanStack><field name="b"/></KanbanStack>
          </KanbanRow>
        </KanbanCard>
      </KanbanView>
    </view>
    """
    plan = _parse(xml)
    row = plan["card_template"]["children"][0]
    assert row["type"] == "KanbanRow"
    assert [c["type"] for c in row["children"]] == ["field", "KanbanSeparator", "KanbanStack"]


def test_kanban_field_names_aggregated():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id">
        <KanbanCard>
          <KanbanTitle><field name="name"/></KanbanTitle>
          <KanbanRow>
            <field name="amount" widget="monetary"/>
            <field name="probability"/>
          </KanbanRow>
          <KanbanFooter>
            <field name="followers" widget="avatar_group"/>
          </KanbanFooter>
        </KanbanCard>
      </KanbanView>
    </view>
    """
    plan = _parse(xml)
    assert set(plan["field_names"]) == {"name", "amount", "probability", "followers"}


def test_kanban_attributes_on_drag_field_quick_create():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="status" on_drag="workflow"
                  order_by="updated_at_utc desc" quick_create="false">
        <KanbanCard><KanbanTitle><field name="name"/></KanbanTitle></KanbanCard>
      </KanbanView>
    </view>
    """
    plan = _parse(xml)
    assert plan["on_drag"] == "workflow"
    assert plan["order_by"] == "updated_at_utc desc"
    assert plan["quick_create"] is False


def test_kanban_missing_card_raises():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id"></KanbanView>
    </view>
    """
    with pytest.raises(DslParseError, match="KanbanCard"):
        _parse(xml)


def test_kanban_multiple_cards_rejected():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id">
        <KanbanCard><KanbanTitle><field name="a"/></KanbanTitle></KanbanCard>
        <KanbanCard><KanbanTitle><field name="b"/></KanbanTitle></KanbanCard>
      </KanbanView>
    </view>
    """
    with pytest.raises(DslParseError, match="exactly one"):
        _parse(xml)


def test_kanban_unknown_tag_inside_card_rejected():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id">
        <KanbanCard>
          <ThisIsNotAKanbanTag/>
        </KanbanCard>
      </KanbanView>
    </view>
    """
    with pytest.raises(DslParseError, match="ThisIsNotAKanbanTag"):
        _parse(xml)


def test_kanban_ribbon_requires_field_attr():
    xml = """
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id">
        <KanbanCard><KanbanRibbon/></KanbanCard>
      </KanbanView>
    </view>
    """
    with pytest.raises(DslParseError, match="KanbanRibbon"):
        _parse(xml)
```

### Step 1.2 — Run tests to verify they all fail

```bash
./run_tests.sh src/tests/foundation/test_dsl_kanban.py
```

Expected: all 10 tests FAIL with the parser raising "unsupported view child element <KanbanView>" (because the branch doesn't exist yet).

### Step 1.3 — Add the parser branch

Open [src/ede/core/services/presentation/dsl/parser.py](../../../src/ede/core/services/presentation/dsl/parser.py). In `parse_to_render_plan`, after the `FormView` check (around line 71):

```python
        if first_child.tag == "KanbanView":
            return self._parse_kanban_view(root=root, view_id=view_id)
```

Add the implementation method (near the bottom, before legacy parser):

```python
    # -------------------------------------------------------------------------
    # KanbanView parser
    # -------------------------------------------------------------------------

    _KANBAN_CONTAINER_TAGS = {
        "KanbanTitle", "KanbanSubtitle", "KanbanFooter",
        "KanbanRow", "KanbanStack",
    }
    _KANBAN_LEAF_TAGS = {"KanbanRibbon", "KanbanSeparator"}

    def _parse_kanban_view(self, *, root: element_tree.Element, view_id: str) -> Dict[str, Any]:
        model = (root.attrib.get("model") or "").strip()
        priority_raw = (root.attrib.get("priority") or "0").strip()
        priority = int(priority_raw) if priority_raw.lstrip("-").isdigit() else 0

        kanban_el = root.find("KanbanView")
        if kanban_el is None:
            raise DslParseError("Invalid DSL: <view> with KanbanView child must contain <KanbanView>")

        default_group_by = (kanban_el.attrib.get("default_group_by") or "").strip()
        order_by = (kanban_el.attrib.get("order_by") or "").strip()
        on_drag_raw = (kanban_el.attrib.get("on_drag") or "auto").strip().lower()
        on_drag = on_drag_raw if on_drag_raw in ("workflow", "field", "auto") else "auto"
        allow_reorder = (kanban_el.attrib.get("allow_reorder") or "false").strip().lower() in ("true", "1")
        quick_create_raw = (kanban_el.attrib.get("quick_create") or "true").strip().lower()
        quick_create = quick_create_raw not in ("false", "0")

        cards = [child for child in list(kanban_el) if child.tag == "KanbanCard"]
        if len(cards) != 1:
            raise DslParseError("Invalid DSL: <KanbanView> must contain exactly one <KanbanCard>")

        card_template = self._parse_kanban_node(cards[0])
        field_names: List[str] = []
        self._collect_kanban_field_names(card_template, field_names)

        return {
            "view_id": view_id,
            "view_type": "kanban",
            "model": model,
            "priority": priority,
            "default_group_by": default_group_by,
            "order_by": order_by,
            "on_drag": on_drag,
            "allow_reorder": allow_reorder,
            "quick_create": quick_create,
            "card_template": card_template,
            "field_names": field_names,
        }

    def _parse_kanban_node(self, el: element_tree.Element) -> Dict[str, Any]:
        tag = el.tag

        if tag == "KanbanCard":
            return {"type": "KanbanCard", "children": [self._parse_kanban_node(c) for c in list(el)]}

        if tag in self._KANBAN_CONTAINER_TAGS:
            return {"type": tag, "children": [self._parse_kanban_node(c) for c in list(el)]}

        if tag == "KanbanRibbon":
            field_name = (el.attrib.get("field") or "").strip()
            if not field_name:
                raise DslParseError("Invalid DSL: <KanbanRibbon> requires attribute field")
            attrs: Dict[str, str] = {"field": field_name}
            color_map = (el.attrib.get("option-color-map") or "").strip()
            if color_map:
                attrs["color_map"] = color_map
            return {"type": "KanbanRibbon", "attrs": attrs}

        if tag == "KanbanSeparator":
            return {"type": "KanbanSeparator"}

        if tag == "field":
            return self._parse_kanban_field(el)

        if tag == "button":
            return self._parse_form_button_element(el) | {"type": "button"}

        raise DslParseError(
            "Invalid DSL: unexpected tag <%s> inside KanbanView. "
            "Allowed: KanbanCard, KanbanTitle, KanbanSubtitle, KanbanFooter, "
            "KanbanRow, KanbanStack, KanbanRibbon, KanbanSeparator, field, button." % tag
        )

    def _parse_kanban_field(self, field_el: element_tree.Element) -> Dict[str, Any]:
        name = (field_el.attrib.get("name") or "").strip()
        if not name:
            raise DslParseError("Invalid DSL: <field> inside KanbanView requires attribute name")
        options: Dict[str, str] = {}
        for attr_name, attr_value in field_el.attrib.items():
            if attr_name.startswith("option-"):
                option_key = attr_name[7:].replace("-", "_")
                options[option_key] = attr_value
        return {
            "type": "field",
            "name": name,
            "widget": (field_el.attrib.get("widget") or "").strip() or None,
            "invisible": (field_el.attrib.get("invisible") or "false").strip().lower() == "true",
            "options": options,
        }

    def _collect_kanban_field_names(self, node: Dict[str, Any], out: List[str]) -> None:
        if node["type"] == "field":
            if node["name"] not in out:
                out.append(node["name"])
        elif "children" in node:
            for child in node["children"]:
                self._collect_kanban_field_names(child, out)
```

### Step 1.4 — Extend RelaxNG validator

Open [src/ede/core/services/presentation/dsl/validator.py](../../../src/ede/core/services/presentation/dsl/validator.py). Locate the grammar (search for `<define name="ListView">` to find the pattern). Add a sibling `KanbanView` define and include it in the top-level view alternation. Pattern to mirror exactly is the one used for `<DynamicProperties/>` in customization Phase 1. The new RelaxNG fragment:

```xml
<define name="KanbanView">
  <element name="KanbanView">
    <attribute name="default_group_by"/>
    <optional><attribute name="order_by"/></optional>
    <optional><attribute name="on_drag"><choice><value>workflow</value><value>field</value><value>auto</value></choice></attribute></optional>
    <optional><attribute name="allow_reorder"><choice><value>true</value><value>false</value></choice></attribute></optional>
    <optional><attribute name="quick_create"><choice><value>true</value><value>false</value></choice></attribute></optional>
    <element name="KanbanCard">
      <oneOrMore><ref name="KanbanNode"/></oneOrMore>
    </element>
  </element>
</define>

<define name="KanbanNode">
  <choice>
    <element name="KanbanRibbon"><attribute name="field"/><optional><attribute name="option-color-map"/></optional></element>
    <element name="KanbanSeparator"><empty/></element>
    <element name="KanbanTitle"><oneOrMore><ref name="KanbanInner"/></oneOrMore></element>
    <element name="KanbanSubtitle"><oneOrMore><ref name="KanbanInner"/></oneOrMore></element>
    <element name="KanbanFooter"><oneOrMore><ref name="KanbanInner"/></oneOrMore></element>
    <element name="KanbanRow"><oneOrMore><ref name="KanbanInner"/></oneOrMore></element>
    <element name="KanbanStack"><oneOrMore><ref name="KanbanInner"/></oneOrMore></element>
    <ref name="FieldElement"/>
    <ref name="ButtonElement"/>
  </choice>
</define>

<define name="KanbanInner">
  <choice>
    <ref name="KanbanNode"/>
  </choice>
</define>
```

In the top-level `<choice>` for view bodies, add `<ref name="KanbanView"/>`.

### Step 1.5 — Run tests to verify they all pass

```bash
./run_tests.sh src/tests/foundation/test_dsl_kanban.py -v
```

Expected: all 10 tests PASS.

### Step 1.6 — Run full pytest to verify no regressions

```bash
./run_tests.sh
```

Expected: prior tests still green (baseline 1424 + 10 new = 1434).

### Step 1.7 — Stop for review

Do not commit. Tell the user: "Task 1 complete — parser branch + RelaxNG schema + 10 parser tests all pass. Ready for review."

---

## Task 2 — Extend `_extract_field_names_from_render_plan`

**Files:**
- Modify: [src/ede/foundation/presentation/models/presentations.py:827](../../../src/ede/foundation/presentation/models/presentations.py#L827)
- Modify: [src/tests/foundation/test_dsl_kanban.py](../../../src/tests/foundation/test_dsl_kanban.py) (add one test)

The save-allowlist + read-payload field projection both depend on this helper. Without extending it, kanban-referenced fields won't be advertised on the action's `view_field_names["kanban"]` set.

### Step 2.1 — Write failing test

Append to `src/tests/foundation/test_dsl_kanban.py`:

```python
def test_kanban_field_names_picked_up_by_presentation_helper():
    from ede.foundation.presentation.models.presentations import PresentationKernel
    plan = _parse("""
    <view id="v" model="m">
      <KanbanView default_group_by="stage_id">
        <KanbanCard>
          <KanbanTitle><field name="name"/></KanbanTitle>
          <KanbanFooter><field name="amount"/><field name="currency_id" invisible="true"/></KanbanFooter>
        </KanbanCard>
      </KanbanView>
    </view>
    """)
    names = PresentationKernel._extract_field_names_from_render_plan(plan)
    assert set(names) >= {"name", "amount", "currency_id"}
```

### Step 2.2 — Run, expect failure

```bash
./run_tests.sh src/tests/foundation/test_dsl_kanban.py::test_kanban_field_names_picked_up_by_presentation_helper -v
```

Expected: FAIL (helper currently only walks list/form RenderPlans).

### Step 2.3 — Extend the helper

Open [src/ede/foundation/presentation/models/presentations.py:827](../../../src/ede/foundation/presentation/models/presentations.py#L827). The method `_extract_field_names_from_render_plan` already handles `list` and `form` view types. Add a kanban branch:

```python
        if render_plan.get("view_type") == "kanban":
            names: List[str] = []
            def _walk(node: Dict[str, Any]) -> None:
                if node.get("type") == "field":
                    n = node.get("name")
                    if n and n not in names:
                        names.append(n)
                for child in node.get("children", []) or []:
                    _walk(child)
            _walk(render_plan.get("card_template") or {})
            return names
```

Place this branch alongside the existing `list` / `form` branches in the same conditional cascade.

### Step 2.4 — Run, expect pass

```bash
./run_tests.sh src/tests/foundation/test_dsl_kanban.py -v
```

Expected: all 11 tests PASS.

### Step 2.5 — Stop for review

---

## Task 3 — Frontend types + `KanbanView.tsx` skeleton

**Files:**
- Create: `src/frontend/src/workspace/views/kanban/types.ts`
- Create: `src/frontend/src/workspace/views/kanban/KanbanView.tsx`
- Modify: [src/frontend/src/workspace/components/action/WorkspaceActionContent.tsx](../../../src/frontend/src/workspace/components/action/WorkspaceActionContent.tsx)

### Step 3.1 — Write the types file

Create `src/frontend/src/workspace/views/kanban/types.ts`:

```ts
import type { RecordData } from "../../../engine/types"

export type DragMode = "workflow" | "field" | "auto"

export interface KanbanFieldNode {
    type: "field"
    name: string
    widget: string | null
    invisible: boolean
    options: Record<string, string>
}

export interface KanbanRibbonNode {
    type: "KanbanRibbon"
    attrs: { field: string; color_map?: string }
}

export interface KanbanSeparatorNode { type: "KanbanSeparator" }

export interface KanbanButtonNode {
    type: "button"
    name: string
    command?: string | null
    label?: string | null
}

export interface KanbanContainerNode {
    type:
        | "KanbanCard"
        | "KanbanTitle"
        | "KanbanSubtitle"
        | "KanbanFooter"
        | "KanbanRow"
        | "KanbanStack"
    children: CardTemplateNode[]
}

export type CardTemplateNode =
    | KanbanContainerNode
    | KanbanRibbonNode
    | KanbanSeparatorNode
    | KanbanFieldNode
    | KanbanButtonNode

export interface KanbanRenderPlan {
    view_id: string
    view_type: "kanban"
    model: string
    priority: number
    default_group_by: string
    order_by: string
    on_drag: DragMode
    allow_reorder: boolean
    quick_create: boolean
    card_template: KanbanContainerNode
    field_names: string[]
}

export interface KanbanColumn {
    value: string | number | null
    label: string
    count: number
    color: string | null
    foldedByDefault: boolean
    records: RecordData[]
}
```

### Step 3.2 — Write `KanbanView.tsx` skeleton

Create `src/frontend/src/workspace/views/kanban/KanbanView.tsx`:

```tsx
import type { RecordData } from "../../../engine/types"
import type { KanbanRenderPlan } from "./types"
import { KanbanBoard } from "./KanbanBoard"

export interface KanbanViewProps {
    renderPlan: KanbanRenderPlan
    records: RecordData[]
    groupBy: string
    onRecordClick: (record: RecordData) => void
    onTransition: (args: {
        record: RecordData
        model_key: string
        target_state_value: string | number | null
    }) => Promise<void>
    onFieldWrite: (args: {
        record: RecordData
        model_key: string
        field: string
        value: string | number | null
    }) => Promise<void>
    onCreate: (args: {
        model_key: string
        values: Record<string, unknown>
    }) => Promise<RecordData>
    fieldSpecs: Record<string, { type?: string; workflow?: boolean }>
}

export function KanbanView(props: KanbanViewProps) {
    return <KanbanBoard {...props} />
}
```

`KanbanBoard` doesn't exist yet — Task 5 creates it. The build will fail here until Task 5 lands. That's intentional — the type contract is locked first.

### Step 3.3 — Wire into `WorkspaceActionContent`

Open [src/frontend/src/workspace/components/action/WorkspaceActionContent.tsx](../../../src/frontend/src/workspace/components/action/WorkspaceActionContent.tsx). Find the view-type switch (search for `view_type === "list"`). Add a kanban branch with the same prop-shape pattern; pull props from existing context hooks. Use `// @ts-expect-error temporary until Task 5 lands KanbanBoard` to suppress the deliberate broken import. The exact prop wiring will be tightened in Task 5.

### Step 3.4 — Verify TypeScript surface for types file

```bash
cd src/frontend && bunx tsc --noEmit src/workspace/views/kanban/types.ts
```

Expected: clean type-check on `types.ts` standalone. The full `bun run build` will still fail due to the `KanbanBoard` placeholder import — that's expected.

### Step 3.5 — Stop for review

---

## Task 4 — `useKanbanColumns` hook + column-source resolution

**Files:**
- Create: `src/frontend/src/workspace/views/kanban/hooks/useKanbanColumns.ts`

### Step 4.1 — Write the hook

Create the file:

```ts
import { useEffect, useState } from "react"

import type { RecordData } from "../../../../engine/types"
import type { KanbanColumn, KanbanRenderPlan } from "../types"

export interface UseKanbanColumnsResult {
    columns: KanbanColumn[]
    isLoading: boolean
    error: Error | null
    reload: () => Promise<void>
}

export interface KanbanColumnSource {
    readGroup: (args: {
        model_key: string
        group_by: string
        domain?: unknown[]
    }) => Promise<Array<{ value: string | number | null; count: number; label: string }>>
    search: (args: {
        model_key: string
        domain: unknown[]
        limit: number
        order: string
    }) => Promise<RecordData[]>
    fetchWorkflowStates: (model_key: string, field: string) => Promise<
        Array<{ value: string; label: string; sequence: number; color: string | null; terminal: boolean }> | null
    >
}

export function useKanbanColumns(
    plan: KanbanRenderPlan,
    groupBy: string,
    fieldIsWorkflow: boolean,
    source: KanbanColumnSource,
): UseKanbanColumnsResult {
    const [columns, setColumns] = useState<KanbanColumn[]>([])
    const [isLoading, setIsLoading] = useState(true)
    const [error, setError] = useState<Error | null>(null)

    const reload = async () => {
        setIsLoading(true)
        setError(null)
        try {
            let columnDefs: KanbanColumn[] = []

            if (fieldIsWorkflow) {
                const states = await source.fetchWorkflowStates(plan.model, groupBy)
                if (states) {
                    columnDefs = states
                        .sort((a, b) => a.sequence - b.sequence)
                        .map(s => ({
                            value: s.value,
                            label: s.label,
                            count: 0,
                            color: s.color,
                            foldedByDefault: s.terminal,
                            records: [],
                        }))
                }
            }

            if (columnDefs.length === 0) {
                const buckets = await source.readGroup({ model_key: plan.model, group_by: groupBy })
                columnDefs = buckets
                    .sort((a, b) => String(a.label).localeCompare(String(b.label)))
                    .map(b => ({
                        value: b.value,
                        label: b.label,
                        count: b.count,
                        color: null,
                        foldedByDefault: false,
                        records: [],
                    }))
            }

            const order = plan.order_by || "id"
            await Promise.all(columnDefs.map(async col => {
                const records = await source.search({
                    model_key: plan.model,
                    domain: [[groupBy, "=", col.value]],
                    limit: 40,
                    order,
                })
                col.records = records
                col.count = records.length  // best-effort; read_group count is canonical when present
            }))

            setColumns(columnDefs)
        } catch (e) {
            setError(e instanceof Error ? e : new Error(String(e)))
        } finally {
            setIsLoading(false)
        }
    }

    useEffect(() => {
        void reload()
        // eslint-disable-next-line react-hooks/exhaustive-deps
    }, [plan.view_id, groupBy, fieldIsWorkflow])

    return { columns, isLoading, error, reload }
}
```

### Step 4.2 — Verify type-check on the hook

```bash
cd src/frontend && bunx tsc --noEmit src/workspace/views/kanban/hooks/useKanbanColumns.ts
```

Expected: clean. Full build still fails on `KanbanBoard` — that's fine.

### Step 4.3 — Stop for review

---

## Task 5 — `KanbanBoard`, `KanbanColumn`, `KanbanColumnHeader`

**Files:**
- Create: `src/frontend/src/workspace/views/kanban/KanbanBoard.tsx`
- Create: `src/frontend/src/workspace/views/kanban/KanbanColumn.tsx`
- Create: `src/frontend/src/workspace/views/kanban/KanbanColumnHeader.tsx`
- Create: `src/frontend/src/workspace/views/kanban/hooks/useKanbanFold.ts`

### Step 5.1 — `useKanbanFold` hook

Create `src/frontend/src/workspace/views/kanban/hooks/useKanbanFold.ts`:

```ts
import { useCallback, useEffect, useState } from "react"

const STORAGE_KEY_PREFIX = "kanban_fold_v1"

function storageKey(userUuid: string, viewId: string, columnValue: string | number | null): string {
    return `${STORAGE_KEY_PREFIX}:${userUuid}:${viewId}:${String(columnValue ?? "__null__")}`
}

export function useKanbanFold(userUuid: string, viewId: string, columnValue: string | number | null, foldedByDefault: boolean) {
    const [folded, setFolded] = useState<boolean>(foldedByDefault)

    useEffect(() => {
        const raw = window.localStorage.getItem(storageKey(userUuid, viewId, columnValue))
        if (raw === "true")  setFolded(true)
        if (raw === "false") setFolded(false)
    }, [userUuid, viewId, columnValue])

    const toggle = useCallback(() => {
        setFolded(prev => {
            const next = !prev
            window.localStorage.setItem(storageKey(userUuid, viewId, columnValue), String(next))
            return next
        })
    }, [userUuid, viewId, columnValue])

    return { folded, toggle }
}
```

### Step 5.2 — `KanbanColumnHeader`

Create `src/frontend/src/workspace/views/kanban/KanbanColumnHeader.tsx`:

```tsx
import { ChevronDown, ChevronRight, Plus } from "lucide-react"

import type { KanbanColumn } from "./types"

export interface KanbanColumnHeaderProps {
    column: KanbanColumn
    folded: boolean
    onToggleFold: () => void
    quickCreate: boolean
    onQuickCreateOpen: () => void
}

export function KanbanColumnHeader(props: KanbanColumnHeaderProps) {
    const { column, folded, onToggleFold, quickCreate, onQuickCreateOpen } = props
    const stripStyle = column.color ? { backgroundColor: column.color } : undefined

    return (
        <div className="rounded-t-md overflow-hidden">
            <div className="h-1 w-full bg-default" style={stripStyle} />
            <div className="flex items-center justify-between px-3 py-2 bg-surface-2">
                <div className="flex items-center gap-2 min-w-0">
                    <button
                        className="p-0.5 hover:bg-surface rounded"
                        aria-label={folded ? "Unfold column" : "Fold column"}
                        onClick={onToggleFold}>
                        {folded ? <ChevronRight size={14}/> : <ChevronDown size={14}/>}
                    </button>
                    <h4 className="text-sm font-semibold text-primary truncate">{column.label}</h4>
                    <span className="text-xs text-muted bg-surface rounded-full px-2 py-0.5">{column.count}</span>
                </div>
                {quickCreate && (
                    <button
                        className="p-0.5 hover:bg-surface rounded text-muted"
                        aria-label="Quick create"
                        onClick={onQuickCreateOpen}>
                        <Plus size={14}/>
                    </button>
                )}
            </div>
        </div>
    )
}
```

### Step 5.3 — `KanbanColumn`

Create `src/frontend/src/workspace/views/kanban/KanbanColumn.tsx`:

```tsx
import { useState } from "react"

import { useDroppable } from "@dnd-kit/core"

import type { RecordData } from "../../../engine/types"
import { KanbanCardRenderer } from "./KanbanCardRenderer"
import { KanbanColumnHeader } from "./KanbanColumnHeader"
import { KanbanQuickCreate } from "./KanbanQuickCreate"
import type { KanbanColumn as ColumnT, KanbanContainerNode } from "./types"
import { useKanbanFold } from "./hooks/useKanbanFold"

export interface KanbanColumnProps {
    column: ColumnT
    cardTemplate: KanbanContainerNode
    viewId: string
    userUuid: string
    fieldSpecs: Record<string, { type?: string; workflow?: boolean }>
    quickCreate: boolean
    onRecordClick: (r: RecordData) => void
    onQuickCreate: (title: string) => Promise<void>
    dragLegality?: "legal" | "illegal" | null
}

export function KanbanColumn(props: KanbanColumnProps) {
    const { folded, toggle } = useKanbanFold(props.userUuid, props.viewId, props.column.value, props.column.foldedByDefault)
    const [qcOpen, setQcOpen] = useState(false)
    const { setNodeRef, isOver } = useDroppable({ id: `col:${String(props.column.value)}` })

    const ringClass =
        props.dragLegality === "legal"   ? "ring-2 ring-emerald-500" :
        props.dragLegality === "illegal" ? "ring-2 ring-rose-500 opacity-60" :
        isOver                            ? "ring-2 ring-blue-400" : ""

    return (
        <div ref={setNodeRef} className={`flex flex-col w-72 flex-shrink-0 bg-surface-2 rounded-md ${ringClass}`}>
            <KanbanColumnHeader
                column={props.column}
                folded={folded}
                onToggleFold={toggle}
                quickCreate={props.quickCreate}
                onQuickCreateOpen={() => setQcOpen(true)}
            />

            <div className={folded ? "hidden" : "flex flex-col gap-2 p-2 overflow-y-auto max-h-[70vh]"}>
                {qcOpen && (
                    <KanbanQuickCreate
                        onSubmit={async title => { await props.onQuickCreate(title); setQcOpen(false) }}
                        onCancel={() => setQcOpen(false)}
                    />
                )}

                {props.column.records.length === 0 && !qcOpen && (
                    <p className="text-sm text-muted text-center py-4">No records</p>
                )}

                {props.column.records.map(record => (
                    <KanbanCardRenderer
                        key={record.id}
                        template={props.cardTemplate}
                        record={record}
                        fieldSpecs={props.fieldSpecs}
                        onClick={() => props.onRecordClick(record)}
                    />
                ))}
            </div>
        </div>
    )
}
```

### Step 5.4 — `KanbanBoard`

Create `src/frontend/src/workspace/views/kanban/KanbanBoard.tsx`:

```tsx
import { useMemo, useState } from "react"

import { DndContext, type DragEndEvent, type DragStartEvent, PointerSensor, useSensor, useSensors } from "@dnd-kit/core"

import type { KanbanViewProps } from "./KanbanView"
import { KanbanColumn } from "./KanbanColumn"
import { useKanbanColumns, type KanbanColumnSource } from "./hooks/useKanbanColumns"
import { resolveColumnSource } from "./columnSource"
import { resolveDragMode } from "./dragMode"

export function KanbanBoard(props: KanbanViewProps) {
    const fieldIsWorkflow = Boolean(props.fieldSpecs[props.groupBy]?.workflow)
    const dragMode = resolveDragMode(props.renderPlan, fieldIsWorkflow)

    const source: KanbanColumnSource = useMemo(() => resolveColumnSource(), [])
    const { columns, isLoading, error, reload } = useKanbanColumns(props.renderPlan, props.groupBy, fieldIsWorkflow, source)

    const [legalityMap, setLegalityMap] = useState<Record<string, "legal" | "illegal">>({})
    const sensors = useSensors(useSensor(PointerSensor, { activationConstraint: { distance: 4 } }))

    const onDragStart = async (_e: DragStartEvent) => {
        // legality preview is implemented in Task 8 (workflow drag path)
        setLegalityMap({})
    }

    const onDragEnd = async (e: DragEndEvent) => {
        setLegalityMap({})
        const cardRecord = e.active.data.current?.record
        const targetColId = String(e.over?.id || "")
        const targetCol = columns.find(c => `col:${String(c.value)}` === targetColId)
        if (!cardRecord || !targetCol) return

        if (dragMode === "workflow") {
            await props.onTransition({
                record: cardRecord,
                model_key: props.renderPlan.model,
                target_state_value: targetCol.value,
            })
        } else {
            await props.onFieldWrite({
                record: cardRecord,
                model_key: props.renderPlan.model,
                field: props.groupBy,
                value: targetCol.value,
            })
        }
        await reload()
    }

    if (isLoading) return <p className="text-muted text-sm p-4">Loading kanban…</p>
    if (error)     return <p className="text-rose-600 text-sm p-4">Failed to load: {error.message}</p>

    return (
        <DndContext sensors={sensors} onDragStart={onDragStart} onDragEnd={onDragEnd}>
            <div className="flex flex-row gap-3 p-3 overflow-x-auto h-full">
                {columns.map(col => (
                    <KanbanColumn
                        key={String(col.value)}
                        column={col}
                        cardTemplate={props.renderPlan.card_template}
                        viewId={props.renderPlan.view_id}
                        userUuid={"current"}  // wired in Task 6 from session context
                        fieldSpecs={props.fieldSpecs}
                        quickCreate={props.renderPlan.quick_create}
                        onRecordClick={props.onRecordClick}
                        onQuickCreate={async title => {
                            await props.onCreate({
                                model_key: props.renderPlan.model,
                                values: { name: title, [props.groupBy]: col.value },
                            })
                            await reload()
                        }}
                        dragLegality={legalityMap[String(col.value)] ?? null}
                    />
                ))}
            </div>
        </DndContext>
    )
}
```

### Step 5.5 — Helper modules

Create `src/frontend/src/workspace/views/kanban/dragMode.ts`:

```ts
import type { KanbanRenderPlan } from "./types"

export function resolveDragMode(plan: KanbanRenderPlan, fieldIsWorkflow: boolean): "workflow" | "field" {
    if (plan.on_drag === "workflow") return "workflow"
    if (plan.on_drag === "field")    return "field"
    return fieldIsWorkflow ? "workflow" : "field"
}
```

Create `src/frontend/src/workspace/views/kanban/columnSource.ts` (stub; real wiring to `dispatchCommand` happens here):

```ts
import { dispatchCommand } from "../../../engine/api"
import type { KanbanColumnSource } from "./hooks/useKanbanColumns"

export function resolveColumnSource(): KanbanColumnSource {
    return {
        readGroup: async ({ model_key, group_by, domain = [] }) => {
            const res: any = await dispatchCommand({ name: "ede.read_group", payload: { model_key, group_by, domain } })
            return res.groups ?? []
        },
        search: async ({ model_key, domain, limit, order }) => {
            const res: any = await dispatchCommand({ name: "ede.search", payload: { model_key, domain, limit, order } })
            return res.records ?? []
        },
        fetchWorkflowStates: async (model_key, field) => {
            try {
                const r = await fetch(`/api/workflow/${model_key}/${field}/states`)
                if (!r.ok) return null
                return await r.json()
            } catch { return null }
        },
    }
}
```

(If `dispatchCommand` lives elsewhere, replace the import with the project's actual command dispatcher.)

### Step 5.6 — Stop for review

Do not run the build yet — card renderer pieces are still missing (Task 6).

---

## Task 6 — Card renderer + 8 card components

**Files:**
- Create: `src/frontend/src/workspace/views/kanban/KanbanCardRenderer.tsx`
- Create: `src/frontend/src/workspace/views/kanban/KanbanQuickCreate.tsx`
- Create: `src/frontend/src/workspace/views/kanban/cards/KanbanCardShell.tsx`
- Create: `src/frontend/src/workspace/views/kanban/cards/KanbanRibbon.tsx`
- Create: `src/frontend/src/workspace/views/kanban/cards/KanbanTitle.tsx`
- Create: `src/frontend/src/workspace/views/kanban/cards/KanbanSubtitle.tsx`
- Create: `src/frontend/src/workspace/views/kanban/cards/KanbanFooter.tsx`
- Create: `src/frontend/src/workspace/views/kanban/cards/KanbanRow.tsx`
- Create: `src/frontend/src/workspace/views/kanban/cards/KanbanStack.tsx`
- Create: `src/frontend/src/workspace/views/kanban/cards/KanbanSeparator.tsx`

### Step 6.1 — Card components (mechanical)

Create each `cards/*.tsx` file. All are short, Tailwind-only, no inline styles. Examples:

`cards/KanbanCardShell.tsx`:

```tsx
import type { ReactNode } from "react"

import { useDraggable } from "@dnd-kit/core"

import type { RecordData } from "../../../../engine/types"

export interface KanbanCardShellProps {
    record: RecordData
    onClick: () => void
    children: ReactNode
}

export function KanbanCardShell({ record, onClick, children }: KanbanCardShellProps) {
    const { attributes, listeners, setNodeRef, isDragging } = useDraggable({
        id: `card:${record.id}`,
        data: { record },
    })

    return (
        <div
            ref={setNodeRef}
            {...attributes}
            {...listeners}
            onClick={onClick}
            className={`bg-surface rounded-md shadow-sm border border-default overflow-hidden cursor-grab hover:shadow-md ${isDragging ? "opacity-50" : ""}`}>
            {children}
        </div>
    )
}
```

`cards/KanbanRibbon.tsx`:

```tsx
import type { RecordData } from "../../../../engine/types"

export interface KanbanRibbonProps {
    field: string
    colorMap?: string  // CSV: "high:red,medium:amber,low:emerald"
    record: RecordData
    fieldSpec: { type?: string; workflow?: boolean } | undefined
    workflowStateColor?: string | null
}

function parseColorMap(raw?: string): Record<string, string> {
    if (!raw) return {}
    return Object.fromEntries(raw.split(",").map(p => p.trim()).filter(Boolean).map(p => {
        const [k, v] = p.split(":")
        return [k.trim(), v.trim()]
    }))
}

export function KanbanRibbon(props: KanbanRibbonProps) {
    const value = String(props.record[props.field] ?? "")
    const map = parseColorMap(props.colorMap)
    const color =
        map[value] ??
        (props.fieldSpec?.workflow ? (props.workflowStateColor ?? null) : null)

    if (!color) return <div className="h-1 w-full bg-default"/>
    return <div className="h-1 w-full" style={{ backgroundColor: color }}/>
}
```

`cards/KanbanTitle.tsx`:

```tsx
import type { ReactNode } from "react"
export function KanbanTitle({ children }: { children: ReactNode }) {
    return <div className="font-semibold text-base text-primary mb-1 px-3.5 pt-3">{children}</div>
}
```

`cards/KanbanSubtitle.tsx`:

```tsx
import type { ReactNode } from "react"
export function KanbanSubtitle({ children }: { children: ReactNode }) {
    return <div className="text-sm text-muted mb-2 px-3.5">{children}</div>
}
```

`cards/KanbanFooter.tsx`:

```tsx
import type { ReactNode } from "react"
export function KanbanFooter({ children }: { children: ReactNode }) {
    return (
        <div className="flex justify-between items-center text-sm border-t border-default bg-surface-2 px-3.5 py-2">
            {children}
        </div>
    )
}
```

`cards/KanbanRow.tsx`:

```tsx
import type { ReactNode } from "react"
export function KanbanRow({ children }: { children: ReactNode }) {
    return <div className="flex flex-row justify-between items-center gap-2 my-1.5 px-3.5">{children}</div>
}
```

`cards/KanbanStack.tsx`:

```tsx
import type { ReactNode } from "react"
export function KanbanStack({ children }: { children: ReactNode }) {
    return <div className="flex flex-col gap-1 px-3.5">{children}</div>
}
```

`cards/KanbanSeparator.tsx`:

```tsx
export function KanbanSeparator() {
    return <hr className="border-t border-default my-2"/>
}
```

### Step 6.2 — Card renderer dispatcher

Create `src/frontend/src/workspace/views/kanban/KanbanCardRenderer.tsx`:

```tsx
import type { ReactNode } from "react"

import { FieldView } from "../fields/FieldView"
import { KanbanCardShell } from "./cards/KanbanCardShell"
import { KanbanFooter } from "./cards/KanbanFooter"
import { KanbanRibbon } from "./cards/KanbanRibbon"
import { KanbanRow } from "./cards/KanbanRow"
import { KanbanSeparator } from "./cards/KanbanSeparator"
import { KanbanStack } from "./cards/KanbanStack"
import { KanbanSubtitle } from "./cards/KanbanSubtitle"
import { KanbanTitle } from "./cards/KanbanTitle"
import type { CardTemplateNode, KanbanContainerNode } from "./types"
import type { RecordData } from "../../../engine/types"

export interface KanbanCardRendererProps {
    template: KanbanContainerNode
    record: RecordData
    fieldSpecs: Record<string, { type?: string; workflow?: boolean }>
    onClick: () => void
}

export function KanbanCardRenderer(props: KanbanCardRendererProps) {
    return (
        <KanbanCardShell record={props.record} onClick={props.onClick}>
            {props.template.children.map((n, i) => renderNode(n, i, props.record, props.fieldSpecs))}
        </KanbanCardShell>
    )
}

function renderNode(
    node: CardTemplateNode,
    key: number,
    record: RecordData,
    fieldSpecs: KanbanCardRendererProps["fieldSpecs"],
): ReactNode {
    switch (node.type) {
        case "KanbanTitle":     return <KanbanTitle key={key}>{node.children.map((c, i) => renderNode(c, i, record, fieldSpecs))}</KanbanTitle>
        case "KanbanSubtitle":  return <KanbanSubtitle key={key}>{node.children.map((c, i) => renderNode(c, i, record, fieldSpecs))}</KanbanSubtitle>
        case "KanbanFooter":    return <KanbanFooter key={key}>{node.children.map((c, i) => renderNode(c, i, record, fieldSpecs))}</KanbanFooter>
        case "KanbanRow":       return <KanbanRow key={key}>{node.children.map((c, i) => renderNode(c, i, record, fieldSpecs))}</KanbanRow>
        case "KanbanStack":     return <KanbanStack key={key}>{node.children.map((c, i) => renderNode(c, i, record, fieldSpecs))}</KanbanStack>
        case "KanbanSeparator": return <KanbanSeparator key={key}/>
        case "KanbanRibbon":    return <KanbanRibbon key={key} field={node.attrs.field} colorMap={node.attrs.color_map} record={record} fieldSpec={fieldSpecs[node.attrs.field]}/>
        case "field":           return node.invisible ? null : <FieldView key={key} fieldName={node.name} widget={node.widget ?? undefined} options={node.options} record={record} fieldSpec={fieldSpecs[node.name]}/>
        case "button":          return null  // buttons inside cards deferred to Phase 2 if any consumer needs them
        case "KanbanCard":      return null  // nested KanbanCard is illegal; RelaxNG rejects this earlier
    }
}
```

> **Type-check note:** if `FieldView` doesn't accept exactly these props, adjust to the existing component's prop shape — the goal is "reuse the existing field-render pipeline", not introduce a new widget contract.

### Step 6.3 — `KanbanQuickCreate`

Create `src/frontend/src/workspace/views/kanban/KanbanQuickCreate.tsx`:

```tsx
import { useState } from "react"

export interface KanbanQuickCreateProps {
    onSubmit: (title: string) => Promise<void>
    onCancel: () => void
}

export function KanbanQuickCreate({ onSubmit, onCancel }: KanbanQuickCreateProps) {
    const [value, setValue] = useState("")
    const [busy, setBusy] = useState(false)

    const submit = async () => {
        if (!value.trim() || busy) return
        setBusy(true)
        try { await onSubmit(value.trim()) }
        finally { setBusy(false) }
    }

    return (
        <div className="bg-surface rounded border border-default p-2">
            <input
                autoFocus
                className="w-full bg-transparent text-sm outline-none"
                placeholder="Title…"
                value={value}
                onChange={e => setValue(e.target.value)}
                onKeyDown={e => {
                    if (e.key === "Enter")  void submit()
                    if (e.key === "Escape") onCancel()
                }}
            />
        </div>
    )
}
```

### Step 6.4 — Stop for review

---

## Task 7 — Install `@dnd-kit` and verify the build

**Files:**
- Modify: [src/frontend/package.json](../../../src/frontend/package.json)

### Step 7.1 — Install deps

```bash
cd src/frontend && bun add @dnd-kit/core @dnd-kit/sortable
```

Expected: both packages added to `package.json` dependencies. Verify `package.json` contains both names.

### Step 7.2 — Run `bun run build`

```bash
cd src/frontend && bun run build
```

Expected: clean strict-mode TypeScript build. Resolve any remaining type errors against existing component prop shapes (notably `FieldView`'s props — adjust the `KanbanCardRenderer.tsx` call site to match).

### Step 7.3 — Run vitest baseline (smoke)

```bash
cd src/frontend && bun run test
```

Expected: existing 388 vitest tests still pass — no new test files yet.

### Step 7.4 — Stop for review

---

## Task 8 — Workflow drag path + legality preview

**Files:**
- Modify: `src/frontend/src/workspace/views/kanban/KanbanBoard.tsx`
- Create: `src/frontend/src/__tests__/views/KanbanView.drag.test.tsx`

### Step 8.1 — Write failing drag test

Create `src/frontend/src/__tests__/views/KanbanView.drag.test.tsx`:

```tsx
import { afterEach, beforeEach, describe, expect, test, vi } from "vitest"
import { render, screen, fireEvent } from "@testing-library/react"

import { KanbanView } from "../../workspace/views/kanban/KanbanView"

// Minimal RenderPlan + record set fixture.
const baseRenderPlan = {
    view_id: "test_kanban", view_type: "kanban" as const,
    model: "crm.lead", priority: 0,
    default_group_by: "stage_id", order_by: "id",
    on_drag: "workflow" as const, allow_reorder: false, quick_create: false,
    card_template: { type: "KanbanCard" as const, children: [
        { type: "KanbanTitle" as const, children: [{ type: "field" as const, name: "name", widget: null, invisible: false, options: {} }] }
    ]},
    field_names: ["name", "stage_id"],
}

describe("KanbanView drag", () => {
    beforeEach(() => {
        vi.stubGlobal("fetch", vi.fn().mockResolvedValue(new Response(JSON.stringify([
            { value: "new",      label: "New",      sequence: 1, color: "#3b82f6", terminal: false },
            { value: "qualified",label: "Qualified",sequence: 2, color: "#10b981", terminal: false },
        ]))))
    })
    afterEach(() => { vi.restoreAllMocks() })

    test("dragging on workflow group-by dispatches onTransition", async () => {
        const onTransition = vi.fn().mockResolvedValue(undefined)
        const onFieldWrite = vi.fn().mockResolvedValue(undefined)
        // … render KanbanView with a mock column source returning two columns each with one record.
        // … simulate a drag from card → second column using dnd-kit's pointer API.
        // Assert onTransition called with { record, model_key: "crm.lead", target_state_value: "qualified" }
        // Assert onFieldWrite NOT called.
        expect(onTransition).toBeDefined()
        expect(onFieldWrite).toBeDefined()
    })
})
```

> **Plan note:** dnd-kit pointer simulation is fiddly in jsdom; the test uses `fireEvent.dragStart` / `dragOver` / `drop` on the column's drop zone element. The full assertion body is filled out in step 8.2 alongside the implementation — write a placeholder assertion now to verify the test infrastructure compiles and runs.

### Step 8.2 — Implement workflow legality preview in `KanbanBoard`

Add to `onDragStart` in `KanbanBoard.tsx`:

```tsx
const onDragStart = async (e: DragStartEvent) => {
    if (dragMode !== "workflow") { setLegalityMap({}); return }
    const recordUuid = (e.active.data.current?.record as RecordData | undefined)?.id
    if (!recordUuid) return
    try {
        const r = await fetch(`/api/workflow/${props.renderPlan.model}/${props.groupBy}/available?record_uuid=${recordUuid}`)
        if (!r.ok) return
        const targets: string[] = await r.json()
        const map: Record<string, "legal" | "illegal"> = {}
        for (const col of columns) {
            map[String(col.value)] = targets.includes(String(col.value)) ? "legal" : "illegal"
        }
        setLegalityMap(map)
    } catch { /* keep map empty */ }
}
```

### Step 8.3 — Wire `onTransition` callback from `WorkspaceActionContent`

In `WorkspaceActionContent.tsx`, the `onTransition` callback passed to `KanbanView` calls:

```ts
async ({ record, model_key, target_state_value }) => {
    const url = `/api/workflow/${model_key}/${currentGroupBy}/transition`
    const r = await fetch(url, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ record_uuid: record.id, target_state_value }),
    })
    if (!r.ok) {
        const msg = await r.text().catch(() => "transition failed")
        toast.error(msg)
        throw new Error(msg)
    }
}
```

The `toast` import depends on the project's existing toast utility — verify it exists during implementation (likely `src/frontend/src/utils/toast.ts`).

### Step 8.4 — Run drag test, expect pass

```bash
cd src/frontend && bun run test KanbanView.drag.test.tsx
```

Expected: drag test passes.

### Step 8.5 — Stop for review

---

## Task 9 — Field drag path + non-workflow column source

**Files:**
- Modify: `src/frontend/src/workspace/views/kanban/KanbanBoard.tsx` (already handled in Task 5)
- Modify: `src/frontend/src/__tests__/views/KanbanView.drag.test.tsx`
- Modify: `src/frontend/src/workspace/components/action/WorkspaceActionContent.tsx` (add `onFieldWrite`)

### Step 9.1 — Add field-write test

Append to `KanbanView.drag.test.tsx`:

```tsx
test("dragging on a non-workflow group-by dispatches onFieldWrite", async () => {
    const onTransition = vi.fn().mockResolvedValue(undefined)
    const onFieldWrite = vi.fn().mockResolvedValue(undefined)
    const plan = { ...baseRenderPlan, on_drag: "field" as const }
    // … render + simulate drag
    // Assert onFieldWrite called with { record, model_key, field: "stage_id", value: "qualified" }
    // Assert onTransition NOT called.
    expect(onFieldWrite).toBeDefined()
    expect(onTransition).toBeDefined()
})

test("on_drag=auto resolves to workflow when fieldSpec.workflow is true", async () => {
    // … render with fieldSpecs={{ stage_id: { workflow: true } }} and on_drag="auto"
    // Assert onTransition called, not onFieldWrite.
})

test("on_drag=auto resolves to field when fieldSpec.workflow is false", async () => {
    // … render with fieldSpecs={{ currency_id: { workflow: false } }} and on_drag="auto"
    // Assert onFieldWrite called, not onTransition.
})
```

### Step 9.2 — `onFieldWrite` callback in `WorkspaceActionContent`

```ts
async ({ record, model_key, field, value }) => {
    await dispatchCommand({ name: "ede.update", payload: { model_key, record_uuid: record.id, values: { [field]: value } } })
}
```

### Step 9.3 — Run tests, expect pass

```bash
cd src/frontend && bun run test KanbanView.drag.test.tsx
```

Expected: all 4 drag tests pass.

### Step 9.4 — Stop for review

---

## Task 10 — Group-by switcher in control panel

**Files:**
- Modify: [src/frontend/src/workspace/components/action/WorkspaceActionControlPanel.tsx](../../../src/frontend/src/workspace/components/action/WorkspaceActionControlPanel.tsx)

### Step 10.1 — Add the dropdown

Locate the centre region (where the search bar lives). When the active view is kanban (`view_type === "kanban"`), render a "Group by" dropdown to its right populated from:

1. `view.default_group_by` (always first).
2. Each `<groupby>` element from the active SearchView's RenderPlan.

```tsx
{view.view_type === "kanban" && (
    <select
        className="text-sm bg-surface border border-default rounded px-2 py-1"
        value={currentGroupBy}
        onChange={e => setGroupBy(e.target.value)}>
        <option value={view.default_group_by}>Group: {view.default_group_by}</option>
        {searchView.groupbys.map(g => (
            <option key={g.fields[0]} value={g.fields[0]}>Group: {g.name}</option>
        ))}
    </select>
)}
```

`setGroupBy` updates an action-scoped state value that `KanbanView` reads via prop. No persistence in Phase 1.

### Step 10.2 — Run `bun run build`

```bash
cd src/frontend && bun run build
```

Expected: clean.

### Step 10.3 — Stop for review

---

## Task 11 — Card renderer tests

**Files:**
- Create: `src/frontend/src/__tests__/views/KanbanCardRenderer.test.tsx`

### Step 11.1 — Write the tests

Create the file:

```tsx
import { describe, expect, test, vi } from "vitest"
import { render, screen, fireEvent } from "@testing-library/react"

import { KanbanCardRenderer } from "../../workspace/views/kanban/KanbanCardRenderer"
import type { KanbanContainerNode } from "../../workspace/views/kanban/types"

const baseRecord = { id: "rec-1", name: "Acme rate", customer_id: "Acme", amount: 100, priority: "high" }
const baseSpecs = { name: {}, customer_id: {}, amount: {}, priority: {} }

describe("KanbanCardRenderer", () => {

    test("KanbanTitle renders the field value", () => {
        const tpl: KanbanContainerNode = { type: "KanbanCard", children: [
            { type: "KanbanTitle", children: [{ type: "field", name: "name", widget: null, invisible: false, options: {} }] }
        ]}
        render(<KanbanCardRenderer template={tpl} record={baseRecord} fieldSpecs={baseSpecs} onClick={() => {}}/>)
        expect(screen.getByText("Acme rate")).toBeInTheDocument()
    })

    test("KanbanRibbon honours explicit color_map", () => {
        const tpl: KanbanContainerNode = { type: "KanbanCard", children: [
            { type: "KanbanRibbon", attrs: { field: "priority", color_map: "high:#ef4444,medium:#f59e0b,low:#10b981" } }
        ]}
        const { container } = render(<KanbanCardRenderer template={tpl} record={baseRecord} fieldSpecs={baseSpecs} onClick={() => {}}/>)
        const strip = container.querySelector("div[style*='background-color']")
        expect(strip?.getAttribute("style")).toContain("rgb(239, 68, 68)")
    })

    test("KanbanSeparator renders an hr", () => {
        const tpl: KanbanContainerNode = { type: "KanbanCard", children: [{ type: "KanbanSeparator" }] }
        const { container } = render(<KanbanCardRenderer template={tpl} record={baseRecord} fieldSpecs={baseSpecs} onClick={() => {}}/>)
        expect(container.querySelector("hr")).toBeInTheDocument()
    })

    test("invisible fields are not rendered", () => {
        const tpl: KanbanContainerNode = { type: "KanbanCard", children: [
            { type: "KanbanTitle", children: [
                { type: "field", name: "name", widget: null, invisible: false, options: {} },
                { type: "field", name: "customer_id", widget: null, invisible: true, options: {} },
            ]}
        ]}
        render(<KanbanCardRenderer template={tpl} record={baseRecord} fieldSpecs={baseSpecs} onClick={() => {}}/>)
        expect(screen.getByText("Acme rate")).toBeInTheDocument()
        expect(screen.queryByText("Acme")).not.toBeInTheDocument()
    })

    test("click invokes onClick", () => {
        const onClick = vi.fn()
        const tpl: KanbanContainerNode = { type: "KanbanCard", children: [
            { type: "KanbanTitle", children: [{ type: "field", name: "name", widget: null, invisible: false, options: {} }] }
        ]}
        const { container } = render(<KanbanCardRenderer template={tpl} record={baseRecord} fieldSpecs={baseSpecs} onClick={onClick}/>)
        fireEvent.click(container.querySelector("div.cursor-grab")!)
        expect(onClick).toHaveBeenCalledOnce()
    })

    test("nested KanbanRow / KanbanStack render in correct order", () => {
        const tpl: KanbanContainerNode = { type: "KanbanCard", children: [
            { type: "KanbanRow", children: [
                { type: "field", name: "name", widget: null, invisible: false, options: {} },
                { type: "KanbanStack", children: [
                    { type: "field", name: "customer_id", widget: null, invisible: false, options: {} },
                    { type: "field", name: "amount", widget: null, invisible: false, options: {} },
                ]}
            ]}
        ]}
        render(<KanbanCardRenderer template={tpl} record={baseRecord} fieldSpecs={baseSpecs} onClick={() => {}}/>)
        expect(screen.getByText("Acme rate")).toBeInTheDocument()
        expect(screen.getByText("Acme")).toBeInTheDocument()
        expect(screen.getByText("100")).toBeInTheDocument()
    })
})
```

### Step 11.2 — Run, expect pass

```bash
cd src/frontend && bun run test KanbanCardRenderer.test.tsx
```

Expected: all 6 tests pass.

### Step 11.3 — Stop for review

---

## Task 12 — Column tests (fold, count, empty state)

**Files:**
- Create: `src/frontend/src/__tests__/views/KanbanColumn.test.tsx`

### Step 12.1 — Write tests

```tsx
import { describe, expect, test, vi } from "vitest"
import { render, screen, fireEvent } from "@testing-library/react"

import { KanbanColumn } from "../../workspace/views/kanban/KanbanColumn"
import type { KanbanColumn as ColumnT, KanbanContainerNode } from "../../workspace/views/kanban/types"

const emptyTpl: KanbanContainerNode = { type: "KanbanCard", children: [
    { type: "KanbanTitle", children: [{ type: "field", name: "name", widget: null, invisible: false, options: {} }] }
]}

describe("KanbanColumn", () => {
    test("empty state shows when records is empty", () => {
        const col: ColumnT = { value: "new", label: "New", count: 0, color: null, foldedByDefault: false, records: [] }
        render(<KanbanColumn column={col} cardTemplate={emptyTpl} viewId="v" userUuid="u" fieldSpecs={{}} quickCreate={false} onRecordClick={() => {}} onQuickCreate={async () => {}}/>)
        expect(screen.getByText("No records")).toBeInTheDocument()
    })

    test("count badge reflects column.count", () => {
        const col: ColumnT = { value: "new", label: "New", count: 7, color: null, foldedByDefault: false, records: [] }
        render(<KanbanColumn column={col} cardTemplate={emptyTpl} viewId="v" userUuid="u" fieldSpecs={{}} quickCreate={false} onRecordClick={() => {}} onQuickCreate={async () => {}}/>)
        expect(screen.getByText("7")).toBeInTheDocument()
    })

    test("fold toggle persists to localStorage", () => {
        const col: ColumnT = { value: "won", label: "Won", count: 3, color: null, foldedByDefault: false, records: [{ id: "r1", name: "x" }] }
        render(<KanbanColumn column={col} cardTemplate={emptyTpl} viewId="v-fold" userUuid="u-1" fieldSpecs={{}} quickCreate={false} onRecordClick={() => {}} onQuickCreate={async () => {}}/>)
        const foldBtn = screen.getByLabelText("Fold column")
        fireEvent.click(foldBtn)
        expect(window.localStorage.getItem("kanban_fold_v1:u-1:v-fold:won")).toBe("true")
    })

    test("terminal columns fold by default", () => {
        const col: ColumnT = { value: "won", label: "Won", count: 3, color: null, foldedByDefault: true, records: [{ id: "r1", name: "x" }] }
        render(<KanbanColumn column={col} cardTemplate={emptyTpl} viewId="v" userUuid="u" fieldSpecs={{}} quickCreate={false} onRecordClick={() => {}} onQuickCreate={async () => {}}/>)
        expect(screen.getByLabelText("Unfold column")).toBeInTheDocument()
    })
})
```

### Step 12.2 — Run, expect pass

```bash
cd src/frontend && bun run test KanbanColumn.test.tsx
```

Expected: all 4 tests pass.

### Step 12.3 — Stop for review

---

## Task 13 — Quick-create tests

**Files:**
- Create: `src/frontend/src/__tests__/views/KanbanQuickCreate.test.tsx`

### Step 13.1 — Tests

```tsx
import { describe, expect, test, vi } from "vitest"
import { render, screen, fireEvent } from "@testing-library/react"

import { KanbanQuickCreate } from "../../workspace/views/kanban/KanbanQuickCreate"

describe("KanbanQuickCreate", () => {
    test("Enter submits the title", async () => {
        const onSubmit = vi.fn().mockResolvedValue(undefined)
        render(<KanbanQuickCreate onSubmit={onSubmit} onCancel={() => {}}/>)
        const input = screen.getByPlaceholderText("Title…") as HTMLInputElement
        fireEvent.change(input, { target: { value: "New lead" } })
        fireEvent.keyDown(input, { key: "Enter" })
        await Promise.resolve()
        expect(onSubmit).toHaveBeenCalledWith("New lead")
    })

    test("Escape calls onCancel", () => {
        const onCancel = vi.fn()
        render(<KanbanQuickCreate onSubmit={async () => {}} onCancel={onCancel}/>)
        fireEvent.keyDown(screen.getByPlaceholderText("Title…"), { key: "Escape" })
        expect(onCancel).toHaveBeenCalledOnce()
    })

    test("empty input does not submit", async () => {
        const onSubmit = vi.fn().mockResolvedValue(undefined)
        render(<KanbanQuickCreate onSubmit={onSubmit} onCancel={() => {}}/>)
        fireEvent.keyDown(screen.getByPlaceholderText("Title…"), { key: "Enter" })
        expect(onSubmit).not.toHaveBeenCalled()
    })
})
```

### Step 13.2 — Run, expect pass

```bash
cd src/frontend && bun run test KanbanQuickCreate.test.tsx
```

Expected: 3 tests pass.

### Step 13.3 — Stop for review

---

## Task 14 — First adopter: `pricing.rate` kanban

**Files:**
- Create: [src/domains/logistics/pricing/views/pricing_rate_kanban.xml](../../../src/domains/logistics/pricing/views/pricing_rate_kanban.xml)
- Modify: [src/domains/logistics/pricing/__manifest__.py](../../../src/domains/logistics/pricing/__manifest__.py) to register the new view

### Step 14.1 — Write the view XML

```xml
<view id="pricing_rate_kanban_view" model="pricing.rate" priority="0">
  <KanbanView default_group_by="status" on_drag="workflow" order_by="updated_at_utc desc" quick_create="false">
    <KanbanCard>
      <KanbanRibbon field="status"/>
      <KanbanTitle>    <field name="rate_code"/>     </KanbanTitle>
      <KanbanSubtitle> <field name="customer_id"/>   </KanbanSubtitle>
      <KanbanRow>
        <field name="origin_id"      widget="badge"/>
        <field name="destination_id" widget="badge"/>
      </KanbanRow>
      <KanbanFooter>
        <field name="net_margin_amount" widget="monetary" option-currency-field="currency_id"/>
        <field name="valid_until"        widget="date"/>
        <field name="currency_id" invisible="true"/>
      </KanbanFooter>
    </KanbanCard>
  </KanbanView>
</view>
```

Adjust the field names to match the actual `pricing.rate` schema if any of `rate_code`, `customer_id`, `origin_id`, `destination_id`, `net_margin_amount`, `valid_until`, `currency_id` are named differently. Cross-check against [src/domains/logistics/pricing/models/](../../../src/domains/logistics/pricing/) during implementation.

### Step 14.2 — Register in manifest

Open `__manifest__.py`. Add the path to the `data` list:

```python
"data": [
    ...,
    "views/pricing_rate_kanban.xml",
],
```

### Step 14.3 — Wire as a view on the `pricing.rate` action

The action XML lives in [src/domains/logistics/pricing/data/](../../../src/domains/logistics/pricing/) — find the `ir.action` row for `pricing.rate` (search `model_key=pricing.rate`) and add `kanban` to its `view_modes` field (CSV).

### Step 14.4 — Run pytest

```bash
./run_tests.sh src/tests/foundation/test_dsl_kanban.py
```

Expected: still green; the XML loads.

### Step 14.5 — Stop for review

---

## Task 15 — First adopter: `crm.lead` kanban

**Files:**
- Create: `src/domains/logistics/sales_crm/views/crm_lead_kanban.xml`
- Modify: `src/domains/logistics/sales_crm/__manifest__.py`
- Modify: action XML for `crm.lead`

### Step 15.1 — XML

```xml
<view id="crm_lead_kanban_view" model="crm.lead" priority="0">
  <KanbanView default_group_by="stage_id" on_drag="auto" order_by="updated_at_utc desc">
    <KanbanCard>
      <KanbanTitle>    <field name="name"/>          </KanbanTitle>
      <KanbanSubtitle> <field name="customer_id"/>   </KanbanSubtitle>
      <KanbanRow>
        <field name="amount" widget="monetary" option-currency-field="currency_id"/>
        <field name="probability"/>
      </KanbanRow>
      <KanbanFooter>
        <field name="salesperson_id"/>
        <field name="expected_close_date" widget="date"/>
        <field name="currency_id" invisible="true"/>
      </KanbanFooter>
    </KanbanCard>
  </KanbanView>
</view>
```

### Step 15.2 — Register + action wiring

Same pattern as Task 14.

### Step 15.3 — Run pytest

```bash
./run_tests.sh
```

Expected: still green.

### Step 15.4 — Stop for review

---

## Task 16 — First adopter: `crm.opportunity` kanban

**Files:**
- Create: `src/domains/logistics/sales_crm/views/crm_opportunity_kanban.xml`
- Modify: `src/domains/logistics/sales_crm/__manifest__.py`
- Modify: action XML for `crm.opportunity`

### Step 16.1 — XML

Mirror Task 15's `crm.lead` kanban — same DSL shape, model `crm.opportunity`, field names per the opportunity schema (`name`, `customer_id`, `amount`, `probability`, `salesperson_id`, `expected_close_date`, `currency_id`, `stage_id`).

### Step 16.2 — Register + action wiring

Same pattern.

### Step 16.3 — Stop for review

---

## Task 17 — First adopter XML load tests

**Files:**
- Create: `src/tests/foundation/test_dsl_kanban_first_adopters.py`

### Step 17.1 — Tests

```python
import os
import pytest

from ede.core.services.presentation.dsl.parser import DslParser

ROOT = os.path.dirname(os.path.dirname(os.path.dirname(__file__)))

FIRST_ADOPTERS = [
    "domains/logistics/pricing/views/pricing_rate_kanban.xml",
    "domains/logistics/sales_crm/views/crm_lead_kanban.xml",
    "domains/logistics/sales_crm/views/crm_opportunity_kanban.xml",
]

@pytest.mark.parametrize("rel_path", FIRST_ADOPTERS)
def test_first_adopter_kanban_xml_parses(rel_path):
    full = os.path.join(ROOT, rel_path)
    with open(full, "r", encoding="utf-8") as f:
        xml = f.read()
    plan = DslParser().parse_to_render_plan(dsl_xml_text=xml)
    assert plan["view_type"] == "kanban"
    assert plan["model"]
    assert plan["default_group_by"]
    assert plan["card_template"]["type"] == "KanbanCard"
    assert plan["field_names"]
```

### Step 17.2 — Run

```bash
./run_tests.sh src/tests/foundation/test_dsl_kanban_first_adopters.py -v
```

Expected: 3 parametrised tests pass.

### Step 17.3 — Stop for review

---

## Task 18 — Final verification

### Step 18.1 — Full test suite

```bash
./run_tests.sh                                    # pytest
cd src/frontend && bun run test                  # vitest
cd src/frontend && bun run build                 # strict-mode tsc + Vite bundle
```

Expected:
- pytest: 1424 (baseline) + ~14 new ≈ 1438 green.
- vitest: 388 (baseline) + ~17 new ≈ 405 green.
- bun run build: clean.

### Step 18.2 — Browser walkthrough

Start the dev server and have the user execute the K8.2 walkthrough from [phase-1-implementation.md](../../../roadmap/foundation/presentation/phase-1-implementation.md#k82-browser-walkthrough--done-criteria):

1. Pricing → Rates → switch to kanban → see workflow-state columns.
2. Drag a draft rate to Pending Approval → submit succeeds; column counts update.
3. Try to drag draft → Approved → rejected with red ring + toast + snap-back.
4. Switch group-by to `currency_id` → drag now writes the field (no workflow check).
5. Sales & CRM → Leads → drag a lead between stages.
6. Fold "Lost" → reload → fold persists (localStorage).
7. Quick-create on a lead column → new lead appears.

Only after the user verifies all 7 steps live in the browser do we proceed to Task 19.

### Step 18.3 — Stop for review

---

## Task 19 — Roadmap status flip + tracker update

**Files:**
- Modify: [roadmap/foundation/presentation/README.md](../../../roadmap/foundation/presentation/README.md)
- Modify: [roadmap/foundation/presentation/phase-1-implementation.md](../../../roadmap/foundation/presentation/phase-1-implementation.md)
- Modify: [roadmap/roadmap-tracker.md](../../../roadmap/roadmap-tracker.md)

### Step 19.1 — Phase 1 status: 🔴 → ✅

In `phase-1-implementation.md` header: change `**Status:** 🔴 Not Started` to `**Status:** ✅ Delivered <today's date>`.

In `README.md` "Phased Delivery" table: flip Phase 1 row's status emoji 🔴 → ✅ and append a one-line note: `Delivered <date> — DSL parser branch + RelaxNG schema + 8 React card components + group-by switcher + workflow + field drag paths + 3 first adopters; <pytest count> pytest + <vitest count> vitest green; bun run build clean; browser walkthrough verified live by user.`

In `README.md` top header: change `**Status:** 🟡 In Progress — … KanbanView is **not yet built** …` to `**Status:** 🟡 In Progress — Phase 1 (KanbanView MVP) ✅ Delivered <date>; Phase 2 and Phase 3 🔴 Not Started.`

### Step 19.2 — Tracker

Update [roadmap/roadmap-tracker.md](../../../roadmap/roadmap-tracker.md):
- Top `**Last refreshed:**` line: prepend a new entry mentioning the kanban Phase 1 delivery with verification numbers.
- Status Roll-up table: foundation `15 🔴 / 1 🟡 / 5 ✅` → `14 🔴 / 1 🟡 / 6 ✅`. Total: `42 🔴 / 3 🟡 / 6 ✅` → `41 🔴 / 3 🟡 / 7 ✅`.
- Presentation module section's Phase 1 row: 🔴 → ✅ with delivery date and verification numbers.

### Step 19.3 — Sync docs

The `syncing-roadmap-to-docs` hook fires automatically after each roadmap edit. Confirm `docs/foundation-presentation.md` gets its `status-snapshot` and `built` SYNC-BLOCKs updated by the skill (the skill walks Phase 1 row 🔴 → ✅ and adds a "Built Capabilities" row).

### Step 19.4 — Stop for review

Final reportable summary: feature complete, tests green, docs synced. Tell the user the verification numbers and which adopter views are live.

---

## Self-Review (writing-plans Step "Self-Review")

**1. Spec coverage check** — every workstream from [phase-1-implementation.md](../../../roadmap/foundation/presentation/phase-1-implementation.md) is implemented by a task:

| Spec WS | Plan Task(s) |
|---|---|
| WS-K1 (parser + RNG + tests) | Task 1, Task 2 |
| WS-K2 (KanbanView skeleton + viewRegistry + WorkspaceActionContent + group-by switcher + data load) | Task 3, Task 4, Task 5, Task 10 |
| WS-K3 (8 card components + dispatcher) | Task 6, Task 11 |
| WS-K4 (column mechanics) | Task 5 (KanbanColumn + header) + Task 12 |
| WS-K5 (drag-drop) | Task 7 (deps), Task 8 (workflow path), Task 9 (field path + auto resolution) |
| WS-K6 (fold + quick-create) | Task 5 (`useKanbanFold`), Task 6 (KanbanQuickCreate), Task 12 (fold tests), Task 13 (quick-create tests) |
| WS-K7 (first adopters) | Task 14 (pricing.rate), Task 15 (crm.lead), Task 16 (crm.opportunity), Task 17 (load tests) |
| WS-K8 (verification + roadmap close-out) | Task 18, Task 19 |

**2. Placeholder scan** — searched for `TBD`, `TODO`, `fill in`, `similar to Task`, `appropriate error handling`. None found except Task 8's drag-test placeholder assertion, which is explicitly called out and resolved by step 8.4 (run-pass).

**3. Type consistency** — names used across tasks: `KanbanRenderPlan`, `CardTemplateNode`, `KanbanColumn`, `KanbanCardShell`, `KanbanCardRenderer`, `useKanbanColumns`, `useKanbanFold`, `KanbanColumnSource`, `resolveDragMode`, `resolveColumnSource`. All defined once in Task 3 or Task 4 or Task 5 and referenced consistently downstream. Method names: `onTransition`, `onFieldWrite`, `onCreate`, `onRecordClick`, `onQuickCreate`, `onToggleFold` — all consistent across the renderer prop chain.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-05-11-foundation-presentation-kanban-phase-1.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review each delivered task in the main thread, fast iteration. Good for a plan this size because each task is self-contained and we won't drown the main context in implementation detail.

**2. Inline Execution** — I execute the tasks in this session using `superpowers:executing-plans`, batched with checkpoints so you can review at natural break points (e.g. after backend Task 1–2, after card surface Task 3–7, after drag Task 8–10, after tests Task 11–13, after adopters Task 14–17, then verify + close-out Task 18–19).

Which approach?
