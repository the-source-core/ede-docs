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

## Per-module specs

| Module | Spec | Demo files (planned / shipped) | Status |
|---|---|---|---|
| `foundation.base` | [modules/foundation-base/usecase.md](./modules/foundation-base/usecase.md) | demo_users.xml, demo_partners.xml, demo_org.xml | 🔴 Not yet authored |
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
| `foundation.dashboard` | [modules/foundation-dashboard/usecase.md](./modules/foundation-dashboard/usecase.md) | demo_kpis.xml, demo_dashboards.xml | 🔴 Not yet authored |
| `foundation.agent` | [modules/foundation-agent/usecase.md](./modules/foundation-agent/usecase.md) | demo_agent.xml, demo_automation.xml | ✅ Delivered 2026-06-18 — 8 records (1 agent + 3 AI Automations + 4 actions); smoke + idempotency verified |
| `foundation.presentation` | _no demo data — engine_ | _none_ | n/a |
| `logistics.masters` | [modules/logistics-masters/usecase.md](./modules/logistics-masters/usecase.md) | demo_ports.xml, demo_equipment.xml, demo_commodities.xml | 🔴 Not yet authored |
| `logistics.pricing` | [modules/logistics-pricing/usecase.md](./modules/logistics-pricing/usecase.md) | demo_rates.xml, demo_spot_rfqs.xml, demo_phase2.xml (+3) | ✅ Delivered 2026-07-02 — Phase 1 rates/contracts/spot + Phase 2 (version history, branch override, contract volume-utilization); 55 demo records, smoke + idempotency verified |
| `logistics.sales-crm` | [modules/logistics-sales-crm/usecase.md](./modules/logistics-sales-crm/usecase.md) | demo_partners.xml, demo_pipeline.xml, demo_quotes.xml | ✅ Delivered 2026-05-13 — 16 records (7 partners + 7 pipeline + 2 quotes) |
| `logistics.equipment-operations` | [modules/logistics-equipment-operations/usecase.md](./modules/logistics-equipment-operations/usecase.md) | demo_equipment.xml, demo_operations.xml | ✅ Delivered 2026-05-28 — 13 records (7 fleet units + 2 usage + 1 seal + 2 movement + 1 load-calc) on the INMUM→SGSIN lane |
| `logistics.shipments` | [modules/logistics-shipments/usecase.md](./modules/logistics-shipments/usecase.md) | demo_shipment.xml | ✅ Delivered 2026-06-17 — 8 records (1 shipment + 1 leg + 4 parties + 1 cargo + 1 charge) on the Globex INMUM→SGSIN lane; composed ref CO-SEA-D-2026-000001; postgres `--with-demo` smoke green + idempotent |
| `logistics.operations_execution` | [modules/logistics-operations-execution/usecase.md](./modules/logistics-operations-execution/usecase.md) | demo_execution.xml, demo_operations.xml | ✅ Delivered 2026-06-25 — 24 records (1 SOP + 1 execution + 1 plan + 2 legs + 4 tasks + 5 milestones + 1 completion + 1 provider + 1 pickup + 1 delivery + 1 dispatch + 1 driver + 1 cargo event + 2 cut-offs + 1 exception) — live Control Tower over the Globex INMUM→SGSIN shipment; `--with-demo` smoke green + idempotent |
| `logistics.profitability` | [modules/logistics-profitability/usecase.md](./modules/logistics-profitability/usecase.md) | demo_profitability.xml | ✅ Delivered 2026-06-29 — 10 records (1 header + 5 charge lines + 1 currency line + 1 adjustment + 2 audit entries) — operational profitability over the Globex INMUM→SGSIN shipment (rev 4150 / cost ~3010 / GP ~1140 / margin ~27%, incl. one multi-currency line); `--with-demo` smoke green + idempotent |

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
