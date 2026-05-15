# Your First Module

You have a running EDE server. Now you'll add a brand-new domain — `blog` — with a single model `blog.post`, watch it register at boot, generate its migration, and confirm it shows up in the web client.

This is deliberately the smallest possible module. Chapters 2–8 build on it.

## What you're going to build

```
src/domains/
├── settings.py                 ← ACTIVE_DOMAINS list (edit)
└── blog/                       ← NEW domain
    ├── __init__.py
    ├── settings.py             ← ACTIVE_MODULES for the blog domain
    └── post/                   ← NEW app inside the blog domain
        ├── __init__.py
        ├── __manifest__.py
        ├── models/
        │   ├── __init__.py
        │   └── post.py         ← defines blog.post
        └── migrations/
            └── versions/       ← Alembic puts files here
```

EDE distinguishes a **domain** (a top-level business area you're modelling — `blog`, `inventory`, `support`, whatever your codebase needs) from the **apps** inside it (each a coherent feature with its own manifest, models, views, and migrations). A trivial domain may have a single app — that's what we're doing here.

## 1. Register the new domain

Edit `src/domains/settings.py` to add `blog` to `ACTIVE_DOMAINS`. If the list is currently empty, this will be its only entry; if it already has other domains, append `"blog"` to them:

```python
from __future__ import annotations

ACTIVE_DOMAINS: list = ["blog"]
```

EDE never auto-discovers domains. If a domain isn't in this list, it never boots — even if the files exist.

## 2. Create the domain package

```bash
mkdir -p src/domains/blog/post/models
mkdir -p src/domains/blog/post/migrations/versions
```

Create `src/domains/blog/__init__.py`:

```python
# blog domain — see settings.py for active apps
```

Create `src/domains/blog/settings.py`:

```python
from __future__ import annotations

ACTIVE_MODULES: list = ["post"]
```

## 3. Create the app manifest

Every app needs a `__manifest__.py` — a plain Python dict that declares the app's identity, dependencies, and any seed data files. Create `src/domains/blog/post/__manifest__.py`:

```python
{
    "name": "Blog",
    "summary": "Minimal blog domain — a single Post model.",
    "description": "Tutorial example app. One model, no business logic.",
    "author": "THE_BLACK_BOX",
    "category": "Tutorial",
    "version": "0.1.0",
    "depends": ["base"],
    "data": [],
}
```

The `depends: ["base"]` line is important — it tells the loader to boot `foundation.base` before this app, because `base` ships the `res.*` masters that every domain references (currency, partner, user, etc.).

## 4. Wire the import chain

EDE uses explicit imports — every model file must be reachable from the app's `__init__.py`. Create three short files:

`src/domains/blog/post/__init__.py`:

```python
from . import models
```

`src/domains/blog/post/models/__init__.py`:

```python
from . import post
```

`src/domains/blog/post/models/post.py`:

```python
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model(
    "blog.post",
    description="Blog Post",
    record_name="title",
    default_order="created_at_utc desc",
    name_search_fields=["title"],
)
class BlogPost(DomainModel):
    """A single blog post — the smallest possible domain model."""

    title = fields.Char(
        max_length=200,
        required=True,
        label="Title",
    )
    body = fields.Char(
        multi_line=True,
        label="Body",
        help="Markdown-formatted post content.",
    )
    published = fields.Boolean(
        default=False,
        label="Published",
    )
```

That's the entire model. Note what you **didn't** write:

-   No `dbid`, `record_uuid`, `created_at_utc`, `updated_at_utc`, `revision` — those are injected automatically by `DomainModel`.
-   No `create() / read() / update() / delete()` — generic CRUD commands (`ede.create`, `ede.read_one`, `ede.update`, `ede.delete`, `ede.search`, `ede.count`, `ede.read_group`) are provided for every model by `CrudKernel`.
-   No table-definition SQL — Alembic generates it from the field declarations.

## 5. Generate the migration

```bash
ede migrate generate -m "blog.post initial" --app blog.post
```

Alembic inspects the field specs declared on `BlogPost`, compares them to the current DB schema, and writes a new revision under `src/domains/blog/post/migrations/versions/`. Open the generated file — you'll see `op.create_table("blog_post", ...)` with columns for every field (yours + the auto-injected `dbid`, `record_uuid`, timestamps, revision).

!!! tip "Naming"
    The migration filename is auto-generated and timestamp-prefixed. The `-m` flag is the human description that goes into the file's docstring.

## 6. Apply the migration

```bash
ede migrate upgrade -t system
```

This runs *all* pending migrations against the `system` tenant — including yours. Re-running is idempotent.

## 7. Confirm the model registered

```bash
ede info
```

Look for `blog.post` in the model list and `blog` in the domain list. If it's missing, the most common cause is a missing `from . import …` line in one of the `__init__.py` files.

## 8. See it in the web client

Restart the server (if it was already running):

```bash
ede serve --with-worker
```

Open `http://localhost:8000`. The model is registered and queryable via the generic CRUD HTTP endpoints, e.g.:

```bash
curl -X POST http://localhost:8000/api/data/blog.post \
    -H "Content-Type: application/json" \
    -d '{"title": "Hello world", "body": "First post.", "published": true}'
```

You won't see Blog in the app-switcher yet — that requires registering an `ir.menu` and a view. That's chapter 3.

---

## What you just did

You added a complete domain to the framework in seven files and one settings edit:

```
src/domains/settings.py                            (edited)
src/domains/blog/__init__.py                       (new)
src/domains/blog/settings.py                       (new)
src/domains/blog/post/__init__.py                  (new)
src/domains/blog/post/__manifest__.py              (new)
src/domains/blog/post/models/__init__.py           (new)
src/domains/blog/post/models/post.py               (new)
src/domains/blog/post/migrations/versions/<...>.py (auto-generated)
```

The framework took care of:

-   Registering the model in the global registry at boot.
-   Generating the SQL schema from your field declarations.
-   Wiring CRUD commands and HTTP endpoints automatically.
-   Adding auto-fields (`dbid`, `record_uuid`, timestamps, revision counter).

## Where to look in the source

| Concept | File |
|---|---|
| `@api.model` decorator | `src/ede/core/api.py` |
| `DomainModel` base | `src/ede/core/kernel/model.py` |
| Field types | `src/ede/core/kernel/fields.py` |
| Module loader | `src/ede/core/loader.py` |
| CRUD command kernel | `src/ede/core/orm/` |

## What you can do next

[:octicons-arrow-right-24: **Next — Models & Fields**](02-models-and-fields.md)
