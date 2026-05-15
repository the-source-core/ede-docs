# 5. Relationships

*Tutorial chapter — coming soon.*

This chapter will extend `blog.post` with relationships:

-   `Reference` (many-to-one) to `res.partner` — the post author.
-   `One2Many` to `blog.comment` — comments on the post.
-   `Many2Many` to `blog.tag` — tags on the post.

…and show how `on_delete` ("restrict" / "cascade" / "set null"), cross-domain references to `res.*` masters, and FK column generation interact.

In the meantime:

-   [Domain Model & Fields](../03-domain-model.md) — the Reference / One2Many / Many2Many sections.
-   [Foundation Base](../foundation-base.md) — what `res.partner`, `res.country`, `res.currency` offer that you can point at.
-   [Foundation Model Naming](../foundation-model-naming.md) — why platform masters live under `res.*` and how to keep cross-domain dependencies clean.

[:octicons-arrow-right-24: Continue — Permissions & Security](06-permissions.md)
