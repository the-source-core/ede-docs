# 5. Relationships

`blog.post` is useful on its own, but real models link to other models. This chapter adds three relationships to the example:

-   **`Reference`** (many-to-one) — every post has one author. The author is a `res.partner` — the platform's universal party master.
-   **`One2Many`** — every post has many comments. You'll create a new `blog.comment` model that points back at the post.
-   **`Many2Many`** — every post has many tags, every tag can label many posts. You'll create a `blog.tag` model and a join table will be auto-generated.

```python
@api.model("blog.post")
class BlogPost(DomainModel):
    title       = fields.Char(required=True)
    author_id   = fields.Reference("res.partner", on_delete="restrict", required=True)
    comment_ids = fields.One2Many("blog.comment", backref_field_name="post_id")
    tag_ids     = fields.Many2Many("blog.tag")


@api.model("blog.comment")
class BlogComment(DomainModel):
    post_id = fields.Reference("blog.post", on_delete="cascade", required=True)
    author  = fields.Char(max_length=200, required=True)
    body    = fields.Char(multi_line=True, required=True)


@api.model("blog.tag")
class BlogTag(DomainModel):
    name = fields.Char(max_length=80, required=True, unique=True)
```

By the end you can navigate from a post to its author, list its comments, set its tags, and understand exactly which `on_delete` policy applies in each case.

## What you're going to build

Add the two new models (`blog.comment`, `blog.tag`) and extend `blog.post` with the three relational fields above. After the migration runs you will:

-   Browse a post → `post.author_id.name` returns the author's name from `res.partner`.
-   Browse a post → `post.comment_ids` returns a `RecordSet` of every comment.
-   Add a tag with `RelationalCommand.link(tag.id)`.
-   See the `blog_post_blog_tag_rel` join table that the framework auto-derived.

## 1. The three relational types — and when to reach for each

| Field type | Storage | Use when |
|---|---|---|
| `Reference(...)` | One UUID column on this table, pointing at the target's `record_uuid`. | The child knows its single parent — author of a post, country of a partner, organization of a record. |
| `One2Many(...)` | **Not stored.** Reads find rows on the *other* table whose `Reference` field points back at this record. | The parent wants to walk its children — comments on a post, lines on an order, addresses of a partner. Requires a matching `Reference` on the target. |
| `Many2Many(...)` | An auto-derived join table — `{this_table}_{target_table}_rel`. | An unordered set on each side — tags on a post, roles on a user, organizations shared with a record. |

Critical: **`One2Many` is the mirror of a `Reference`.** It never stores anything itself; it just reads the inverse direction. You declare the FK on the child (as a `Reference`), then declare the reverse-view on the parent (as a `One2Many`).

## 2. Create the two new models

Create `src/domains/blog/post/models/comment.py`:

```python
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model(
    "blog.comment",
    description="Blog Comment",
    record_name="author",
    default_order="created_at_utc desc",
)
class BlogComment(DomainModel):
    """A reader-submitted comment on a blog post."""

    post_id = fields.Reference(
        "blog.post",
        on_delete="cascade",
        required=True,
        index=True,
        label="Post",
    )
    author = fields.Char(max_length=200, required=True, label="Author")
    body   = fields.Char(multi_line=True, required=True, label="Body")
    active = fields.Boolean(default=True)
```

Create `src/domains/blog/post/models/tag.py`:

```python
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model(
    "blog.tag",
    description="Blog Tag",
    record_name="name",
    default_order="name asc",
)
class BlogTag(DomainModel):
    """A categorisation label that can be attached to any post."""

    name   = fields.Char(max_length=80, required=True, unique=True, label="Name")
    active = fields.Boolean(default=True)
```

Update `src/domains/blog/post/models/__init__.py` to import them:

```python
from . import post
from . import comment
from . import tag
```

## 3. Extend `blog.post` with the three relations

Add three fields to `BlogPost` in `src/domains/blog/post/models/post.py`:

```python
@api.model(
    "blog.post",
    record_name="title",
    display_name_format="[{slug}] {title}",
    default_order="created_at_utc desc",
    name_search_fields=["title", "slug"],
)
class BlogPost(DomainModel):
    # ... existing fields ...

    # Many-to-one — every post has one author.
    author_id = fields.Reference(
        "res.partner",
        on_delete="restrict",
        required=True,
        label="Author",
    )

    # One-to-many — every post has many comments.
    comment_ids = fields.One2Many(
        "blog.comment",
        backref_field_name="post_id",
        label="Comments",
    )

    # Many-to-many — posts and tags are independent sets.
    tag_ids = fields.Many2Many(
        "blog.tag",
        label="Tags",
    )
```

The `One2Many`'s `backref_field_name` is the name of the `Reference` field on `blog.comment` that points back here. The framework uses that to find children at read time.

## 4. `on_delete` — what happens when the target is deleted

`Reference` fields take an `on_delete` policy. Pick deliberately:

| Policy | Behaviour | Use when |
|---|---|---|
| `"restrict"` (default) | Deleting the target is rejected as long as any record still references it. | The relationship is structural and breaking it should be a deliberate act. Default — e.g. you can't delete a `res.partner` while posts still cite them as author. |
| `"cascade"` | Deleting the target also deletes every record referencing it. | The child is meaningless without its parent — delete a post and all its comments go with it. |
| `"set null"` | Deleting the target sets the reference to NULL on referencing rows. | The reference is informational and the child should survive — e.g. an "owner" pointer that becomes orphaned. |

In the example: `blog.post.author_id` is `restrict` (an author isn't deletable while their posts exist); `blog.comment.post_id` is `cascade` (delete the post → comments go).

## 5. Generate and apply the migration

```bash
ede migrate generate -m "blog comment + tag + relations" --app blog.post
ede migrate upgrade -t system
```

Open the generated file. You'll see:

-   `op.create_table("blog_comment", ...)` with a `post_id` column (FK to `blog_post.record_uuid`) and `ON DELETE CASCADE`.
-   `op.create_table("blog_tag", ...)` with the unique index on `name`.
-   `op.create_table("blog_post_blog_tag_rel", ...)` — the auto-derived M2M join table with two FK columns: `blog_post_id` and `blog_tag_id`.
-   `op.add_column("blog_post", ...)` for `author_id`, plus an index.

The Many2Many join table name follows a fixed convention: `{this_table}_{target_table}_rel`. The columns are `{this_table}_id` and `{target_table}_id`, each pointing at the relevant `record_uuid`.

## 6. Reading relations — `RecordSet` navigation

In a command handler, an HTTP controller, or a test, you traverse relations as attribute access on the `RecordSet`:

```python
post = env.models["blog.post"].browse(some_uuid)

# Reference — returns a single-record RecordSet of the target model.
author = post.author_id
print(author.name)                  # res.partner.name

# One2Many — returns a (possibly empty) RecordSet of the target model.
comments = post.comment_ids
for c in comments:
    print(c.author, c.body)

# Many2Many — same shape.
tags = post.tag_ids
print([t.name for t in tags])
```

Notes:

-   **Empty references are not `None`** — they're an empty `RecordSet`. `if post.author_id:` works (truthy when non-empty).
-   `post.comment_ids` is a fresh read each time you access it — no stale cache. For loops that re-read, materialise once: `comments = post.comment_ids` and iterate.
-   The `active` filter applies transitively: archived comments and tags are hidden by default from these reads. Use `env.with_active_test(False)` to see them.

## 7. Writing to relations — `RelationalCommand`

You don't assign lists directly to relational fields. Instead, pass a list of `RelationalCommand` tuples that describe what to add, link, or remove:

```python
from ede.core.orm.commands import RelationalCommand as Rel

# Add a tag the user just picked from a dropdown.
post.write({"tag_ids": [Rel.link(tag_uuid)]})

# Replace the entire tag set in one shot.
post.write({"tag_ids": [Rel.set([tag_a, tag_b, tag_c])]})

# Remove a tag (does NOT delete the tag — just unlinks).
post.write({"tag_ids": [Rel.unlink(tag_uuid)]})

# Strip every tag from this post.
post.write({"tag_ids": [Rel.clear()]})

# Create-and-link a brand-new tag in one call.
post.write({"tag_ids": [Rel.create({"name": "Fresh"})]})

# One2Many — create a new comment as a child of this post in one call.
post.write({
    "comment_ids": [
        Rel.create({"author": "Alice", "body": "Loved it"}),
        Rel.update(existing_comment_uuid, {"body": "Edited"}),
        Rel.delete(stale_comment_uuid),
    ]
})
```

The seven factories:

| Factory | Effect |
|---|---|
| `Rel.create(values)` | Create a new related record and link it. For One2Many, the FK is stamped automatically. |
| `Rel.update(id, values)` | Write `values` to the related record `id`. |
| `Rel.delete(id)` | Delete the related record from the database, removing the link. |
| `Rel.unlink(id)` | Remove the link only. The related record stays (Many2Many) or has its FK set to null (One2Many with on_delete="set null"). |
| `Rel.link(id)` | Add an existing related record to the relation (Many2Many only). |
| `Rel.clear()` | Remove every link (Many2Many) or unlink every child (One2Many). |
| `Rel.set([ids])` | Replace the entire relation with exactly this set. |

Why a command list instead of plain values? Because the relation is a multi-row collection — there's no single "value" to assign. The command tuples let you express add / remove / replace atomically in one `write()` call, all inside the same transaction.

## 8. Cross-domain references and the platform layer

`blog.post.author_id` points at `res.partner` — a model that ships with the **platform** (`foundation.base`), not with the `blog` domain. That's the right shape.

The framework's mental model has two layers:

```
DOMAIN LAYER          your code: blog.post, blog.comment, blog.tag
                      depends on the platform; never depends on other domains.
        │
        ▼
PLATFORM LAYER        framework: res.partner, res.organization, res.user,
                      res.country, res.currency, res.uom, ir.session,
                      ir.menu, ir.action, ir.rbac.* …
                      Never references any domain model.
```

Rules of thumb:

-   **A `Reference` from your code to a `res.*` or `ir.*` model is always fine** — that's the platform there for you to use. The `res.partner` master, `res.organization` legal entity, `res.user` user, `res.currency` for money, `res.country` for addresses — all available.
-   **A `Reference` from your domain to another domain is a smell.** If `blog` needs to point at something that lives in another business domain, the model probably belongs in the platform instead.

See [Features: Base — Platform Substrate](../features/base.md) for what the platform models actually offer.

## 9. Add the relations to the form view

Update `src/domains/blog/post/views/blog_post_views.xml` so the form shows the new fields:

```xml
<FormView>
    <header>
        <field name="status" widget="statusbar"/>
    </header>
    <sheet>
        <group>
            <field name="title"/>
            <field name="slug"/>
            <field name="author_id"/>
            <field name="tag_ids" widget="tags"/>
        </group>
        <notebook>
            <page string="Body">
                <field name="body" widget="text"/>
            </page>
            <page string="Comments">
                <field name="comment_ids" widget="one2many">
                    <ListView>
                        <field name="author"/>
                        <field name="body"/>
                        <field name="created_at_utc" widget="datetime"/>
                    </ListView>
                </field>
            </page>
        </notebook>
    </sheet>
</FormView>
```

Two new widgets:

-   `widget="tags"` on a `Many2Many` renders chip-style pills with an inline picker.
-   `widget="one2many"` embeds a list view of the related rows directly inside the form. The inner `<ListView>` declares which columns to show.

Restart the server, open a post in the web client. The Tags pill bar lives on the right; the Comments tab shows an inline editable list — add a row, save, and the framework dispatches `RelationalCommand.create(...)` under the hood.

---

## What you just did

-   Created two new models — `blog.comment` and `blog.tag` — each registered, migrated, and queryable like any other model.
-   Added a **`Reference`**, a **`One2Many`**, and a **`Many2Many`** to `blog.post`, each with the right `on_delete` policy.
-   Read related records by walking the `RecordSet` (`post.author_id.name`, `post.comment_ids`, `post.tag_ids`).
-   Wrote relations using the seven `RelationalCommand` factories — `link` / `unlink` / `create` / `update` / `delete` / `clear` / `set`.
-   Learned the cross-layer rule: domain → platform `Reference`s are fine; domain → domain ones aren't.

[:octicons-arrow-right-24: **Next — Permissions & Security**](06-permissions.md)
