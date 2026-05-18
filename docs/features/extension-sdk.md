# Model & View Extension SDK

Add fields to a model — and patch its views — from a downstream module without editing the upstream source file. One decorator declares the new fields; one DSL element declares the view patch; the registry merges everything at boot.

```python
# In foundation.assistant — extending res.organization owned by foundation.base
from ede.core import api, fields


@api.extend_model("res.organization")
class OrgExtensionForAssistant:
    default_ai_assistant_provider_id = fields.Reference(
        "ai.provider.config",
        on_delete="set null",
        label="Default AI Assistant Provider",
    )
```

```xml
<!-- In foundation.assistant — adding an "AI Assistant" section to the org form -->
<view id="res_organization_form_ai_section" model="res.organization">
    <extend ref="foundation.res_organization_form_view">
        <xpath expr=".//section[@string='Status']" position="after">
            <section string="AI Assistant">
                <field name="default_ai_assistant_provider_id"/>
            </section>
        </xpath>
    </extend>
</view>
```

The base model's source file stays universal; the downstream module owns its additions.

---

## What you get

-   **`@api.extend_model("target.key", scope=None)`** — decorator that turns any class with `fields.*` attributes into a passive field-contribution against a base model. The class is **not** a `DomainModel`; the registry merges its fields into the target at registration time.
-   **`<extend ref="parent.view.id">`** DSL element — wraps one or more `<xpath expr="..." position="..."/>` patches. The view renderer splices the patches into the base view's RenderPlan; the base view file stays untouched.
-   **Four xpath positions** — `after`, `before`, `inside`, `replace`. Any other value fails at load time with a `DslParseError`.
-   **`ir.model.extension`** — read-only mirror model. One row per registered extension class, populated at boot. Admins introspect "what does each module add to what" through **Settings → Customization → Extensions**.
-   **Soft-FK metadata builder** — `fields.Reference("comodel.key", ...)` declared on an extension class still resolves correctly even if the consumer module is loaded after the target. The metadata builder records the FK as a soft reference and binds it once both ends are present.
-   **Boot-time validator** — rejects extensions whose target model does not exist, whose FK targets are unresolvable, or whose decorated class declares no fields.
-   **Scope hookpoint** — `@api.extend_model("...", scope=<callable>)`. The default `scope=None` applies the extension unconditionally. Consumer modules can ship scope factories (per-country, per-tenant, per-organization) that gate where extensions take effect. The SDK stores the callable verbatim and exposes it for consumer engines to read; no built-in factory ships in Phase 1.
-   **Per-app migration story** — extension columns live in Alembic revisions owned by the *contributing* module, not the target. `ede migrate generate --app foundation.assistant` autogens a revision adding `res_organization.default_ai_assistant_provider_id` under the assistant's `version_locations`.

## How to use it

### Extend a model with new fields

Declare a class — *not* a `DomainModel` subclass — with `fields.*` attributes:

```python
from ede.core import api, fields


@api.extend_model("blog.post")
class PostExtensionForAnalytics:
    last_indexed_at = fields.DateTime(
        label="Last Indexed At",
        index=True,
    )
    seo_score = fields.Integer(
        label="SEO Score",
        default=0,
    )
```

At decoration time the registry copies `last_indexed_at` and `seo_score` into the target model's `__ede_fields__` and `__ede_field_specs__`. `env.models["blog.post"].browse(uuid).last_indexed_at` works from anywhere as if the field had been declared on the original `Post` class.

If the consumer module's import order happens *before* the target model loads, the registry remembers the contribution and merges on the target's eventual registration. The merge is idempotent — re-importing the extension class doesn't double-add fields.

Generate the migration from the extension's owning module:

```bash
ede migrate generate --app foundation.analytics -m "blog.post: SEO indexing fields"
```

The autogen places the `ALTER TABLE blog_post ADD COLUMN ...` lines under `foundation.analytics`'s `version_locations`, so the target module's revision history stays clean.

### Patch a view

```xml
<view id="blog_post_form_analytics_section" model="blog.post">
    <extend ref="blog.blog_post_form_view">
        <xpath expr=".//notebook" position="inside">
            <page string="Analytics">
                <field name="last_indexed_at"/>
                <field name="seo_score"/>
            </page>
        </xpath>
    </extend>
</view>
```

`ref` points at the parent view's `view_id` (the loader resolves it through `ir.data.reference`, so the canonical name is `<owner_module>.<view_id>`). The renderer parses the base view into a RenderPlan, applies each `<xpath>` patch in order, and ships the resulting plan to the React web client. The base view file never changes.

A single `<extend>` may contain any number of `<xpath>` patches against the same parent.

### Pick an xpath position

The position attribute decides how a patch's children splice into the base view at the matched node:

| `position` | Effect |
|---|---|
| `inside` | Append children as the last children of the matched node. |
| `before` | Insert children immediately before the matched node. |
| `after` | Insert children immediately after the matched node. |
| `replace` | Remove the matched node and put the patch's children in its place. |

Any other value fails parsing at module load with a clear error — typos surface at boot, never at runtime.

### Inspect what's registered

Navigate to **Settings → Customization → Extensions** to see the list of registered model extensions, who owns each one, the target model, and the scope kind. The view is read-only — the source of truth is the decorator on the contributing class. Run `ede migrate upgrade` after deploying a new contributor and the mirror picks up the new row on next boot.

Programmatically:

```python
extensions = env.models["ir.model.extension"].search([
    ("target_model_key", "=", "res.organization"),
])
for ext in extensions:
    print(ext.owner_module_key, "→", ext.extension_class_path)
```

### Use the scope hookpoint

A consumer module that wants its fields to apply conditionally passes a `scope` callable:

```python
def production_orgs_only(env, record=None):
    return record is None or record.environment == "production"


@api.extend_model("res.organization", scope=production_orgs_only)
class OrgExtensionForProductionMetrics:
    deploy_pipeline_url = fields.Char(label="Deploy Pipeline URL")
```

The SDK stores `scope` on the extension class and surfaces it through `ir.model.extension.scope_kind`. It is the *consumer* module's responsibility to wire an engine that reads the callable — required-field gating, workflow guard rules, form-view show/hide logic. The SDK itself does not enforce the scope; it provides the hookpoint and the mirror metadata so engines built on top can.

## How it composes with other features

-   [Customization (Properties)](customization.md) — `@api.extend_model` is the static analogue to per-tenant `properties`: extensions add real DDL columns at deploy time, properties add tenant-scoped JSON keys at run time. Use the SDK for cross-module fixed schemas; use properties for tenant-tunable schemas.
-   [Presentation (View DSL)](presentation.md) — `<extend>` is a regular element of the same parser; the view loader picks up extension views from any module's `data:` manifest list.
-   [AI Assistant](assistant.md) — the AI Assistant's per-org provider preference is shipped by an `@api.extend_model("res.organization")` + a paired `<extend ref="...">` view patch; the assistant module never touches `res.organization`'s source file.

## Reference

-   Decorator: `src/ede/core/kernel/extensions.py` (re-exported as `api.extend_model`)
-   Registry merge step: `src/ede/core/registry.py` (`register_extension`)
-   Metadata builder (soft-FK handling): `src/ede/core/adapters/persistence/sqlalchemy/metadata_builder.py`
-   DSL parser branch: `src/ede/core/services/presentation/dsl/parser.py` (`_parse_extend_view`)
-   View renderer composition: `src/ede/core/services/presentation/view_registry.py`
-   Mirror model: `src/ede/foundation/base/models/ir_model_extension.py`
-   Admin views: `src/ede/foundation/base/views/ir_model_extension_views.xml`
-   Worked example (real consumer): `src/ede/foundation/assistant/models/organization_extension.py` + `src/ede/foundation/assistant/views/res_organization_ai_extension.xml`
