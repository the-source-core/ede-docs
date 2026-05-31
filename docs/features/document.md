# Documents & Reports

Render PDFs from a markup template. You author a document in DML (a small XML grammar), bind values at render time, and the engine produces format-specific bytes through a two-stage pipeline. Every render is audited.

```python
pdf = env.dispatch(Command("ede.document.render", payload={
    "key": "blog.post.summary",
    "params": {"post": post_dict, "author": author_dict},
    "format": "pdf",
}))["bytes"]
```

Templates support variables, computed formulas, conditionals, loops, includes, inheritance, and data-driven tables — so one template covers many records without per-record code.

---

## What you get

-   **`ir.document.template`** — a named DML template (with inheritance and a document-type tag).
-   **`ir.document.type`** — a taxonomy that lets an action pick the right template by type.
-   **`ir.document.render.audit`** — append-only log of every render (success or failure, size, applied extensions).
-   **`ir.document.config`** — per-organization header/footer chrome binding.
-   **`ede.document.render`** command + **`DocumentService.render_by_key(...)`** — render entrypoints.
-   **`ir.action.report`** + **`/api/reports/*`** — wire a print action to a model and surface it in a print menu.
-   **`@api.report_engine`** — register a rendering engine; the document engine registers under `engine_key="dre"`.
-   **DML grammar** — `<var>`, `<field>`, `<page>`, `<if>`, `<for-each>`, `<include>`, `<rows>`, `<bind-chrome>`.

## How to use it

### Author a template

A template's body is DML. Declare variables, substitute placeholders, and format fields inline:

```xml
<DocumentTemplate>
    <var name="vat_rate" type="decimal" value="0.18"/>
    <var name="gross" type="decimal" formula="net * (1 + vat_rate)"/>
    <pages>
        <page id="main">
            <heading><field name="post.title" format="title"/></heading>
            <paragraph>By <field name="author.name"/> on <field name="post.published_on" format="date:YYYY-MM-DD"/></paragraph>
            <paragraph>Total: <field name="gross" format="currency:USD" default="0.00"/></paragraph>
        </page>
    </pages>
</DocumentTemplate>
```

`<field>` accepts a `format` (e.g. `currency:USD`, `date:YYYY-MM-DD`, `percent:2`, `upper`) and a `default` for missing values.

### Loop, branch, and include

```xml
<page id="row" for="items" as="item" if="item.active">
    <paragraph>{item.name} — {item.qty}</paragraph>
</page>

<if test="show_terms">
    <include src="blog.common.terms"/>
</if>
```

`<page for=>` clones a page per item; `<if test=>` drops a page or block when its predicate is falsy; `<include>` inlines another template by key.

### Compute variables with formulas

`<var formula="...">` evaluates against a safe expression evaluator with a whitelisted function set — math (`abs`, `round`, `min`, `max`), aggregates (`sum`, `avg`, `count`), logic (`iff`, `coalesce`, `is_empty`), dates (`today`, `add_days`, `days_between`), and formatting (`currency`, `percent`). No attribute access, imports, or arbitrary calls.

### Pull rows from a metric

`<rows datasource="...">` runs a registered metric at render time and repeats its children once per returned row:

```xml
<rows datasource="blog.monthly_views" as="row">
    <paragraph>{row.month}: {row.views}</paragraph>
</rows>
```

The `datasource` is a metric key — see [Datasets](dataset.md) for how metrics are defined.

### Reference page numbers

Stage 2 substitutes render-time tokens after the page count is known — `{PageNumber}`, `{TotalPages}`, `{LoopIndex}`, `{LoopCount}`, `{LoopKey}` — so headers and footers can show "Page 2 of 7" without the template knowing the count up front.

### Wire a print action

Create an `ir.action.report` bound to a model and the document engine, then list and run it:

```python
# list the print actions available for a record
GET  /api/reports/_list?model=blog.post&record_id=<uuid>
# render one
POST /api/reports/<action_key>/run?format=pdf
```

Point the action at an `ir.document.type` and the engine selects the highest-priority matching template — so localizations or per-organization variants resolve automatically without changing the caller.

### Configure header/footer chrome

`<bind-chrome from-config="default_header_template_id"/>` resolves at render time against the active organization's `ir.document.config` row (falling back to the system default), so each company's letterhead is applied without editing the template.

## Configuration

The document engine has no global feature flags — its configuration is data:

| Where | What it controls |
|---|---|
| `ir.document.config` (Settings → Documents) | Per-organization default header / footer templates. |
| `strict` flag on `render_by_key(...)` | When set, an unbound placeholder without a `default` raises instead of rendering blank. |
| `ir.document.type` + template `priority` / `domain` | Which template an action resolves to for a given record. |

## Reference

-   Source: `src/ede/foundation/document/`
-   Render: `ede.document.render` command, or `DocumentService.render_by_key(key, params, format=...)`.
-   Related: [Datasets](dataset.md) (metric datasources), [Storage](storage.md) (persisting rendered output), [Communication (Chatter)](communication.md) (attach a render to a record).
