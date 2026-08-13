# `logistics_nvocc_agency.agency_masters` — Demo Use-Case

**Module:** NVOCC / Shipping Agency (`src/domains/logistics_nvocc_agency/agency_masters/`)
**Manifest key:** `demo: ["demo/demo_agency.xml"]`
**Load:** `ede migrate upgrade -t <tenant> --with-demo=logistics_nvocc_agency.agency_reports`
**Roadmap:** [roadmap/logistics/nvocc-agency/README.md](../../../../roadmap/logistics/nvocc-agency/README.md)

---

## The story this data tells

Acme Forwarding Ltd. (Mumbai HQ) — the shared demo organisation — does not only forward
freight. It also acts as the **local shipping agency** for two principals, which is exactly
the NVOCC/agency business BRD N1 describes:

| Principal | Type | Why it is in the demo |
|---|---|---|
| **Maersk Demo Line** (`PRN-MAERSK`) | Shipping Line | The main principal — carries the agent network, a numbering rule and an integration partner. |
| **Bluewater NVOCC** (`PRN-BLUEWATER`) | NVOCC Principal | A second principal, so entitlement and switching have something to distinguish between. Ships with `suspension_reason` pre-filled. |

**Both load in `draft`.** `nvocc.principal.status` is `workflow=True`, so the ORM guard
forbids writing it directly and demo data must not try — a fixture that bypassed the
guard would demonstrate a lifecycle that does not exist. Driving them onward is the
reviewer's first exercise:

```
nvocc.principal.submit  → pending_approval   (guard: mandatory fields complete)
nvocc.principal.approve → active             (guard: effective date reached)
nvocc.principal.suspend → suspended          (guard: reason captured — pre-filled on Bluewater)
```

Each principal's **legal identity lives on a shadow `res.partner` row that the
delegation kernel creates automatically** — the demo sets `name`, `country_id` and
`email`, and those route through `partner_id` rather than onto the principal's own
table. Verified in a loaded tenant: `nvocc_principal.name` is NULL and the value sits on
the partner. Only the agency-owned columns (`principal_code`, `principal_type`,
`document_prefix`, `is_active`) are written locally.

That is also why the code field is `principal_code` and the flag is `is_active`: a field
named `code` or `active` would be captured by `res.partner` and this model's own column
would never be written.

## Records shipped

| Model | Records | Illustrates |
|---|---|---|
| `nvocc.principal` | 2 — both `draft`, driven by workflow commands | NAM-01, NAM-24, BR-NAM-05/06 |
| `nvocc.principal.entitlement` | 3 | NAM-02, NAM-03, NAM-04 |
| `nvocc.principal.agent` | 2 | NAM-13 |
| `nvocc.numbering.rule` | 2 | NAM-19, BR-NAM-12 |
| `nvocc.integration.partner` | 2 — one `active` production, one `testing` | NAM-18, BR-NAM-16 |
| `ir.rbac.role.binding` | 3 | NAM-23, NAM-25 — see below |

### Role bindings — why the demo ships them and production must not

The maker-checker step routes to the `agency_principal_admin` RBAC role.
`AssignmentService.resolve` looks up bindings for that **exact** role code and does
**not** walk role inheritance — so a user bound only to `agency_system_admin` (which
inherits from principal admin) still leaves the step unassigned, and Submit for
Approval fails with *"No one is assigned to approve this step yet"*.

The demo therefore binds `Administrator` directly to `agency_principal_admin`
(GLOBAL) so the flow is demonstrable end to end, plus `agency_system_admin` for
full control, and gives Priya `agency_operations_admin` (ORG-scoped) — keeping
maker and checker as different people, which is the point of maker-checker.

**Production seeds none of these.** A module that granted approval authority on
install would be a security anti-pattern; the administrator assigns roles under
Settings → Security → Role Bindings, which is exactly what the error message says.

### Entitlements — the point of the three rows

The three entitlement rows exist to make the *rules* visible, not just the model:

1. `base.admin_user` → Maersk, **no branch** (`organization_id` empty), `full` — the
   head-office administrator who acts for the principal from anywhere.
2. `base.demo_user_priya` → Maersk, **branch-scoped** to `base.default_organization`,
   `booking` — the branch operator, limited in both scope and level.
3. `base.demo_user_priya` → Bluewater, `finance`, **`can_switch=false`** — visibility
   without the ability to transact as that principal.

Together they demonstrate: branch-agnostic vs branch-scoped grants, the access-level
ladder, and the switch/visibility distinction — the three things NAM-02/03/04 turn on.

### Integration partners — why one is `testing`

`Mumbai Port Terminal` ships `active` + `production` and may exchange. `Singapore PSA`
ships `testing` — so BR-NAM-16 ("EDI/API partner setup must be active before automated
exchange is enabled", and *testing does not count*) has a record that demonstrates the
refusal rather than only the pass.

### Numbering rules — why two

One global fallback (no principal, no branch) for `bl`, and one Maersk-specific `cro`
rule carrying a `prefix_override`. Together they demonstrate most-specific-wins
resolution and the prefix-override mechanism that lets one sequence serve several
principals without cloning it.

## What this demo deliberately does NOT ship

- **No tax, GST/PAN, or finance-mapping records.** Those are NAM-15/16/17 — blocked on
  `foundation.l10n` and an open finance-layer decision. Shipping placeholder data would
  imply a capability that does not exist.
- **No container records.** Container demo data is base-tier and ships with
  `logistics_base.equipment_operations`. This module contributes only the `principal_id`
  link, which the equipment demo sets where relevant.
- **No transactional records** (bookings, CROs, BLs). BRD N1 is the *foundation* layer;
  those belong to downstream BRDs and their own modules.

## Cross-module references consumed

| Ref | Owner | Used for |
|---|---|---|
| `base.default_organization` | foundation.base | Branch scoping on entitlements and numbering |
| `base.admin_user` | foundation.base | Head-office entitlement subject (production seed) |
| `base.demo_user_priya` | foundation.base | Branch-scoped + finance entitlement subject (demo seed) |
| `base.demo_partner_co_001` / `_co_002` | foundation.base | Origin and destination agent parties |
| `base.demo_partner_co_003` | foundation.base | Mumbai Port Terminal integration party |
| `base.country_dk` / `base.country_sg` | foundation.base | Principal country of registration |
| `agency_numbering.sequence_bl` / `agency_numbering.sequence_cro` | this module | Numbering-rule bindings |
| `agency_integration.msgtype_codeco` / `msgtype_coarri` | this module | Message types on the integration partners |

## Smoke test

```bash
ede migrate upgrade -t <tenant> --with-demo=logistics_nvocc_agency.agency_reports
```

Expect **14 records** created on a fresh tenant; a re-run is idempotent
(`created=0 updated=11`). The agency demo references `foundation.base` demo users and
partners, so run it as part of `--with-demo=all` (or after the base demo) — on its own
the cross-module refs cannot resolve.

Then check **Agency → Principals** shows both principals in Draft, and
**Agency → Entitlements** shows the three grants.
