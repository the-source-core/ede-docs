# Foundation Base — Team Substrate (Enhancement 06)

**Status:** ✅ Delivered 2026-05-25
**Roadmap:** [roadmap/foundation/base/enhancements/06-team-substrate.md](../roadmap/foundation/base/enhancements/06-team-substrate.md)
**Source:** [src/ede/foundation/base/models/team.py](../src/ede/foundation/base/models/team.py), [src/ede/foundation/base/services/team_role_service.py](../src/ede/foundation/base/services/team_role_service.py), [migration `a2e7f4b3c819`](../src/ede/foundation/base/migrations/versions/a2e7f4b3c819_team_substrate.py)

> Generic team primitive distinct from legal org-unit hierarchy. Team-functional roles decoupled from RBAC security roles. Sequence-aware role assignment for primary/backup escalation. Shared `TeamRoleService` consumed by `foundation.approval` and `foundation.workflow` (and any future automation consumer).

---

## 1. Why this exists

Real organizations have two orthogonal "who's who" vocabularies:

1. **RBAC security** — what can you DO? `ir.role` with `sales_rep`, `sales_manager`, `system_admin`. Drives the permission boundary at every CRUD callsite.
2. **Team-functional responsibility** — what's your JOB in the team's process? Examples: `MANAGER`, `PRICING_APPROVER`, `FINANCE_APPROVER`, `COUNTRY_HEAD`, `ACCOUNT_MANAGER`, `REVIEWER`.

A single RBAC `sales_rep` may correspond to several team-functional sub-roles (`LEAD_HUNTER`, `ACCOUNT_MANAGER`, `INSIDE_SALES`) that drive different workflow behavior with the same permission set. Conflating the two would force every new responsibility to require a new RBAC permission. Independent vocabularies grow at the pace of their owners — security review controls one, business workflow design controls the other.

Until Enhancement 06, the platform had `ir.role` (RBAC) and `ir.org.unit` (legal/branch hierarchy) but no concept of:
- A team distinct from the legal HQ/Division/Branch tree
- A team-functional role distinct from a security role
- A user holding multiple sub-roles on the same team with priority ordering
- A reusable lookup that says "who plays role X on team Y, walk up if missing?"

This enhancement ships those four primitives plus a shared lookup service.

---

## 2. The four models

### `res.team` — generic team primitive

```python
@api.model("res.team", ...)
class Team(DomainModel):
    name             # "Mumbai West Sales"
    code             # unique — "TEAM_MUM_WEST"
    parent_id        # Reference → res.team (self) — Country → Region → Local
    organization_id  # Reference → res.organization (required, tenant scope)
    org_unit_id      # Reference → ir.org.unit (optional branch link)
    type_id          # Reference → res.team.type (required)
    description, active, role_assignment_ids (O2M)
```

A team is hierarchical (`parent_id`), tenant-scoped (`organization_id`), optionally branch-aligned (`org_unit_id`), and typed (`type_id`). The hierarchy drives `WALK_UP_HIERARCHY` escalation in approval flows — when Mumbai West has no PRICING_APPROVER assigned, the engine walks up to India Region.

### `res.team.type` — extensible type vocabulary

```python
@api.model("res.team.type", ...)
class TeamType(DomainModel):
    code         # unique — "SALES", "PRICING_DESK", "FINANCE_APPROVERS", "HR", ...
    name, description, active
```

Foundation ships **zero seed rows**. Each consuming module seeds its own types in its `data/` directory and registers them in its manifest. Tenants can add types at runtime under **Settings → Teams & Roles → Team Types**.

Domain pickers filter by `type.code` so a CRM `team_id` dropdown shows only sales-typed teams:

```xml
<field name="team_id" domain="[['type_id.code', '=', 'SALES']]"/>
```

### `ir.team.role` — team-functional role master (decoupled from RBAC)

```python
@api.model("ir.team.role", ...)
class TeamRole(DomainModel):
    code            # unique — "MANAGER", "PRICING_APPROVER", "FINANCE_APPROVER", "COUNTRY_HEAD"
    name, description, active
    is_managerial   # soft hint — used by flow-template wizards as default filter
```

Note the `ir.*` prefix — `ir.team.role` is platform substrate vocabulary, same shelf as `ir.role` (RBAC). Foundation ships **zero seed rows**. Each consuming module seeds its own. The `is_managerial` flag is a soft hint for UX wizards (e.g. "show only approver-style roles when picking a flow step assignee").

### `res.team.role.assignment` — (team, role, user, sequence) join

```python
@api.model("res.team.role.assignment", ...)
class TeamRoleAssignment(DomainModel):
    team_id     # Reference → res.team
    role_id     # Reference → ir.team.role (NOT ir.role)
    user_id     # Reference → res.user
    sequence    # Integer default=10 — lower = higher priority
    active
```

Unique constraint on `(team_id, role_id, user_id)` — same person can't be listed twice for the same role on the same team. Multiple people CAN hold the same role on the same team — that's the whole point of `sequence`.

**`sequence=1` is primary. `sequence=2` is backup. `sequence=3` is tertiary.** Drives `NEXT_IN_SEQUENCE` escalation in the approval engine and `mode='all'` ordering in `TeamRoleService.resolve`.

---

## 3. `res.user` extension

Two new fields ship directly on `res.user` (foundation.base owns the model, so no `@api.extend_model` needed):

| Field | Type | Purpose |
|---|---|---|
| `team_ids` | Many2Many → `res.team` | All teams the user belongs to. Drives "show me records I can see" RBAC scoping for team-scoped consumer modules. |
| `primary_team_id` | Reference → `res.team` | Default team stamped on new records this user creates. Read by `pre.create` hooks in consumer models (`record.team_id = principal.user.primary_team_id`). |

---

## 4. `TeamRoleService` — shared lookup API

```python
from ede.foundation.base.services.team_role_service import TeamRoleService

svc = TeamRoleService(env)

# Primary user only (sequence=1) — empty list if none
users = svc.resolve(team, "PRICING_APPROVER", mode="primary")

# All users ordered by sequence asc
users = svc.resolve(team, "PRICING_APPROVER", mode="all")

# Walk up team.parent_id chain when direct lookup is empty
users = svc.resolve(team, "PRICING_APPROVER", mode="primary", walk_up=True)

# Walk up bounded by max_walk (safety cap on cyclic data; default 5)
users = svc.resolve(team, "PRICING_APPROVER", mode="primary", walk_up=True, max_walk=3)

# True/False — does user hold role on team?
svc.is_assigned(team, "MANAGER", user)

# Reverse lookup — all (team, role) pairs this user has
pairs = svc.all_assignments_for_user(user)
```

The service is **stateless beyond the bound `env`** and used by both `foundation.approval` (case routing — Enh 01) and `foundation.workflow` (guards + transition actions — Enh 01). One source of truth, no logic drift between the two engines.

---

## 5. Consumer adoption checklist

For any model that wants team-driven approval / workflow routing:

1. **Declare the field**

```python
@api.model("crm.quote")
class Quote(DomainModel):
    team_id = fields.Reference(
        "res.team",
        on_delete="restrict",
        required=True,
        label="Sales Team",
    )
```

2. **Seed your team type and roles** in your module's `data/` directory:

```csv
# data/res.team.type.csv
id,code,name,description,active
crm.team_type_sales,SALES,Sales Team,Sales reps grouped under a manager,True
```

```csv
# data/ir.team.role.csv
id,code,name,is_managerial,active
crm.team_role_sales_manager,SALES_MANAGER,Sales Manager,True,True
crm.team_role_pricing_approver,PRICING_APPROVER,Pricing Approver,True,True
```

Register both in your manifest's `data` list.

3. **Stamp team_id on create** via a pre-create hook (or a foundation-provided mixin when one ships):

```python
@api.on_hook("pre.crm.quote.create")
def stamp_team_id(self, cmd: Command) -> None:
    values = (cmd.payload or {}).get("values", {})
    if values.get("team_id"):
        return
    principal = self.env.principal or {}
    primary = principal.get("primary_team_id")
    if primary:
        values["team_id"] = primary
```

4. **Filter the picker** in your form view to show only sales-typed teams:

```xml
<field name="team_id" domain="[['type_id.code', '=', 'SALES']]"/>
```

5. **Wire approval** by writing a flow template bound to your model (foundation.approval Enh 01):

```xml
<record id="crm.flow_pricing_approval" model="ir.approval.flow.template">
    <field name="subject_model_id" ref="ir.model.crm_quote"/>
    <field name="subject_team_field_id" ref="ir.model.field.crm_quote_team_id"/>
    <!-- ...steps with assignment_kind=TEAM_ROLE, team_role_id=... -->
</record>
```

(Pending foundation.approval Enh 01 ship — see [`roadmap/foundation/approval/enhancements/01-team-role-routing.md`](../roadmap/foundation/approval/enhancements/01-team-role-routing.md).)

---

## 6. Admin UI

**Settings → Teams & Roles** parent menu with four child leaves:

| Leaf | Model |
|---|---|
| Teams | `res.team` |
| Team Types | `res.team.type` |
| Team Roles | `ir.team.role` |
| Team Role Assignments | `res.team.role.assignment` |

The Teams form view embeds a **Role Assignments** notebook tab showing the team's assigned (role, user, sequence) rows grouped by role.

**RBAC:** all four models are readable by `internal_user` (so pickers populate everywhere) and writable only by `system_admin`. To grant write access to a non-admin role (e.g. `data_steward`), add an `ir.role.binding` row in your tenant's seed data.

---

## 7. Configuration

| `ir.config` key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `res.team.walk_up_default_enabled` | system | boolean | `false` | Default value for `walk_up` when `TeamRoleService.resolve` is called without explicit kwarg. Consumers pass it explicitly per-call today; this key is reserved for future tenant-level defaults. |
| `res.team.walk_up_max_depth` | system | integer | `5` | Safety cap on hierarchy walk depth. Prevents infinite loops on cyclic team data. |

Surfaced under **Settings → Teams & Roles → Team Configuration** (panel ships via `views/team_settings.xml` — currently the keys are declared but the panel is empty; future enhancement may add UI tunables here).

---

## 8. Why team-functional role ≠ RBAC role

| RBAC `ir.role` | Team-functional `ir.team.role` |
|---|---|
| Security boundary — what you CAN do | Process responsibility — what you DO |
| `sales_rep`, `sales_manager`, `system_admin` | `MANAGER`, `PRICING_APPROVER`, `ACCOUNT_MANAGER`, `REVIEWER` |
| Gates `ede.create`, `ede.update`, etc. at the ORM | Targeted by approval flow steps and workflow guards |
| Used by `AuthorizationService.can(resource, action)` | Used by `TeamRoleService.resolve(team, role_code)` |
| One role per permission boundary | Many functional roles can share a single RBAC role |
| Adding a new role triggers permission-design review | Adding a new role is a data row, no security review |

A single user holding RBAC `sales_manager` can simultaneously be the team-functional `MANAGER` on Mumbai West Sales (sequence=1), the `PRICING_APPROVER` on India Region (sequence=2), and the `COUNTRY_HEAD` on India HQ (sequence=1). The two vocabularies grow independently.

---

## 9. The M2M soft-FK degradation fix that came with this enhancement

The Phase 2 SDK shipped soft-FK degradation for Reference fields (when target table is absent from the metadata build batch, the column is created but no FK constraint emitted). It did NOT cover M2M join tables — those still hard-failed with `NoReferencedTableError` when the right-side target was absent.

This enhancement bundled the platform fix: `SqlAlchemyMetadataBuilder._build_join_tables` now skips the entire join table when either FK target is missing from the build batch. Same architectural contract as the Reference-field soft-degrade. Lets test fixtures that register `res.user` without `res.team` (or vice versa) build cleanly — the M2M field column won't be queryable in that test, but the metadata builds without exceptions.

See [src/ede/core/adapters/persistence/sqlalchemy/metadata_builder.py:_build_join_tables](../src/ede/core/adapters/persistence/sqlalchemy/metadata_builder.py).

---

## 10. Related

- [Roadmap: foundation.base Enhancement 06 — Team Substrate](../roadmap/foundation/base/enhancements/06-team-substrate.md)
- [Roadmap: foundation.approval Enhancement 01 — Team-Role Routing](../roadmap/foundation/approval/enhancements/01-team-role-routing.md) (consumer, hard prereq)
- [Roadmap: foundation.workflow Enhancement 01 — Team-Role Integration](../roadmap/foundation/workflow/enhancements/01-team-role-integration.md) (consumer, hard prereq)
- [Roadmap: logistics.sales-crm Enhancement 09 — Team Adoption](../roadmap/logistics/sales-crm/enhancements/09-team-adoption.md) (first user-visible consumer)
- [docs/foundation-base-extensions.md](foundation-base-extensions.md) (Phase 2 SDK — soft-FK degradation precedent)
