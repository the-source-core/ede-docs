# DB-Backed Views, Per-Property Field Binding, and AI-Driven Customization — Design

**Date:** 2026-06-11
**Status:** Draft for review
**Scope:** Combined A + B + C (single spec, sequenced delivery)
**Affected modules:** `foundation.base`, `foundation.presentation`, `foundation.customization`, `foundation.assistant`, `src/ede/core/services/presentation/*`, `src/frontend/*`

---

## 1. Context & Problem

Today the EDE Framework has two disconnected facts:

- **Custom properties are tenant-defined data.** Models opt in with `@api.model(..., custom_properties=True)`, which injects a `properties` JSON column. Tenants declare ad-hoc fields as rows in `ir.model.property.definition` (+ `ir.model.property.selection`). `PropertiesValidator` coerces/FK-checks those values on every write via `pre.{model}.create|update` hooks. This half is complete and tested.
- **Views are developer-authored static XML, file-only.** `ViewRegistry` indexes `<ede><view>` files at boot; `<extend ref="...">` patches are merged at request time by `compose_view_xml()`. **No view content lives in the database.** `ir.action`, `ir.action.view` (a `Char` view_id, explicitly *not* a reference), `ir.menu`, and `ir.data.reference` are DB-persisted, but the *view layout itself is never persisted*.

The two halves meet at a single DSL element, `<DynamicProperties/>`, which is an **all-or-nothing lump**: every custom property is meant to render in one fixed slot — and today not even that, because the frontend renderer (`FormSheet.tsx`) explicitly `return null`s it as "RESERVED."

### The three things this design changes

1. **Per-property placement.** Retire `<DynamicProperties/>`. Place one property per field via `<field name="<bag>:<key>"/>` (e.g. `properties:gst_treatment`), anywhere in the form, like any normal field. The binding is generic over a **property-bag registry** so future JSONB bags (`l10n_properties`, …) reuse the same machinery.
2. **DB-backed view store (`ir.application.view`).** Every view becomes a database record: a **primary** view per (model, type) plus **extension** views (parent → child) carrying xpath patches, each with **scope** (global / organization) and **role** gating. Views compose from the DB at request time.
3. **AI-driven customization.** The assistant, aware of the current action/view, proposes new custom fields and (after confirmation) writes them as **scoped extension rows** + property definitions, which render live.

### Decisions already locked (via brainstorming)

| Decision | Choice |
|---|---|
| View-store approach | **B2 — Full DB migration.** All view content moves into the DB; XML files become the seed; the read path reads from the DB. |
| Spec scope | **One combined spec** for A + B + C. |
| Ownership / sync rule | **Runtime = extension rows only.** Primary views are developer-owned and **re-synced from their XML files on every `migrate upgrade`** (the DB primary row is overwritten from file). Tenant/AI customization may ONLY create extension rows — never edit a primary. |
| `ir.action.view` binding | **Convert `view_id` Char → Reference** to `ir.application.view` (with a backfill migration). |
| Field binding syntax | **`<field name="<bag>:<key>"/>` where `<bag>` is the JSONB column name** (`properties:gst_treatment`, `l10n_properties:gst_number`). Generic over a **property-bag registry**, not a hardcoded prefix. |
| Bag column name | **Keep `properties`** for the custom-properties bag (no rename). Future `l10n_properties` bag parallels it. |
| Bag scope this spec | **Generic seam + the `properties` bag only.** `l10n_properties` is the documented proving second consumer, built later by `foundation.l10n`. |
| AI write boundary | **Assistant proposes; it never writes.** Confirm-gated → real writes go through the command bus with RBAC. |

---

## 2. Goals & Non-Goals

### Goals
- A custom property can be placed at any position in a form via `<field name="<bag>:<key>"/>` (e.g. `properties:gst_treatment`), rendering with the correct typed widget and saving into the named JSONB bag.
- The binding is **bag-agnostic**: a property-bag registry lets `foundation.l10n` later add an `l10n_properties` bag with zero changes to the shared parse / FieldDef / frontend / save code.
- All views (primary + extension) are records in `ir.application.view`; composition reads from the DB.
- Extension views carry xpath patches, a scope (global/org), and role gating; composition filters by the request principal's org + roles.
- The assistant can add a custom field + place it (as an org-scoped or global extension) through a confirm-gated, RBAC-checked write path — without ever directly mutating data itself.
- Developer authoring stays in XML files; `migrate upgrade` upserts them into primary rows (system-owned, re-synced).

### Non-Goals
- No visual drag-and-drop form designer in this spec (the AI proposal flow + admin records are the editing surface).
- No cross-tenant view sharing (tenants are DB-isolated; org-scoping is *within* a tenant).
- No change to the `PropertiesValidator` coercion/validation contract — it already handles `properties` correctly.
- No removal of the `@api.model(custom_properties=True)` opt-in mechanism.

---

## 3. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│ AUTHORING (developers)                                                 │
│  <ede><view id="..."> XML files in module data/                        │
└───────────────┬──────────────────────────────────────────────────────┘
                │ ede migrate upgrade  →  view-sync loader (upsert)
                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ ir.application.view  (DB, per tenant)                                  │
│   primary rows  (owner=system, re-synced from file, NEVER runtime-edited) │
│   extension rows (owner=system from file  OR  owner=user from runtime/AI)  │
│     parent_id → primary | scope=global|organization | organization_id | roles │
└───────────────┬──────────────────────────────────────────────────────┘
                │ request: compose(view_id, principal)
                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ View Composer (core/foundation.presentation)                          │
│   load primary arch → apply matching extension xpatches (scope+roles)  │
│   → DslParser.parse_to_render_plan → strip predicate gates             │
│   cache by (view_id, org, principal-role-fingerprint)                  │
└───────────────┬──────────────────────────────────────────────────────┘
                │  RenderPlan (+ synthesized property FieldDefs)
                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ React webclient                                                        │
│   <property name="bag:key"/> → widget registry → record[bag][key]        │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Phase A — `<property>` Element + Property-Bag Registry

### A.0 Property-bag registry (the framework primitive)
A new registry — `registry.property_bags` (lowest reusable layer, in `src/ede/core/registry.py`) — maps each JSONB **bag column** to its provider + validator:

```
PropertyBag = {
    "column":           "properties",                 # the JSONB column on the host model
    "schema_provider":  <callable(model_key) -> [FieldDef-like defs]>,
    "validator":        <PropertiesValidator-style coercer>,
}
```

- The `@api.model(..., custom_properties=True)` decorator registers the **`properties`** bag: provider = `ir.model.property.definition` lookup (existing `_load_schema`), validator = existing `PropertiesValidator`. This is the **only bag implemented in this spec.**
- Future: `foundation.l10n` calls the same registration API to add the **`l10n_properties`** bag with its own provider (country-specific definitions) + validator — **no change** to any shared parse / FieldDef / frontend / save code.
- Boot-time validation: a `<property name="bag:key"/>` referencing an unregistered bag fails view validation with a clear error.

### A.1 DSL & parser
- **New element:** `<property name="<bag>:<key>"/>` — e.g. `<property name="properties:gst_treatment"/>`. `<bag>` is a registered bag column; `<key>` matches `^[a-z][a-z0-9_]*$`. Confirmed conflict-free against the parser, RNG, frontend element union, and widget registry.
- The element **mirrors `<field>`'s attribute set**: `widget`, `readonly`, `required`, `invisible`, `visible-when`/`hidden-when`, `nolabel`, etc.
- **`view.rng`:** add the `<property>` element definition (attributes referencing the shared `field-attrs` + `gate-attrs` groups); **remove** the `<DynamicProperties>` definition.
- **`parser.py`:** add a `tag == "property"` branch emitting a distinct node `{ "type": "property", "bag": "<bag>", "key": "<key>", ...field-style attrs }`. Remove the `DynamicProperties` branch (lines ~480–491). The distinct `type: "property"` is the explicit signal the frontend renderer branches on.

### A.2 FieldDef synthesis (backend)
- When `load_action` walks a view and hits a `property` node, it resolves the bag from `registry.property_bags`, calls the bag's `schema_provider(model_key)`, finds the matching `key`, and synthesizes a **FieldDef** (`name="<bag>:<key>"`, `type`, `label`, `required`, `help`, `selection`, `comodel_key`). Appended to `field_defs` so the existing widget registry renders it with zero special-casing.
- Each referenced bag **column** is added to the fetch field list whenever a `property` node for it is present (replaces the `has_dynamic_properties` logic).
- A `property` node whose key is unknown/inactive in its bag: render-time **skip with a logged warning** (don't hard-fail the view).

### A.3 Frontend value mapping
- `mappers.ts`: map the `property` node → a typed `FormPropertyElement { type: "property"; bag; key }`.
- `FormSheet.tsx`: add a `case "property"` renderer. It looks up the synthesized FieldDef by `"<bag>:<key>"`, reads the current value from `record[bag]?.[key]`, and on edit calls `onWriteField` which routes the value into the `bag` sub-object of the submit payload.
- Widget resolution is by the synthesized FieldDef `type` — reuses existing read/edit widgets in `widgets/builtins.tsx`; reference/many2many reuse relation widgets via `comodel_key`.
- The renderer can carry a subtle "custom field" affordance since the node type is explicit — optional, not required.

### A.4 Validation & save (unchanged contract)
- The bag's registered validator runs in the existing `pre.{model}.create|update` hook path. For the `properties` bag this is the current `PropertiesValidator` — **no behavioural change**. A second bag simply registers its own validator on the same seam.

### A.5 Retire `<DynamicProperties/>`
- Remove: `view.rng` element, `parser.py` branch, `presentations.py` detection + `properties_schema` inlining (`_has_properties_element`, `_load_properties_schema`), frontend `FormDynamicPropertiesElement` type + mapper case + `FormSheet.tsx` `return null` branch.
- Convert the only live usage (`pricing_rate_form.xml`) to explicit `<property name="properties:<key>"/>` elements.
- Update tests: `test_dsl_dynamic_properties.py` → rewritten as `test_dsl_property_element.py`; keep `test_property_validator.py` as-is.

---

## 5. Phase B — `ir.application.view` (Full DB-Backed View Store)

### B.1 Model (placement: `foundation.base`, alongside `ir.action` / `ir.menu`)

`ir.application.view` (field names snake_case; model key dotted per convention):

| Field | Type | Notes |
|---|---|---|
| `view_key` | Char, required, index | Logical view id (was the XML `id`); unique per (tenant). |
| `model_id` | Reference(`ir.model`), index, nullable | Bound model (null for extension-only/abstract). **Reference, not a Char** — mirrors `ir.model.property.definition.model_id`; resolve `model_id.model_key` for composition. |
| `view_type` | Enum | `list` / `form` / `kanban` / `search` / `clientaction`. |
| `mode` | Enum | `primary` / `extension`. |
| `parent_id` | Reference(`ir.application.view`), on_delete=cascade | Extension → its base. The `parent="module.xml_id"` attribute on `<view>` resolves to this at sync; runtime/AI rows set it directly. Null for primary. |
| `arch` | Text | DSL XML body: full `<view>` for primary; the `<xpath expr="..." position="...">` patch fragment(s) for extension. |
| `priority` | Integer, default=10 | Primary resolution + extension apply order. |
| `scope` | Enum | `global` / `organization`. |
| `organization_id` | Reference(`res.organization`), nullable | Set when `scope=organization`. |
| `role_keys` | JSON (list) | Optional role gate — extension applies only if principal has a listed role. |
| `owner` | Enum | `system` (file-synced) / `user` (runtime/AI). |
| `active` | Boolean, default=True | Soft-archive. |

**Invariants (enforced by hooks):**
- `owner=system` rows are write-protected against runtime mutation (only the view-sync loader may upsert them).
- Runtime/AI creation is **always** `mode=extension`, `owner=user`. A pre-create hook rejects `mode=primary` + `owner=user`.
- `scope=organization` requires `organization_id`; `scope=global` requires it null.
- Extension `arch` xpath must resolve against the current parent at create time (else reject — see C.3).

### B.2 Authoring grammar (unified `<view>` wrapper)

The `<extend ref="...">` nested element is **retired**. Both primary and extension views use the **same `<view>` wrapper**; an extension is simply a `<view>` that carries a `parent="module.xml_id"` attribute and holds `<xpath>` patches directly:

```xml
<!-- Primary -->
<ede version="1.0">
    <view id="logistics_shipment_form" model="logistics.shipment">
        <FormView> ... </FormView>
    </view>
</ede>

<!-- Extension (no <extend> element; parent is an attribute, xpath is a direct child) -->
<ede version="1.0">
    <view id="shipment_form_tax_props" model="logistics.shipment"
          parent="logistics_shipments.logistics_shipment_form">
        <xpath expr="//section[@string='General']" position="after">
            <section string="Tax">
                <property name="properties:gst_treatment"/>
            </section>
        </xpath>
    </view>
</ede>
```

- **`parent` attribute** → resolves to `parent_id`; its presence sets `mode=extension`. Absence → `mode=primary`.
- **xpath uses `//`, not `.//`** — the leading `.` is not required. The composer roots the search at the view's `arch` tree, so `//section[...]` is relative to it.
- `view.rng` + `parser.py` updated: add the `parent` attribute on `<view>`, allow `<xpath>` as a direct `<view>` child, remove the `<extend>` element. The **5 existing `<extend ref>` files migrate** to this form as part of Phase 4 (assistant · sales_crm · booking · equipment ×2).

### B.3 Sync
- A **view-sync service** (mirrors `RegistrySync` for `ir.model`) runs during `migrate upgrade`: for each file-authored view, **upsert** an `ir.application.view` row keyed by its xml_id (via `ir.data.reference`), `owner=system`. `parent` attr → resolved `parent_id`; `model` attr → resolved `model_id` (via `ir.model`); the `<view>` **body** (the view-type tree for primary, the `<xpath>` fragments for extension) → `arch`. Primary rows and file-authored extensions are **overwritten from file** every migrate. `owner=user` rows are never touched.

### B.4 Composition (read path moves to DB)
- New **ViewComposer** (in `core/services/presentation` or `foundation.presentation`): `compose(view_key, principal) -> RenderPlan`.
  1. Load the primary `arch` from `ir.application.view` (resolve by `view_key`, or by (`model_id`, `view_type`, lowest priority) when an action doesn't pin one).
  2. Gather extension rows where `parent_id = primary` AND (`scope=global` OR `organization_id = principal.org`) AND (`role_keys` empty OR principal has a listed role), ordered by `(owner system-first, priority, dbid)`.
  3. Apply each extension's xpath patch to the base tree (reuse existing `_apply_xpath_patch` logic, lifted from `ViewRegistry`). xpath is rooted at the view's `arch` tree, so `//`-prefixed exprs resolve relative to it.
  4. `DslParser.parse_to_render_plan(...)` → strip predicate gates (existing pass).
- **Caching:** by `(view_key, organization_id, principal-role-fingerprint)`, reusing the existing predicate fingerprint-cache invalidation from `using-predicate-view-gates`.
- `ViewRegistry`'s file-indexing role is reduced to **feeding the sync loader**; the request-time read path no longer touches files.

### B.5 `ir.action.view` binding
- `view_id` Char → `view_ref` Reference(`ir.application.view`) (FK to `record_uuid`).
- **Backfill migration** (generator-produced): match existing Char `view_id` values to the synced `ir.application.view` rows and populate the reference.

### B.6 Admin surface
- List + form views for `ir.application.view` under Settings → Customizations, so admins can inspect/manage extension rows (primaries shown read-only).

---

## 6. Phase C — AI-Driven Customization

### C.1 Context awareness
- The assistant already knows the current action/view via `ActionContext` (see `extending-assistant-skill-packs`). A new **customization skill pack** exposes the host `model_key`, current `view_key`, and the existing property schema to the model.

### C.2 Propose, never write
- `foundation.ai` is **read-only by contract.** The pack's tools are **proposers**: they emit a structured proposal op, e.g.
  ```json
  {
    "op": "propose_custom_field",
    "model_key": "logistics.shipment",
    "field": { "key": "risk_band", "label": "Risk Band", "type": "selection",
               "options": [{"key":"low","label":"Low"}, ...], "required": false },
    "placement": { "view_key": "logistics_shipment_form", "xpath": "//page[@name='other']", "position": "inside" },
    "scope": "organization"
  }
  ```
- The frontend renders a **confirm card** (✨ marker, consistent with view-intent ops). On confirm, the frontend dispatches real commands through the **command bus with RBAC**:
  1. `ede.create` on `ir.model.property.definition` (+ selection rows).
  2. `ede.create` on `ir.application.view` (`mode=extension`, `owner=user`, `parent_id`, `scope`, `organization_id`, `arch` = the `<xpath>` fragment placing `<property name="properties:risk_band"/>`).
- The assistant itself issues **no writes** — it only returns the proposal.

### C.3 xpath safety
- A pre-create hook on `ir.application.view` (extension mode) **applies the patch against the current parent arch in a dry run**; unresolvable xpath → `ValueError` (rejected before persist).
- At compose time, an extension whose xpath no longer resolves (base drifted) is **skipped with a logged warning**, never crashing the view.

---

## 7. Migration Plan (all generator-produced — no hand-authored Alembic)

1. **`foundation.base`:** new `ir_application_view` table (model → `ede migrate generate`). New columns on `ir_action_view` for `view_ref`; drop `view_id` Char after backfill. Backfill step matches Char → UUID.
2. **Per opt-in host model:** no schema change for Phase A (the `properties` column already exists on opt-in models).
3. **Demo data:** `ir.application.view` ships records → per the CLAUDE.md rule, a `demo/` file + usecase doc is required for the customization module before any ✅ flip (use the `adding-demo-data` skill).

---

## 8. Testing Strategy

- **Phase A:** parser emits `property` nodes; bag-registry resolution; FieldDef synthesis (each property type → correct widget); save round-trips into `properties`; unknown-key skip; `pricing_rate` form conversion; `<DynamicProperties>` fully removed (grep-clean).
- **Phase B:** view-sync upsert (primary re-sync overwrites file changes; `owner=user` untouched); composer applies extensions filtered by scope + roles; cache keying/invalidation; `ir.action.view` reference backfill; ownership-invariant hooks (reject `primary`+`user`, reject org-scope w/o org).
- **Phase C:** proposal op shape; confirm → command-bus writes with RBAC; xpath dry-run rejection at create; graceful skip at render; assistant issues zero direct writes (read-only contract preserved).
- **E2E:** define a property via AI proposal → confirm → field renders at the placed position → value saves. Extend `test_customization_property.py`.
- Run via `./run_tests.sh`; frontend via `bun run build` + `bun run test`.

---

## 9. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Moving the view read path from files to DB destabilizes every rendered screen. | Keep `DslParser` + xpath-merge logic identical; only the *source* of arch changes. Land the sync loader + composer behind the existing compose entry point; full-suite gate. |
| Many existing tests use the in-memory `ViewRegistry` with files. | Provide a test fixture that runs the sync loader into a test DB; migrate tests incrementally. (Tracked as explicit work in the plan.) |
| AI/tenant xpath breaks when a developer changes a primary view. | Create-time dry-run validation + render-time graceful skip + logged warning. |
| Composition cost per request (DB reads + merge + parse). | Cache by `(view_key, org, role-fingerprint)`, reuse predicate fingerprint invalidation. |
| Module dependency direction (`foundation.presentation` ↔ `foundation.base`). | Confirm during planning: model in `foundation.base`, composer in `foundation.presentation`/core; verify no cycle. |
| Scope/role gating overlaps existing predicate gates + RBAC. | View-level scope decides *which extensions apply*; element-level `visible-when`/RBAC decides *within* a composed view. Keep the two layers distinct and documented. |

---

## 10. Delivery Sequencing

Although specced together, implementation lands A → B → C (B and C depend on A's field node; C depends on B's extension store). This work touches `src/ede/foundation/**`, so implementation proceeds under **`roadmap-driven-delivery`** (roadmap entry per phase), with **demo data** shipped for the customization records before any status flip.

---

## 11. Open Questions (to resolve during planning)

- `arch` column type: `Text` vs `JSON` (raw XML string vs pre-parsed). Leaning `Text` (raw DSL) to keep authoring and diffing simple.
- Whether `view_key` must be globally unique or unique-per-(model,type); affects action resolution.
- Exact role-gate semantics: any-of vs all-of for `role_keys`.
- Whether search/kanban/list views also accept `<property>` elements in Phase A or form-only first.
