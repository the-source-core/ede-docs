# Demo Use-Cases — Per-Module Catalogue

> **Source of truth for what `ede migrate upgrade -t <tenant> --with-demo=...` ships per module.**
>
> Read this before authoring or reviewing a `demo/*.xml` file in any app.

---

## What this directory is

Every active EDE module declares an opt-in `demo: [...]` list in its `__manifest__.py` pointing at XML/CSV files under `<app_root>/demo/`. Those files are loaded by the platform demo-data loader ([roadmap/platform/03-demo-data-loader.md](../../roadmap/platform/03-demo-data-loader.md)) when a migration is run with `--with-demo=all` or `--with-demo=<app_key>`.

This directory is the **specification** for *what* each module's demo files contain — the use-case the data illustrates, the records produced, and how those records knit together across modules. The XML/CSV files themselves live next to the code (`<app_root>/demo/`); this catalogue exists so a reviewer or new contributor can answer "what will happen if I run `--with-demo=all`?" without grepping every manifest.

## Why per-module spec docs

Demo data is part of every shipped feature, not a follow-up. The `preparing-demo-data` skill enforces that no Phase / Enhancement flips ✅ Delivered without (a) updated demo data under `<app_root>/demo/`, (b) the matching `demo: [...]` manifest entry, and (c) an updated entry in the corresponding `modules/<module>/usecase.md` here. Without the spec doc, demo files drift into ad-hoc fixtures — exactly what `--with-demo` was built to replace.

---

## The unifying scenario — Acme Forwarding Ltd. (Mumbai HQ)

All modules write demo records against **one** consistent scenario so cross-module refs work and a `--with-demo=all` run produces a coherent demo tenant, not a Frankenstein of unrelated fixtures.

| Actor | Role | Used by |
|---|---|---|
| `Acme Forwarding — Mumbai` | Demo organisation unit | foundation.base, logistics.* |
| `demo_user_admin` | System admin (full RBAC) | foundation.base |
| `demo_user_sales_rep` | Sales rep — owns CRM records | foundation.base, logistics.sales-crm |
| `demo_user_ops_manager` | Operations manager — pricing approver | foundation.base, foundation.approval, logistics.pricing |
| `demo_user_finance` | Finance approver — high-value rates | foundation.base, foundation.approval |
| `Globex Inc.` (`demo_partner_customer_globex`) | Customer | foundation.base, logistics.sales-crm |
| `Maersk Demo Line` (`demo_partner_carrier_maersk`) | Ocean carrier | foundation.base, logistics.pricing |
| `Mumbai Port Terminal` (`demo_partner_vendor_mpt`) | Vendor / terminal | foundation.base |
| Lane `INMUM → SGSIN` | Mumbai (India) → Singapore | logistics.masters, logistics.pricing, logistics.sales-crm |
| Lane `SGSIN → INMUM` | Singapore → Mumbai (return) | logistics.masters, logistics.pricing |

Every module's demo XML must declare its records as part of (or in support of) this scenario. New scenarios are added by extending the table above — not by parallel storylines that confuse the loaded state.

## External-ID naming convention

All demo external IDs are prefixed `demo_` so they are easy to spot in `ir.data.reference` and can be `grep`-ed across all manifests:

```
<module-short-name>.demo_<entity>_<theme>
```

Examples:

```
base.demo_user_sales_rep
base.demo_partner_customer_globex
masters.demo_port_inmum
pricing.demo_rate_inmum_sgsin_lcl
sales_crm.demo_inquiry_globex_001
```

The same ID can be referenced from any module's demo XML once it is loaded — the demo pass shares the loader's `_local_refs` cache across apps in dependency order, so foundation.base demo IDs resolve from logistics.sales-crm demo XML without an explicit search.

## Cross-module dependency order

Demo files run in the dependency order set by `registry.sorted_app_specs()` — foundation first, then domains alphabetically. A demo file in app B can `ref=` records from app A iff `A` is loaded before `B`. The unifying-scenario table above is ordered so dependencies cascade cleanly.

## Shared fixtures that are deliberately UNAVAILABLE

Some fixtures exist to be refused. They are placed in a blocked state on purpose
so a reviewer can walk a real refusal, and a later module that reuses them
innocently gets that refusal instead of a working demo.

**This has already cost two sessions the same defect**, so it is recorded here
rather than in each module's file:

| Fixture | Placed by | State | What happens if you reuse it |
|---|---|---|---|
| `equipment_operations.demo_equipment_42gp_002` — **MSKU4200002** | BRD N4 demo | open customs hold ("selected for customs scanning at JNPT") | Any allocation is refused: *"container is on hold and cannot be allocated"*. Bit BRD N5's demo, then BRD N7's, both at `--with-demo` time. |
| `equipment_operations.demo_equipment_carrier_42gp_001` — **TCLU5500001** | BRD N7 demo | usage `USG-DND-0002`, deliberately no empty return | Reusing it for a *complete* cycle destroys the N7-24 missing-event walkthrough, which is the whole point of that box. |
| `equipment_operations.demo_equipment_42gp_001` — **MSKU4200001** | BRD N4 demo | usage `USG-DEMO-N4-01`, 01 Jul → 05 Aug 2026 | Not blocked, but **scheduled**: an overlapping window is rejected by the BR-USG-02..04 overlap rule. |

**The general rule this implies: prefer shipping your own container to reusing a
shared one.** A demo that allocates a shared box depends on another module's
*scheduling*, not merely on its data — so a future module adding a usage in your
window breaks your fixture silently, and neither file mentions the other. Owning
the record removes a coupling nobody declares. `agency_booking` set the precedent
with its own principal; `agency_charges` follows it with **SEAU4200103**.

If you place a fixture in a deliberately-blocked state, add a row above.

## Per-module specs

| Module | Spec | Demo files (planned / shipped) | Status |
|---|---|---|---|
| `foundation.base` | [modules/foundation-base/usecase.md](./modules/foundation-base/usecase.md) | demo_partners.xml ✅, demo_application_view.xml ✅, demo_user_access.xml ✅ (Enh 14); demo_org.xml + demo_users.xml pending | 🟡 Partial |
| `foundation.approval` | [modules/foundation-approval/usecase.md](./modules/foundation-approval/usecase.md) | demo_approval_rules.xml | 🔴 Not yet authored |
| `foundation.communication` | [modules/foundation-communication/usecase.md](./modules/foundation-communication/usecase.md) | _records on demo subjects in domain demo files_ | 🔴 Not yet authored |
| `foundation.customization` | [modules/foundation-customization/usecase.md](./modules/foundation-customization/usecase.md) | demo_properties.xml | 🔴 Not yet authored |
| `foundation.i18n` | [modules/foundation-i18n/usecase.md](./modules/foundation-i18n/usecase.md) | demo_language_extras.xml | 🔴 Not yet authored |
| `foundation.notifications` | [modules/foundation-notifications/usecase.md](./modules/foundation-notifications/usecase.md) | demo_preferences.xml | ✅ Delivered (Phase 2 preferences) |
| `foundation.workflow` | [modules/foundation-workflow/usecase.md](./modules/foundation-workflow/usecase.md) | _via demo records in domain modules_ | 🔴 Not yet authored |
| `foundation.jobs` | [modules/foundation-jobs/usecase.md](./modules/foundation-jobs/usecase.md) | (TBD — see usecase doc) | 🔴 Not yet authored |
| `foundation.dataset` | [modules/foundation-dataset/usecase.md](./modules/foundation-dataset/usecase.md) | demo_blueprints.xml, demo_metrics_seed_data.xml | 🔴 Not yet authored |
| `foundation.reporting` | [modules/foundation-reporting/usecase.md](./modules/foundation-reporting/usecase.md) | demo_reports.xml, demo_reports_categories.xml | 🔴 Not yet authored |
| `foundation.document` | [modules/foundation-document/usecase.md](./modules/foundation-document/usecase.md) | demo_document_templates.xml, demo/templates/*.dml | 🔴 Not yet authored |
| `foundation.dashboard` | [modules/foundation-dashboard/usecase.md](./modules/foundation-dashboard/usecase.md) | demo_kpis.xml, demo_dashboards.xml | ✅ Delivered (2026-07-10) — 4 KPIs + 2 dashboards on self-contained `base.*` metrics; Ops-Desk widget team-scoped |
| `foundation.agent` | [modules/foundation-agent/usecase.md](./modules/foundation-agent/usecase.md) | demo_agent.xml, demo_automation.xml | ✅ Delivered 2026-06-18 — 8 records (1 agent + 3 AI Automations + 4 actions); smoke + idempotency verified |
| `foundation.tax` | [modules/foundation-tax/usecase.md](./modules/foundation-tax/usecase.md) | demo_tax.xml | ✅ Delivered 2026-07-24 · **rewritten 2026-07-31 (D-10 / D-11)** — 18 records (6 tax codes + 4 rates + 2 compound components + 3 fiscal positions + 3 map rows) — India operational-tax setup on the Acme INMUM→SGSIN lane: ONE catalogue tax `IN-GST-18` remapped per jurisdiction (export zero-rated · intra-state CGST+SGST as a code **set** · inter-state IGST), plus a compound cess on the preceding total for the tax-on-tax path; `--with-demo` smoke green + idempotent |
| `foundation.presentation` | _no demo data — engine_ | _none_ | n/a |
| `logistics.masters` | [modules/logistics-masters/usecase.md](./modules/logistics-masters/usecase.md) | demo_ports.xml, demo_equipment.xml, demo_commodities.xml | 🔴 Not yet authored |
| `logistics.pricing` | [modules/logistics-pricing/usecase.md](./modules/logistics-pricing/usecase.md) | demo_rates.xml, demo_spot_rfqs.xml, demo_phase2.xml (+3) | ✅ Delivered 2026-07-02 — Phase 1 rates/contracts/spot + Phase 2 (version history, branch override, contract volume-utilization); 55 demo records, smoke + idempotency verified |
| `logistics.sales-crm` | [modules/logistics-sales-crm/usecase.md](./modules/logistics-sales-crm/usecase.md) | demo_partners.xml, demo_pipeline.xml, demo_quotes.xml | ✅ Delivered 2026-05-13 — 16 records (7 partners + 7 pipeline + 2 quotes) |
| `logistics.equipment-operations` | [modules/logistics-equipment-operations/usecase.md](./modules/logistics-equipment-operations/usecase.md) | demo_equipment.xml, demo_operations.xml | ✅ Delivered 2026-05-28 — 13 records (7 fleet units + 2 usage + 1 seal + 2 movement + 1 load-calc) on the INMUM→SGSIN lane |
| `logistics.shipments` | [modules/logistics-shipments/usecase.md](./modules/logistics-shipments/usecase.md) | demo_shipment.xml | ✅ Delivered 2026-06-17 — 8 records (1 shipment + 1 leg + 4 parties + 1 cargo + 1 charge) on the Globex INMUM→SGSIN lane; composed ref CO-SEA-D-2026-000001; postgres `--with-demo` smoke green + idempotent |
| `logistics.operations_execution` | [modules/logistics-operations-execution/usecase.md](./modules/logistics-operations-execution/usecase.md) | demo_execution.xml, demo_operations.xml | ✅ Delivered 2026-06-25 — 24 records (1 SOP + 1 execution + 1 plan + 2 legs + 4 tasks + 5 milestones + 1 completion + 1 provider + 1 pickup + 1 delivery + 1 dispatch + 1 driver + 1 cargo event + 2 cut-offs + 1 exception) — live Control Tower over the Globex INMUM→SGSIN shipment; `--with-demo` smoke green + idempotent |
| `logistics.profitability` | [modules/logistics-profitability/usecase.md](./modules/logistics-profitability/usecase.md) | demo_billing_readiness.xml, demo_profitability.xml, demo_proforma.xml | ✅ Delivered 2026-06-29 — 11 records (1 header + 6 charge lines + 1 currency line + 1 adjustment + 2 audit entries) — operational profitability over the Globex INMUM→SGSIN shipment (rev 4400 / cost 3190.24 / GP 1209.76 / margin 27.49%, incl. one multi-currency line). **Proforma billing (WS-3) ✅ 2026-07-27** — +13 records (2 customer proforma invoices split-billing the freight charge 2,000 + 1,200 → 2,550 / 1,500 after invoice 2's document discount + two vendor proforma bills, 2,400 and 60.24), linked to the charge lines; on load the freight charge syncs to `fully_invoiced`. **One-click billing (WS-4.1) ✅ 2026-07-29** — the BAF charge is deliberately left unbilled so the header's "Create Proforma Invoice" button succeeds on first click; the PFM-08 prerequisites it needs (partner role assignment, billing address, payment term) ship from `logistics.sales_crm`'s demo. `--with-demo=all` smoke green + idempotent. **Line currency translation + FX provenance (WS-4.3) ✅ 2026-08-10** — the second vendor bill carries the demo's one cross-currency line (5,000 INR → 60.24 USD), and the header's full PFM-11 provenance block (rate · type · date · source) derives rather than landing null. **Operational tax (WS-5 / WS-13) ✅ 2026-08-12 — inputs only, no new rows:** `tax_ids = [IN-GST-18]` on the six sea/ocean charge products in `logistics.pricing`'s demo is the source set, the three India fiscal positions come from `foundation.tax`'s demo, and the four documents load `tax_amount = 0.00` **by design** — `…apply_taxes` is an explicit operator command, so the figure materialises on **Apply Tax** rather than at load. **Billing readiness (WS-8) ✅ 2026-08-12** — `demo_billing_readiness.xml` ships the gate's two dials as real `ir.config` rows (`billing_readiness_enforced` / `zero_total_blocks`, both **advisory**) so they are visible and flippable on the Settings panel. The evaluator writes nothing, so there is no readiness table to seed — its demonstrable output is the **not-ready verdict** the deliberately mid-lifecycle Globex shipment produces, naming each failing check. Enforcing it in demo data would make grouping refuse and `demo_proforma.xml` fail to load. **Proforma numbering (WS-1) 🟡 — no demo rows**, which is exactly what holds it off ✅ |

| `logistics_nvocc_agency.agency_charges` | [modules/logistics-nvocc-agency-charges/usecase.md](./modules/logistics-nvocc-agency-charges/usecase.md) | demo_agency_charges.xml | ✅ Delivered 2026-08-18 — 20 records (1 42GP container + 1 free-time term + 2 usages + 5 movement events + 1 pricing rate with 2 lines and 5 slabs + 3 demo role bindings) on the Sealink INNSA import cycle. **Seeds inputs, not outputs:** no calculation, waiver, dispute or hold is shipped, because each is a lifecycle and the number is the engine's product — the reviewer runs `nvocc.charge.calculate` and walks the refusals on purpose (an unapproved tariff, then a container that never came back). ✅ `--with-demo` smoke **green and idempotent** on PostgreSQL (2026-08-20) — the migration blocker is cleared, revision `738d3d45c3fb` applies from empty and re-generates to zero drift. Two consecutive demo runs: `created=20` then `created=0 updated=8 skipped=12`, zero loader errors. The run found a defect the module's unit tests could not: the complete import cycle allocated **MSKU4200002**, the container BRD N4's demo deliberately holds for customs, and 13 records failed as cascades. Fixed by shipping this module's own 42GP box **SEAU4200103** rather than borrowing a shared one — see [Shared fixtures that are deliberately UNAVAILABLE](#shared-fixtures-that-are-deliberately-unavailable). A live `nvocc.dnd.calculation` on a real tenant now returns non-NULL money columns (`calculated_amount` 2400.00, `payable_exposure_amount` 1200.00), which is the first end-to-end proof of the stored-compute fix. |

| `logistics_nvocc_agency.agency_commercial` | [modules/agency-commercial/usecase.md](./modules/agency-commercial/usecase.md) | demo_agency_commercial.xml | ✅ Delivered 2026-11-05 (re-filed from `agency_reports`) — 25 records: the BRD N2 principal → rates → quote → accepted-baseline chain. Every row sits on a model this module *extends* (`crm.quote`, `crm.quote.version`, `pricing.contract`, `pricing.rate`, `pricing.margin.rule`) with the agency columns it contributes filled in, which is why the file belongs here and not to the reporting module. Under `--with-demo=all`: pass 1 `created=25`, pass 2 `created=0 / updated=25` — **idempotent**. A single-app `--with-demo` is **not** equivalent: it takes app keys literally and does not expand along `depends`, so a file that `ref=`s the principal needs `agency_masters` in the same selection. |
| `logistics_nvocc_agency.agency_documentation` | [modules/logistics-nvocc-agency-documentation/usecase.md](./modules/logistics-nvocc-agency-documentation/usecase.md) | demo_documentation.xml | 🟡 Built 2026-08-20 — 20 records (2 role bindings + 2 partner-role assignments + 2 numbering rules + 1 integration partner + 2 equipment usages + 3 seal applications + 1 shipping instruction with 1 cargo and 2 container lines + 1 draft B/L + 1 manifest + 2 stowage lists + 1 readiness row) continuing the Sealink INNSA→USNYC story `agency_booking` began. **Plants two deliberate errors and ships no issued document:** the customer's SI quotes the seal cut off at a customs examination and a VGM contradicting the declared gross weight by 42%, and both are the exact failures N5-07 exists to catch. Nothing is issued, no manifest line is written and no numbering rule is active, because `nvocc.bl.status` and `nvocc.manifest.status` are `workflow=True` and a fixture that wrote a governed stage would demonstrate post-issuance immutability that does not exist. ✅ `--with-demo` smoke **green and idempotent** on PostgreSQL (2026-08-20) — two consecutive runs, zero loader errors, no duplicated rows. The run found **three defects 156 unit tests did not**: a role-name collision on the unique `ir.rbac.role.name` that took all 75 permission rows with it, a container the BRD N4 demo deliberately holds for customs, and append-only seal rows in an updatable data block that loaded clean on run 1 and failed on run 2. All fixed; see [phase-5/05](../../roadmap/logistics/nvocc-agency/phase-5/05-repository-dependency-and-reports.md#three-defects-the-demo-smoke-test-caught-that-156-unit-tests-did-not) |
| `logistics_nvocc_agency.agency_integration` | [modules/logistics-nvocc-agency/usecase.md](./modules/logistics-nvocc-agency/usecase.md) | demo_agency_integration.xml | ✅ Delivered 2026-11-05 (re-filed from `agency_reports`) — 2 records (NAM-18): one terminal feed **active in production** that may exchange, and one still in **test** that may not, so BR-NAM-16's refusal has a record and not only its pass. Neither stores credentials — `endpoint_ref` is a connector reference. Measured under `--with-demo=all`: pass 1 `created=2`, pass 2 `updated=2`. A single-app `--with-demo` is not equivalent — it takes app keys literally and does not expand along `depends`. |
| `logistics_nvocc_agency.agency_masters` | [modules/logistics-nvocc-agency/usecase.md](./modules/logistics-nvocc-agency/usecase.md) | demo_agency_masters.xml | ✅ Delivered 2026-11-05 (re-filed from `agency_reports`) — 8 records: 2 principals (both in `draft`, because `status` is `workflow=True` and seeding it would bypass the guard the lifecycle depends on), 3 entitlements chosen to make the rules visible (branch-agnostic + full · branch-scoped + booking · finance with `can_switch=false`), and 3 demo-only role bindings so the approval step is assignable at all. Measured under `--with-demo=all`: pass 1 `created=8`, pass 2 `updated=8`. A single-app `--with-demo` is not equivalent — it takes app keys literally and does not expand along `depends`. |
| `logistics_nvocc_agency.agency_network` | [modules/logistics-nvocc-agency/usecase.md](./modules/logistics-nvocc-agency/usecase.md) | demo_agency_network.xml | ✅ Delivered 2026-11-05 (re-filed from `agency_reports`) — 2 records (NAM-13): the origin and destination agents of the Mumbai → Singapore lane, because one agent cannot show that `agent_type` is what distinguishes the two sides. Measured under `--with-demo=all`: pass 1 `created=2`, pass 2 `updated=2`. A single-app `--with-demo` is not equivalent — it takes app keys literally and does not expand along `depends`, so `agency_masters` must be in the same selection for the principal `ref=`. |
| `logistics_nvocc_agency.agency_numbering` | [modules/logistics-nvocc-agency/usecase.md](./modules/logistics-nvocc-agency/usecase.md) | demo_agency_numbering.xml | ✅ Delivered 2026-11-05 (re-filed from `agency_reports`) — 2 records (NAM-19): a **global** B/L fallback and a **Maersk-specific** CRO rule with a `prefix_override`, because most-specific-wins resolution needs two rows to be visible at all. Measured under `--with-demo=all`: pass 1 `created=2`, pass 2 `updated=2`. A single-app `--with-demo` is not equivalent — it takes app keys literally and does not expand along `depends`, so `agency_masters` must be in the same selection for the principal `ref=`. |
| `logistics_nvocc_agency.agency_reports` | [modules/agency-reports/usecase.md](./modules/agency-reports/usecase.md) | demo_agency_reports_n9.xml | ✅ Delivered 2026-08-18 (BRD N9 slice) — 7 new records (4 demo role bindings + 2 reconciliation exceptions + 1 open August reporting period) turning Maersk Demo Line's finished month into a month that **will not close, and says exactly why**: USD 1,840 marked billing-ready and never handed to accounting, with **no owner**, plus a hand-off accounting rejected with a real reason. **Ships the gate's inputs, never its output:** no closure checklist (eleven rows are what *Run Closure Checks* produces — seeding them would teach the reviewer that a control is a fixture), no report submission (its `content_hash` is a snapshot of real bytes, and an invented one breaks the only property the model has), no resolved exception. The N9 verbs are gated on ACTION resources rather than model CRUD, which is why four role bindings ship at all. ✅ **`--with-demo` smoke PASSED 2026-08-20** on a fresh tenant built from empty. Migration `2353f1a5cf27` creates the five tables. Pass 1 `created=7`, pass 2 `created=0 / updated=7` — **idempotent**. Getting there took two real defects that only a live install could surface: the 8 report-template seed rows failed with `operator does not exist: character varying = boolean` (a `("col", "=", False)` domain leaf against a *Reference* column — `generic_repo` special-cases that idiom to `IS NULL` for BOOLEAN columns only), and 3 demo rows loaded clean on pass 1 then failed on pass 2 with `Direct write to workflow-controlled field 'status' is forbidden` — seeding a `workflow=True` field is a no-op on create and a violation on re-load. Neither was visible to the test suite or the migration-graph gate, both of which were green throughout. **Demo re-filed 2026-11-05:** the BRD N1 records (principals, agents, numbering rules, EDI partners) and the BRD N2 quote chain shipped from here although this module owns neither; each now ships from its owning module, so `--with-demo=<that app>` produces its own records. This module keeps the N9 file only. |
| `logistics_nvocc_agency.agency_release` | [modules/logistics-nvocc-agency-release/usecase.md](./modules/logistics-nvocc-agency-release/usecase.md) | demo_release.xml | ✅ Delivered 2026-08-20 — 14 records (2 facilities + 2 role bindings + 1 numbering rule + 1 haulier partner with its pickup-party role + 1 pending customs entry + 1 draft delivery order with 2 release lines and 3 party rows) picking the Sealink INNSA→USNYC story up at the discharge end, where Acme's New York desk releases to Stark's nominated haulier. **The pending customs entry is the point:** everything else about the DO is correct, so the first readiness run fails on exactly one interesting condition rather than a wall of missing data — and it fails three times over, on three different desks, in one message. **Ships no issued DO, no release hold and no movement event:** `status` is `workflow=True`; a hold hangs off a document envelope that does not exist until the B/L is issued; and discharge, gate-out and empty return belong to Phase 4's append-only stream with its own sequence and duplicate detection. ✅ `--with-demo=all` smoke **green and idempotent** on PostgreSQL from a FRESH tenant (2026-08-20), after the run found two defects 126 unit tests did not — a `res.partner` NOT NULL violation, and a release-party guard that **refused every haulier while reading as if it worked** because `config_csv` upper-cases its tokens and partner role codes are lower-case. Both fixed and guarded by tests |
| `logistics_nvocc_agency.agency_voyage` | [modules/agency-voyage/usecase.md](./modules/agency-voyage/usecase.md) | demo_agency_voyage.xml | ✅ Delivered 2026-08-19 — 28 records (1 service + 1 feeder vessel + 2 voyages + 4 rotation lines + 3 port calls + 1 operation plan + 1 booking link + 1 load list with 3 container lines + 1 feeder connection + 3 port services + 2 cost references + 1 communication + 1 vendor party + 1 config + 2 role bindings) on Sealink's IGX feeder loop out of Nhava Sheva. **Ships the plan, never the outcome:** no `status` on any voyage or port call (both are `workflow=True`, and a fixture that pre-set them would show a port call `arrived` with no ATA), no recorded actual (the create hook enforces chronology, the departure policy and the correction contract — pre-recording skips all three), and no discrepancy (short load, short landed and over landed are DERIVED by reconciliation; one shipped as a fixture is the hand-typed discrepancy the design refuses). The 20ft box on the load list is deliberately the one the walkthrough leaves unloaded. ✅ `--with-demo` smoke **green and idempotent** on PostgreSQL (2026-08-19), after the run found five defects 152 unit tests did not — including **twenty stored computes that silently produced NULL** because a compute cannot call a helper method on its RecordSet `self`. All fixed and guarded by tests; see [phase-8/README.md](../../roadmap/logistics/nvocc-agency/phase-8/README.md#five-defects-the-demo-smoke-test-caught--and-one-worth-the-whole-exercise) |

Modules with no usecase row (e.g. `foundation.auth`, `foundation.connectors`, `foundation.storage`, `foundation.email`) are pure engines — they consume records produced by other modules' demo files. They get a usecase doc the first time one of their phases introduces records of its own.

## Workflow — when does demo data get authored?

The canonical process is the [`adding-demo-data`](../../.claude/skills/adding-demo-data/SKILL.md) skill — the full end-to-end guide with three explicit phases:

1. **Phase 1 — Author the usecase doc** at `docs/demo-usecase/modules/<layer>-<short>/usecase.md` (the per-module spec).
2. **Phase 2 — Register the demo-data deliverable in the roadmap** — pick the right surface:
   - Demo ships with a new feature → acceptance-criteria lines on the feature file.
   - Demo retrofits onto a delivered module → new Enhancement file.
   - Demo is a cross-module rollout → new Platform Change.
3. **Phase 3 — Author the demo XML/CSV** under `<app_root>/demo/`, declare it in `__manifest__.py demo: [...]`, smoke-test with `ede migrate upgrade -t <tenant> --with-demo=<app_key>`, then flip status across the five sync sites in lockstep.

The companion [`preparing-demo-data`](../../.claude/skills/preparing-demo-data/SKILL.md) skill is the lightweight gate that runs only at ✅-flip time — it catches misses by verifying the demo XML + manifest + smoke-test + five-site sync are all in place before the status changes. Use the gate skill at PR review; use `adding-demo-data` when authoring.

## Out of scope

- Per-tenant variations of demo data — every tenant gets the same set.
- Localized demo data (translated names) — Phase 2 of `foundation.i18n` enhancement.
- Demo data for development branches that are 🔴/🟡 — only ✅ Delivered work needs demo coverage. A 🟡 module may ship partial demo data alongside the partial feature; that is fine.
- Production seed data (the existing `data: [...]` list) — that ships in every tenant unconditionally and is *not* tagged `is_demo=True`. Keep the two channels separate.

---

*This catalogue is hand-authored. Each `modules/<module>/usecase.md` lists the records that module's demo pass produces. The actual XML/CSV files live at `<app_root>/demo/`.*
