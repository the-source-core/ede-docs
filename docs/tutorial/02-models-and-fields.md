# 2. Models & Fields

In Chapter 1 you declared `blog.post` with three fields. This chapter extends it into a fuller schema — Char with constraints, Enum, Decimal, Reference, JSON, and a computed field — and walks through every field option you'll use day-to-day.

```python
@api.model("blog.post")
class BlogPost(DomainModel):
    title        = fields.Char(max_length=200, required=True, index=True)
    slug         = fields.Char(max_length=200, required=True, unique=True)
    body         = fields.Char(multi_line=True, help="Markdown body.")
    status       = fields.Enum(
        selection=[("draft", "Draft"), ("review", "Review"), ("published", "Published")],
        default="draft",
        required=True,
    )
    published_on = fields.Date()
    view_count   = fields.Integer(default=0)
    rating       = fields.Decimal(precision=3, scale=2)
    author_id    = fields.Reference("res.partner", on_delete="restrict", required=True)
    word_count   = fields.Integer(method="compute_word_count", depends_on=["body"], store=True)
    metadata     = fields.JSON(default_factory="dict")
    active       = fields.Boolean(default=True)

    def compute_word_count(self):
        return len((self.body or "").split())
```

By the end of the chapter you'll know which field type to reach for, how to constrain it, and how to compute one field from another.

## What you're going to build

Take the three-field `blog.post` from Chapter 1 and grow it into the model above. Each new line teaches one concept; nothing is added without explanation.

```
blog.post
├── title         Char   (required, indexed)
├── slug          Char   (required, unique)
├── body          Char   (multi-line)
├── status        Enum   (default "draft")
├── published_on  Date
├── view_count    Integer (default 0)
├── rating        Decimal (precision 3, scale 2)
├── author_id     Reference → res.partner
├── word_count    Integer (computed from body)
├── metadata      JSON
└── active        Boolean (soft-archive flag)
```

## 1. The field catalog

The kernel ships exactly these field types — declared in `src/ede/core/kernel/fields.py`. There's a deliberate one-line summary because the catalog is intentionally small:

| Type | One-line use | Storage |
|---|---|---|
| `Char` | Strings of any length. Add `multi_line=True` for paragraph text. | `VARCHAR` / `TEXT` |
| `Integer` | Whole numbers. | `INTEGER` |
| `Decimal` | Exact numeric values — money, measurements. Never use floats. | `NUMERIC(precision, scale)` |
| `Boolean` | True / False. | `BOOLEAN` |
| `Date` | A date (no time component). | `DATE` |
| `DateTime` | A point in time. Always UTC by contract. | `TIMESTAMP` |
| `Enum` | One of a fixed list. Stores the key, UI shows the label. | `VARCHAR` |
| `JSON` | Structured or free-form payload. | `TEXT` (SQLite) / `JSONB` (Postgres) |
| `UUID` | Raw UUID. Rarely needed by hand — auto-fields use it. | `VARCHAR(36)` / native `UUID` |
| `Reference` | Many-to-one foreign key. | `VARCHAR` (FK → `record_uuid`) |
| `One2Many` | Reverse of a Reference — children pointing back. Not stored. | — |
| `Many2Many` | Set-valued association via an auto-derived join table. | join table |

`Reference`, `One2Many`, and `Many2Many` get their own chapter — see [Relationships](05-relationships.md). The rest you'll use here.

## 2. Common options on every field

Every field, regardless of type, accepts these keyword arguments:

| Option | Effect |
|---|---|
| `required=True` | The field must be set on every record. Validated before INSERT. |
| `unique=True` | A unique index is generated on the column. |
| `index=True` | A non-unique index is generated. Use for fields you filter on. |
| `readonly=True` | Direct writes via `RecordSet.write()` are rejected. |
| `default=<value>` | Literal default applied when no value is provided. |
| `default_factory="dict"` | Name of a factory (e.g. `"dict"`, `"list"`) called per-record to produce the default. Use this for mutable defaults. |
| `label="..."` | Human-readable name shown in the web client. Auto-derived from the field name if omitted. |
| `help="..."` | Tooltip / inline help shown in the form view. |

Use these freely. A field can combine any number of them:

```python
slug = fields.Char(
    max_length=200,
    required=True,
    unique=True,
    label="URL slug",
    help="Lowercase, dash-separated. Used in the post URL.",
)
```

## 3. Per-type options worth knowing

**`Char`** — `max_length`, `min_length`, `regex` for validation; `multi_line=True` for paragraph storage.

```python
title = fields.Char(max_length=200, required=True)
body  = fields.Char(multi_line=True)
phone = fields.Char(regex=r"^\+?[0-9\-\s]{7,}$")
```

**`Integer`** — `min_value`, `max_value` for range validation.

```python
view_count   = fields.Integer(default=0, min_value=0)
priority     = fields.Integer(min_value=1, max_value=5)
```

**`Decimal`** — `precision` (total digits) and `scale` (digits after the point). Use for money or measurements where rounding matters.

```python
rating       = fields.Decimal(precision=3, scale=2)   # 0.00 – 9.99
list_price   = fields.Decimal(precision=12, scale=2)  # up to 99,999,999.99
```

**`DateTime`** — `auto_now_add=True` stamps on INSERT, `auto_now=True` stamps on every UPDATE. The framework already injects `created_at_utc` and `updated_at_utc` for you (see step 5), so you rarely need this.

**`Enum`** — `selection` is a list of `(key, label)` tuples. The DB stores the key, the web client shows the label.

```python
status = fields.Enum(
    selection=[
        ("draft", "Draft"),
        ("review", "In Review"),
        ("published", "Published"),
    ],
    default="draft",
    required=True,
)
```

**`JSON`** — use `default_factory="dict"` or `default_factory="list"` for mutable defaults; never `default={}` (the same dict instance would be shared across all records).

```python
metadata = fields.JSON(default_factory="dict")
tags     = fields.JSON(default_factory="list")
```

## 4. References — one quick word now, full story in Chapter 5

You don't need to wait until Chapter 5 to use a foreign key. The shape is simple:

```python
author_id = fields.Reference("res.partner", on_delete="restrict", required=True)
```

- The first argument is the **target model key** (a string).
- `on_delete` controls what happens when the target is deleted: `"restrict"` (default — block the delete), `"cascade"`, or `"set null"`.
- Reference fields are **always indexed** automatically.
- The column stores the target's `record_uuid`. From a `RecordSet`, `post.author_id` returns a single-record `RecordSet` of `res.partner` — read its fields directly.

`res.partner` is the canonical platform party master (people, companies, contacts). It's shipped by the framework — see [Features: Base — Platform Substrate](../features/base.md).

The full mechanics of `One2Many`, `Many2Many`, and write semantics for relations come in [Chapter 5 — Relationships](05-relationships.md).

## 5. Auto-fields the framework injects for you

You did not declare `dbid`, `record_uuid`, `created_at_utc`, `updated_at_utc`, or `revision` on `blog.post` — and you don't need to. Every `DomainModel` subclass gets them automatically:

| Auto-field | Type | What it does |
|---|---|---|
| `dbid` | `Integer` | Auto-increment primary key. DB-managed. Never set it in INSERT payloads. |
| `record_uuid` | `UUID` | External identity. Used in HTTP URLs and as the target of every foreign key. |
| `created_at_utc` | `DateTime` | Stamped by the repository on INSERT. Indexed. |
| `updated_at_utc` | `DateTime` | Stamped on every UPDATE. Indexed. |
| `revision` | `Integer` | Starts at 0; increments on every write. Optimistic-concurrency hint. |

`.id` is kept as an alias for `.record_uuid` — use whichever feels natural. The numeric `.dbid` is for internal joins and never appears in external APIs.

You can opt out for an unusual model by setting `__ede_disable_auto_fields__ = True` on the class — but it's rarely the right choice.

## 6. Computed fields — derive one field from others

Any field can be **computed** by giving it a `method=` and listing the fields it depends on. There are two flavours:

```python
# Unstored — computed on every read. The default.
preview = fields.Char(
    method="compute_preview",
    depends_on=["body"],
)

# Stored — persisted to a column; recomputed on every write to its dependencies.
word_count = fields.Integer(
    method="compute_word_count",
    depends_on=["body"],
    store=True,
)

def compute_preview(self):
    text = self.body or ""
    return text[:140]

def compute_word_count(self):
    return len((self.body or "").split())
```

Rules:

- `method` is the name (a string) of an instance method on the same class.
- `depends_on` is the list of fields the computed value reads. The framework uses it to invalidate / recompute correctly.
- Computed fields are **always read-only** — direct writes are rejected.
- Default `store=False` is right when the computation is cheap; choose `store=True` when you query or sort on the value frequently.

## 7. Model options — the `@api.model` decorator

Every model class is preceded by `@api.model(...)`. The decorator accepts these keyword arguments; pick what's useful for each model and omit the rest:

| Option | What it does |
|---|---|
| `model_key` | **Required.** The stable identifier — `"blog.post"`. First positional argument. |
| `record_name` | Field name used as the row's display label (in lists, M2O dropdowns, breadcrumbs). |
| `description` | Human-readable label shown in the admin model browser. |
| `default_order` | SQL ORDER BY clause for `search()` calls without explicit ordering, e.g. `"created_at_utc desc"`. |
| `name_search_fields` | List of fields the M2O picker matches against using `ilike`. Defaults to `["name"]`. |
| `display_name_format` | `str.format_map` template for the row's display name. Every `{key}` must be a declared field — validated at decoration time. Example: `"[{slug}] {title}"`. |
| `table` | Override the auto-derived SQL table name. Rarely needed. |
| `auto_fields` | Set `False` to opt out of the seven auto-fields. Almost never the right choice. |
| `abstract` | Set `True` for mixin classes that contribute fields but never get a table of their own. |
| `custom_properties` | Set `True` to opt the model into the Properties feature — a JSON `properties` column is auto-injected and tenant-scoped property definitions validate on every write. See [Features: Customization](../features/customization.md). |
| `company_scope` | `"strict"` / `"optional"` / `"multi"` — auto-injects an `organization_id` (or `organization_ids`) field and filters every read to the user's active organization. Covered in [Chapter 6 — Permissions & Security](06-permissions.md). |

Example combining several:

```python
@api.model(
    "blog.post",
    record_name="title",
    display_name_format="[{slug}] {title}",
    default_order="created_at_utc desc",
    name_search_fields=["title", "slug"],
    description="Blog Post",
)
class BlogPost(DomainModel):
    ...
```

## 8. Soft-archive — the `active` Boolean

When a model needs the equivalent of "soft delete", add an `active` Boolean. The framework auto-recognises it:

```python
active = fields.Boolean(default=True)
```

The ORM then automatically hides `active=False` rows from `search` / `count` / `read_group` and from relational reads. The web client search panel auto-renders an **Archived** toggle. Bypass the filter in code with `env.with_active_test(False)` when you genuinely need every record.

You don't have to declare `active` — it's a per-model choice. Without it, every record is always visible.

## 9. Assemble the full model

Replace the body of `src/domains/blog/post/models/post.py` from Chapter 1 with the assembled version:

```python
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model(
    "blog.post",
    description="Blog Post",
    record_name="title",
    display_name_format="[{slug}] {title}",
    default_order="created_at_utc desc",
    name_search_fields=["title", "slug"],
)
class BlogPost(DomainModel):
    """A blog post — the running example used throughout the tutorial."""

    title        = fields.Char(max_length=200, required=True, index=True)
    slug         = fields.Char(max_length=200, required=True, unique=True)
    body         = fields.Char(multi_line=True, help="Markdown body.")
    summary      = fields.Char(max_length=500, help="One-paragraph preview.")

    status       = fields.Enum(
        selection=[
            ("draft", "Draft"),
            ("review", "In Review"),
            ("published", "Published"),
            ("archived", "Archived"),
        ],
        default="draft",
        required=True,
    )
    published_on = fields.Date(label="Published on")
    view_count   = fields.Integer(default=0, min_value=0, label="Views")
    rating       = fields.Decimal(precision=3, scale=2, help="Average rating, 0.00–5.00.")

    author_id    = fields.Reference("res.partner", on_delete="restrict", required=True)

    word_count   = fields.Integer(
        method="compute_word_count",
        depends_on=["body"],
        store=True,
    )

    metadata     = fields.JSON(default_factory="dict")
    active       = fields.Boolean(default=True)

    def compute_word_count(self) -> int:
        return len((self.body or "").split())
```

## 10. Regenerate and apply the migration

The change to the model has not yet hit the database. Generate a new migration revision and apply it:

```bash
ede migrate generate -m "blog.post fields expansion" --app blog.post
ede migrate upgrade -t system
```

Open the generated file under `src/domains/blog/post/migrations/versions/`. Alembic has diffed your field declarations against the current schema and emitted `op.add_column(...)` calls for every new field, plus a new index for `title` and the unique index for `slug`.

The repository auto-stamps `created_at_utc` / `updated_at_utc` / `revision` on each write — you don't write migration code for those.

## 11. Confirm via the API

The generic CRUD HTTP endpoints work against the new schema with no extra code:

```bash
curl -X POST http://localhost:8000/api/data/blog.post \
    -H "Content-Type: application/json" \
    -d '{
        "title":     "Hello fields",
        "slug":      "hello-fields",
        "body":      "Writing a longer post to exercise word_count.",
        "status":    "draft",
        "author_id": "<record_uuid of a res.partner>",
        "metadata":  {"source": "tutorial-ch02"}
    }'
```

The response includes every declared field plus the auto-injected ones. `word_count` is filled in by the framework — you never set it directly.

---

## What you just did

-   Walked the **full field-type catalog** and the options each type exposes.
-   Tuned the model with `@api.model` options: `record_name`, `display_name_format`, `default_order`, `name_search_fields`.
-   Added a **computed field** (`word_count`) that derives its value from `body`.
-   Declared an **`active` Boolean** so the ORM treats archived posts as hidden by default.
-   Generated a follow-up Alembic revision and applied it — the schema now matches the expanded model.

[:octicons-arrow-right-24: **Next — Views & the Web Client**](03-views-and-web-client.md)
