# `foundation.base` — Demo Use-Case

**Module:** `ede.foundation.base`
**App key:** `foundation.base`
**Demo manifest entries** (shipped): `demo/demo_partners.xml`, `demo/demo_application_view.xml`, `demo/demo_user_access.xml`
**Demo manifest entries** (target): `demo/demo_org.xml`, `demo/demo_users.xml`
**Status:** 🟡 Partial — `demo_partners.xml`, `demo_application_view.xml`, `demo_user_access.xml` (Enh 14) shipped; org/users cohort pending

---

## Use-case

Bootstrap the **Acme Forwarding Ltd. (Mumbai HQ)** demo tenant with the minimum identity + party records every downstream demo file refers to:

- One demo organisation unit (the company you log into as).
- A demo user per role used by the rest of the scenario (admin, sales rep, ops manager, finance approver) with stable known passwords for walkthrough demos.
- Three demo partners (one customer, one carrier, one vendor) so the CRM and pricing demo files have someone to transact with.

Without this module's demo data nothing downstream resolves — every other demo file `ref=`s at least one record from here.

## Records produced

### `demo/demo_org.xml`

| External ID | Model | Notes |
|---|---|---|
| `base.demo_org_acme_mumbai` | `res.organization` | `name="Acme Forwarding — Mumbai"`, `country_id=ref(res_country.IN)`, `default_language_id=ref(base.res_language_en)` |

### `demo/demo_users.xml`

| External ID | Model | Role binding(s) |
|---|---|---|
| `base.demo_user_admin` | `res.user` | `system.admin` |
| `base.demo_user_sales_rep` | `res.user` | `sales_crm.sales_rep` |
| `base.demo_user_ops_manager` | `res.user` | `pricing.ops_manager` + workflow approver |
| `base.demo_user_finance` | `res.user` | `pricing.finance_approver` |

Every demo user belongs to `base.demo_org_acme_mumbai`. Passwords are `demo` (with `is_demo=True` tagging this is acceptable; production seeds never ship default passwords).

### `demo/demo_partners.xml`

| External ID | Model | Notes |
|---|---|---|
| `base.demo_partner_customer_globex` | `res.partner` | role: customer; `country_id=IN` |
| `base.demo_partner_carrier_maersk` | `res.partner` | role: carrier; `country_id=DK` |
| `base.demo_partner_vendor_mpt` | `res.partner` | role: vendor; `country_id=IN`; address: Mumbai Port |

Plus one address row per partner under `res.partner.address` (billing / pickup / delivery as needed) so logistics pricing + sales-CRM demo records can `ref=` them.

### `demo/demo_application_view.xml` (customization Phase 4B)

| External ID | Model | Notes |
|---|---|---|
| `base.demo_appview_org_form_internal_notes` | `ir.application.view` | `mode=extension`, `owner=user`, `parent_id=ref(base.view_res_organization_form_view)` — adds an "Internal Notes" section after "General" on the organization form, proving DB-backed composition over the file-synced primary. Full scenario: [`foundation-customization/usecase.md`](../foundation-customization/usecase.md). |

### `demo/demo_user_access.xml` (User Management 360 — Enhancement 14) ✅

Populates one demo user with a **full access picture** so the "Manage access" 360 console is non-empty on a demo tenant. Reuses the seeded roles (`rbac.role_internal_user`, `rbac.role_portal_user`), a seeded permission (`base.p_res_partner_update`), the default organization, the admin user, and the Asia/Kolkata timezone.

| External ID | Model | Notes |
|---|---|---|
| `base.demo_org_west_branch` | `res.organization` | Branch under `base.default_organization` (`parent_id` set) — gives BRANCH scope a real target per Enh 07 root/branch model |
| `base.demo_user_priya` | `res.user` | Priya Deshmukh; `res.user.register`; org = `base.default_organization`; password `demo` |
| `base.demo_team_type_desk` | `res.team.type` | `DESK` team-type vocabulary row |
| `base.demo_team_role_approver` | `ir.team.role` | `APPROVER`, `is_managerial=true` |
| `base.demo_team_pricing_west` | `res.team` | "Pricing Desk – West" on the default org, type = DESK |
| `base.demo_seat_priya_pricing` | `res.team.role.assignment` | Priya seated as APPROVER on the Pricing Desk team |
| `base.demo_binding_priya_global` | `ir.rbac.role.binding` | GLOBAL-scope internal_user role binding |
| `base.demo_binding_priya_org` | `ir.rbac.role.binding` | ORG-scope (`scope_id=default_organization`) internal_user binding |
| `base.demo_binding_priya_branch` | `ir.rbac.role.binding` | BRANCH-scope (`scope_id=demo_org_west_branch`) portal_user binding |
| `base.demo_grant_priya_partner_update` | `ir.rbac.access.grant` | Time-limited direct grant of `p_res_partner_update`, expires 2026-12-31 |
| `base.demo_ooo_priya` | `ir.user.out_of_office` | Out-of-office window (2026-07-20 → 07-27), delegate = admin |

## Out of scope

- Country / language / UOM / currency seeds — those are **production** masters and stay in `data/*.csv`. They are not demo data.
- RBAC role definitions — production seed in `data/rbac_roles.xml`. Demo only adds *bindings* between demo users and existing roles.
- RBAC permission rows — also production seeds (`data/ir.rbac.permission.csv`).
- Sample chatter / followers — those land in `foundation.communication`'s demo file once the demo users + partners exist here.

## Dependencies (must already be loaded)

- Production data of `foundation.base` itself: `res.country.csv`, `res.language.csv`, `res.uom.csv`, `rbac_roles.xml`, `ir.rbac.permission.csv` — all run before any `demo/*.xml` because the demo pass executes strictly after the data pass.

## Verification

```
ede migrate upgrade -t demo --with-demo=foundation.base
# Expect:
#   demo_load: 7 created, 0 updated, 0 skipped (scope=foundation.base)
# (1 org + 4 users + 3 partners — adjust counts as scope grows)
```

Query check:
```sql
SELECT module, name, model_key, is_demo FROM ir_data_reference
 WHERE module='base' AND is_demo=true
 ORDER BY name;
```

## Authoring order

1. `demo_org.xml` first — every user references it.
2. `demo_users.xml` — uses `ref=` to bind to roles + the org above.
3. `demo_partners.xml` last — independent of users but loaded after for readability.

Manifest order in `__manifest__.py`:

```python
"demo": [
    "demo/demo_org.xml",
    "demo/demo_users.xml",
    "demo/demo_partners.xml",
],
```

---

*Back to [demo-usecase index](../../README.md).*

## Recorded e2e tests

### TestRateExampleTest
- File: [src/tests/e2e/foundation/base/test_rate-example-test.py](../../../../src/tests/e2e/foundation/base/test_rate-example-test.py)
- Class: `TestRateExampleTest` · Method: `test_rate_example_test`
- Module: `foundation.base`
- Recorded via `ede e2e record` — re-record with `ede e2e record foundation_base/rate-example-test --force`.

