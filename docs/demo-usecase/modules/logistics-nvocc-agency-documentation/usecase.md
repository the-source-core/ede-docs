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
| 7 | Manifests → `assemble` → `mark_ready` → `submit` → `record_acceptance` | The issued B/L lands; any draft is **skipped and named** in `skipped_not_issued`. Filing refuses while ICEGATE is in `testing`: on the demo tenant, set ICEGATE's `environment=production` and activate it (BR-NAM-16). The record has no endpoint or credential, so nothing is actually sent |
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

---

## Enhancement 01 — demo-data expansion

**Status:** ✅ Delivered 2026-11-22 — 15 records in 2 files; `--with-demo=all` smoke green and idempotent (see the end of this section).
**Roadmap:** [nvocc-agency Enhancement 01](../../../../roadmap/logistics/nvocc-agency/enhancements/01-demo-data-expansion.md). The cross-module walkthrough order lives there, and step IDs match it.
**New demo files** (appended **after** `demo/demo_documentation.xml`, whose ICEGATE partner the IGM files to):
`demo/demo_documentation_bluewater_import.xml` (9) · `demo/demo_documentation_special_cargo_si.xml` (6)

### Use case 1 — the import B/L and the IGM at Nhava Sheva

> Bluewater's Singapore office drafted the house B/L for Wayne Imports' two boxes: two 40ft of LED
> driver boards on *Maersk Madrid* 2614W. At Nhava Sheva, Acme has to get that B/L onto the
> **Import General Manifest** and the IGM accepted by Indian Customs before any delivery order can
> issue. Then the customs examination finds that the carton marks say NHAVA SHEVA where the B/L
> says MUMBAI. That makes a post-issue amendment (Use case 3) and an IGM amendment necessary.

The IGM is the **same model** as the export manifest. `manifest_type = import` makes
`nvocc.manifest.assemble` match B/Ls on **port of discharge** instead of port of loading, so the
two ends of one voyage file against one object type. Nothing in the current demo has exercised
the import branch.

#### Records produced — `demo/demo_documentation_bluewater_import.xml` (9)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_documentation.demo_partner_role_lionasia_shipper` | `res.partner.role` | `base.demo_partner_co_047` (Lion Asia Trading, SG) · `sales_crm.role_master_shipper` | `validate_parties` checks the **role**, not just that the partner exists |
| `agency_documentation.demo_partner_role_wayne_consignee` | `res.partner.role` | `sales_crm.demo_partner_customer_wayne` · `sales_crm.role_master_consignee` | Wayne is classified only as a customer today |
| `agency_documentation.demo_partner_role_wayne_notify` | `res.partner.role` | Wayne · `sales_crm.role_master_notify_party` | A notify party that is **set but unclassified is a block**, not a warning |
| `agency_documentation.demo_numbering_rule_manifest_bluewater` | `nvocc.numbering.rule` | `document_type=manifest` · Bluewater · default org · `agency_numbering.sequence_manifest` · `prefix_override` BLWM · draft | The IGM's number series. The B/L deliberately has **no** Bluewater rule: it numbers from the global fallback `agency_numbering.demo_numbering_bl_global`, so most-specific-wins resolution falls through to the fallback for the first time on demo data |
| `agency_documentation.demo_bl_bluewater_import` | `nvocc.bl` | `bl_type=nvocc_bl` · Bluewater · `agency_booking.demo_booking_bluewater_import` · `booking_reference` BKG-BLWT-IMP-001 · shipper Lion Asia · consignee + notify Wayne · receipt / loading `masters.demo_port_sgsin` · discharge / delivery `masters.demo_port_innsa` · `masters.demo_vessel_maersk_madrid` · `voyage_number` 2614W · ETD 2026-10-01T18:00 · ETA 2026-10-06T06:00 · `freight_term=collect` · `release_type_code=telex` · 3 originals · place of issue SGSIN · **`marks_and_numbers` "WAYNE IMPORTS / MUMBAI / C/NO 1-960"** | Draft. **The marks are planted wrong**: the cartons say NHAVA SHEVA, and customs finds it at examination. It is a *header* field on purpose, because `nvocc.bl.amendment.apply` writes header fields only |
| `agency_documentation.demo_bl_cargo_bluewater_import` | `nvocc.bl.cargo` | `masters.demo_commodity_electronics` · "LED driver boards" · HS 8504.40 · `carton` 960 · 18 400 kg · 110 m³ · marks (same planted text) | Seeded directly, unlike the export B/L, because no SI exists at this end: the origin desk drafted the B/L. Lines on a **draft** B/L are writable; the guards bite only after issue |
| `agency_documentation.demo_bl_container_bluewater_box_1` / `_box_2` | `nvocc.bl.container` | `equipment_usage_id` → `agency_booking.demo_usage_bluewater_box_1` / `_2` · VGM 9 450 / 9 430 kg · `vgm_method=method_1` · 480 packages and 9 200 kg each | **Container and seal numbers are not typed.** `stamp_from_usage` reads them through from the usage and its live seal (BR-DOC-03) |
| `agency_documentation.demo_manifest_innsa_import_bluewater` | `nvocc.manifest` | **`manifest_type=import`** · Bluewater · default org · Maersk Madrid · 2614W · `port_id` INNSA · `arrival_date` 2026-10-06 · `cut_off_at` 2026-10-05T18:00 · `integration_partner_id` `agency_documentation.demo_integration_partner_indcus` | The IGM: draft, unnumbered, no lines. Lines assemble from **issued** B/Ls only (BR-N5-11) |

**Block split.** The three role rows and the numbering rule go in `noupdate="0"`. The B/L, its
three lines and the IGM go in **`noupdate="1"`**. Once the walkthrough issues the B/L,
`pre.nvocc.bl.update` refuses any rewrite on the loader path too (BR-N5-09 has no loader
carve-out). A second `--with-demo` on a walked tenant would then fail on an updatable B/L, and it
would also revert the amendment the reviewer applied.

### Use case 2 — two special-cargo shipping instructions, one gap each

> Globex's portal SI for the paint booking arrives complete, except that the cargo line says
> *hazardous, class 3, packing group II, flash point 18 °C* and **leaves the UN number blank**. Its
> emailed SI for the reefer booking quotes the **tolerance band** (−20 / −16 °C) and **drops the
> set-point**. Both bookings had the right values. No B/L exists yet, and that is exactly when
> this should be caught.

These are the two conditional cargo rules in `services/validation.py`, one fixture each:
`dg_details_incomplete` (class *and* UN number) and `reefer_setpoint_missing` (a band with no
set-point). There is no reefer flag anywhere: the set-point *is* the declaration, so a reefer
line that lacks one would be carried dry and plugged in by nobody.

#### Records produced — `demo/demo_documentation_special_cargo_si.xml` (6)

| External ID | Model | Key fields | Why it exists |
|---|---|---|---|
| `agency_documentation.demo_si_dg_globex` | `nvocc.shipping.instruction` | `SI-DEMO-0002` · `customer_reference` GLBX-DG-0412 · Sealink · `agency_booking.demo_booking_dg_sealink` · default org · `source=portal` · `status=received` · `received_at` 2026-11-30T10:15 · from Globex · shipper Globex · consignee + notify Stark · `nvocc_bl` · prepaid · 3 originals | SI `status` is a plain Enum, so demo data legitimately places it. Globex and Stark already hold shipper / consignee / notify roles from `demo_documentation.xml` and `sales_crm` |
| `agency_documentation.demo_si_cargo_dg` | `nvocc.si.cargo` | "Paint, flammable (industrial coatings in steel drums)" · HS 3208.90 · `drum` 80 · 16 000 kg · 28 m³ (identical to the booking, so SI↔booking reconciliation stays silent) · `hazardous=true` · `hazard_class` 3 · `packing_group` II · `flash_point_c` 18 · **`hazmat_un_number` empty** · marks "GLOBEX / SINGAPORE / DR 1-80" | **The planted error.** It is the only blocking finding the SI produces |
| `agency_documentation.demo_si_container_dg` | `nvocc.si.container` | `container_number` SEAU2200402 (text only) · seal SLK-DG-0402 · `equipment_type_code` 22GP · VGM 16 600 vs gross 16 000 (3.75%, inside the 5% tolerance) · `vgm_method=method_1` · 80 packages | Without a container line the SI would add a second blocker (`containers_missing`). With no usage it adds one **warning** (`container_not_allocated`), which is true: the box has not been allocated yet |
| `agency_documentation.demo_si_reefer_globex` | `nvocc.shipping.instruction` | `SI-DEMO-0003` · `customer_reference` GLBX-RF-0413 · Sealink · `agency_booking.demo_booking_reefer_sealink` · default org · `source=email` · `status=received` · `received_at` 2026-11-30T11:40 · from Globex · shipper Globex · consignee + notify Stark · `nvocc_bl` · prepaid · 3 originals | The reefer booking's instruction |
| `agency_documentation.demo_si_cargo_reefer` | `nvocc.si.cargo` | `masters.demo_commodity_perishable` · "Frozen shrimp, IQF, glazed" · HS 0306.17 · `carton` 1 200 · 24 000 kg · 58 m³ (identical to the booking) · `temperature_min_c` −20 · `temperature_max_c` −16 · **`temperature_setting_c` empty** · marks "GLOBEX / SINGAPORE / CTN 1-1200" | **The planted error.** A band with no set-point is a block, not a warning |
| `agency_documentation.demo_si_container_reefer` | `nvocc.si.container` | `container_number` SEAU4500501 (text only) · seal SLK-RF-0501 · `equipment_type_code` 42RT · VGM 24 800 vs gross 24 000 (3.3%) · `vgm_method=method_1` · 1 200 packages | Same reasoning as the DG container line: one true warning, no second blocker |

**Block split.** All six SI records sit in **`noupdate="1"`**. The walkthrough validates and
applies both instructions and fixes their cargo lines, and a reload would put them back to
`received` with the planted errors. **Known gap, unchanged:** the older `SI-DEMO-0001` in
`demo_documentation.xml` is updatable, so a reload after its own walkthrough reverts it the same
way.

### What this slice deliberately does NOT ship

| Not shipped | Why |
|---|---|
| An issued B/L, a filed or accepted IGM, or any manifest line | `nvocc.bl.status` and `nvocc.manifest.status` are `workflow=True`, and manifest lines assemble only from issued B/Ls (BR-N5-11) |
| A B/L amendment | It cannot exist in a fixture by construction: `pre.nvocc.bl.amendment.create` refuses against a non-issued B/L ("edit the draft directly"). The walkthrough walks into that refusal on purpose, then raises the real one after issue |
| A B/L for the DG booking | It would exist *before* the check whose whole point is to precede it. The walkthrough creates it with `nvocc.bl.create_from_booking` |
| A corrected DG or reefer SI | Fixing them is the exercise |
| A Bluewater-specific B/L numbering rule | Leaving it out is what makes the global fallback carry the import B/L |

### Walkthrough — the steps this module owns

| Step | Do this | What it shows |
|---|---|---|
| UC1-1 | Documentation → Manifests → the Bluewater IGM (INNSA, *Maersk Madrid* 2614W) → `nvocc.manifest.assemble`, **before** the B/L is issued | `bls_added: 0`. `skipped_not_issued` **names** the draft B/L rather than silently omitting it (BR-N5-11) |
| UC1-2 | Try to create an `nvocc.bl.amendment` against the import B/L | **Refused**: *"…is not yet a document of title — edit the draft directly"*. This sets up Use case 3 |
| UC1-3 | B/L: `submit_for_review` → `send_to_customer` → `record_customer_decision(approved)` → `approve` → approve the case → `issue` → `nvocc.bl.telex_release` | Stands in for origin issuance. The number comes from the **global** B/L rule. `telex_released` counts as released for the readiness contract, because the originals were surrendered at Singapore |
| UC1-5 | IGM → `assemble` → `mark_ready` → `submit` → `record_acceptance` | Two lines (one per container), matched on port of **discharge**. Filing is refused while ICEGATE is in `test`: on the demo tenant, set ICEGATE's `environment=production` and activate it (BR-NAM-16). The record has no endpoint or credential, so nothing is actually sent |
| UC3-3 | *(during the customs examination, release step UC1-10)* Edit the B/L's marks directly. Then raise `nvocc.bl.amendment`: `amendment_type=cargo`, `requested_by=customs`, reason, `new_values {"marks_and_numbers": "WAYNE IMPORTS / NHAVA SHEVA / C/NO 1-960"}` → `submit` → `record_decision(approved)` → `apply` | The direct edit is **refused** and names the amendment path (BR-N5-09). The amendment lands with `revision_number` 2 and the old value captured from the live record |
| UC3-4 | IGM → `assemble` again → then `nvocc.manifest.amend` with a reason | `assemble` is **refused**: the IGM is filed, so its lines are frozen. `amend` raises a new manifest linked by `amends_manifest_id` (BR-N5-13). This is the IGM amendment the examination made necessary |
| UC2-4 | Shipping Instructions → SI-DEMO-0002 → `validate`. No B/L exists for this booking | *(Verified live.)* Status `under_validation`, with **one** blocking finding, `dg_details_incomplete`: *"cargo line 1 is declared hazardous but is missing UN number. A terminal will refuse the box, and the refusal happens after the B/L is issued."* There is one warning, `container_not_allocated`. Validation is a report; it never raises |
| UC2-5 | Switch the principal switcher to **Sealink** (try it first without: refused, *"No active principal… `nvocc.principal_mandatory` is enabled (BR-NAM-01)"*). Then `nvocc.bl.create_from_booking` on the DG booking, which is **still draft**, and `apply_to_bl` with SI-DEMO-0002 | *(Verified live.)* The B/L is created as a draft and the booking stays draft: B/L creation does not read booking status. The apply is **refused** with the same message as the finding. The block bites at the act |
| UC2-6 | Set UN 1263 on the SI cargo line → `validate` → `apply_to_bl` | *(Verified live.)* `validated`, then `applied`. The draft B/L's cargo line carries UN 1263 / class 3 / PG II / flash point. Nothing was issued, so nothing needs amending, which is the whole case for catching it at SI stage |
| UC2-7 | Shipping Instructions → SI-DEMO-0003 → `validate`. Then set `temperature_setting_c` −18 on its cargo line and validate again | *(First run verified live.)* **One** blocking finding, `reefer_setpoint_missing`: *"cargo line 1 declares a temperature tolerance band but no set-point… without it the box will be carried dry."* Plus the same single `container_not_allocated` warning. With the set-point added it becomes `validated`. The booking had −18 all along; the gap was on the document the B/L is built from |

### Dependencies and load order

`agency_booking` (the Bluewater booking, usages and seals; the DG and reefer bookings). This
module declares `agency_booking` in its manifest `depends`. Enhancement 01 added the edge; before
that, booking loading first was only an alphabetical tiebreak. ·
`agency_numbering` (`sequence_manifest`, the global B/L rule) · this module's own
`demo_documentation.xml` (ICEGATE; Globex's shipper and Stark's notify roles) · `sales_crm`
(Wayne, Stark, Globex, role masters) · `foundation.base` (`base.demo_partner_co_047`) ·
`logistics_base.masters` (ports, vessel, commodity). Load with `--with-demo=all`.

### Smoke test — ✅ green and idempotent, 2026-11-22

`ede migrate upgrade -t agencydemo3 --with-demo=all` on a fresh PostgreSQL tenant, run twice, exit
0, zero loader errors. Actual counts match the prediction exactly:

| File | Pass 1 | Pass 2 |
|---|---|---|
| `demo_documentation_bluewater_import.xml` | created=9 | created=0 · updated=4 · skipped=5 (B/L, 3 lines, IGM) |
| `demo_documentation_special_cargo_si.xml` | created=6 | created=0 · updated=0 · skipped=6 |

All 15 external ids are `is_demo = true`. The loaded B/L container lines read
`BLWU4200308 / BLW-771201` and `BLWU4200313 / BLW-771202`, stamped from the usages. The IGM is
`import`, `draft`, voyage 2614W.

**Reload after the walkthrough (agencydemo3):** with the Bluewater tariff approved, the reefer booking confirmed and the DG SI validated through the real commands, a third `--with-demo=all` run exits 0 with zero loader errors, and all three keep their walked state.

Then check: **Manifests** shows a second manifest of type *Import Manifest* for INNSA;
**Shipping Instructions** shows SI-DEMO-0002 and SI-DEMO-0003 in `received`; the import B/L shows its two
containers with seal numbers `BLW-771201` / `BLW-771202` that nobody typed.
