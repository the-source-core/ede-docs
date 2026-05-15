# `company_scope` — Active-Organization Scoping for DomainModels

**Module:** `foundation.security` Phase 2
**Roadmap:** [roadmap/foundation/security/phase-2/](../roadmap/foundation/security/phase-2/README.md)
**Status:** ✅ Delivered 2026-05-13

---

## What it does

One kwarg on `@api.model(...)` opts a DomainModel into active-organization
scoping end-to-end. After Phase 2 ships:

```python
@api.model("crm.opportunity", company_scope="strict")
class Opportunity(DomainModel):
    ...
```

…is the **entire change** required to make `crm.opportunity` org-scoped:

1. **Auto-injects** the right ownership field if the developer didn't declare it.
2. **Auto-filters** every read (`search`, `count`, `read_group`, One2Many, Many2Many)
   to the env's active organization.
3. **Stamps** the ownership field from `env.active_organization_id` on every
   create where the caller didn't supply it.

The pattern mirrors the existing `__ede_has_active__` soft-archive mechanism —
no new architecture, one parallel filter, one parallel registrar.

---

## The three modes

| Mode | Auto-injected field | Read filter | Create stamping | When to use |
|---|---|---|---|---|
| `"strict"` | `organization_id` **required**, `on_delete="restrict"`, indexed | `["organization_id", "=", active_org]` | Stamp `active_org`; reject `None` after stamping | CRM, sales, finance, HR records that belong to exactly one legal entity |
| `"optional"` | `organization_id` **nullable**, `on_delete="restrict"`, indexed | `["organization_id", "=", active_org] OR ["organization_id", "=", None]` (own + tenant-global) | Stamp `active_org` when missing; `None` is valid | Reference / master data that may be company-owned OR tenant-global (e.g. `res.partner`, master tables) |
| `"multi"` | `organization_ids = Many2Many("res.organization")` | `["organization_ids", "in", [active_org]]` (containment) | Stamp `[active_org]` when empty | Records explicitly shared across organizations (e.g. shared price lists, joint accounts) |

### Mode conflicts

The decorator validates at decoration time and raises `InvalidHandler` when a
mode conflicts with what the developer declared:

- `company_scope="strict"` (or `"optional"`) + an explicit `organization_ids = fields.Many2Many(...)` → conflict.
- `company_scope="multi"` + an explicit `organization_id = fields.Reference(...)` → conflict.

A developer-declared field of the *expected* kind is preserved verbatim
(e.g. `company_scope="strict"` + an explicit `organization_id` with custom
`label` or `required=False` keeps the developer's declaration).

---

## Bypassing the filter / stamping

Three escape hatches, narrowing in scope:

```python
# 1. PER-CALL — caller's domain mentions the ownership field → auto-filter skipped
env.models["crm.opportunity"].search([["organization_id", "=", other_org]])

# 2. PER-ENV — clone the env to bypass the filter for the rest of the chain
env.with_company_test(False).models["crm.opportunity"].search([])

# 3. SUDO — system principal bypasses both the filter AND the create-time
#    stamping (sudo writes do not stamp organization_id automatically).
#    Mostly useful for migrations, workers, and seed loaders that intentionally
#    span every org. Combine with `.with_active_organization_id(org_id)` to
#    stamp into a specific org from system code.
env.sudo().models["crm.opportunity"].search([])
env.sudo().with_active_organization_id(org_id).models["crm.opportunity"].create({...})
```

The bypass list — `res.user`, `res.organization`, `ir.session`, `ir.menu`,
`ir.action`, `ir.org.unit`, `ir.data.reference`, every `ir.rbac.*`, every
`ir.model*` — is hardcoded in `domain_filter._is_bypass_model`. These models
never participate in company scoping even if a downstream patch were to set
their `company_scope`.

---

## What happens when there's no active org?

`env.active_organization_id` can be `None` in three situations:

- A non-request env (migrations, seed loaders, workers, bare bootstrap)
- A request env from a user who hasn't picked an org yet
- A system principal (`env.sudo()`) that didn't chain `with_active_organization_id`

Behaviour, per mode:

| Mode | Read | Create |
|---|---|---|
| `strict` | No rows (fail closed) | Raises `ValueError("organization_id_required: …")` |
| `optional` | Returns only tenant-global rows (`organization_id IS NULL`) | Creates with `organization_id=NULL` (legal) |
| `multi` | No rows visible | Creates with empty `organization_ids` (legal but no-one can read) |

The setting `SECURITY_REQUIRE_ACTIVE_ORG=True` (Phase 1 default) is the
backstop: login is rejected when a user has no `organization_id` AND no
`organization_ids`, so request envs almost always carry an active org.

---

## Migration recipe for opting an existing model into `strict`

Adding `company_scope="strict"` to a model that *already has data* needs a
backfill step before the column can become `NOT NULL`. Recipe:

```python
# 1. Add the column nullable (the decorator's auto-inject won't actually
#    drive the schema — you author this migration by hand).
op.add_column("crm_opportunity",
    sa.Column("organization_id", sa.String(length=36), nullable=True))
op.create_index("idx_crm_opportunity_organization_id",
    "crm_opportunity", ["organization_id"])
op.create_foreign_key(
    op.f("fk_crm_opportunity_organization_id_res_organization"),
    "crm_opportunity", "res_organization",
    ["organization_id"], ["record_uuid"],
    ondelete="restrict",
)

# 2. Backfill existing rows. Pick a strategy that fits your data:
#    a) creator's default org:
#       UPDATE crm_opportunity SET organization_id = (
#           SELECT u.organization_id FROM res_user u
#           WHERE u.record_uuid = crm_opportunity.created_uid
#       )
#    b) tenant-wide fallback:
#       UPDATE crm_opportunity SET organization_id = '<tenant-default-org-uuid>'

# 3. Flip NOT NULL once every row has a value:
op.alter_column("crm_opportunity", "organization_id", nullable=False)
```

For `company_scope="optional"` (like `res.partner` did in Phase 2), no backfill
is needed — `NULL` is the legal "tenant-global" state, and legacy rows keep
their pre-Phase-2 cross-org visibility.

For `company_scope="multi"`, the join table is auto-generated by the metadata
builder; the migration only needs to create the join table — no backfill of
the parent model's columns.

---

## Settings → Technical → Models view

`ir.model.company_scope` (`Char(20)`, nullable) is auto-populated by
`RegistrySync` on every `ede migrate upgrade`. Values: `null` (default),
`strict`, `optional`, `multi`. The Settings UI can filter the model picker
by company_scope to find every opted-in model in a tenant.

---

## Reference example: `res.partner` (Phase 2 proof-of-life)

```python
@api.model(
    "res.partner",
    description="Partner",
    name_search_fields=["name", "code", "email"],
    display_name_format="[{code}] {name}",
    company_scope="optional",   # ← opt-in
)
class Partner(Chatterable):
    # `organization_id` auto-injected as nullable Reference to res.organization
    member_type = fields.Enum(...)
    code = fields.Char(...)
    name = fields.Char(...)
    ...
```

After this opt-in:
- New partners created from a request env with an active org are stamped on create.
- Search returns the active org's partners **plus** all tenant-global partners (`organization_id IS NULL`).
- Cross-org admin tools chain `env.with_company_test(False)`.
- The Alembic migration `c2a5b9d4e8f3_res_partner_organization_id.py` adds the column nullable with no backfill — every legacy partner row stays tenant-global.

---

## Related

- [foundation-security.md](foundation-security.md) — module overview + Phase 1
- [18-permissions.md](18-permissions.md) — the ABAC `$principal.*` reference that complements `company_scope` filtering
- [roadmap/foundation/security/phase-2/](../roadmap/foundation/security/phase-2/README.md) — design rationale + acceptance criteria
