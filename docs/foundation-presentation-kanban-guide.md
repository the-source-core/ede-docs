# Foundation Presentation — Kanban Authoring Guide

> **For developers adding a kanban view to a model.** Hand-authored. The auto-maintained doc that mirrors built state is [`foundation-presentation.md`](./foundation-presentation.md); this file is the *how do I use it* companion.

A kanban view groups records into columns by a single field (workflow stage, Enum, or Many2one) and lets the user drag cards between columns to change that field. This guide walks through the DSL surface, how the renderer interprets it, what each card-template tag does, and the most common authoring gotchas.

---

## TL;DR — the minimum viable kanban

```xml
<view id="my_model_kanban_view" model="my.model" priority="0">
  <KanbanView default_group_by="status" on_drag="auto">
    <KanbanCard>
      <KanbanRibbon field="status"/>
      <KanbanTitle>    <field name="name"/>          </KanbanTitle>
      <KanbanSubtitle> <field name="customer_id"/>   </KanbanSubtitle>
      <KanbanFooter>
        <field name="amount" widget="monetary" option-currency-field="currency_id"/>
        <field name="currency_id" invisible="true"/>
      </KanbanFooter>
    </KanbanCard>
  </KanbanView>
</view>
```

Then register the file in `__manifest__.py` `data: [...]` and add `kanban` to the model's `ir.action.available_views` CSV. That's the whole authoring contract.

---

## `<KanbanView>` attributes

| Attribute | Required | Default | Meaning |
|---|---|---|---|
| `default_group_by` | **yes** | — | Field name to group records by when the user has no explicit groupby. Becomes the column field; drag-drop writes to this field. |
| `order_by` | no | `""` | SQL-style order applied within each column (e.g. `"updated_at_utc desc"`). |
| `on_drag` | no | `"auto"` | `workflow` / `field` / `auto` — see [Drag modes](#drag-modes) below. |
| `allow_reorder` | no | `false` | Phase 2 opener — intra-column reorder via a `sequence` Integer field. Not honoured by Phase 1. |
| `quick_create` | no | `true` | When `true`, a `+` icon appears on each column header; clicking opens an inline `name`-only create input. Set `false` when the model has multiple required fields a title-only create can't satisfy. |

---

## Drag modes

The kanban supports two drag semantics depending on the group-by field:

### `on_drag="workflow"`

Drag-drop dispatches a **workflow transition** via `WorkflowService.transition()`. The board hover-fetches `workflowService.available()` at drag-start and paints every column with a legality hint:

- **Green ring (`ring-emerald`)** — a legal transition exists from the current stage to this column's stage.
- **Red ring + dimmed (`ring-rose-500/60 opacity-60`)** — no legal transition; drop will be rejected by the backend.
- **No ring** — neutral; user hasn't started dragging or the column is the source.

Only valid when the group-by field has `workflow=True` declared on the model. The backend's workflow ORM guard rejects direct writes, so this is the **only** drag path that works for workflow fields.

### `on_drag="field"`

Drag-drop dispatches a **plain field write** via `ede.update`. No legality preview — every drop is legal as long as the user has write permission on the field.

Use this for non-workflow Enum or Many2one group-by (e.g. salesperson, priority, category).

### `on_drag="auto"` *(recommended)*

The view leaves the choice to the runtime. At each drag-start the resolver looks up the group-by field's `FieldDef.workflow` flag:

- `workflow=true` → workflow path
- otherwise → field path

This is the right default for any kanban whose group-by might switch at runtime (e.g. user toggles between `status` and `salesperson_id` via the search panel's `<groupby>`).

---

## The card template

Every kanban card is built from a recursive tree of `Kanban*` tags inside `<KanbanCard>`. **You compose freely** — there is no fixed "header / body / footer" slot order; the tags carry their own typography and the renderer arranges them in source order.

### Tag vocabulary

| Tag | Body | Purpose |
|---|---|---|
| `<KanbanCard>` | one-or-more `Kanban*` children | Card root. Exactly one per view. |
| `<KanbanRibbon field="…" [option-color-map="csv"]/>` | leaf | 4px colored stripe at top edge. Color resolution: explicit `color_map` → workflow stage's `kanban_color` → neutral grey. |
| `<KanbanTitle>` | `<field>`, `<button>`, or layout primitives | Headline typography (font-semibold, text-base). Typically wraps the record's `name`. |
| `<KanbanSubtitle>` | same | Secondary muted text under the title (e.g. customer name). |
| `<KanbanFooter>` | same | Bottom band with separator above and slightly muted background. Renders last regardless of source position. |
| `<KanbanRow>` | same | Horizontal flex row — children laid side-by-side, justified between. |
| `<KanbanStack>` | same | Vertical stack — children laid top-to-bottom. |
| `<KanbanSeparator/>` | leaf | Thin horizontal divider. |
| `<field name="…" widget="…" [option-* …] [invisible="true"]/>` | leaf | Same shape as ListView/FormView field; same widget registry. `invisible="true"` skips render but keeps the field in the action's `field_names` allowlist (use this for `currency_id` anchors on monetary widgets). |
| `<button name="…" …/>` | leaf | Phase 1 deferral — renders `null` in MVP; Phase 2 will wire these. |

### Color-map syntax

`option-color-map` on `<KanbanRibbon>` is CSV: `"value:#hex,…"`. Example:

```xml
<KanbanRibbon field="priority"
              option-color-map="high:#ef4444,medium:#f59e0b,low:#10b981"/>
```

For workflow fields you typically **don't** need a color map — set `kanban_color` on each `ir.workflow.stage` row in your `<workflow>` DSL instead, and let the ribbon read it automatically. Use the explicit map only for non-workflow Enums.

### What renders read-only

Card fields use the existing `FieldView` pipeline in read-only mode. All widgets (`monetary`, `badge`, `progressbar`, `avatar_group`, `relative_date`, `date`, etc.) work inside cards. **There is no edit mode on a kanban card** — clicking the card opens the form view.

---

## Column behaviour

### Every legal column appears, even empty

The kanban advertises the **full universe** of column values from the field's metadata, not just the values that currently have records:

- **Workflow field:** columns = all `ir.workflow.stage` rows ordered by `sequence`. Terminal stages (`is_terminal=true`) fold by default.
- **Enum field:** columns = all `FieldDef.selection` options. No backend roundtrip — the data already arrived with the action's metadata.
- **Many2one field:** columns = all records of `ref_model` (searched with `limit=200`, ordered by `name`).
- **Anything else:** fallback — derive columns from the records present. Only data-present columns appear (because we cannot enumerate an open domain).

Records whose group value matches no column spec (stale enum value after a model migration, archived M2O target, etc.) drop into a synthetic `(none)` column appended at the end. This is intentional — invisible orphans are worse than a visible reminder.

### Fold / unfold

Users can fold any column. Folded columns collapse to a **36px-wide vertical strip** with the label rendered sideways (`writing-mode: vertical-rl rotate-180`). The column count badge stays visible on the strip; the color stripe stays at the top. Clicking the strip unfolds.

Fold state persists per `(user, view_id, column_value)` in `localStorage` under the key `kanban_fold_v1:<user_uuid>:<view_id>:<column_value>`. The persisted value is `"1"` (folded) or `"0"` (unfolded). Terminal workflow stages start folded automatically.

### Quick-create

When `quick_create="true"` (the default), each column header shows a `+` icon. Clicking it opens an inline input above the cards. Pressing Enter dispatches `ede.create` with `{name: <title>, <group_by_field>: <column_value>}`. Escape cancels.

The quick-create currently assumes the model has a `name` field that satisfies all required columns. If your model has additional required fields that a single-line input can't supply, set `quick_create="false"` on the `<KanbanView>` and authors will use the form view to create (the kanban falls back to opening the form view in create mode with the group-by field pre-filled).

### Group-by switcher

There is **no kanban-specific group-by dropdown** — the kanban honours the same `groupByField` state that the search panel writes for ListView. To make a field switchable at runtime, declare it as a `<groupby>` in the corresponding `<SearchView>`. The drag mode auto-resolves at runtime, so a kanban grouped by `status` (workflow) will reconfigure to `on_drag="field"` automatically when the user switches groupby to a non-workflow field.

---

## Authoring checklist for a new adopter

1. **Verify the model has a stable group-by field.** Workflow `status` (Enum with `workflow=True`) or a real `stage_id` Many2one is ideal. A boolean is a poor choice (only two columns; rarely useful).
2. **Write the XML** under `views/<model>_kanban.xml` next to the model's existing `list` / `form` view files. Use the model key (not table name) in the `model="…"` attribute.
3. **Register in `__manifest__.py`** — add the relative path to the module's `data: [...]` list near the other view files.
4. **Add `kanban` to the action's `available_views` CSV.** Find the `ir.action` row for the model (usually in `data/<module>_menus.xml` or `data/<module>_actions.xml`) and edit the `available_views="list,form"` field to include `kanban`.
5. **Run the parser tests** — `pytest src/tests/foundation/test_dsl_kanban_first_adopters.py` does a load-test on each adopter XML. Add your view to the `FIRST_ADOPTERS` list if it warrants regression coverage.
6. **Run the build** — `cd src/frontend && bun run build && bun run test`.
7. **Browser-walk it.** Switch to the kanban view, drag a card, fold a column, reload, quick-create if enabled.

---

## Common gotchas

### "My column for stage X is missing"

Two causes:
- **Workflow stage doesn't exist.** The kanban shows every `ir.workflow.stage` row for the workflow definition; if a stage is missing from your workflow XML, it won't appear as a column. Add the stage to the `<workflow>` DSL and re-seed.
- **Backend workflow-definition query field name drift.** The column source queries `ir.workflow.definition` by `(model_key, field_name)`. If a refactor renames either column on the definition model, the query falls through to the fallback bucketing (only data-present columns appear). Fix at [`src/frontend/src/workspace/views/kanban/columnSource.ts`](../src/frontend/src/workspace/views/kanban/columnSource.ts).

### "I get a `(none)` column with one or two records in it"

That column holds records whose group-by value doesn't match any legal stage / option / M2O record. Usually means a record was created against a value that was later removed from the field's domain. Cleanup is a domain concern — write a migration or let the user re-categorise via the form view.

### "Drag-drop says transition forbidden but the workflow allows it"

The legality preview reads `workflowService.available(model, field, record_uuid)`. If you just changed the workflow definition (added a transition) and a stale page is open, the preview lags until the user refreshes. The actual transition dispatch is correct — it's a frontend cache problem, not a workflow engine bug.

### "Quick-create dispatched but no record appeared"

`ede.create` returned an error that the kanban silently caught (the row currently re-loads on success only). Check the browser network panel for the `POST /api/.../create` response. The most common cause is required fields the title-only quick-create can't supply — set `quick_create="false"` and let the user create via form view.

### "The monetary widget shows no currency symbol"

The `option-currency-field="…"` on a `<field widget="monetary">` references a sibling field on the record. That sibling needs to be in the card template too (usually `invisible="true"` so it doesn't render):

```xml
<KanbanFooter>
  <field name="amount" widget="monetary" option-currency-field="currency_id"/>
  <field name="currency_id" invisible="true"/>
</KanbanFooter>
```

Without the invisible sibling, the action's field allowlist (`view_field_names["kanban"]`) doesn't fetch `currency_id` for each record, and the widget falls back to no symbol.

### "Folded column flashes back to unfolded on every reload"

The localStorage key is per-user-per-view-per-column. If you regenerated the view with a different `view_id` (e.g. you renamed the XML id), the fold history doesn't carry over — that's a feature, not a bug. The old keys are orphaned but harmless.

---

## File layout reference

```
src/frontend/src/workspace/views/kanban/
├── types.ts                    KanbanRenderPlan, CardTemplateNode, KanbanColumn
├── KanbanView.tsx              Top-level — picks groupBy, derives columns
├── KanbanBoard.tsx             DndContext + horizontal column container
├── KanbanColumn.tsx            One column — fold logic + drop target
├── KanbanColumnHeader.tsx      Color strip + label + count + chevron + quick-create
├── KanbanQuickCreate.tsx       Inline create input
├── KanbanCardRenderer.tsx      Recursive tag-dispatch interpreter
├── dragMode.ts                 resolveDragMode(plan, groupBy, fieldDefs)
├── columnSource.ts             fetchWorkflowStages / fetchEnumOptions / fetchM2OOptions
├── hooks/
│   ├── useKanbanColumns.ts     Column source resolution + record bucketing
│   └── useKanbanFold.ts        localStorage fold state
└── cards/
    ├── KanbanCardShell.tsx     Draggable card wrapper
    ├── KanbanRibbon.tsx        4px color stripe
    ├── KanbanTitle.tsx         Headline text
    ├── KanbanSubtitle.tsx      Muted secondary text
    ├── KanbanFooter.tsx        Bottom band
    ├── KanbanRow.tsx           Horizontal flex
    ├── KanbanStack.tsx         Vertical stack
    └── KanbanSeparator.tsx     Horizontal divider
```

---

## What this guide does not cover

- **Backend parser internals.** See [`parser.py`](../src/ede/core/services/presentation/dsl/parser.py) `_parse_kanban_view` and the RelaxNG grammar in [`view.rng`](../src/ede/core/services/presentation/dsl/schemas/view.rng).
- **Phase 2 features** (intra-column reorder, sum aggregates, per-column pagination, cover image, kanban menu, conditional ribbons). See the roadmap [`phase-1-implementation.md`](../roadmap/foundation/presentation/phase-1-implementation.md) → "Out of Phase 1 Scope" table.
- **`@dnd-kit/core` internals.** The board's drag handler is a thin wrapper; the library's docs are authoritative for advanced pointer semantics.

---

## Related

- [foundation-presentation.md](./foundation-presentation.md) — auto-maintained built-state mirror for the module
- [foundation-workflow.md](./foundation-workflow.md) — the workflow engine the kanban consumes for drag legality + transitions
- [10-presentation-dsl.md](./10-presentation-dsl.md) — legacy DSL reference (list / form / search authoring)
- [roadmap/foundation/presentation/phase-1-implementation.md](../roadmap/foundation/presentation/phase-1-implementation.md) — Phase 1 implementation plan (now code-complete 🟡, awaiting browser walkthrough)

*Hand-authored 2026-05-11. Not auto-synced from roadmap — update directly when the kanban authoring surface changes.*
