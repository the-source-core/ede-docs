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
<view id="res_organization_form_ai_section" model="res.organization"
      parent="foundation.res_organization_form_view">
    <xpath expr=".//section[@string='Status']" position="after">
        <section string="AI Assistant">
            <field name="default_ai_assistant_provider_id"/>
        </section>
    </xpath>
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

## Where the column comes from

The Python class declaration creates the field in the ORM at boot. The DB column arrives on the
next `ede migrate upgrade`: the schema comparison sees a column the models declare and the
database lacks, and adds it. **There is nothing to write and nothing to commit alongside the
extension class** — see [Runtime Schema Upgrades](runtime-schema-upgrades.md).

The important property for this SDK is that the column is **attributed to the extending module**
in exactly the way the declaration is. `foundation.assistant` declares
`default_ai_assistant_provider_id` on `res.organization`; the comparison reads the merged
`MetaData`, so the column and its foreign key appear because the *consuming* module declared them.
`foundation.base` neither knows nor needs to know.

Ordering is a non-problem here, and deliberately so. The comparison creates every table first and
applies **all** foreign keys afterwards, so an extension's FK into a table another module owns
cannot be applied before that table exists — regardless of which module is processed first, or
whether the target module is even active in this deployment.

```python
# src/ede/foundation/assistant/models/organization_ext.py
@api.extend_model("res.organization")
class OrganizationAssistantExtension:
    default_ai_assistant_provider_id = fields.Reference(
        "ai.provider.config", on_delete="set null", label="Default AI Provider",
    )
```

That is the whole change. Check what it will do before applying it:

```bash
ede migrate check -t <tenant>     # reports res_organization.default_ai_assistant_provider_id missing
ede migrate upgrade -t <tenant>   # adds the column and its FK
```

!!! note "Retiring an extension field"

    Deleting the declaration removes the field from the ORM. The **column is not dropped** — the
    comparison never drops, so a deactivated module is indistinguishable from a deleted one only
    in the ORM, never in the data. The column keeps its contents and loses its NOT NULL.

## Soft-FK degradation

When test fixtures register only a subset of models (e.g. just `res.organization` without `ai.provider.config`), the SqlAlchemy metadata builder used to crash with `NoReferencedTableError` for any FK whose target table wasn't in the batch. Phase 2's builder now **degrades the FK to a soft reference**: the column is still created, the FK constraint is silently omitted, and the Reference still resolves at runtime via the registry.

This means: writing tests that target the base model **continues to work** even if you've added extension fields whose FK targets live in modules the test doesn't load. No test fixture update required.

## View inheritance — supported xpath positions

A `<view id="..." model="..." parent="<prefix>.<parent_view_id>">` carries one or more `<xpath expr="..." position="...">` patches as **direct children**. (The older nested `<extend ref="...">` element was retired in `foundation.customization` Phase 4B and now raises a `DslParseError`.) **The `parent` MUST be qualified** with a dotted prefix identifying the owning module of the base view — e.g. `foundation.res_organization_form_view` (the `res_organization_form_view` lives in `foundation.base`). The SDK strips the leading prefix and looks up the suffix as a registered view_id. If the suffix doesn't resolve, an `InvalidViewDefinition` raises at boot — bare refs are not supported in production code (test fixtures using bare ids are accepted via a literal-hit fallback to keep the harness tight).

```xml
<view id="res_organization_form_ai_section" model="res.organization"
      parent="foundation.res_organization_form_view">
    <xpath expr=".//section[@string='Status']" position="after">
        <section string="AI Assistant">
            <field name="default_ai_assistant_provider_id"/>
        </section>
    </xpath>
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

## What an extension may contribute

**Fields, and the methods those fields need.** A computed field declares its
behaviour as a method name, and the compute runtime resolves that name on the
*target* model — so the method is carried across with the field:

```python
@api.extend_model("crm.quote")
class QuoteTaxExtension:
    gross_total = fields.Decimal(
        method="_compute_gross_total", store=True, depends_on=["net_total", "tax_rate"],
    )

    def _compute_gross_total(self, env):
        return (self.net_total or 0) * (1 + (self.tax_rate or 0))
```

Contributed method names must be **free** on the target. A name already held by
the model's own method, by something inherited from `DomainModel` (`save`,
`validate`, `emit`), or by another extension raises `ExtensionMethodCollision`
at registration. Collisions are refused rather than resolved by precedence:
letting load order decide whose `_compute_total` wins produces wrong data with
no error attached to it.

## Boot-time validation

The boot pipeline calls `registry.validate_extensions()` after every app loads.
Failures surface immediately:

| Refused | Why |
|---|---|
| Target model not registered | Usually a missing manifest `depends` on the owning module. |
| Target opted out via `__ede_no_extension__=True` | The model has declared itself closed. |
| **The extending app does not depend on the app owning the target** | Extending a model *is* a dependency on its owner — the same dependency as importing from it, expressed in the schema instead of in Python. Without the manifest edge the loader has nothing to order the two modules by, and the graph on paper stops matching the real one. Transitive edges count: a module need not restate what its own dependency already declares. |
| **A field declares `method=` that does not resolve on the target** | The compute runtime looks the method up by name and, finding nothing, logs and returns — leaving the column at its default. Green boot, existing column, quietly wrong value. Refused at boot instead. |

Field- and method-name collisions raise `ExtensionFieldCollision` /
`ExtensionMethodCollision` at registration, before boot completes.

## Inspecting extensions at runtime

| API | Purpose |
|---|---|
| `registry.list_extensions()` | All registered extension classes. |
| `registry.list_extensions(target_model_key)` | Extensions targeting a specific model. |
| `view_registry.list_extensions_for_view(view_id=...)` | View extensions registered against a specific parent view. |
| Settings → Technical → Resources → **Extensions** | Admin UI surface listing all `@api.extend_model` registrations from the live registry mirror. |

`ir.model.extension` is the registry mirror behind that screen, filled by `ExtensionRegistrySync` on every `ede migrate upgrade` — one row per extension, recording target model, owning module, contributed field names and scope. An extension removed from the code is soft-deactivated, never deleted.

## Consumer adoption checklist

For a downstream module wanting to extend a base model:

1. **Manifest `depends`** — list the owning module (e.g. `foundation.assistant` depends on `foundation.base` + `foundation.ai`).
2. **Declare the extension class** — anywhere in `src/ede/foundation/<module>/models/` works; the file just needs to be imported during app load.
3. **Nothing to ship for the schema** — the column is added by the schema comparison on the next `ede migrate upgrade`, attributed to your module because that is where it is declared.
4. **(Optional) Ship a view extension** — a `<view ... parent="...">` with `<xpath>` children, in an XML file under the consuming module's `views/` folder; list it in the consuming module's `__manifest__.py` `data:` list.
5. **(Optional) Provide a `scope` predicate** — only needed for country-scoped / tenant-scoped / custom-gated extensions.
6. **Tests** — write fixtures that boot just your module + the upstream module; assert the extension fields appear on the target. If you contributed a compute, assert the *value*, not just the column — that is the part that used to fail silently.

That's the whole contract. No other infrastructure pieces are needed.

## Related

- [Phase 2 roadmap](../roadmap/foundation/base/phase-2/README.md) — the full workstream breakdown.
- [`foundation.l10n` roadmap](../roadmap/foundation/l10n/README.md) — the country-scope predicate consumer; Phase 1 lands the `country_scope("CC")` factory on top of this SDK.
- [Runtime Schema Upgrades](runtime-schema-upgrades.md) — how an extension's column reaches a tenant, and where a rename or backfill goes.
- [CLAUDE.md Model Placement Test](../CLAUDE.md) — when to extend vs lift to platform.

---

## A shipped example

`foundation.storage` contributes the file slot on `foundation.connectors`'
parameter model. It is the reference implementation of both halves.

| Piece | File |
|---|---|
| Field | `src/ede/foundation/storage/models/connector_param_ext.py` |
| Registration | `src/ede/foundation/storage/models/__init__.py` (import it, or the decorator never runs) |
| Views | `src/ede/foundation/storage/views/connector_param_file_ext.xml` (three `parent="connectors.…"` patches) |
| Manifest | `views/connector_param_file_ext.xml` listed in `foundation.storage`'s `data:` |

The reason it is an extension rather than a plain field is the same reason the
SDK exists: `ir.connector.param` belongs to `foundation.connectors`, which
depends only on `foundation.base`, while `foundation.storage` depends on
*connectors*. Declaring `file_id` on the connectors model would have connectors
reference a table storage owns, closing a cycle in the module graph.

**One thing the consumer has to remember:** a contributed field is absent when
its contributing module is not active. Read it with `getattr(record, "field", None)`,
not `record.field`.
