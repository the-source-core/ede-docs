# `<related/>` — `Related ▾` Form-Toolbar Dropdown

**Status:** ✅ Delivered 2026-06-02 (Enhancement 06)
**Roadmap:** [06-related-records-dropdown.md](../roadmap/foundation/presentation/enhancements/06-related-records-dropdown.md)
**Substrate:** `foundation.presentation` + `foundation.base` (`env.ref`)

> A declarative dropdown that surfaces every record cross-referenced from the
> current record — converted leads, linked quotes, attached tags, child
> shipments — in one consistent slot in the form toolbar. Two authoring
> modes; reload-safe URLs; consistent visual language driven by the field's
> target model.

---

## TL;DR

```xml
<view id="crm_inquiry_form" model="crm.inquiry">
    <FormView>
        <header>
            <related>
                <!-- Mode A — action-less. Parser derives target model from the
                     field spec. M2O drills straight to the linked record. -->
                <field name="converted_lead_id"/>

                <!-- Mode B — action-bound. Reload-safe via the action path. -->
                <field name="tag_ids" action="sales_crm.action_tags_from_inquiry"/>
            </related>
            ...
        </header>
        ...
    </FormView>
</view>
```

That's the whole DSL surface. Everything below is the *why* and the
operating envelope.

---

## DSL grammar

`<related>` is a child of `<header>` on a `<FormView>`. It holds one or more
`<field/>` chips; each chip becomes a row in the dropdown:

| Attribute     | Required | Notes |
|---------------|----------|-------|
| `name`        | **yes**  | Relational field on the current model (Reference / Many2Many / One2Many). |
| `action`      | no       | Dotted xml_id of a bespoke `ir.action`. Omit for Mode A. |
| `icon`        | no       | Lucide icon name; falls back to menu-entry icon → default per arity. |
| `string`      | no       | Overrides the chip label. Else derived from the field's `label=` on the model. |
| `hide-empty`  | no       | `"1"` (default) hides the chip when count=0. `"0"` keeps it visible as `(0)`. |
| `visible-when` / `hidden-when` | no | Predicate-gating per Enhancement 04. |

The `<related>` element itself also accepts the gate-attrs — set them on the
parent to gate the whole group (e.g. by role).

---

## Mode A vs Mode B — when to pick which

| | **Mode A — action-less** | **Mode B — action-bound** |
|---|---|---|
| **DSL** | `<field name="X"/>` | `<field name="X" action="module.xml_id"/>` |
| **Backend round-trip** | None for M2O (parent's value carries everything). | One `related_counts` call per chip per parent record. |
| **Count semantics for multi** | Length of `record[field]` array. | `count(domain)` server-side. |
| **Drill URL** | M2O: `/<targetModelKey>/form/<recordRef>` (action-less record frame). | List/Record: `/<actionPath>/<viewMode>[/<recordRef>]`. |
| **Reload-safe** | Yes for M2O (model key + uuid in URL). | Yes (action path in URL). |
| **Custom filter** | No — the chip lists/counts the parent's M2M values verbatim. | Yes — the action's `domain` template controls what's counted/listed. |
| **Use it when…** | The chip's job is "show me what's linked to this record" with no custom view, no filter beyond the relation itself. | You want a domain template (`active_id`, `$active_record.<field>`), a specific list/kanban view choice (`default_view`), or non-relational filtering of related records. |

**Rule of thumb:** start with Mode A. Upgrade a chip to Mode B only when you
need to ship a bespoke list view (column set, sort, search filters) for the
drilled-into records, or a non-trivial domain filter.

---

## Mode B — bespoke `ir.action` shape

A "related action" is a normal `ir.action` row that just isn't menu-bound:

```xml
<!-- src/domains/logistics/sales_crm/data/related_actions.xml -->
<record id="sales_crm.action_tags_from_inquiry" model="ir.action">
    <field name="name">Tags</field>
    <field name="path">inquiry-tags</field>           <!-- URL slug -->
    <field name="model_key">logistics.tag</field>
    <field name="default_view">list</field>
    <field name="available_views">list,form</field>
    <field name="domain">[["record_uuid", "in", "$active_record.tag_ids"]]</field>
</record>
```

Required field reference:

| Field             | Purpose |
|-------------------|---------|
| `id` (xml_id)     | What `<field action="…"/>` references. |
| `path`            | URL slug used in the drill URL (`/inquiry-tags/list`). |
| `model_key`       | Target model for the drilled list / form. |
| `default_view`    | `list` / `kanban` / `calendar`. |
| `available_views` | CSV; what view types the chip's drill can switch between. |
| `domain`          | JSON-encoded domain template. May reference `active_id` and/or `$active_record.<field>` tokens (see next section). |

The action is **not** mounted under any `ir.menu`. It exists purely as a
drill target — that's the convention. Conventionally co-locate the action's
XML next to the views that consume it; the manifest's `data` list order
ensures the action seeds *before* its consumer view loads.

---

## Domain templates — `active_id` and `$active_record.<field>`

The action's `domain` is a JSON-encoded list with two substitution
placeholders, both resolved at chip-render / drill-render time:

### `active_id`

The parent record's `record_uuid`. Used for natural FK back-references:

```xml
<!-- Inquiry → its converted leads. crm.lead has a FK `inquiry_id`. -->
<field name="domain">[["inquiry_id", "=", active_id]]</field>
```

Substitution: bare identifier in value position, quoted to a JSON string.
Regex enforces no quotes / underscores / dots around the identifier.

### `$active_record.<field>`

JSON-serialized value of `<field>` on the parent record. Used when the target
has **no inverse field** pointing back at the parent — most commonly M2M.

```xml
<!-- Inquiry → its tags. logistics.tag has no `inquiry_ids` field. -->
<field name="domain">[["record_uuid", "in", "$active_record.tag_ids"]]</field>
```

Resolution:

- **Reference** (`{id, display_name}`) → bare UUID string.
- **M2M / O2M array of refs** → list of UUIDs.
- **Scalar** → as-is.

The substituting regex consumes the surrounding double-quotes so a JSON
array doesn't end up wrapped in quotes. You can omit the quotes (rarely
useful — won't be valid JSON pre-substitution) and it still works.

Both substitutions run on the server in `related_counts` (the count query)
**and** on the client in `ActionManager.drillDomain` (the list-drill query),
twin implementations that must stay in step.

---

## URL grammar

The workspace URL is `/wc/<orgCode>/<key1>/<viewMode1>[/<recordRef1>]/<key2>/<viewMode2>[/<recordRef2>]/…` — each frame is a triple. Concretely:

| Frame | URL fragment |
|-------|--------------|
| Root action, list view | `/crm-inquiries/list` |
| …with a record open | `/crm-inquiries/list/<inq>` |
| Drilled into a list (Mode B) | `/inquiry-tags/list` |
| …with a row open (drilled-list form) | `/inquiry-tags/list/<tag>` |
| Drilled into an M2O record (Mode A, no action) | `/crm.lead/form/<lead>` |

Worked example — Inquiry's tag drill opened to a single tag:

```
/wc/myorg/crm-inquiries/list/<inquiry>/inquiry-tags/list/<tag>
```

Parsing rule (see [stack.ts](../src/frontend/src/core/navigation/stack.ts)
`parseNavigationStack`): walking left to right, after each `key/viewMode`
pair, peek the segment *two ahead* — if it's a known viewMode keyword
(`list|kanban|calendar|form`), the segment in between is the next frame's
key and this frame has no recordRef. Otherwise it's this frame's recordRef.

---

## Dropdown UI — what the user sees

Inside the open menu, chips partition into two groups separated by a
divider:

- **Singles first** (M2O — `record[field]` is `{id, display_name}` or null).
- **Multis after** (M2M / O2M — `record[field]` is an array).

Each row is `[marker] | TITLE \n SUBTITLE?`. Marker is a 22×22 square with
slightly rounded corners; both variants reuse the existing `--app-color-N`
palette tokens (0..10) hashed deterministically from the chip's target
model via [`colorForModel(modelKey)`](../src/frontend/src/managers/colorForModel.ts) — so every `crm.lead` chip
across the app wears the same colour.

| Variant | Background | Icon colour | Title | Subtitle |
|---------|------------|-------------|-------|----------|
| M2O (`--tinted`) | Transparent | Palette colour | Linked record's `display_name` | Field label (`string=` ?? `field_label` from spec, uppercase, muted `--fs-2xs`) |
| Multi (`--filled`) | Palette colour | White | Bare bold count | Field label (same chain) |

Default icons (override per-chip with `icon="..."`):

- M2O default: `bookmark` (visually "this record is bookmarked from here").
- Multi default: `git-branch` (one-to-many split).

The trigger button shares `.btn.form-toolbar__actions` chrome with Print ▾
and Actions ▾ (chevron rotates on open), with an accent-orange tint layer
so the slot is visually distinct in the toolbar.

---

## Server-resolved chip metadata

The DSL parser does not have model-registry access, so the chip dict it
emits has placeholder slots that `load_action` fills via a post-parse pass
([`_enrich_related_chips`](../src/ede/foundation/presentation/models/presentations.py)):

| Slot              | Source |
|-------------------|--------|
| `target_model_key` | M2O → `reference_model_key`; M2M/O2M → `target_model_key` |
| `field_label`     | `spec.label or field_name.replace("_", " ").title()` |

The frontend reads these via the wire shape and uses them for:

- **Mode A drill** — `target_model_key` is the only thing needed to build the
  drill URL (parent's record value carries the linked id/display_name).
- **Chip subtitle** — `field_label` keeps the dropdown from reading
  "tag_ids"; it shows "Tags".

---

## Click behaviour

| Chip type | Outcome |
|-----------|---------|
| M2O (Mode A) | Push a record frame: `/<targetModelKey>/form/<linked_id>` |
| Multi count=1 (Mode B) | Short-circuit to the lone record: `/<actionPath>/form/<lone_uuid>` |
| Multi count>1 (Mode B) | Push a list-drill frame: `/<actionPath>/list` |
| Multi (Mode A) | Push a list-drill via `targetModelKey` — list view of the target, no custom filter beyond the parent's M2M ids |

When the user is *already* on a drilled list and clicks a row, the
`OPEN_RECORD` handler mutates the top list-drill frame's `recordRef` in
place (no new frame, no stack collapse) so the breadcrumb chain survives.

---

## Adding `<related/>` to your form — checklist

1. Decide **Mode A or Mode B** per the table above.
2. (Mode B only) Author an `<record model="ir.action">` with `path`, `model_key`, `domain`. Place it in a manifest data file that loads *before* the view file (or in the same file, above the view).
3. Add a `<related>` block under `<header>` with one `<field/>` per chip.
4. Run `ede migrate upgrade -t <tenant>` so the action record (if new) loads.
5. Restart the backend so the parser + load_action enrichment run against the latest views.

If the chip uses `$active_record.<field>` in its domain, the **target's
table must not need an inverse field** — the substitution gives you the
parent's values inline. Don't add inverse fields just to make a chip work;
prefer `$active_record` for cross-domain M2M edges.

---

## Backend command reference

### `presentation.related_counts`

`POST /api/web/_related-counts`

**Payload:**
```json
{
  "queries": [
    {
      "action_key":       "sales_crm.action_tags_from_inquiry",
      "active_id":        "<parent uuid>",
      "active_model_key": "crm.inquiry"
    },
    ...
  ]
}
```

`active_model_key` is **required** when the action's domain references
`$active_record.<field>` tokens (so the backend can fetch the parent record's
field values). Optional otherwise.

**Response:**
```json
{
  "results": [
    {
      "action_key": "sales_crm.action_tags_from_inquiry",
      "model_key":  "logistics.tag",
      "path":       "inquiry-tags",
      "count":      2,
      "lone_uuid":  null
    }
  ]
}
```

`lone_uuid` is set only when `count == 1` — drives the count-1 short-circuit
on the frontend (drill straight to the record, skip the list step).

---

## Test patterns

- `tests/foundation/test_env_ref.py` — `env.ref` happy path, dangling
  refs, malformed input, raise/None semantics, cross-model resolution.
- `tests/foundation/test_dsl_parser_related.py` — `<related>` parsing.
- Frontend list/form views can stub `env.ref()` via the `_StubRecordSet` +
  `_make_ref_stub` pattern in `test_presentation_record_api.py` — no need to
  seed real `ir.data.reference` rows.

---

## Things developers commonly get wrong

- **Reaching for Mode B when Mode A works.** If the chip just lists the
  parent's M2M, `<field name="tag_ids"/>` is enough. Mode B is for custom
  domain / view / RBAC.
- **Adding an inverse field on the target.** `$active_record.<field>` exists
  precisely to avoid this — polluting upstream models with consumer-aware
  fields creates ownership drift.
- **Quoting `$active_record.<field>` differently.** Inside JSON-encoded
  domain strings the token must be `"$active_record.X"` (quoted to be valid
  JSON pre-substitution). The substituting regex consumes the surrounding
  quotes so the JSON-encoded replacement value isn't re-wrapped in quotes.
- **Authoring a related action without a `path`.** The action loader and
  Mode B drill URL both rely on the path slug; an empty `path` makes the
  drill URL unresolvable.
- **Putting `<related>` outside `<header>`.** The DSL parser only walks
  `<related>` blocks inside `<header>` — the form toolbar is the only render
  slot today.

---

## See also

- [foundation-presentation.md](foundation-presentation.md) — module-level doc.
- [Enhancement 06 roadmap source](../roadmap/foundation/presentation/enhancements/06-related-records-dropdown.md) — design rationale + delivery log.
- [foundation-presentation-predicate-gates.md](foundation-presentation-predicate-gates.md) — `visible-when` / `hidden-when` you can put on `<related>` or its children.
- [Foundation Model Naming](foundation-model-naming.md) — dotted xml_id conventions.

*Last updated: 2026-06-02.*
