# Presentation (View DSL)

Declare list, form, kanban, and search views in XML. The React web client renders them — no per-view component code, no JSX.

```xml
<!-- src/domains/blog/post/views/post_views.xml -->
<list model="blog.post">
    <field name="title"/>
    <field name="published"/>
    <field name="created_at_utc" widget="datetime"/>
</list>

<form model="blog.post">
    <header>
        <button name="publish" string="Publish" type="object"
                attrs="{'invisible': [('published','=',True)]}"/>
        <field name="state" widget="statusbar"/>
    </header>
    <sheet>
        <group>
            <field name="title"/>
            <field name="published"/>
        </group>
        <notebook>
            <page string="Body">
                <field name="body" widget="text"/>
            </page>
        </notebook>
    </sheet>
    <chatter/>
</form>
```

Declare the file in `__manifest__.py` under `data:` — the loader registers the views at boot and the web client picks them up.

---

## What you get

-   **Four view types**: `<list>`, `<form>`, `<kanban>`, `<search>`.
-   **DSL elements**: `<field>`, `<header>`, `<sheet>`, `<group>`, `<notebook>`/`<page>`, `<button>`, `<chatter>`, `<statusbar>`.
-   **Widgets** for fields: `text`, `email`, `phone`, `currency`, `monetary`, `datetime`, `date`, `boolean`, `selection`, `many2one`, `one2many`, `many2many`, `progress`, `radio`, `tags`, `image`, `binary`, `html`, `statusbar`.
-   **Domain filter attrs**: `attrs="{'invisible': [...], 'readonly': [...]}"` to conditionally hide / lock fields.
-   **Action wiring**: `<button name="cmd_name" type="object">` calls a command on the record.
-   **Auto-generation** for missing views: the engine produces a default list / form when a model has none registered.
-   **`ViewRegistry`** — runtime view lookup keyed by model + view type.
-   **`DslParser`** — parses XML into a `RenderPlan` dict consumed by the React client.

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
<search model="blog.post">
    <filter name="published"   string="Published"
            domain="[('published','=',True)]"/>
    <filter name="my_drafts"   string="My Drafts"
            domain="[('author_id','=',$principal.user_id), ('published','=',False)]"/>
    <group_by name="author_id" string="By Author"/>
</search>
```

The web client renders filter pills + a groupby dropdown on every list and kanban automatically.

### Add a kanban view

```xml
<kanban model="blog.post" group_by="state">
    <field name="title"/>
    <field name="author_id"/>
    <templates>
        <t t-name="card">
            <div class="card">
                <strong><field name="title"/></strong>
                <small><field name="author_id"/></small>
            </div>
        </t>
    </templates>
</kanban>
```

### Conditionally hide a field

```xml
<field name="approver_id"
       attrs="{'invisible': [('state','!=','pending')]}"/>
```

The expression is evaluated client-side against the current form values.

### Override another module's view

Use `<extend ref="...">` to add / replace nodes:

```xml
<extend ref="base.res_partner_form">
    <xpath expr="//field[@name='email']" position="after">
        <field name="loyalty_tier"/>
    </xpath>
</extend>
```

## Reference

-   Source: `src/ede/foundation/presentation/`
-   `DslParser`: `src/ede/core/services/presentation/dsl/parser.py`
-   `ViewRegistry`: `src/ede/core/services/presentation/view_registry.py`
-   Architecture: [Presentation DSL](../10-presentation-dsl.md).
