# `ir.rbac.record.rule` — Record-Rule Engine (Developer Guide)

**Module:** `foundation.security` Phase 5
**Roadmap:** [roadmap/foundation/security/phase-5/](../roadmap/foundation/security/phase-5/README.md)
**Status:** ✅ Delivered 2026-05-13

---

## Mental Model

Two layers of RBAC, one model gates each:

| Layer | Model | Question it answers |
|---|---|---|
| **Permission** (action gate) | `ir.rbac.permission` | "Can role X perform action A on model M?" |
| **Record rule** (row filter) | `ir.rbac.record.rule` | "On model M, which subset of rows can role X see / write / create / delete?" |

Permissions decide IF; record rules decide WHICH. The two coexist — both are AND-merged at every ORM read callsite and per-record write gate.

---

## The composition algorithm

Per request × `user` × `model` × `action`:

```
GLOBAL_1 AND GLOBAL_2
  AND (
        (ROLE_A_RULE_1 OR ROLE_A_RULE_2)
     OR (ROLE_B_RULE_1 OR ROLE_B_RULE_2)
  )
```

- **Globals** (rules with empty `role_ids`) → AND together. Universal restrictions that no role can bypass.
- **Role-scoped rules** (rules with non-empty `role_ids`) → OR within each role bucket the user actually holds.
- **Multi-role users** → cross-role OR.
- **Final expression** → AND-merged with the caller's domain at every ORM read callsite.

The fully composed domain then chains with the existing filters at every callsite, in this order:

```
caller_domain
   AND  apply_active_filter(...)          ← soft-archive
   AND  apply_company_filter(...)         ← Phase 2 multi-company
   AND  apply_record_rules_filter(...)    ← THIS phase
```

---

## Authoring a rule (admin UI)

**Settings → Security → Roles & Permissions → Record Rules**

| Field | Notes |
|---|---|
| `name` | Human-readable label ("Sales agents see only own leads"). |
| `model_id` | Target model (FK to `ir.model`). Cascade-deletes when the model is unregistered. |
| `role_ids` | Empty = GLOBAL rule. Non-empty = role-scoped. |
| `domain` | Domain-filter JSON with `$principal.*` resolution. Example: `[["owner_id", "=", "$principal.user_id"]]`. |
| `perm_read` / `perm_create` / `perm_update` / `perm_delete` | Per-action gating. A rule with `perm_read=True, perm_update=False` shapes search but is ignored on update. |
| `sequence` | Render order in admin list. Does NOT affect evaluation — composition is set-theoretic and order-independent. |
| `active` | Soft-archive — inactive rules don't compose. |

### `$principal.*` variables resolved in domain values

See [docs/18-permissions.md](18-permissions.md) Section 7 for the full list. The common ones:

- `$principal.user_id` — UUID of the current user
- `$principal.active_organization_id` — currently-selected org (Phase 1)
- `$principal.allowed_organization_ids` — list of orgs the user can switch to
- `$principal.org_ids` / `$principal.branch_ids` / `$principal.department_ids` — split per role-binding scope_type

---

## Worked example

**Goal:** Sales agents only see their own leads. Team leads see leads in their team. System admin sees everything.

```
Rule 1 (role-scoped):
  name        = "Sales agents see own leads"
  model_id    = crm.lead
  role_ids    = [role_sales_agent]
  domain      = [["owner_id", "=", "$principal.user_id"]]
  perm_read   = True   perm_update = True   perm_delete = True   perm_create = True

Rule 2 (role-scoped):
  name        = "Team leads see team leads"
  model_id    = crm.lead
  role_ids    = [role_team_lead]
  domain      = [["team_id", "in", "$principal.branch_ids"]]
  perm_read   = True   perm_update = True   perm_delete = False  perm_create = True
```

- **System admin** — `system_admin` role short-circuits the engine. Sees everything.
- **Sales agent (no team lead)** — sees rows where `owner_id = self`.
- **Team lead (no sales agent)** — sees rows where `team_id ∈ branches_user_leads`.
- **Both roles** — OR-combined: `(owner_id = self OR team_id ∈ branches)`.
- **User with neither role** — model has role-scoped rules but no matching bucket → **empty set** (`[["record_uuid","=",None]]`). Fail-closed.

---

## Bypass / escape hatches

| Layer | Trigger | Effect |
|---|---|---|
| **system_admin role** | User holds `system_admin` | Engine no-ops; user sees + writes everything. |
| **env.sudo()** | `principal.is_system=True` | Engine no-ops; system code is unconstrained. |
| **Platform bypass list** | Model is `res.user` / `res.organization` / `ir.session` / `ir.rbac.*` / `ir.model*` / `ir.menu` / `ir.action` / `ir.org.unit` / `ir.data.reference` | Engine never filters these regardless of rules in the DB. Admins can't lock themselves out. |
| **No rules for `(model, action)`** | DB has no rows matching the filter | Engine returns `[]` — back-compat no-op. Adding the first rule is what flips the model into opt-in restricted mode. |

There is NO env-level `with_record_rules_test(False)` clone — record rules are intentionally not user-disableable from request env. If you need to read across the rule boundary in code, use `env.sudo()`.

---

## Per-record gates (write side)

`AuthorizationService._enforce_record_rules(...)` runs **after** the existing `ir.rbac.permission` action check. On `create` / `update` / `delete` / `read_one`, it evaluates the composed rule expression against the live record dict:

- For `read_one` / `update` / `delete`, the live row is fetched once per env (cached in `_record_rule_record_cache`) and tested against the rule.
- For `create`, the new values dict is tested before persistence.
- A miss raises `PermissionDeniedError(reason="record_rule_violation")` → HTTP 403.

---

## Performance characteristics

Per request × model × action, the engine:
1. **Cache lookup** on `env._record_rules_cache[(user_id, model_key, action)]` — sub-microsecond.
2. **Miss path**: one `ir.model` lookup + one `ir.rbac.record.rule` search + one M2M join-table search — three DB hits total, all on indexed columns.

Idempotent across multiple lookups within the same env. Env clones (`with_principal`, `with_active_organization_id`, etc.) start with a fresh cache.

The composite index `idx_ir_rbac_record_rule_model_active_read` covers the hot path for read operations. The other three perm flags share `idx_ir_rbac_record_rule_model_id` + `idx_ir_rbac_record_rule_active`.

---

## Debugging cookbook

**"My rule isn't applying."**

1. Confirm `active=True` and the right `perm_<action>` flag is set.
2. Confirm `model_id` resolves: `SELECT model_key FROM ir_model WHERE record_uuid = <rule.model_id>` should return the expected model.
3. Confirm role membership: `SELECT scope_type, scope_id, role_id FROM ir_rbac_role_binding WHERE user_id = <uuid>`.
4. Check the env cache: `env._record_rules_cache` shows what the engine resolved per `(user_id, model_key, action)` tuple.
5. Malformed `domain` JSON drops the rule silently (logged at WARNING). Tail the app log for `record_rule: invalid domain JSON` lines.

**"Everyone is seeing everything despite my rule."**

Most likely cause: the rule's role is bound, but `system_admin` is also bound on that user. The system_admin bypass short-circuits the engine before any rule is loaded. Strip the system_admin role to test rule behaviour.

**"My role-scoped rule blocks even me."**

The fail-closed contract: a model with **any** role-scoped rule becomes opt-in restricted. A user with no matching role bucket sees zero rows. Add a global rule with `domain='[]'` if you want universal-allow plus role-narrowing.

---

## Related

- [foundation-security.md](foundation-security.md) — module overview + Phases 1-4
- [foundation-company-scope.md](foundation-company-scope.md) — `company_scope` ORM auto-filter (Phase 2)
- [18-permissions.md](18-permissions.md) — `ir.rbac.permission` action gates + `$principal.*` variable reference
- [roadmap/foundation/security/phase-5/](../roadmap/foundation/security/phase-5/README.md) — design rationale + acceptance criteria
