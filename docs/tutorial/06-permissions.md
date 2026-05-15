# 6. Permissions & Security

Right now anyone with a valid token can read, update, and delete every `blog.post` — the API is wide open. This chapter closes that gate. You'll declare roles, attach permissions to each role, assign roles to users, and add a **record rule** so that authors only see their own drafts. By the end, the same `blog.post` model behaves differently for an admin, a regular author, and a portal user, with no extra code in the model itself.

```python
# A role binding — the foundation of every access decision.
env.models["ir.rbac.role.binding"].create({
    "user_id":  alice.id,
    "role_id":  blog_author_role.id,
})

# Inside a command handler, the engine has already gated this call.
@api.on_command("blog.post.publish")
def publish(self, cmd):
    # If Alice lacks the `blog.post.publish` permission, this method
    # never runs — `PermissionDeniedError` is raised upstream.
    self.write({"status": "published"})
```

Four moving parts:

| Concept | Model | Purpose |
|---|---|---|
| **Role** | `ir.rbac.role` | A bundle of permissions. Hierarchical — child roles inherit. |
| **Permission** | `ir.rbac.permission` | A single `resource × action` grant tied to one role. |
| **Binding** | `ir.rbac.role.binding` | Assigns a user to a role. |
| **Record rule** | `ir.rbac.record.rule` | A per-row visibility / write filter scoped to a role. |

The platform ships three default roles you can inherit from: `portal_user`, `internal_user`, `system_admin`.

## What you're going to build

-   Declare a `blog_author` role that inherits from the built-in `internal_user`.
-   Declare permissions for read / create / update / delete / publish of `blog.post`.
-   Assign the role to a user via a `ir.rbac.role.binding`.
-   Add a **record rule** so authors only see drafts they own; published posts are visible to everyone.
-   Optionally opt `blog.post` into **company scope** with one decorator argument so every post is automatically owned by the user's active organization.

## 1. The shape of an access decision

Two engines compose to answer "can this user do this on that row?":

1.  **The permission engine** answers the *coarse* question — *can this role perform action X on resource M at all?* Resources are model keys; actions are `read` / `create` / `update` / `delete` / `execute` (where execute covers domain commands like `blog.post.publish`).
2.  **The record-rule engine** answers the *fine* question — *given that this role is allowed action X on M, which **subset of rows** can it touch?* Rules are stored as domain-filter expressions with `$principal.*` variable substitution.

Both engines run before every command handler. A request needs to pass both gates to proceed.

For reads, the rules layer composes into the ORM filter automatically — `search()` returns only rows the user is allowed to see, no extra code in the handler.

## 2. Declare a custom role

Roles live under `ir.rbac.role`. The platform ships three default ones already, ranked from least to most powerful:

| Role | Purpose | Inherits |
|---|---|---|
| `portal_user` | Customer / vendor self-service | — |
| `internal_user` | Day-to-day operational staff | `portal_user` |
| `system_admin` | Full platform access | `internal_user` |

A child role automatically gets every permission its parent has, and the parent's parent, and so on. So `system_admin` has every permission of `internal_user` and `portal_user` too.

Create a custom role for the blog app. Add a new data file `src/domains/blog/post/data/blog_roles.xml`:

```xml
<ede>
    <data noupdate="1">

        <record id="blog.role_author" model="ir.rbac.role">
            <field name="code">blog_author</field>
            <field name="name">Blog Author</field>
            <field name="role_type">JOB</field>
            <field name="parent_id" ref="rbac.role_internal_user"/>
            <field name="description">Writes, edits, and publishes blog posts.</field>
            <field name="is_active">true</field>
        </record>

    </data>
</ede>
```

`role_type` is a label (`JOB`, `ADMIN`, `FUNCTIONAL` — used for grouping in admin UI). `parent_id` references the role this one inherits from.

`noupdate="1"` because roles are seed data — created once on first install, never overwritten on subsequent upgrades (so admin tweaks survive).

Register the file in the manifest's `data` list.

## 3. Declare permissions

Permissions are stored in `ir.rbac.permission`. They're typically loaded from a CSV — one row per `resource × action × role`. Create `src/domains/blog/post/data/blog_permissions.csv`:

```csv
id,name,code,resource,action,role_id/id,domain
blog.p_blog_post_read,Read Blog Posts,blog.post.read,blog.post,read,rbac.role_portal_user,
blog.p_blog_post_create,Create Blog Posts,blog.post.create,blog.post,create,blog.role_author,
blog.p_blog_post_update,Update Blog Posts,blog.post.update,blog.post,update,blog.role_author,
blog.p_blog_post_publish,Publish Blog Posts,blog.post.publish,blog.post,execute,blog.role_author,
blog.p_blog_post_delete,Delete Blog Posts,blog.post.delete,blog.post,delete,rbac.role_system_admin,
blog.p_blog_comment_read,Read Comments,blog.comment.read,blog.comment,read,rbac.role_portal_user,
blog.p_blog_comment_create,Post Comments,blog.comment.create,blog.comment,create,rbac.role_portal_user,
blog.p_blog_tag_read,Read Tags,blog.tag.read,blog.tag,read,rbac.role_portal_user,
```

The CSV columns:

| Column | Purpose |
|---|---|
| `id` | External ID — `<your_module>.<unique_suffix>`. |
| `name` | Human label. |
| `code` | A globally unique short code; pattern is `<resource>.<action>`. |
| `resource` | The model key (`blog.post`). |
| `action` | One of `read` / `create` / `update` / `delete` / `execute`. The `execute` action is the one custom commands like `blog.post.publish` check. |
| `role_id/id` | External ID of the role granted this permission. Leave **empty** for a global permission everyone has. |
| `domain` | Optional domain-filter JSON literal. When set, the permission is row-scoped. Supports `$principal.*` variables. |

For example, `base.p_res_user_read_self` is a real ship-default permission with a row-scoped domain:

```
"[[""record_uuid"",""="",""$principal.user_id""]]"
```

That grants every portal user read access to their own user record only.

Register the CSV in the manifest's `data` list. The loader picks up CSV files automatically — filename **must** match the target model key (`blog.tag.csv` → `blog.tag`).

## 4. Available action verbs

The framework recognises these actions on every model:

| Action | Triggered by |
|---|---|
| `read` | `ede.read_one`, `ede.search`, `ede.count`, `ede.read_group`. |
| `create` | `ede.create`. |
| `update` | `ede.update`. |
| `delete` | `ede.delete`. |
| `execute` | Any domain command like `blog.post.publish`. The command name is the permission code. |

You don't have to enumerate every `execute` permission you ever add — the framework matches by command name. So a permission with `code: blog.post.publish` gates the `blog.post.publish` command.

## 5. Assign the role to a user

A `ir.rbac.role.binding` row links a user to a role:

```python
env.models["ir.rbac.role.binding"].create({
    "user_id":         alice.id,
    "role_id":         blog_author_role.id,
    "organization_id": acme.id,    # optional — scopes the binding to one org
})
```

`organization_id` is optional. When set, the binding is **scoped** — Alice has the `blog_author` role only when her active organization is `acme`. When omitted, the binding is global to Alice across every organization she belongs to.

For testing, you can also create bindings declaratively in an XML data file:

```xml
<record id="blog.binding_alice_author" model="ir.rbac.role.binding">
    <field name="user_id" ref="base.user_alice"/>
    <field name="role_id" ref="blog.role_author"/>
</record>
```

## 6. The `$principal.*` ABAC variables

Permission domains and record-rule domains both support a small set of variables that the engine substitutes per request:

| Variable | Resolved from |
|---|---|
| `$principal.user_id` | The current user's `record_uuid`. |
| `$principal.active_organization_id` | The org currently bound to the request (from the JWT). |
| `$principal.allowed_organization_ids` | The set of organization IDs the user can switch into. |
| `$principal.org_ids` / `$principal.branch_ids` / `$principal.department_ids` | Split by org-unit kind. Useful for finer-grained scoping. |

A typical domain leaf using these:

```
[["owner_id", "=", "$principal.user_id"]]
```

The engine substitutes the variable before evaluating the filter. The result is identical to writing `[["owner_id", "=", "<alice's uuid>"]]` by hand — but it works for every user without per-user permissions.

## 7. Record rules — row-level filters

A record rule narrows what a role can *see* (or *write*) on a model. Add a rule that says "authors only see their own drafts; published posts are visible to everyone":

```xml
<ede>
    <data noupdate="1">

        <!-- GLOBAL rule — applies to every user, AND-merged with all other globals.
             Says "draft posts are filtered to the owner; published posts are wide-open". -->
        <record id="blog.rr_post_visibility" model="ir.rbac.record.rule">
            <field name="name">Author sees own drafts; published is public</field>
            <field name="model_id" ref="ir.model_blog_post"/>
            <field name="domain">[
                "|",
                ["status", "=", "published"],
                ["author_id", "=", "$principal.user_id"]
            ]</field>
            <field name="perm_read">true</field>
            <field name="perm_create">true</field>
            <field name="perm_update">true</field>
            <field name="perm_delete">true</field>
        </record>

    </data>
</ede>
```

Anatomy of a record rule:

| Field | Purpose |
|---|---|
| `name` | Human label. |
| `model_id` | Reference to the `ir.model` registry row for the target model. Always present once a model is registered. |
| `domain` | Domain-filter expression. Same syntax as `env.models[...].search([...])`. `$principal.*` variables resolved per request. |
| `role_ids` | Many-to-many to roles. **Empty** = the rule is GLOBAL (applies to every user, AND-merged with other globals). **Non-empty** = the rule is role-scoped (OR-merged within each role bucket; OR'd across role buckets for multi-role users). |
| `perm_read` / `perm_create` / `perm_update` / `perm_delete` | Booleans selecting which operations the rule gates. A `perm_read=true` rule narrows reads; a `perm_update=true` rule blocks updates on rows outside the filter. |
| `sequence` | Display order. |
| `active` | Standard soft-archive flag. |

### Composition algorithm

The engine composes every active rule on a (model, action) into one final domain:

```
GLOBAL_1 AND GLOBAL_2 AND … AND ((ROLE_A_RULE_1 OR ROLE_A_RULE_2) OR (ROLE_B_RULE_1 OR …))
```

The final composition is AND-merged with the model's `active` filter (Chapter 2) and any `company_scope` filter (next section), then applied to every read.

**Boundary cases worth knowing:**

-   **No rules at all** → no narrowing; the user sees every row the permission engine allows.
-   **Globals only, no role rules** → the AND of every global is the universal floor.
-   **A user with NO matching role bucket on a model that has role-scoped rules** → fail-closed empty set. Once any role rule exists for a model, only matching roles can see rows.
-   **System admin (role code `system_admin`)** → bypasses record rules entirely. Admins can't lock themselves out.

## 8. Company scope — automatic per-org isolation

For multi-organization deployments, every `blog.post` should be owned by one organization and visible only to users currently scoped to that org. You get this for free by opting the model in:

```python
@api.model(
    "blog.post",
    record_name="title",
    company_scope="strict",          # ← one keyword
)
class BlogPost(DomainModel):
    title = fields.Char(required=True)
    ...
```

What `company_scope="strict"` does:

-   **Auto-injects** a required `organization_id` Reference field pointing at `res.organization`.
-   **Filters every read** to the user's active organization. Posts owned by other orgs are simply not returned.
-   **Stamps the field** automatically on every create from `env.active_organization_id`.
-   **Rejects writes** that target an organization the user doesn't have in `$principal.allowed_organization_ids`.

Three modes:

| Mode | Behaviour |
|---|---|
| `"strict"` | Required `organization_id`. Reads filter to the active org only. Create stamps from the active org. |
| `"optional"` | Nullable `organization_id`. Reads include the active org's rows *or* rows where `organization_id is None` (tenant-global). Useful for catalogue-like models. |
| `"multi"` | Auto-injects an `organization_ids` Many2Many. Reads return rows whose set includes the active org. Use for records shared across multiple orgs. |

The migration adds the FK column for you. Re-run `ede migrate generate` after adding `company_scope=...` and the diff shows the new column with the right index.

## 9. Switching the active organization at runtime

A user with access to multiple organizations can switch between them. The HTTP endpoint:

```
POST /api/auth/switch-organization
{
    "organization_id": "<target_org_uuid>"
}
```

The endpoint re-issues the JWT with a new `active_organization_id` claim, provided the target is in the user's `allowed_organization_ids`. Subsequent requests are scoped to the new org automatically — every read filters to it, every create stamps it.

## 10. Inspecting "why was I denied"

Every denial writes a row to `ir.rbac.decision.log` with a structured reason code:

| `reason` | Meaning |
|---|---|
| `permission_missing` | No role the user holds grants the requested action on the resource. |
| `record_rule_violation` | The user has the permission but the row isn't in any visible bucket. |
| `wrong_organization` | A write targets an org outside the user's `allowed_organization_ids`. |

Open **Settings → Roles & Permissions → Decision Log** in the web client and filter by `reason = "record_rule_violation"` when debugging an "I can't see this row" issue. The log also captures the active organization at the time of the request — useful for diagnosing scope mismatches.

## 11. Apply and verify

Wire the new files in `src/domains/blog/post/__manifest__.py`:

```python
"data": [
    "views/blog_post_views.xml",
    "data/blog_post_actions.xml",
    "data/blog_post_menus.xml",
    "data/blog_roles.xml",
    "data/blog_permissions.csv",
    "data/blog_record_rules.xml",
],
```

Apply:

```bash
ede migrate upgrade -t system
```

If you added `company_scope=...`, run a `migrate generate` first to produce the column migration, then `upgrade`.

Verify in the web client:

1.  Log in as Alice. The Blog app appears. She sees only her own drafts and every published post.
2.  Log in as a portal-only user. The Blog app still appears (portal can read posts), but the **Create** button is hidden — no `blog.post.create` permission.
3.  Look at **Settings → Roles & Permissions → Permissions**. Filter `resource = blog.post`. Every row you authored should appear, tied to its role.

---

## What you just did

-   Declared a `blog_author` role inheriting from `internal_user`.
-   Authored a CSV of permissions covering read / create / update / delete / publish for the blog models.
-   Bound a user to the role.
-   Wrote a **global record rule** that combines public-published with private-drafts in one composed domain filter.
-   Saw how `$principal.*` variables let one rule serve every user without per-user duplication.
-   Learned the difference between **permissions** (coarse, role × resource × action) and **record rules** (fine, row-level filter with composition).
-   Optionally enabled **company scope** with one decorator argument and let the framework do per-org isolation for you.

[:octicons-arrow-right-24: **Next — Migrations**](07-migrations.md)
