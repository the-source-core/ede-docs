# `foundation.base` — Demo Use-Case

**Module:** `ede.foundation.base`
**App key:** `foundation.base`
**Demo manifest entries** (target): `demo/demo_org.xml`, `demo/demo_users.xml`, `demo/demo_partners.xml`
**Status:** 🔴 Not yet authored

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

