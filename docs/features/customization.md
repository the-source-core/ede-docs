# Customization (Properties)

Add user-defined fields to any record without writing code or migrations. Administrators declare properties through the Settings UI; the values land in `ir.model.property` and surface on the record's form view alongside the static fields.

```python
# Read a property value off any record
post = env.models["blog.post"].browse(post_uuid)
extra = post.properties.get("seo_keywords", default=[])
```

Property values are typed (`string`, `integer`, `decimal`, `boolean`, `date`, `datetime`, `many2one`, `selection`, `tag`) and validated against the property definition.

---

## What you get

-   **`ir.model.property`** — definitions: which model, field name, type, label, optional constraints.
-   **`ir.model.property.value`** — per-record values.
-   **`record.properties`** descriptor — dict-like accessor on any `DomainModel`.
-   **Settings UI** — "Properties" tab on the per-model configuration page.
-   **Form-view auto-render** — properties appear in a dedicated "Custom" section on the record form.
-   **Search & filter** — properties are searchable in `SearchPanel` like any static field.
-   **Lifecycle hook** — properties participate in `pre.*` / `post.*` hooks (`cmd.record.properties` is mutable in pre-create).

## How to use it

### Define a property (XML)

```xml
<record id="prop_blog_post_seo_keywords" model="ir.model.property">
    <field name="model_key">blog.post</field>
    <field name="name">seo_keywords</field>
    <field name="property_type">tag</field>
    <field name="label">SEO Keywords</field>
    <field name="help">Comma-separated keywords for search engines.</field>
</record>
```

### Define a property (Settings UI)

Navigate to **Settings → Models → Blog Post → Properties → New**, fill in the type and label, save. The property is live immediately — no server restart.

### Read a property value

```python
keywords = post.properties.get("seo_keywords", default=[])
```

### Write a property value

```python
post.properties["seo_keywords"] = ["domain-driven-design", "python", "fastapi"]
```

The setter dispatches `ede.update` so events, hooks, and audit logs see the change as a regular write.

### Search by a property

```python
posts = env.models["blog.post"].search([
    ("properties.seo_keywords", "in", ["python"]),
])
```

The domain-filter DSL accepts `properties.<name>` paths the same way it accepts dotted-path field paths.

## Property types

| `property_type` | Stored as | Notes |
|---|---|---|
| `string` | `Char` | Single-line. |
| `text` | `Text` | Multi-line. |
| `integer` | `Integer` | |
| `decimal` | `Decimal(20, 6)` | Adjustable precision via constraint. |
| `boolean` | `Boolean` | |
| `date` | `Date` | |
| `datetime` | `DateTime` | UTC. |
| `selection` | `Char` | With a `selection_values` constraint. |
| `many2one` | `Reference` | With a `target_model_key` constraint. |
| `tag` | `JSON` array | Free-form labels. |

## How it composes with other features

-   **[Search](presentation.md)** — properties are first-class in the search panel.
-   **[Datasets & Metrics](dataset.md)** — properties can be columns in a dataset.
-   **[Security](security.md)** — per-property RBAC permissions are supported.

## Reference

| Concept | Where it lives |
|---|---|
| `ir.model.property` | `src/ede/foundation/customization/models/property.py` |
| Property descriptor | `src/ede/foundation/customization/runtime/properties.py` |
| HTTP API | `src/ede/foundation/customization/api/` |
