# 6. Permissions & Security

*Tutorial chapter — coming soon.*

This chapter will cover gating access to `blog.post`: declaring `ir.rbac.permission` rows, binding roles to users via `ir.rbac.binding`, scoping records to a company with `company_scope`, and writing a record rule so authors can only see their own drafts.

In the meantime:

-   [Permissions](../18-permissions.md) — full RBAC reference.
-   [Foundation Security](../foundation-security.md) — active-org, company-scope, record rules.
-   [Foundation Record Rules](../foundation-record-rules.md) — the `ir.rbac.record.rule` engine.
-   [Foundation Company Scope](../foundation-company-scope.md) — `@api.model(company_scope="strict|optional|multi")`.
-   [Foundation Archivable Models](../foundation-archivable-models.md) — soft-delete via the `active` field.

[:octicons-arrow-right-24: Continue — Migrations](07-migrations.md)
