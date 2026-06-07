# Customization (Properties)

Add tenant-defined fields to any record without writing code or running a schema migration. Administrators declare properties from the Settings UI; values land in a JSON column on the host record and surface alongside the static fields.

```python
@api.model("blog.post", custom_properties=True)
class Post(DomainModel):
    title = fields.Char(required=True)
    body = fields.Text()
```

The `custom_properties=True` flag auto-injects a `properties` JSONB column on the table and a validator hook on the model. From that point on, an administrator can attach custom fields to `blog.post` from **Settings → Customizations → Custom Fields** — no Python change, no Alembic migration, no redeploy.

---

## What you get

-   **`@api.model(..., custom_properties=True)`** — opt-in flag that auto-injects the `properties` JSON column plus a pre-create / pre-update validator hook.
-   **`ir.model.property.definition`** — per-tenant property schema: target model, key, label, type, default, optional comodel.
-   **`ir.model.property.selection`** — option rows for `selection`-typed properties.
-   **`RecordSet.get_property(key)` / `set_property(key, value)`** — typed accessors that read and write the `properties` JSON.
-   **`<DynamicProperties/>` DSL element** — drop into a form view to render the editable Properties tab.
-   **`ir.model` / `ir.model.field` / `ir.model.field.selection`** — read-only persistent mirror of the runtime registry, refreshed on every `ede migrate upgrade`.
-   **`env.ref("base.model_res_partner")`** — resolver from a stable external ID to the live record. Use it in XML data files or runtime code to reference any seeded row by name instead of UUID.
-   **Settings → Customizations → Custom Fields** — admin list/form UI for managing definitions.

## How to use it

### Mark a model as customizable

```python
from ede.core import api, fields
from ede.core.kernel.model import DomainModel

@api.model("blog.post", custom_properties=True)
class Post(DomainModel):
    title = fields.Char(required=True)
    body = fields.Text()
```

Generate and apply an Alembic migration — `ede migrate generate -m "blog.post: enable custom_properties"` adds a `blog_post.properties` JSONB column defaulting to `{}`. The model is now ready to receive tenant-defined properties.

### Define a property (Settings UI)

Navigate to **Settings → Customizations → Custom Fields → New**. Pick the target model from the `model_id` selector (the picker is filtered to models that opted into `custom_properties=True`), enter a key, label, and type, then save. The property is live immediately — every form view that includes `<DynamicProperties/>` shows the new field on next reload.

### Define a property (XML data file)

```xml
<record id="prop_blog_post_topic_tags" model="ir.model.property.definition">
    <field name="model_id" ref="base.model_blog_post"/>
    <field name="key">topic_tags</field>
    <field name="label">Topic Tags</field>
    <field name="property_type">many2many</field>
    <field name="comodel_id" ref="base.model_blog_tag"/>
</record>
```

`model_id` and `comodel_id` are many-to-one references resolved by external ID. The registry sync seeds those external IDs automatically — see *Resolve a registry entry by external ID* below.

### Read and write property values

```python
post = env.models["blog.post"].browse(post_uuid)

# Read
tags = post.get_property("topic_tags")

# Write — dispatches ede.update, so events, hooks, and audit logs all see it
post.set_property("topic_tags", [tag_uuid_1, tag_uuid_2])
```

`set_property` dispatches `ede.update` like any other field write — see [Commands and events](../tutorial/04-commands-and-events.md). The validator hook fires automatically before the write commits.

### Render the Properties tab on a form view

```xml
<form>
    <sheet>
        <group>
            <field name="title"/>
            <field name="body"/>
        </group>
        <notebook>
            <page string="Custom Properties">
                <DynamicProperties/>
            </page>
        </notebook>
    </sheet>
</form>
```

`<DynamicProperties/>` is a self-closing leaf. The presentation engine inlines a `propertiesSchema` block into the action's RenderPlan, and the React `PropertiesEditor` widget reuses the existing field-editor registry to render the right widget per type.

### Resolve a registry entry by external ID

```python
post_model = env.ref("base.model_blog_post")          # → ir.model record
title_field = env.ref("base.field_blog_post__title")  # → ir.model.field record
```

`env.ref()` is the read path for any row registered through `ir.data.reference`. The registry sync seeds canonical external IDs on every `ede migrate upgrade`:

| Pattern | Resolves to |
|---|---|
| `<app_key>.model_<key>` | `ir.model` row for the model |
| `<app_key>.field_<key>__<name>` | `ir.model.field` row for the field |
| `<app_key>.selection_<key>__<field>__<option>` | `ir.model.field.selection` row for an Enum option |

`<key>` is the model key with dots replaced by underscores (`res.partner` → `res_partner`). Pass `required=True` to raise instead of returning `None` on a miss.

## Property types

| `property_type` | Stored as | Notes |
|---|---|---|
| `char` | string | Single-line text. |
| `integer` | int | |
| `decimal` | decimal | Full precision preserved in JSON. |
| `boolean` | bool | |
| `date` | ISO date string | |
| `datetime` | ISO datetime, UTC | |
| `selection` | option key (string) | Declare options as `ir.model.property.selection` rows linked back to the definition. |
| `reference` | target `record_uuid` | Requires a `comodel_id` pointing at the target `ir.model`. Single-valued (many-to-one). |
| `many2many` | list of target `record_uuid`s | Same `comodel_id` requirement; multi-valued. |

The validator hook coerces every incoming value to the declared type and, for `reference` / `many2many`, checks that every UUID actually exists. A bad UUID raises a 422 with field path `properties.<key>`.

`property_type` cannot be changed once a definition is saved — both the UI and the backend hook reject the update. Delete and recreate the definition instead.

## How it composes with other features

-   [Form views](../tutorial/03-views-and-web-client.md) — drop `<DynamicProperties/>` into any notebook page.
-   [Commands & events](../tutorial/04-commands-and-events.md) — `set_property` dispatches `ede.update`, so all hooks and events fire normally.
-   [Permissions](security.md) — RBAC permissions on `ir.model.property.definition` control who can manage the schema.

## CLI

| Flag | Command | Purpose |
|---|---|---|
| `--no-registry-sync` | `ede migrate upgrade` | Skip the `ir.model*` mirror sync. Mirrors the existing `--no-cleanup` flag; useful in CI or when debugging upgrade-time issues. |

## Reference

-   Models: `src/ede/foundation/base/models/ir_model_property.py`, `src/ede/foundation/base/models/ir_model.py`
-   Registry sync: `src/ede/foundation/base/services/registry_sync.py`
-   Validator hook: `src/ede/foundation/base/services/property_validator.py`
-   DSL parser branch: `src/ede/core/services/presentation/dsl/parser.py` (`DynamicProperties` element)
-   Frontend widget: planned in `src/frontend/src/managers/PropertiesEditor.tsx` (DynamicProperties — placeholder slot is wired in `FormSheet.tsx`)
-   Admin menus: `src/ede/foundation/base/data/customization_menus.xml`
-   RBAC seed: `src/ede/foundation/base/data/ir.rbac.permission.csv`
