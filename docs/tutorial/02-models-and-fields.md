# 2. Models & Fields

*Tutorial chapter — coming soon.*

This chapter will walk through every field type in the kernel (`Char`, `Text`, `Integer`, `Decimal`, `Boolean`, `Date`, `DateTime`, `Enum`, `JSON`), validation options (`required`, `min_value`, `max_value`, `index`, `unique`), computed fields, default values, and the lifecycle of auto-fields.

In the meantime:

-   [Domain Model & Fields](../03-domain-model.md) — full reference of the field system, auto-fields, and `@api.model` options.
-   [ORM Layer](../05-orm-layer.md) — how `RecordSet` and `ModelProxy` expose those fields at runtime.
-   [Computed Field Runtime](../platform-02-compute-field-runtime.md) — `fields.Field(method=..., depends_on=..., store=...)`.

[:octicons-arrow-right-24: Continue — Views & the Web Client](03-views-and-web-client.md)
