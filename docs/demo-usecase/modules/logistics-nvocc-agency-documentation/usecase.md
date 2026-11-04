# Demo Use-Case — `logistics_nvocc_agency.agency_documentation`

**Module:** `agency_documentation` (BRD N5 — BL, Manifest & Documentation Control)
**Demo file:** `src/domains/logistics_nvocc_agency/agency_documentation/demo/demo_documentation.xml`
**Scenario:** Acme Forwarding Ltd. (Mumbai HQ) — see [the unifying scenario](../../README.md)

---

## The story this data tells

`agency_booking` left Globex's booking under **Sealink Container Line** on the tenant: two 40ft dry
containers, Nhava Sheva → New York. This module continues from there:

> Globex emails its **shipping instruction** for the booking. The agency has already stuffed and
> sealed both boxes — and one of them was re-sealed after a customs examination at Nhava Sheva, which
> Globex does not know. The instruction quotes the **old seal**, and its VGM for that box contradicts
> the declared gross weight by 42%.
>
> The documentation desk validates the instruction, catches both errors *before* anything is issued,
> gets them corrected, turns the instruction into a bill of lading, issues it, files the export
> manifest to Indian Customs, and works the load list at the quay — where one of the two boxes is
> short-loaded.

**The two errors are the point of the fixture.** A demo whose data is already perfect never shows you
the control. These are the exact failures N5-07 exists to catch, and catching them at SI stage costs
a phone call while catching them after issuance costs an amendment fee and a week.

---

## Records shipped (19)

| # | Record | Model | Notes |
|---|---|---|---|
| 1–2 | `demo_binding_admin_documentation_manager` / `_executive` | `ir.rbac.role.binding` | Admin ↔ the two new roles. Required because `AssignmentService` resolves a ROLE approval step by **exact code** and does not walk role inheritance — a user bound only to `agency_system_admin` still leaves the Documentation Manager step unassigned. **Demo only**: a module must never grant approval authority on install |
| 3 | `demo_partner_role_globex_shipper` | `res.partner.role` | Globex as **shipper**. The base demo classifies it as customer + billing party only, and `validate_parties` checks the role, not just existence |
| 4 | `demo_partner_role_stark_notify` | `res.partner.role` | Stark as **notify party** — same reason |
| 5 | `demo_numbering_rule_bl_sealink` | `nvocc.numbering.rule` | `bl` × Sealink × default org → the `BL` sequence, `prefix_override = SLKB`. Lands in **draft** on purpose |
| 6 | `demo_numbering_rule_manifest_sealink` | `nvocc.numbering.rule` | `manifest` × Sealink, `prefix_override = SLKM`. Also **draft** |
| 7 | `demo_integration_partner_indcus` | `nvocc.integration.partner` | Indian Customs ICEGATE, `environment=test`. Lands in **draft** so step 7 walks into the BR-NAM-16 refusal |
| 8–9 | `demo_usage_box_001` / `_002` | `logistics.equipment.usage` | The two boxes' allocations. Container identity on a B/L is **read through** from these, never re-typed (BR-DOC-03) |
| 10 | `demo_seal_box_001` | `logistics.equipment.seal.application` | `SLK-334455`, still on the box |
| 11 | `demo_seal_box_002_original` | `logistics.equipment.seal.application` | `SLK-445566`, applied then **removed** at the customs examination |
| 12 | `demo_seal_box_002_current` | `logistics.equipment.seal.application` | `SLK-556677`, the successor. **This trio is why "the current seal" means the newest row with no `removed_at`, not simply the newest row** — reading it the naive way would print a seal cut off days ago |
| 13 | `demo_si_globex_001` | `nvocc.shipping.instruction` | `SI-DEMO-0001`, source `email`, status **`received`** — the SI's status is a plain Enum, not `workflow=True`, so demo data legitimately places it |
| 14 | `demo_si_cargo_001` | `nvocc.si.cargo` | Mirrors the booking line exactly, so the SI↔booking reconciliation passes until you change it |
| 15 | `demo_si_container_001` | `nvocc.si.container` | MSKU4200001 — clean: correct seal, VGM 11 050 kg against 10 750 kg declared (2.8%, inside the 5% tolerance) |
| 16 | `demo_si_container_002` | `nvocc.si.container` | TCLU5500001 — **the two planted errors**: seal `SEAL-99999` (the cut-off one) and VGM 30 500 kg against 10 750 kg declared |
| 17 | `demo_bl_sealink_001` | `nvocc.bl` | **Draft**, route + vessel only, **no cargo and no containers**. Those arrive from the SI in step 4, which is the exercise |
| 18 | `demo_manifest_innsa_export` | `nvocc.manifest` | Export manifest for the sailing. **Draft, unnumbered, no lines** — lines assemble from *issued* B/Ls |
| 19a | `demo_load_list_innsa` | `nvocc.stowage.list` | `list_type=load` at INNSA |
| 19b | `demo_discharge_list_usnyc` | `nvocc.stowage.list` | `list_type=discharge` at USNYC — **the same model**, seen from the other end of the voyage |
| 20 | `demo_readiness_globex` | `nvocc.documentation.readiness` | `is_ready=false`. A **cache** for the register; nothing reads it as authority |

---

## What this demo deliberately does not ship

**1. No issued B/L, and no manifest lines.**
`nvocc.bl.status` and `nvocc.manifest.status` are `workflow=True`, so the ORM guard rejects a direct
status write. Every record lands in its initial stage and the reviewer drives it with the real
commands. A fixture that issued a document of title by writing a column would demonstrate
post-issuance immutability that does not exist — which is the one thing this phase is for.

Manifest lines are assembled from **issued** B/Ls by `nvocc.manifest.assemble`, and there are none
until step 5. A hand-written line would show a manifest that skipped BR-N5-11.

**2. No active numbering rules or integration partner.**
All three land in `draft`. Steps 6 and 7 refuse until they are activated, naming what is missing
(BR-NAM-12 / BR-NAM-16). **Walk into those refusals once on purpose** — they are the control, not a
defect.

**3. No corrected SI.**
The errors stay in the fixture. Fixing them is step 3.

---

## Walkthrough

```bash
ede migrate upgrade -t <tenant> --with-demo=logistics_nvocc_agency.agency_documentation
```

| # | Do this | What it shows |
|---|---|---|
| 1 | Agency → Setup → Principals → SEALINK: `submit` → approve → `active` | Only needed if the `agency_booking` demo has not been walked |
| 2 | Agency → Configuration → Numbering Rules: submit + approve **both** the B/L and manifest rules | Skip it and step 6 refuses, naming the missing rule (BR-NAM-12) |
| 3 | Documentation → Shipping Instructions → SI-DEMO-0001 → `validate` | **Four findings, no exception.** Validation is a *report*, not a refusal — the desk sees every problem at once. Two block: `vgm_out_of_tolerance` (naming 30 500 kg vs 10 750 kg, 183.7% drift against a 5% tolerance) and `seal_mismatch` (naming `SEAL-99999` and `SLK-556677`). Fix both on the container line and re-run |
| 4 | `nvocc.shipping.instruction.apply_to_bl` → the draft B/L | Parties, cargo and containers copy across. **Container and seal numbers do not** — the B/L re-reads them live from the usage (BR-DOC-03), so even a stale number on the SI lands correct. Try applying to an issued B/L instead: refused, naming the amendment path |
| 5 | B/L: `submit_for_review` → `send_to_customer` → `record_customer_decision(approved)` → `approve` → approve the case → `issue` | Issue allocates the number through the rule from step 2. **Then try editing the consignee — REFUSED**, naming the amendment path (BR-N5-09) |
| 6 | `nvocc.bl.amendment` → create → `submit` → `record_decision(approved)` → `apply` | `revision_number` → 2, and the result reports `customer_approval_invalidated: true`, because the customer approved revision 1 |
| 7 | Manifests → `assemble` → `mark_ready` → `submit` → `record_acceptance` | The issued B/L lands; any draft is **skipped and named** in `skipped_not_issued`. Filing refuses while ICEGATE is in `testing` — activate it first (BR-NAM-16) |
| 8 | `nvocc.bl.cancel` on the manifested B/L | **REFUSED, naming the manifest** that depends on it (BR-N5-10) |
| 9 | Load / Discharge Lists → the INNSA load list → `assemble` → `record_outcome(TCLU5500001, short_loaded)` → `finalise`. Then record an over-landed box on the discharge list | Discrepancy count is 1. The over-landed box is **appended, not refused** — a container on no list is the definition of over-landed |
| 10 | Dispatch `nvocc.documentation.readiness` with the booking id, **before and after step 7** | Before: `ready: false` with `missing[]` naming the unaccepted manifest. After: `ready: true`. **This is the contract Phase 6 consumes before it releases cargo** (N5-22) |

---

## Refs consumed

`base.default_organization` · `base.admin_user` · `masters.demo_port_innsa` / `_usnyc` ·
`masters.demo_vessel_maersk_madrid` · `masters.eq_type_42gp` · `masters.demo_commodity_fak` ·
`masters.seal_type_bolt` · `sales_crm.demo_partner_customer_globex` / `_stark` ·
`sales_crm.role_master_shipper` / `_consignee` / `_notify_party` ·
`equipment_operations.demo_equipment_42gp_001` / `_carrier_42gp_001` ·
`agency_booking.demo_principal_sealink` · `agency_booking.demo_booking_sealink_001` ·
`agency_numbering.sequence_bl` / `sequence_manifest`

---

## Smoke test — green, and what it caught

```
ede migrate upgrade -t <tenant> --with-demo=all
```

✅ Two consecutive runs on PostgreSQL, exit 0, **zero loader errors, no duplicated rows**.

Note the flag: **`--with-demo=all`**, not this module alone. This fixture builds on
`sales_crm`'s partners, `equipment_operations`' containers and `agency_booking`'s principal and
booking — loading it in isolation leaves every one of those refs unresolved.

The run found three defects the module's 156 unit tests did not, each of a kind only a real database
surfaces:

1. **A role-name collision.** `ir.rbac.role.name` is unique and the base `documentation` module
   already seeds *"Documentation Manager"*. The clash failed on `uidx_ir_rbac_role_name` and took
   **all 75 permission rows** with it, because each refs its role by xml id and the role never
   landed. Both roles are now prefixed *"Agency …"*.
2. **A container another fixture deliberately holds.** BRD N4's demo puts `MSKU4200002` on a customs
   hold *on purpose*, so a reviewer can watch an allocation be refused — and this fixture tried to
   allocate it. Box 2 is now `TCLU5500001`, the carrier-owned 40GP.
3. **Append-only rows in an updatable block.** `logistics.equipment.seal.application` refuses every
   update by design, so `noupdate="0"` loaded clean on run 1 and failed three records on run 2. The
   seal rows now sit in their own `noupdate="1"` block.

A fourth was self-inflicted: the XML comment written to explain finding 1 contained the literal
`--with-demo`, and **an XML comment may not contain a double-hyphen** — the roles file stopped
parsing entirely.
