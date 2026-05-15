# 3. Views & the Web Client

You have a `blog.post` model and you can hit it via the generic CRUD endpoints — but the web client doesn't show it anywhere yet. In this chapter you'll declare list / form / search views in the XML DSL, wire an `ir.action` + `ir.menu`, and watch a **Blog** app appear in the top-bar app switcher with your posts navigable from the menu.

```xml
<!-- src/domains/blog/post/views/blog_post_views.xml -->
<ede version="1.0">
    <view id="blog_post_view_list" model="blog.post">
        <ListView>
            <field name="title"/>
            <field name="slug"/>
            <field name="status"/>
            <field name="view_count"/>
            <field name="author_id"/>
        </ListView>
    </view>

    <view id="blog_post_view_form" model="blog.post">
        <FormView>
            <header>
                <field name="status" widget="statusbar"/>
            </header>
            <sheet>
                <group>
                    <field name="title"/>
                    <field name="slug"/>
                    <field name="author_id"/>
                    <field name="published_on"/>
                </group>
                <notebook>
                    <page string="Body">
                        <field name="body" widget="text"/>
                    </page>
                    <page string="Metadata">
                        <field name="rating"/>
                        <field name="view_count"/>
                        <field name="word_count"/>
                    </page>
                </notebook>
            </sheet>
        </FormView>
    </view>

    <view id="blog_post_view_search" model="blog.post">
        <SearchView>
            <field name="title"/>
            <field name="author_id"/>
            <filter name="published" string="Published"
                    domain='[["status", "=", "published"]]'/>
            <filter name="drafts" string="Drafts"
                    domain='[["status", "=", "draft"]]'/>
            <group_by name="status" string="By status"/>
            <group_by name="author_id" string="By author"/>
        </SearchView>
    </view>
</ede>
```

## What you're going to build

By the end of this chapter, the web client at `http://localhost:8000` shows a **Blog** entry in the app switcher. Clicking it opens a list of posts. Clicking a row opens a form view with the body in a notebook tab. The search bar has "Published" / "Drafts" filters and a group-by toggle.

You will author three files and edit one:

```
src/domains/blog/post/
├── __manifest__.py                          ← edited (add "data" entries)
├── views/
│   └── blog_post_views.xml                  ← NEW
└── data/
    ├── blog_post_actions.xml                ← NEW (ir.action records)
    └── blog_post_menus.xml                  ← NEW (ir.menu records)
```

## 1. Author the view XML

Create `src/domains/blog/post/views/blog_post_views.xml` with the contents shown at the top of this chapter.

The DSL grammar:

| Tag | Purpose |
|---|---|
| `<ede version="1.0">` | Root element; every view file uses this wrapper. |
| `<view id="..." model="...">` | Container for one view. The `id` is the external identity used to reference / override it from another module. |
| `<ListView>` | Tabular list. `<field>` children declare visible columns. |
| `<FormView>` | Edit view. Composes `<header>`, `<sheet>`, `<group>`, `<notebook>`/`<page>`, `<chatter>`, and `<button>`. |
| `<KanbanView>` | Column-based board. Add `group_by="status"` to the root tag. |
| `<SearchView>` | The dynamic filter bar at the top of list / kanban views. Declares `<field>` (live ilike fields), `<filter>` (one-click domain filters), `<group_by>` (groupby chips). |

Field elements accept the same widgets the React client supports — `text`, `email`, `phone`, `currency`, `datetime`, `date`, `boolean`, `selection`, `many2one`, `one2many`, `many2many`, `progress`, `radio`, `tags`, `image`, `html`, `statusbar`. The kernel picks a sensible default when no `widget=` is given.

**A note on attribute quoting**: domain filters inside `<filter domain="...">` are valid JSON literals embedded in XML. Use single quotes around the attribute value so the inner JSON can use double quotes — `domain='[["status", "=", "draft"]]'`.

## 2. Register the views in the manifest

Open `src/domains/blog/post/__manifest__.py` and add the view file to the `data` list:

```python
{
    "name": "Blog",
    "summary": "Minimal blog domain — a single Post model.",
    "description": "Tutorial example app.",
    "author": "THE_BLACK_BOX",
    "category": "Tutorial",
    "version": "0.1.0",
    "depends": ["base"],
    "data": [
        "views/blog_post_views.xml",
        "data/blog_post_actions.xml",
        "data/blog_post_menus.xml",
    ],
}
```

Files in `data` are loaded in order at boot. Views must be loaded before the actions that reference them.

## 3. Declare the action

An `ir.action` tells the web client *what to open* when a menu is clicked. Create `src/domains/blog/post/data/blog_post_actions.xml`:

```xml
<ede>
    <data>

        <record id="blog.action_blog_post" model="ir.action">
            <field name="name">Posts</field>
            <field name="path">posts</field>
            <field name="model_key">blog.post</field>
            <field name="default_view">list</field>
            <field name="available_views">list,form,search</field>
        </record>

    </data>
</ede>
```

What the fields mean:

| Field | Purpose |
|---|---|
| `name` | Label shown on the menu and in the browser tab. |
| `path` | URL segment under `/wc/` — here it produces `/wc/posts`. |
| `model_key` | Which model the action operates on. The web client uses this to look up views and CRUD endpoints. |
| `default_view` | View shown first when the action loads. |
| `available_views` | Comma-separated list of view types the user can switch between via the top-right toggle. |

The action's `id` (`blog.action_blog_post`) is an **external ID** — a stable string that other XML records can reference via `<field ... ref="..."/>`.

## 4. Declare the menu

Menus are the navigation tree on the left, plus the app icons in the top-bar app switcher. Create `src/domains/blog/post/data/blog_post_menus.xml`:

```xml
<ede>
    <data noupdate="0">

        <!-- Top-level app entry — visible in the app switcher. -->
        <record id="blog.menu_blog_root" model="ir.menu">
            <field name="name">Blog</field>
            <field name="sequence">60</field>
            <field name="icon">notebook-text</field>
            <field name="icon_type">lucide</field>
            <field name="category">application</field>
        </record>

        <!-- Leaf — opens the posts list. -->
        <record id="blog.menu_blog_posts" model="ir.menu">
            <field name="name">Posts</field>
            <field name="parent_id" ref="blog.menu_blog_root"/>
            <field name="action_id" ref="blog.action_blog_post"/>
            <field name="sequence">10</field>
        </record>

    </data>
</ede>
```

`ir.menu` fields:

| Field | Purpose |
|---|---|
| `name` | The label that appears in the menu. |
| `parent_id` | Reference to the parent menu (omit for top-level entries). |
| `action_id` | Reference to the `ir.action` to run when this menu is clicked. Leaf menus have one; category menus don't. |
| `sequence` | Display order. Lower values come first. |
| `icon` / `icon_type` | Icon for app-switcher entries. `icon_type="lucide"` uses [lucide.dev](https://lucide.dev) icon names. |
| `category` | `"application"` for business apps (top section of app switcher) or `"system"` for utility apps like Settings. |

`noupdate="0"` means the loader will re-apply user changes from XML on every upgrade — useful while developing. Use `noupdate="1"` for seed data you only want to install once.

## 5. Restart and apply

Two of the three new files are XML data — they load on every upgrade:

```bash
ede migrate upgrade -t system
ede serve --with-worker
```

`ede migrate upgrade` re-runs the data loader for every module in dependency order. Your new `ir.action` and `ir.menu` records land in the DB; the existing `blog.post` table is untouched (no migration needed).

## 6. See it in the browser

Open `http://localhost:8000`. The app-switcher in the top bar now has a **Blog** icon. Click it. Click **Posts**. You see the list view, with whatever rows exist in `blog.post`.

Things to try:

-   Click a row → form view opens, with the body in the "Body" notebook tab and metadata in the "Metadata" tab.
-   Click the **Search** icon → search panel appears with the *Published* and *Drafts* filter chips.
-   Toggle **Group by → By status** → rows collapse into status buckets.
-   Click **Create** → empty form opens. Fill it in. Save → a new row.

Behind the scenes, every list-row click and every save dispatches a generic CRUD command (`ede.read_one`, `ede.update`, etc.) and renders the result against the `RenderPlan` the DSL parser produced.

## 7. Extending someone else's view

When a foundation module ships a view (say, `res.partner`'s form) and you want to add a field, override it with `<extend>`:

```xml
<extend ref="base.res_partner_form">
    <xpath expr="//field[@name='email']" position="after">
        <field name="loyalty_tier"/>
    </xpath>
</extend>
```

The expression syntax is XPath; `position` is one of `before`, `after`, `inside`, or `replace`. The web client merges your changes on top of the original — no source patching, no fork.

This is how every customisation should layer in: declare your override, let the framework compose it at boot, and the upstream view is free to evolve underneath you.

## 8. Buttons that trigger commands

Form views can call a command on the current record with `<button>`:

```xml
<header>
    <button name="blog.post.publish" string="Publish" type="object"
            attrs='{"invisible": [["status", "=", "published"]]}'/>
    <field name="status" widget="statusbar"/>
</header>
```

The `name` is the **command name** (a string registered via `@api.on_command` — see [Chapter 4](04-commands-and-events.md)). `type="object"` dispatches the command against the current record's model. The `attrs` expression hides the button when the record is already published.

Buttons are the bridge between the visual form and the command bus. The next chapter is where commands earn their keep.

---

## What you just did

-   Wrote a view file that declares **list**, **form**, and **search** views for `blog.post`.
-   Registered an `ir.action` so the web client knows how to open the model, and an `ir.menu` so a **Blog** entry appears in the app switcher.
-   Saw the model in the React web client end-to-end — list, search, group-by, form, create.
-   Learned the `<extend>` override pattern that lets you compose changes on top of foundation views without forking them.
-   Previewed how form buttons hand off to commands — picked up next chapter.

[:octicons-arrow-right-24: **Next — Commands & Events**](04-commands-and-events.md)
