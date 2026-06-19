# Presentation (View DSL)

Declare list, form, kanban, and search views in XML. The React web client renders them — no per-view component code, no JSX.

```xml
<!-- src/domains/blog/post/views/post_views.xml -->
<ede version="1.0">
    <view id="blog_post_list" model="blog.post">
        <ListView order_by="created_at_utc desc">
            <field name="title"/>
            <field name="published"/>
            <field name="created_at_utc" widget="datetime"/>
        </ListView>
    </view>

    <view id="blog_post_form" model="blog.post">
        <FormView>
            <header>
                <button name="publish" command="blog.post.publish" label="Publish"
                        icon="check" invisible="published == true"/>
                <field name="status" widget="statusbar"/>
            </header>
            <sheet>
                <section string="Post" cols="2">
                    <field name="title" widget="record_title"/>
                    <field name="published"/>
                </section>
                <notebook>
                    <page label="Body">
                        <field name="body" widget="text"/>
                    </page>
                </notebook>
            </sheet>
            <chatter/>
        </FormView>
    </view>
</ede>
```

Declare the file in `__manifest__.py` under `data:` — the loader registers the views at boot and the web client picks them up.

---

## What you get

-   **Root + wrapper** — every view file is an `<ede version="1.0">` document; each view is a `<view id="..." model="...">` wrapping one renderable element.
-   **Renderable elements** — `<ListView>`, `<FormView>`, `<KanbanView>`, `<SearchView>`, `<ClientActionView>`.
-   **Form layout** — `<header>`, `<sheet>`, `<section string="..." cols="N">`, `<notebook>` / `<page label="...">`, `<field>`, `<button>`, `<statusbar>`, `<chatter>`.
-   **Field widgets** — pass `widget="..."` on a `<field>` (e.g. `text`, `datetime`, `date`, `boolean`, `currency`, `many2one`, `record_title`, `chips`, `statusbar`, `domain_selector`, `image`).
-   **Conditional display** — bare boolean expressions: `invisible="published == true"` on fields, sections, and buttons.
-   **Action buttons** — `<button name="..." command="..." label="..." icon="..."/>` dispatches a command on the record.
-   **Search** — `<filter>`, `<groupby>`, and searchable `<field>` inside `<SearchView>`.
-   **View extension** — `<view parent="...">` + `<xpath>` to patch another module's view.
-   **`ViewRegistry`** + **`DslParser`** — runtime view lookup and XML-to-`RenderPlan` parsing.

## How to use it

### Register a view in the manifest

```python
# src/domains/blog/post/__manifest__.py
{
    "name": "Blog Post",
    "depends": ["base"],
    "data": [
        "views/post_views.xml",
    ],
}
```

### Add a search view

```xml
<view id="blog_post_search" model="blog.post">
    <SearchView>
        <field name="title"/>
        <filter name="published" string="Published" domain='[["published","=",true]]'/>
        <filter name="drafts" string="Drafts" domain='[["published","=",false]]' is_default="true"/>
        <groupby name="by_author" string="By Author" groups="author_id"/>
    </SearchView>
</view>
```

The web client renders filter pills + a groupby dropdown on every list and kanban automatically.

### Add a kanban view

Kanban cards are composed from dedicated elements — `<KanbanRibbon>` for a status stripe, `<KanbanTitle>` / `<KanbanSubtitle>` / `<KanbanRow>` / `<KanbanFooter>` for the card body:

```xml
<view id="blog_post_kanban" model="blog.post">
    <KanbanView default_group_by="status" order_by="created_at_utc desc">
        <KanbanCard>
            <KanbanRibbon field="status"/>
            <KanbanTitle>
                <field name="title"/>
            </KanbanTitle>
            <KanbanSubtitle>
                <field name="author_id" widget="many2one"/>
            </KanbanSubtitle>
            <KanbanFooter>
                <field name="created_at_utc" widget="date"/>
            </KanbanFooter>
        </KanbanCard>
    </KanbanView>
</view>
```

Set `default_group_by` to the column field. Add `on_drag="workflow"` to make card drag run workflow transitions, and `quick_create="true"` for inline record creation.

### Conditionally hide a field

```xml
<field name="approver_id" invisible="status != 'pending'"/>
```

The expression is evaluated client-side against the current record values — no `attrs` dict, just the boolean expression.

### Override another module's view

An extension view carries its own `id`, names the view it patches with `parent`, and applies `<xpath>` patches:

```xml
<view id="blog_partner_form_loyalty" parent="base.res_partner_form_view" model="res.partner">
    <xpath expr="//field[@name='email']" position="after">
        <field name="loyalty_tier"/>
    </xpath>
</view>
```

`position` is one of `after`, `before`, `inside`, `replace`. See [Model & View Extension SDK](extension-sdk.md) for the full cross-module pattern.

## Reference

-   Source: `src/ede/foundation/presentation/`
-   `DslParser`: `src/ede/core/services/presentation/dsl/parser.py`
-   `ViewRegistry`: `src/ede/core/services/presentation/view_registry.py`
-   Grammar (RelaxNG): `src/ede/core/services/presentation/dsl/schemas/view.rng`
-   Architecture: [Presentation DSL](../10-presentation-dsl.md).
