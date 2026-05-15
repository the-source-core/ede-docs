<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation — Presentation (View DSL & Web Client) — Implementation Docs

**Module:** `foundation.presentation` (`src/ede/foundation/presentation/` for the backend DSL element registry; `src/frontend/src/workspace/views/` for the React renderers)
**Roadmap:** [roadmap/foundation/presentation/](../roadmap/foundation/presentation/README.md)
**Status:** 🟡 In Progress — Phase 1 KanbanView MVP ✅ Delivered 2026-05-11 (verified live via user browser walkthrough); Phases 2 & 3 🔴 Not Started. ListView, FormView, SearchView, `<DynamicProperties/>`, `<workflow>` statusbar previously delivered via `foundation.workflow` and `foundation.customization`.
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
The view DSL grammar plus the React renderers that interpret it. Authors declare views as XML; the parser emits a frontend-consumable RenderPlan; the React webclient looks up a renderer by `view_type` and draws it. Today three view types ship (`list`, `form`, `search`); Phase 1 of this module adds the fourth (`kanban`).
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
View-DSL changes have so far ridden on the back of consumer phases — `<statusbar/>` lived in `foundation.workflow` Phase 2; `<DynamicProperties/>` lived in `foundation.customization` Phase 1. That is fine for one-off element additions but creates a maintenance gap when the work is itself a major surface, like adding a wholly new view type. Pulling the presentation module out gives the next view types (kanban now; pivot, calendar, gantt later) a clear owning home, and makes cross-cutting DSL evolution (conditional invisibility, expression-driven attributes, element composition rules) reviewable in one place.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- Module authors declare a view as XML in their `views/*.xml` file, register it via `__manifest__.py` "data", and the renderer surfaces it automatically when an action is invoked.
- The React webclient looks up `view_type` (`list` / `form` / `search` / `kanban`) in `src/frontend/src/workspace/views/viewRegistry.ts` and dispatches to the matching component.
- For kanban (Phase 1), a workflow-aware drag-drop calls `WorkflowEngine.transition_by_command`; a non-workflow drag falls back to `ede.update`.
- Integration boundary — produces: parsed RenderPlans + React renderers. Consumes: kernel field metadata, workflow engine transitions, `ede.search` / `ede.read_group` for data.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Author]                          [foundation.presentation]                    [React webclient]
─────────                         ─────────────────────────                    ──────────────────
view XML in module's              ViewRegistry.discover()              ──►   GET /api/web/action/:id
data manifest                     reads __manifest__.py "data"                returns { views: [...] }
       │                                  │                                          │
       ▼                                  ▼                                          ▼
DslParser.parse_to_render_plan(xml) ──►  RenderPlan dict (view_type, fields, …)   viewRegistry[view_type]
                                                                                      │
                                                                                      ▼
                                                                              renders the screen
                                                                              (ListView / FormView /
                                                                               SearchView / KanbanView)
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `presentations.presentation` | Persists registered view DSL XML keyed by `view_id` and `view_type` | `src/ede/foundation/presentation/models/presentations.py` |
| `presentations.data_filter` | User-saved data filter (search domain + group-by) per view | `src/ede/foundation/presentation/models/data_filter.py` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `DslParser` | Parses XML DSL into a frontend-consumable RenderPlan; dispatches on first child element (`ListView` / `FormView` / `SearchView` / `KanbanView` Phase 1 / `page` legacy) | `src/ede/core/services/presentation/dsl/parser.py` |
| `DslValidator` | RelaxNG-validates DSL XML before parsing | `src/ede/core/services/presentation/dsl/validator.py` |
| `DslLoader` | Loads view XML files from app `__manifest__.py` "data" entries | `src/ede/core/services/presentation/dsl/loader.py` |
| `ViewRegistry` | Indexes registered views by `(model_key, view_type, priority)`; resolves the highest-priority view at lookup time | `src/ede/core/services/presentation/view_registry.py` |
| Frontend `viewRegistry.ts` | Per-view-type metadata (`showSearchPanel`, `showListPager`) consumed by the workspace controller | `src/frontend/src/workspace/views/viewRegistry.ts` |
| Frontend `WorkspaceActionController` | Looks up the renderer by `view_type` and forwards the RenderPlan + record set | `src/frontend/src/workspace/components/action/WorkspaceActionController.tsx` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ — presentation is read-mostly; data commands flow through `ede.search`, `ede.read_group`, `ede.create`, `ede.update`, `ede.delete` and through workflow / domain commands | | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `GET /api/web/bootstrap` | Returns session + apps + menus payload for the React webclient on boot | `src/ede/foundation/presentation/api/web_client.py` |
| `GET /api/web/action/:id` | Returns the action + its views (parsed RenderPlans) for a menu leaf | `src/ede/foundation/presentation/api/web_client.py` |
| `POST /api/web/data_filter` | Saves a user data filter | `src/ede/foundation/presentation/api/controllers.py` |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_ — presentation does not own state; it surfaces records whose state machines live in their owning modules.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` (in `src/ede/foundation/settings.py`): `"presentation"` — already active. No change for the kanban Phase 1 work.
- `ACTIVE_DOMAINS`: n/a (foundation engine).
- Manifest `depends`: `["base", "workflow"]` — workflow is a soft dep (only the workflow drag path calls it) but `__manifest__.py` declares it explicitly so the boot order is deterministic.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `PRESENTATION_VIEW_VALIDATION` | `str` | `strict` | `PRESENTATION_VIEW_VALIDATION` | Load-time view referential-validator mode (Enhancement 03 — anchored on Phase 1, 🔴 Not Started). `strict` blocks `ede migrate upgrade` on any view-validation failure (missing `ir.action` for inline relational targets, field-name typos, broken `ref=` lookups). `warn` logs and continues — rollout-friendly default while existing modules clean up violations. `off` skips the validator entirely. |
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
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| _none_ — all kanban configuration is per-view DSL XML in consumer modules; this module ships no seed rows | |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | KanbanView MVP — DSL + Renderer + Workflow Drag | ✅ Delivered 2026-05-11 | [phase-1-implementation.md](../roadmap/foundation/presentation/phase-1-implementation.md) |
| Phase 2 | Kanban Power Features                            | 🔴 Not Started | _scoped when Phase 1 ships_ |
| Phase 3 | Kanban At Scale + Future View Types              | 🔴 Not Started | _scoped when Phase 2 ships_ |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| ListView DSL + React renderer | `presentations.presentation` | `src/ede/core/services/presentation/dsl/parser.py` (`_parse_list_view`) · `src/frontend/src/workspace/views/ListView.tsx` | predates the presentation roadmap |
| FormView DSL + React renderer | `presentations.presentation` | `src/ede/core/services/presentation/dsl/parser.py` (`_parse_form_view`) · `src/frontend/src/workspace/views/FormView.tsx` | predates the presentation roadmap |
| SearchView DSL + React renderer | `presentations.presentation` | `src/ede/core/services/presentation/dsl/parser.py` (`_parse_search_view`) | predates the presentation roadmap |
| `<DynamicProperties/>` element | `ir.model.property.definition` | `parser.py` (`_parse_form_elements`) · `src/frontend/src/workspace/views/PropertiesEditor.tsx` | [foundation/customization Phase 1](../roadmap/foundation/customization/README.md) |
| `<statusbar/>` element + `WorkflowController` HTTP routes | `ir.workflow.*` | `parser.py` Phase 2 work · workflow-routes module | [foundation/workflow Phase 2](../roadmap/foundation/workflow/phase-2-implementation.md) |
| `<KanbanView>` DSL element family + React renderer + workflow drag + legality preview + fold-to-vertical-strip + 3 first adopters | `presentations.presentation` · consumes `ir.workflow.*` for the workflow drag path · `pricing.rate.status`, `crm.lead.status`, `crm.opportunity.status` first adopters | `src/ede/core/services/presentation/dsl/parser.py` (`_parse_kanban_view`) · `src/ede/core/services/presentation/dsl/schemas/view.rng` (KanbanView grammar) · `src/ede/foundation/presentation/models/presentations.py` (`_extract_field_names_from_render_plan` kanban walker) · 19 files under `src/frontend/src/workspace/views/kanban/` · `src/frontend/src/workspace/components/action/WorkspaceActionContent.tsx` (kanban branch + `KanbanWorkspaceSection`) · 3 first-adopter view XMLs · developer guide [docs/foundation-presentation-kanban-guide.md](foundation-presentation-kanban-guide.md) | [foundation/presentation Phase 1](../roadmap/foundation/presentation/phase-1-implementation.md) ✅ 2026-05-11 |
| Enhancement 01 — dynamic field-ref domain on Reference pickers (client-side substitution of bare identifiers in `domain="..."` against current form state; missing field values drop their tuple, loose semantics; literals untouched) | n/a — pure frontend evaluator, no model changes | `src/frontend/src/workspace/views/fields/substituteDomain.ts` (pure utility) · `src/frontend/src/workspace/views/fields/ReferenceField.tsx` (calls `substituteDomain` before `searchRelated()`) · 4 sales-crm consumer forms updated (inquiry / lead / opportunity / quote) + `crm.quote.communication.recipient_contact_id` | [enhancements/01-dynamic-field-ref-domain.md](../roadmap/foundation/presentation/enhancements/01-dynamic-field-ref-domain.md) ✅ 2026-05-12 |
| Enhancement 02 — M2O templated display + multi-field name search (`@api.model(..., display_name_format="[{code}] {name}", name_search_fields=["code","name"])` decorator wiring honored end-to-end through `/related/<field>` + first-class `<field display_format="..."/>` XML override; 6 first-adopter masters opted in declaratively with no view-XML changes) | declarative on every code-bearing master — opted in: `res.currency` · `logistics.incoterm.master` · `crm.quote.stage` · `crm.lost.reason` · `logistics.commodity.master` · `logistics.document.type.master` | `src/ede/core/kernel/decorators.py` (decorator + validation + `_extract_format_keys` + `__ede_display_data_fields__` precompute) · `src/ede/core/services/presentation/display_format.py` (shared `render_with_missing` helper) · `src/ede/foundation/presentation/api/controllers.py` (multi-field OR-domain + per-row `data` payload) · `src/ede/foundation/presentation/models/presentations.py` `_resolve_reference_fields` (chip parity) · `src/ede/core/services/presentation/dsl/parser.py` (`display_format` extraction in form/list/kanban) · `src/ede/core/services/presentation/dsl/schemas/view.rng` (RelaxNG additive) · `src/frontend/src/workspace/views/fields/formatDisplay.ts` (pure helper) · `src/frontend/src/workspace/views/fields/ReferenceField.tsx` (override precedence) | [enhancements/02-m2o-display-format-and-multi-field-search.md](../roadmap/foundation/presentation/enhancements/02-m2o-display-format-and-multi-field-search.md) ✅ 2026-05-12 |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Phase 2 power features (intra-column reorder, `<KanbanAggregate/>` header, per-column pagination, `<KanbanCover/>`, `<KanbanMenu>`, conditional ribbons/colors) | 🔴 | _scoped when Phase 2 starts_ |
| Phase 3 at-scale + future view types (virtualisation, swimlanes, saved filters, backend fold persistence, realtime, mobile gestures) | 🔴 | _scoped when Phase 2 ships_ |
<!-- /SYNC-BLOCK -->

### Dynamic field-ref domain on Reference pickers
<!-- HAND-AUTHORED — preserved across syncs -->

Reference field `domain="..."` attributes support bare identifiers that resolve at render time against the current form record. This unblocks parent → child cascading pickers without a server-side endpoint per filter.

**Syntax** — author the domain as a string in your view XML. Quoted strings, numbers, JSON keywords (`true` / `false` / `null`), and list literals pass through verbatim. Any bare alphanumeric token in a value position is treated as a record field name.

```xml
<!-- Static (works on every release) -->
<field name="contact_id"
       domain='[["member_type","=","individual"]]' />

<!-- Dynamic — partner → contact cascade -->
<field name="contact_id"
       domain='[["member_type","=","individual"],["parent_partner_id","=",partner_id]]' />
```

**Resolution rules** (applied client-side, before the request leaves the browser):
- A bare identifier resolves to the matching field in the current record. Reference values (`{ id, display_name }` or `{ record_uuid, display_name }`) substitute to the UUID string; scalars substitute to their JSON-encoded value.
- If the referenced field is `null`, `undefined`, or a Reference with an empty id, the **whole tuple is dropped** from the domain — the picker stays usable while showing the looser candidate set. This is intentional: "no customer selected yet → show all individuals" beats "no customer selected yet → zero results".
- Field names (the left-hand operand of every tuple) must be quoted strings — only the value position is substituted.

**Out of scope** — boolean composition (`partner_id and [...] or []`), token interpolation inside string literals (`"%{partner_id}-suffix"`), traversal on the resolved value (`partner_id.parent_id`). If a picker on a child record needs to scope against a parent record's field, denormalise the field onto the child or expose it as a computed field.

**Implementation**: [`src/frontend/src/workspace/views/fields/substituteDomain.ts`](../src/frontend/src/workspace/views/fields/substituteDomain.ts) — pure utility called from `ReferenceField.renderEditMode` on every render of the edit widget. The substituted string is shipped to `/api/web/<action>/related/<field>?domain=...` exactly as a static domain would be — the backend search endpoint is unchanged and stateless.

### M2O templated display + multi-field name search
<!-- HAND-AUTHORED — preserved across syncs -->

Code-bearing master M2O pickers (currency, incoterms, commodity, lost-reason, quote-stage, document-type, …) declare two pieces of metadata on their `@api.model(...)` and every picker in every view automatically inherits the behaviour. No view-XML edits required.

**Model declaration** — author once on the target model:

```python
@api.model(
    "res.currency",
    record_name="name",
    name_search_fields=["code", "name"],          # search hits both fields with ilike
    display_name_format="[{code}] {name}",        # render label as "[USD] US Dollar"
)
class Currency(DomainModel):
    code = fields.Char(max_length=3, required=True, ...)
    name = fields.Char(max_length=100, required=True, ...)
```

Both kwargs are validated at decoration time — every `{key}` in the format and every entry in `name_search_fields` must resolve to a declared field on the model (auto-injected fields like `record_uuid` and `dbid` count). Unknown references raise `InvalidHandler` with the model key in the message; the boot does not start with a misconfigured model.

**Per-field XML override** — the rare case where one form needs different formatting:

```xml
<!-- Form-view chip: full label "[USD] US Dollar" (model default kicks in, no override needed) -->
<field name="currency_id"/>

<!-- Compact list cell: just the code "USD" -->
<field name="currency_id" display_format="{code}"/>

<!-- Customs filing form: format with extra fields -->
<field name="incoterms_id" display_format="[{term_code}] {name} ({version_year})"/>
```

`display_format` is a first-class attribute on `<field>` (sibling to `domain`, `widget`, `string`, `optional`) in form, list, and kanban view contexts.

**Wire format** — search response from `GET /api/web/<action>/related/<field>?q=USD`:

```json
[
  { "id": "uuid-1",
    "display_name": "[USD] US Dollar",
    "data": { "code": "USD", "name": "US Dollar" } },
  ...
]
```

- `display_name` — pre-rendered server-side using the model's `display_name_format` (or falls back to `record_name` field when the format is unset).
- `data` — present iff the model declared `display_name_format`; carries the union `name_search_fields ∪ {record_name_field} ∪ format-keys`. Used by the frontend when an XML `display_format` override needs to re-render with a different shape.
- The shape stays a strict superset of the legacy `{id, display_name}` — un-opted-in models keep the old behaviour exactly.

**Frontend rendering precedence** — both the dropdown options and the selected chip use the same path:

1. XML `display_format` (interpolated against `data` via [`formatDisplay`](../src/frontend/src/workspace/views/fields/formatDisplay.ts)) — wins when both are present.
2. Server-rendered `display_name` (already encodes the model's `display_name_format`).
3. `name` — last-ditch fallback for unusual payload shapes.

The chip rendering on form load and the dropdown items in the picker share `_resolve_reference_fields` on the backend, so the format applied to the dropdown is always the same as the one applied to the chip after a selection — no parity gap.

**Search behaviour** — when the target has `name_search_fields=[...]`, the `/related/<field>` endpoint builds a polish-prefix OR-domain across the listed fields with `ilike` semantics:

| `name_search_fields` | Domain shape |
|---|---|
| (unset) | `[["name", "ilike", "%q%"]]` |
| `["code"]` | `[["code", "ilike", "%q%"]]` |
| `["code", "name"]` | `["|", ["code", "ilike", "%q%"], ["name", "ilike", "%q%"]]` |
| `["code", "name", "hs_reference"]` | `["|", "|", ["code", …], ["name", …], ["hs_reference", …]]` |

User-supplied `domain="..."` filters merge with the search domain via implicit AND — the search clauses come first, followed by the user filter.

**Out of scope** (permanent — not even a Phase 3 item):

- **Per-locale formats** (e.g. `[USD] US Dollar` vs `[USD] Dollar américain`) — i18n of code-bearing master *names* lives in `foundation.i18n`. The format string is single-locale; the `name` field's translation handling is unchanged.
- **Conditional / computed formats** (e.g. `{code|upper}`, branching on length) — keeps the format trivially regex-replaceable on both Python and TypeScript sides. Sourcing data with the right casing is the model's responsibility.
- **Server-side rendering of XML overrides** — the override always renders client-side using `data`. Pushing the override into the HTTP request would require shipping a per-request format string to the search endpoint and would defeat HTTP-cache friendliness.
- **Format strings that traverse relations** (e.g. `{country_id.code}`) — only resolves keys in the immediate row's `data` payload. Authors who need a related field surface it in `name_search_fields` or as a stored compute on the model.

**Implementation map**:
- [`src/ede/core/kernel/decorators.py`](../src/ede/core/kernel/decorators.py) — `display_name_format` kwarg, validation, `_extract_format_keys`, `__ede_display_data_fields__` precompute.
- [`src/ede/core/services/presentation/display_format.py`](../src/ede/core/services/presentation/display_format.py) — shared `render_with_missing` (used by both the controller and the kernel for parity).
- [`src/ede/foundation/presentation/api/controllers.py`](../src/ede/foundation/presentation/api/controllers.py) — `/related/<field>` multi-field search + format rendering + `data` payload.
- [`src/ede/foundation/presentation/models/presentations.py`](../src/ede/foundation/presentation/models/presentations.py) — `_resolve_reference_fields` chip rendering with same precedence.
- [`src/ede/core/services/presentation/dsl/parser.py`](../src/ede/core/services/presentation/dsl/parser.py) — `display_format` attribute extraction on `<field>` in form/list/kanban contexts.
- [`src/frontend/src/workspace/views/fields/formatDisplay.ts`](../src/frontend/src/workspace/views/fields/formatDisplay.ts) — pure helper mirroring Python `format_map` semantics.
- [`src/frontend/src/workspace/views/fields/ReferenceField.tsx`](../src/frontend/src/workspace/views/fields/ReferenceField.tsx) — `renderLabel` precedence helper used by chip + option mapper.

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- A Reference picker with a dynamic-domain bare identifier (e.g. `parent_partner_id=partner_id`) silently drops its tuple when the referenced form field is empty — that is intentional loose semantics, not a bug. If you want strict behaviour (empty input → zero options), gate the picker behind a guard rather than relying on the domain.
- Bare identifiers in `domain="..."` are resolved against the **current record** only, not via traversal. `partner_id.parent_id` will not work — denormalise or expose a computed field on the child record instead.
- `display_name_format` references field names — if you rename a field on a model that declares the format, the decorator validation raises at boot (with the model key in the message); rename the format key in the same diff. The format is not a free-form string; it is part of the model's declared interface.
- `display_format` on a `<field>` references fields the *target* model carries in its `data` payload, not the form's own fields. The payload includes the union `name_search_fields ∪ {record_name_field} ∪ format-keys` of the *target* model — references to fields outside that set silently render as empty string and the override falls back to the server `display_name`. If you need an extra field in the format, add it to the target model's `name_search_fields`.
- Migrating a model from `name`-only to multi-field search by adding `name_search_fields=["code", "name"]` does NOT change the existing on-disk data — it only changes how the picker queries. Existing chips re-render with the new format on the next form load (no migration needed).

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1 ships no Alembic migrations on its own — consumer modules adopting kanban add no schema changes for the MVP. Phase 2 introduces a `sequence Integer` field convention on opt-in models, requiring per-consumer Alembic migrations at that time.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ — kanban inherits the model's existing read/write/create/delete permissions; no kanban-specific RBAC keys | |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Kanban Authoring Guide](foundation-presentation-kanban-guide.md) — **developer-facing** how-to for adding a kanban view to a model (DSL tag reference, drag modes, gotchas, file layout)
- [M2O Display & Search Guide](foundation-presentation-m2o-display-guide.md) — **developer-facing** how-to for `name_search_fields` + `display_name_format` (model-level default + per-`<field>` XML override) for code-bearing masters
- [`foundation.workflow`](foundation-workflow.md) — workflow drag path; `<statusbar/>` precedent for a workflow-aware DSL element
- [`foundation.customization`](foundation-customization.md) — `<DynamicProperties/>` precedent for adding new DSL elements
- [Foundation Apps overview](11-foundation-apps.md) — legacy hand-authored guide (mentions the presentation app at a high level)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-13 (Enhancement 03 🔴 Not Started authored — **Load-Time View Referential Validator**. Moves "missing `ir.action` for inline relational target" / "field-name typo" / "broken `ref=`" from runtime to `ede migrate upgrade` time. Motivated by the Phase 4 qa-automation rollout where operators hit `"No ir.action seeded for qa.usecase.step"` in the browser instead of at upgrade time. New `view_referential_validator.py` service under `src/ede/foundation/presentation/services/` runs after RNG; new `validate_all_views(env) -> ValidationReport` aggregates errors into one report; wired into `ede migrate upgrade` post-data-load step; new `PRESENTATION_VIEW_VALIDATION` foundation setting (`strict` blocks · `warn` logs · `off` skips) shipped to the foundation-settings table; new `ede presentation validate-views [--tenant <name>]` CLI subcommand for CI. RNG kept in place; runtime `presentation.load_action` keeps its existing fallback warning as a last-mile safety net. Implementation deferred to a dedicated session.

Earlier: 2026-05-12 — Enhancement 02 — M2O templated display + multi-field name search — flipped 🔴 → ✅ Delivered same day. Built Capabilities row added; Known Gaps row removed. Hand-authored prose section "M2O templated display + multi-field name search" added with model-declaration syntax, XML-override syntax, wire format, frontend precedence, search-domain shape table, out-of-scope list, implementation map. Three new "things developers commonly get wrong" entries added. 1589 pytest + 494 vitest green; bun run build clean.). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
