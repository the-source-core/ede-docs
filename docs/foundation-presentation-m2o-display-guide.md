# M2O Display & Search — Developer Guide

**Audience:** module authors who own a code-bearing master, or view authors who consume an M2O picker.
**Sister doc:** [`docs/foundation-presentation.md`](foundation-presentation.md#m2o-templated-display--multi-field-name-search) — full implementation reference.
**Sister doc:** [`docs/foundation-model-naming.md`](foundation-model-naming.md#code-bearing-masters--always-declare-name_search_fields--display_name_format) — naming + when-to-declare guidance.
**Status:** Shipped 2026-05-12 (foundation.presentation Enhancement 02).

---

## TL;DR

If your master has a *code* and a *name*, declare both kwargs on the model and you're done — every picker on every form / list / kanban / search bar in the entire product gains:

- Search by code OR name (typing `USD` matches; typing `Dollar` matches).
- Dropdown items + selected chips render as `[USD] US Dollar`.

```python
@api.model(
    "res.currency",
    record_name="name",
    name_search_fields=["code", "name"],
    display_name_format="[{code}] {name}",
)
class Currency(DomainModel): ...
```

No view-XML edits. Done.

---

## When to use this

### Use it (declare both kwargs on the model)
- Currency, country, language, timezone, unit-of-measure, port, airport.
- Incoterms, commodities, document types, equipment types, facility types.
- CRM stages, lost-reason categories, approval policy sets.
- Any master that ships with a short technical code AND a human-readable name.

### Don't use it
- Pure transactional records (you don't pick a *quote* from a dropdown — you create it).
- Masters with no short code (party records keyed only by display name).
- Masters where the only "code" is an internal UUID slug (defeats readability).

### Use the per-field XML override (`display_format="..."`)
- A specific list cell needs to be compact — `<field name="currency_id" display_format="{code}"/>` shows just `USD`.
- A specific form needs extra context — `<field name="incoterms_id" display_format="[{term_code}] {name} ({version_year})"/>` shows `[FOB] Free on Board (2020)`.
- 95% of consumers will never need this; the model default suffices.

---

## Step 1 — Declare on the model

Open the model file (`src/ede/foundation/<app>/models/<m>.py` or `src/domains/<domain>/<module>/models/<m>.py`).

Add two kwargs to `@api.model(...)`:

```python
@api.model(
    "logistics.incoterm.master",
    description="Incoterm",
    record_name="name",
    name_search_fields=["term_code", "code", "name"],     # NEW
    display_name_format="[{term_code}] {name}",           # NEW
)
class IncotermMaster(DomainModel):
    code = fields.Char(...)              # internal: FOB_2020
    term_code = fields.Char(...)         # user-facing: FOB
    name = fields.Char(...)              # Free on Board
    version_year = fields.Enum(...)
```

### How to choose the fields

**`name_search_fields`** — every field the user might *type* into the picker.
- Each entry becomes an `ilike '%q%'` clause in an OR-domain on the backend.
- Order matters for nothing functional but reads better when the most-queried field comes first.
- Keep the list short (3 or fewer); every field is a separate index hit.
- Auto-injected fields (`record_uuid`, `dbid`, etc.) are valid entries — admit if you really want UUID search.

**`display_name_format`** — what the user *sees* in the dropdown and chip.
- Pick the field combination that uniquely identifies a row at a glance.
- Standard pattern: `"[{code}] {name}"`. Bracketed code anchors the eye; name disambiguates.
- For incoterms: use `term_code` (`FOB`), not the disambiguated `code` (`FOB_2020`) — the user thinks `FOB`.
- For commodity with HS code: use `[{hs_code}] {name}` if HS is the recognised key in your domain; else `[{code}] {name}`.
- For CRM stages: include the workflow stage code so users see `[QUOTE_DRAFT] Draft`.

### Validation kicks in at boot

Both kwargs are validated at decoration time against `__ede_fields__`. Unknown references raise `InvalidHandler` with the model key in the message:

```
InvalidHandler: kernel.decorators.model(res.currency): display_name_format references unknown field '{nonexistent}' — must be a declared field on the model
```

If you rename a field, the format / search list rename in the same diff. The format is part of the model's declared interface, not a free-form string buried in a view file.

---

## Step 2 — Done

Really. Every existing `<field name="currency_id"/>` in every existing view file across the entire product gains the new behaviour on the next page reload.

**No frontend changes needed**. **No view XML changes needed**. **No migration needed** (it's runtime metadata, not schema).

The first-adopter PR (Enhancement 02) added the kwargs to 6 models and shipped zero view-XML changes.

---

## Step 3 (optional) — Per-field XML override

When a specific view needs different formatting from the model default, the `display_format` attribute on `<field>` wins:

```xml
<!-- Quote form: full label "[USD] US Dollar" (model default applies, no override) -->
<field name="currency_id"/>

<!-- Quote line list cell (narrow column): just "USD" -->
<field name="currency_id" display_format="{code}"/>

<!-- Customs filing form: extra context -->
<field name="incoterms_id" display_format="[{term_code}] {name} ({version_year})"/>

<!-- Reporting form: name only, no code -->
<field name="currency_id" display_format="{name}"/>
```

The override is a first-class `<field>` attribute (sibling to `domain`, `widget`, `string`, `optional`). Works in form, list, and kanban view contexts.

### What the override has access to

The override interpolates against the *target model's* `data` payload — which carries the union `name_search_fields ∪ {record_name_field} ∪ format-keys` of the target model.

If the override references a field that's NOT in this union, the format silently falls back to the server-rendered `display_name`. To extend the data payload, add the field to the target model's `name_search_fields`.

---

## How it works under the hood (skim if you ever debug)

### Search path — `GET /api/web/<action>/related/<field>?q=USD`

1. Backend resolves the target model from the field descriptor (Reference's `reference_model_key`).
2. Reads `__ede_name_search_fields__` from the target model class.
3. Builds a polish-prefix OR-domain via `PresentationController._build_name_search_domain`:

   | `name_search_fields` | Domain shape |
   |---|---|
   | (unset) | `[["name", "ilike", "%q%"]]` |
   | `["code"]` | `[["code", "ilike", "%q%"]]` |
   | `["code", "name"]` | `["|", ["code", "ilike", "%q%"], ["name", "ilike", "%q%"]]` |
   | `["code", "name", "hs_reference"]` | `["|", "|", ["code", …], ["name", …], ["hs_reference", …]]` |

4. Merges with any user-supplied `domain="..."` filter via implicit AND.
5. Dispatches `ede.search`, gets back the rows.
6. For each row, renders `display_name` via shared helper `render_with_missing` (`str.format_map` + missing-key tolerance).
7. Ships per-row `data: {field: value}` payload covering the union fields, IFF the format is set.

### Selected-chip path — `GET /api/web/<action>/records/<uuid>`

`PresentationKernel._resolve_reference_fields` uses the **same** render helper, so the chip and the dropdown always look identical. No parity gap.

### Frontend — `ReferenceField.tsx`

Precedence (used by both the chip in `renderViewMode` and each dropdown option in the search widget):

1. XML `display_format` from the view (if set) AND the option carries `data` → interpolate via `formatDisplay(template, data)`.
2. Else use server-rendered `display_name` (already encodes the model default).
3. Else use `name`.
4. Else empty.

The helper is pure and free of React state — see [`src/frontend/src/workspace/views/fields/formatDisplay.ts`](../src/frontend/src/workspace/views/fields/formatDisplay.ts).

---

## Format string syntax (both sides)

Mirrors Python `str.format_map` semantics, implemented identically in Python and TypeScript.

| Syntax | Meaning |
|---|---|
| `{key}` | Substitutes `data["key"]`. Missing keys, `null`, and `undefined` all render as empty string. |
| `{{` and `}}` | Literal `{` and `}` characters. |
| `{key.attr}` | Collapses to `{key}` (only the head identifier is honored). |
| `{0}`, `{1}` | Numeric/positional placeholders are NOT supported (field-name keys only). |
| `{key|filter}` | Filter syntax NOT supported. Source the data with the right shape on the model. |

### Examples

```python
"[{code}] {name}"                            # → "[USD] US Dollar"
"[{stage_code}] {stage_name}"                # → "[DRAFT] Draft"
"[{term_code}] {name} ({version_year})"      # → "[FOB] Free on Board (2020)"
"{name}"                                      # → "US Dollar"  (just the name)
"{code}"                                      # → "USD"        (just the code)
"{name} ({code})"                            # → "US Dollar (USD)"
```

---

## Out of scope (won't ever ship)

These were considered and explicitly excluded — don't ask:

- **Per-locale formats** — i18n of master names lives in `foundation.i18n`. The format string is single-locale.
- **Conditional / computed formats** — keep the format trivially regex-replaceable on both sides.
- **Server-side rendering of XML overrides** — defeats HTTP-cache friendliness.
- **Format strings that traverse relations** (`{country_id.code}`) — only resolves immediate-row keys. If you need a related field, surface it in `name_search_fields` or as a stored compute on the target model.

---

## Common gotchas

- **The format references a field that doesn't exist** → boot raises `InvalidHandler`. Fix the format or the field name.
- **The XML override references a field NOT in the target model's data payload** → silently falls back to server `display_name`. Extend the target's `name_search_fields` to include the field.
- **An existing test does an exact-equality match on a kanban field's render plan and now fails** → add `"display_format": None` to the expected dict. Every kanban field render plan now carries the field (default `None`).
- **A migration is required to add `display_name_format` to a model** → no, it's runtime metadata only. Ship the decorator change, restart, done.
- **Multi-field search hits a field that's not indexed** → backend uses `ilike`, no full-text index needed. For very large masters (10k+ rows) consider a trigram or partial index, but the existing `name`-only index already covers the common path.

---

## Adding a new code-bearing master — checklist

When you add a brand-new master with a code/name pair, declare both kwargs from day one:

- [ ] `record_name="name"` (or whichever field is the human-readable label)
- [ ] `name_search_fields=["code", "name"]` (or however your code-bearing fields are named)
- [ ] `display_name_format="[{code}] {name}"` (or your domain's preferred shape)
- [ ] Verify boot: `python -c "from ede.runtime import bootstrap_environment; bootstrap_environment()"` — should not raise.
- [ ] (For seeded masters) Ship the demo CSV with realistic codes — see the `adding-demo-data` skill.
- [ ] (For consumers) Existing `<field name="<your_field>"/>` declarations gain the behaviour automatically — verify in browser.

---

## Reference index

- [Implementation reference](foundation-presentation.md#m2o-templated-display--multi-field-name-search) — full doc with implementation map.
- [Naming guidance](foundation-model-naming.md#code-bearing-masters--always-declare-name_search_fields--display_name_format) — when to use, when not to use.
- [Enhancement 02 spec](../roadmap/foundation/presentation/enhancements/02-m2o-display-format-and-multi-field-search.md) — design decisions + rejected alternatives.
- [`formatDisplay`](../src/frontend/src/workspace/views/fields/formatDisplay.ts) — pure helper used by the frontend.
- [`render_with_missing`](../src/ede/core/services/presentation/display_format.py) — shared backend helper used by the controller and the kernel.
