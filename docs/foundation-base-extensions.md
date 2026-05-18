# Model & View Extension SDK — Developer Guide

> **Status:** Shipped in [`foundation.base` Phase 2](../roadmap/foundation/base/phase-2/README.md) — 2026-05-18.
> **Roadmap:** [phase-2/README.md](../roadmap/foundation/base/phase-2/README.md)

The Model & View Extension SDK lets a downstream module add fields to a model declared upstream — and patch existing form/list/kanban views — without editing the upstream module's source. EDE's equivalent of Odoo's `_inherit` for fields and `inherit_id` + `<xpath>` for views.

## The three-line example

```python
# foundation.assistant adds an org-level default provider to res.organization,
# without touching the foundation.base file that defines Organization.

from ede.core import api
from ede.core.kernel import fields

@api.extend_model("res.organization")
class OrgExtForAssistant:
    default_ai_assistant_provider_id = fields.Reference(
        "ai.provider.config", on_delete="set null",
        label="Default AI Assistant Provider",
    )
```

That's it. After boot:

- `res.organization` has the `default_ai_assistant_provider_id` field.
- `Organization.__ede_field_specs__` includes it.
- The SqlAlchemy metadata builder generates a column for it.
- A migration in `foundation.assistant` ships the actual `ALTER TABLE` for the column + FK (see "Migrations" below).

The view side uses XML inheritance:

```xml
<!-- foundation.assistant ships its own view file; no edit to res_organization_views.xml -->
<view id="res_organization_form_ai_section" model="res.organization">
    <extend ref="res_organization_form_view">
        <xpath expr=".//section[@string='Status']" position="after">
            <section string="AI Assistant">
                <field name="default_ai_assistant_provider_id"/>
            </section>
        </xpath>
    </extend>
</view>
```

Now `res_organization_form_view` shows an "AI Assistant" section right after the "Status" section — when the assistant module is loaded.

## Why extend instead of lift?

Three bad alternatives that this SDK replaces:

| Approach | Problem |
|---|---|
| Add `default_ai_assistant_provider_id` directly to `Organization` in `foundation.base` | Couples `foundation.base` to a model (`ai.provider.config`) it doesn't know about; breaks the dependency graph. Hit a real regression: 78 pytest cases failed with `NoReferencedTableError` when this was attempted. |
| Create a side-car `assistant.org.preference` model with 1-to-1 FK to org | Balloons the schema, requires UI plumbing to join two records on every form, breaks the "one record per business entity" mental model. |
| Hardcode every variant on every base model (country, tenant, app-specific) | Pollutes the universal definition, drowns the user in fields they don't care about, makes the base unstable over time. |

Field-extension is the standard ERP answer to this — it just hadn't been in EDE before Phase 2.

## When to extend vs lift to platform

Use the EDE Model Placement Test from [CLAUDE.md](../CLAUDE.md):

| Question | Answer | Action |
|---|---|---|
| Would two or more ERP domains (logistics, CRM, HR, finance) need this field with the **same** semantics? | Yes | **Lift to `foundation.base`** — make it a base field. |
| Is the field specific to one country / one tenant / one consumer module? | Yes | **`@api.extend_model`** from that module. |
| Is it specific to a domain (only `logistics`) but multiple modules within that domain need it? | Yes | Add it directly to the domain model in the domain's home file. |

Extensions are for **cross-module additions to platform models**. They're not a replacement for lifting genuinely-universal fields to the platform.

## Scope pluggability

The SDK stores an optional `scope` callable that consumers can use to gate when an extension applies at runtime. The SDK itself doesn't read it — it just attaches it to every contributed field as `field._ede_extension_scope`. Engines that read it land in their respective consumer modules:

| Consumer | Scope predicate | Behavior |
|---|---|---|
| `foundation.l10n` | `country_scope("CC")` | Field shown / required only when the active org's country pack matches |
| `foundation.marketplace` | `tenant_extension_scope("ext_key")` | Field shown only when the tenant has the extension installed |
| `foundation.assistant` | `None` (default) | Field always present — unconditional extension |

The predicate is metadata only in Phase 2. Implementations land alongside the consuming modules.

## Migrations

The Python class declaration creates fields in the ORM at boot. The actual DB column has to ship in an Alembic migration. **The migration lives in the extending module, not the base module:**

```python
# src/ede/foundation/assistant/migrations/versions/X_org_default_provider.py
"""
EDE-App-Key: foundation.assistant
"""
from alembic import op
import sqlalchemy as sa

revision = "X..."
down_revision = "<previous assistant head>"

def upgrade():
    with op.batch_alter_table("res_organization", schema=None) as batch_op:
        batch_op.add_column(sa.Column("default_ai_assistant_provider_id", sa.String(length=36), nullable=True))
        batch_op.create_foreign_key(
            op.f("fk_res_organization_default_ai_assistant_provider_id_ai_provider_config"),
            "ai_provider_config",
            ["default_ai_assistant_provider_id"],
            ["record_uuid"],
            ondelete="set null",
        )

def downgrade():
    with op.batch_alter_table("res_organization", schema=None) as batch_op:
        batch_op.drop_constraint(
            op.f("fk_res_organization_default_ai_assistant_provider_id_ai_provider_config"),
            type_="foreignkey",
        )
        batch_op.drop_column("default_ai_assistant_provider_id")
```

Per the `writing-alembic-migrations` skill: wrap constraint ALTERs in `op.batch_alter_table` for SQLite parity; wrap FK names in `op.f(...)` for cross-dialect deterministic truncation.

## Soft-FK degradation

When test fixtures register only a subset of models (e.g. just `res.organization` without `ai.provider.config`), the SqlAlchemy metadata builder used to crash with `NoReferencedTableError` for any FK whose target table wasn't in the batch. Phase 2's builder now **degrades the FK to a soft reference**: the column is still created, the FK constraint is silently omitted, and the Reference still resolves at runtime via the registry.

This means: writing tests that target the base model **continues to work** even if you've added extension fields whose FK targets live in modules the test doesn't load. No test fixture update required.

## View inheritance — supported xpath positions

`<extend ref="<prefix>.<parent_view_id>">` can contain one or more `<xpath expr="..." position="...">` patches. **The `ref` MUST be qualified** with a dotted prefix identifying the owning module of the base view — e.g. `foundation.res_organization_form_view` (the `res_organization_form_view` lives in `foundation.base`). The SDK strips the leading prefix and looks up the suffix as a registered view_id. If the suffix doesn't resolve, an `InvalidViewDefinition` raises at boot — bare refs are not supported in production code (test fixtures using bare ids are accepted via a literal-hit fallback to keep the harness tight).

```xml
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

| `position` | Effect |
|---|---|
| `inside` (default) | Append patch children to the matched element. |
| `after` | Insert patch children as siblings AFTER the matched element. |
| `before` | Insert patch children as siblings BEFORE the matched element. |
| `replace` | Replace the matched element's children with the patch children (preserves the target element itself + its attributes). |

`expr` is evaluated with Python's `xml.etree.ElementTree` XPath subset — `.//tag[@attr='value']` works; advanced predicates do not.

A patch whose xpath doesn't match the base view raises `InvalidViewDefinition` at compose time. Silent drops would hide authoring bugs.

## What NOT to extend

The base model can opt out:

```python
@api.model("ir.session")
class IrSession(DomainModel):
    __ede_no_extension__ = True
    # ... fields
```

Any `@api.extend_model("ir.session")` declaration then fails at boot with `ExtensionValidationError`. Reserved for kernel-internal models where extensions would be unsafe.

## Boot-time validation

The boot pipeline calls `registry.validate_extensions()` after every app loads. Failures surface immediately:

- `@api.extend_model("foo.bar")` where `foo.bar` doesn't exist → likely a missing manifest `depends` entry.
- `@api.extend_model("foo.bar")` where `foo.bar` is opted out via `__ede_no_extension__=True`.

Field-name collisions (extension declares a field name that already exists on the base or on another extension) raise `ExtensionFieldCollision` at decoration time, before boot completes.

## Inspecting extensions at runtime

| API | Purpose |
|---|---|
| `registry.list_extensions()` | All registered extension classes. |
| `registry.list_extensions(target_model_key)` | Extensions targeting a specific model. |
| `view_registry.list_extensions_for_view(view_id=...)` | View extensions registered against a specific parent view. |
| Settings → Technical → Resources → **Extensions** | Admin UI surface listing all `@api.extend_model` registrations from the live registry mirror. |

The `ir.model.extension` model + table exist as the registry mirror. Population at boot is currently manual (rows seeded by demo data / admin import); a future enhancement will sync rows automatically from `registry.list_extensions()` at every boot.

## Consumer adoption checklist

For a downstream module wanting to extend a base model:

1. **Manifest `depends`** — list the owning module (e.g. `foundation.assistant` depends on `foundation.base` + `foundation.ai`).
2. **Declare the extension class** — anywhere in `src/ede/foundation/<module>/models/` works; the file just needs to be imported during app load.
3. **Ship the migration** — in the consuming module's `migrations/versions/` folder. Chain it off the consuming module's most recent revision.
4. **(Optional) Ship a view extension** — `<extend ref="...">` in an XML file under the consuming module's `views/` folder; list it in the consuming module's `__manifest__.py` `data:` list.
5. **(Optional) Provide a `scope` predicate** — only needed for country-scoped / tenant-scoped / custom-gated extensions.
6. **Tests** — write fixtures that boot just your module + the upstream module; assert the extension fields appear on the target.

That's the whole contract. No other infrastructure pieces are needed.

## Related

- [Phase 2 roadmap](../roadmap/foundation/base/phase-2/README.md) — the full workstream breakdown.
- [`foundation.l10n` roadmap](../roadmap/foundation/l10n/README.md) — the country-scope predicate consumer; Phase 1 lands the `country_scope("CC")` factory on top of this SDK.
- [`writing-alembic-migrations` skill](../.claude/skills/writing-alembic-migrations/SKILL.md) — the patterns extension-column migrations must follow.
- [CLAUDE.md Model Placement Test](../CLAUDE.md) — when to extend vs lift to platform.
