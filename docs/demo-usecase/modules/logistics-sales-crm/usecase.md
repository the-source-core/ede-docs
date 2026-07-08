# `logistics.sales-crm` — Demo Use-Case

**Module:** `domains.logistics.sales_crm`
**App key:** `logistics.sales_crm`
**Demo manifest entries** (target): `demo/demo_teams.xml` (Enh 09), `demo/demo_partners.xml`, `demo/demo_pipeline.xml`, `demo/demo_quotes.xml`, `demo/demo_spot_linkage.xml`
**Status:** ✅ Delivered 2026-05-13

---

## Use-case

A freight-forwarder running on EDE wants to demo the full Phase MVP pipeline — inquiry → lead → opportunity → quote → handover — in a single tenant so a prospect or new hire opens the Sales & CRM app and immediately sees populated kanban + list + form views across every Phase MVP feature. This demo seeds the operational spine for the unifying scenario from [docs/demo-usecase/README.md](../../README.md) — **Acme Forwarding Ltd. (Mumbai HQ)** quoting freight services to multiple Asian customers — exercising all four delivered MVP features in their characteristic states.

Three customer storylines run in parallel so the user can see how the same workflow handles different outcomes:

1. **Globex Inc.** — full happy path: inquiry → lead → opportunity → quote-in-draft. The user can drive forward by clicking Submit → Request Approval → Send → Accept in the browser to exercise the rest of the lifecycle (and trigger the auto-create-on-Accept handover).
2. **Stark Industries Demo** — mid-flight commercial work: opportunity at `OPP_NEGOTIATION`, quote sitting at `QUOTE_SENT` waiting on customer decision. Shows what an in-flight quote looks like.
3. **Wayne Imports Demo + Hooli Demo** — entry-point and lost states: a cold `NEW_PROSPECT` inquiry plus a `PROSPECT_DISQUALIFIED` inquiry and a `OPP_LOST` opportunity with a structured lost-reason. Shows the unhappy paths.

**XML can only seed records in their INITIAL workflow stage.** The `foundation.workflow.bootstrap` registers a `pre.{model_key}.create` hook (`make_create_initial_stage_filler`) for every `workflow=True` field that **overwrites the user-supplied value with the workflow's initial-stage `target_value`**. So `status="OPP_LOST"` at create-time would be silently rewritten to `OPP_PROPOSAL_SENT`. Workflow advances (Submit → Send → Accept; Drop → Disqualified; Mark Lost) are the user's job in the browser walkthrough — that's the demo, not setup. Skipping the create-filler for demo loads is a platform enhancement filed elsewhere.

**Self-contained partners:** because [Platform 04 (foundation-rollout)](../../../../roadmap/platform/04-demo-data-foundation-rollout.md) is still 🔴 Not Started, this module declares its own customer / contact records inline using `sales_crm.demo_partner_*` xml-ids. When Platform 04 lands and ships `base.demo_partner_customer_globex` (etc.), this file shrinks to refs and a redirect note.

## Records produced

### `demo/demo_partners.xml` — Self-contained partner master (7 records)

| External ID | Model | Notes |
|---|---|---|
| `sales_crm.demo_partner_customer_globex` | `res.partner` | Globex Inc. — primary demo customer (company; matches the unifying-scenario actor). |
| `sales_crm.demo_partner_contact_globex_jane` | `res.partner` | Jane Procurement — individual at Globex; `parent_partner_id`=globex; procurement contact. |
| `sales_crm.demo_partner_contact_globex_raj` | `res.partner` | Raj Logistics — individual at Globex; ops-side contact. |
| `sales_crm.demo_partner_customer_stark` | `res.partner` | Stark Industries Demo — secondary customer. |
| `sales_crm.demo_partner_contact_stark_pepper` | `res.partner` | Pepper Potts — individual at Stark; chief procurement officer. |
| `sales_crm.demo_partner_customer_wayne` | `res.partner` | Wayne Imports Demo — tertiary customer (cold inquiry only). |
| `sales_crm.demo_partner_contact_wayne_alfred` | `res.partner` | Alfred Pennyworth — individual at Wayne; senior buyer. |

### `demo/demo_pipeline.xml` — Inquiry → Lead → Opportunity chain (7 records, all in initial stage)

Workflow create-filler forces all records into their initial stage (see above). Records carry forward-cross-refs so the conversion lock fields are populated even though no actual transitions have fired.

| External ID | Model | Stage | Notes |
|---|---|---|---|
| `sales_crm.demo_inquiry_globex_001` | `crm.inquiry` | `NEW_PROSPECT` | Inbound enquiry from Jane (Globex) — 5 × 40' HC containers Mumbai → Singapore; `converted_lead_id` → `demo_lead_globex_001` (forward ref — resolved by the loader's `_local_refs` cache). |
| `sales_crm.demo_inquiry_stark_001` | `crm.inquiry` | `NEW_PROSPECT` | Stark inquiry from Pepper — Singapore → Mumbai ocean FCL; `converted_lead_id` → `demo_lead_stark_001`. |
| `sales_crm.demo_inquiry_wayne_001` | `crm.inquiry` | `NEW_PROSPECT` | Cold web-form inquiry from Alfred (Wayne) — Mumbai → Dubai air freight; no lead conversion yet (Convert button visible in form). |
| `sales_crm.demo_lead_globex_001` | `crm.lead` | `LEAD_QUALIFIED` | Qualified lead from inquiry 001 — subject "Globex Q3 Asia Ocean FCL", estimated_revenue $50000 USD, `inquiry_id`=globex_001, `converted_opportunity_id`=globex_001. |
| `sales_crm.demo_lead_stark_001` | `crm.lead` | `LEAD_QUALIFIED` | Stark lead — `inquiry_id`=stark_001, subject "Stark Industries SG-IN trade lane", $80K revenue, `converted_opportunity_id`=stark_001. |
| `sales_crm.demo_opportunity_globex_001` | `crm.opportunity` | `OPP_PROPOSAL_SENT` | Promoted from `demo_lead_globex_001`. expected_revenue 50000 USD, expected_margin_percent 12, close_date +30 days. |
| `sales_crm.demo_opportunity_stark_001` | `crm.opportunity` | `OPP_PROPOSAL_SENT` | Promoted from `demo_lead_stark_001`. expected_revenue 80000 USD, expected_margin_percent 10, close_date +20 days. |

### `demo/demo_quotes.xml` — Quote header + lines (2 quotes + ~7 lines = 9 records)

The auto-init `post.crm.quote.create` hook creates v1 for each quote with the parent's currency_id and `version_status="draft"`. Auto-numbering generates `QT-2026-NNNNNN` for each — exact numbers determined by the `ir.sequence` row's current_value at load time.

| External ID | Model | Stage | Notes |
|---|---|---|---|
| `sales_crm.demo_quote_globex_001` | `crm.quote` | `QUOTE_DRAFT` | Created on `demo_opportunity_globex_001`; partner=globex, contact=jane, currency=USD, quote_issue_date today, quote_expiry_date today+30. v1 auto-created via `post.crm.quote.create`. |
| `sales_crm.demo_quote_stark_001` | `crm.quote` | `QUOTE_SENT` | Created on `demo_opportunity_stark_001`; partner=stark, contact=pepper, currency=USD; status="QUOTE_SENT" at create (valid — the ORM guard fires on UPDATE not CREATE). v1 auto-created. Customer has the quote in hand awaiting decision — exercises the post-Send state. |
| `sales_crm.demo_quote_line_globex_001_ocean_freight` | `crm.quote.line` | n/a | Charge line on globex v1 — Ocean Freight FCL HC, buy 32000 USD, sell 40000 USD, mandatory_flag=true. `product_id` left NULL because `logistics.product.master` ships no demo records (will resolve when masters demo lands). |
| `sales_crm.demo_quote_line_globex_001_thc` | `crm.quote.line` | n/a | Terminal Handling — buy 800, sell 1200. |
| `sales_crm.demo_quote_line_globex_001_baf` | `crm.quote.line` | n/a | Bunker Adjustment Factor — buy 600, sell 900. |
| `sales_crm.demo_quote_line_globex_001_doc` | `crm.quote.line` | n/a | Documentation — buy 60, sell 100. |
| `sales_crm.demo_quote_line_stark_001_ocean_freight` | `crm.quote.line` | n/a | Stark v1 Ocean Freight — buy 28000, sell 35000. |
| `sales_crm.demo_quote_line_stark_001_thc` | `crm.quote.line` | n/a | Stark THC — buy 700, sell 1000. |
| `sales_crm.demo_quote_line_stark_001_doc` | `crm.quote.line` | n/a | Stark Documentation — buy 60, sell 100. |

**Cross-ref note**: lines reference `quote_version_id` which is set by the auto-init hook. Because the loader processes records in declared order within a file, and the `post.crm.quote.create` hook fires synchronously during quote creation (creating v1 then writing `current_version_id` on the quote), the quote's v1 UUID is available the moment line records are processed. The lines `ref=` the quote's `current_version_id` via a small XML helper: `<field name="quote_version_id" eval="ref('sales_crm.demo_quote_globex_001').current_version_id.id"/>` — **wait, the data loader doesn't support `eval=`**. The pattern that DOES work: lines `ref=` an explicit version xml-id. Since the version is auto-created (no xml-id), the loader has no name to look up. **Resolution**: lines are seeded by querying the just-created quote's version inside a `pre.crm.quote.line.create` helper hook? No — too clever. Pragmatic answer: lines declare `quote_version_id` via a small Python `data_post_hook` registered for this demo file (sales_crm's `_helpers.py` already has utility scope), OR the demo seeds lines as plain CSVs that resolve `quote_version_id` server-side using `sales_crm.demo_quote_*` parent xml-id. **Simpler path adopted here**: drop charge lines from MVP demo — v1 of each quote ships empty; user adds lines by clicking "Add" in the Versions tab during the walkthrough. Margin compute fields stay at 0 USD which is correct given no lines.

**Revised line count**: 16 records total (7 partners + 7 pipeline + 2 quotes).

### `demo/demo_teams.xml` — Sales team hierarchy (Enhancement 09, 10 records)

A 3-level sales hierarchy plus a pricing desk and a finance-approvers team, on the
default organization, with the admin user holding an approver role at each level so
the TEAM_ROLE approval chains + WALK_UP_HIERARCHY escalation resolve a person
end-to-end:

| External ID | Model | Notes |
|---|---|---|
| `sales_crm.demo_team_india_hq` | `res.team` | SALES, root; COUNTRY_HEAD level |
| `sales_crm.demo_team_india_region` | `res.team` | SALES, child of HQ; PRICING_APPROVER level |
| `sales_crm.demo_team_mumbai_west` | `res.team` | SALES, child of Region; SALES_MANAGER + ACCOUNT_MANAGER |
| `sales_crm.demo_team_pricing_desk` | `res.team` | PRICING_DESK |
| `sales_crm.demo_team_finance_approvers` | `res.team` | FINANCE_APPROVERS; FINANCE_APPROVER level |
| `sales_crm.demo_assign_*` (5) | `res.team.role.assignment` | admin → SALES_MANAGER / ACCOUNT_MANAGER / PRICING_APPROVER / COUNTRY_HEAD / FINANCE_APPROVER |

The demo pipeline + quote records (above) are stamped with a `team_id` automatically
by the Enh-09 pre-create stampers (org-scoped default-team resolution), so every CRM
demo record carries a team. Real multi-user role assignments land once demo users exist.

### `demo/demo_pricing_thresholds.xml` — Pricing-approval threshold matrix (Phase 2 · 02 WS4, 4 records)

Configurable margin floors that select the low-margin approval chain, keyed by
`transport_mode × trade_lane × customer_tier` (blank = wildcard). They demonstrate
most-specific-wins resolution; the demo customers carry a `customer_tier` so the
bands actually apply.

| External ID | Match (mode / lane / tier) | Floor % | Seq |
|---|---|---|---|
| `sales_crm.demo_pricing_threshold_sea` | SEA / — / — | 12 | 30 |
| `sales_crm.demo_pricing_threshold_air` | AIR / — / — | 18 | 30 |
| `sales_crm.demo_pricing_threshold_sea_platinum` | SEA / — / Platinum | 6 | 20 |
| `sales_crm.demo_pricing_threshold_sea_inmum_sgsin_gold` | SEA / INMUM→SGSIN / Gold | 9 | 10 (most specific) |

Demo customer tiers: Globex = Platinum, Stark = Gold, Wayne = Silver. A submitted
quote with no matching band falls back to the global
`ir.config['crm.quote.margin_threshold_percent']` (default 5.00).

## Out of scope

- **Quote lines, routes, packages, communications** — seeding these in XML is blocked by the cross-ref pattern described above; v1 of each demo quote ships empty. User exercises line-adding by clicking the embedded list "Add" button. Solving this requires either a `post_load` Python hook OR a small data-loader extension to support `eval=` / dynamic field-ref — out of scope for the MVP demo.
- **`crm.quote.acceptance` + `crm.handover` pre-seeded records** — the auto-create-on-Accept flow is the demo for these. Seeding them as fixtures defeats the purpose. The user clicks Accept on `demo_quote_globex_001` (after Submit + Request Approval + Send) and the handover materializes.
- **Approval cases on demo quotes** — depend on `foundation.approval` Platform 04 rollout. The standard policy seed shipped in `data/approval_rules_crm_quote.xml` is already in place, so clicking Request Approval works; the case lands on `sales_crm.role_sales_manager` role-bound users.
- **Logistics master records** (`logistics.product.master`, `logistics.unlocode.master`, `logistics.commodity.master`, `logistics.trade.lane`) — owned by `logistics.masters` demo rollout. Until then, demo records leave those optional refs NULL.
- **`crm.organization.profile` / `crm.contact.profile` sidecars** — declared empty for MVP; the underlying `res.partner` carries enough demo context.
- **Chatter messages + notifications** — owned by `foundation.communication` / `foundation.notifications` Platform 04 rollouts.

## Dependencies

- `foundation.base` production seeds: `res.currency.usd` (USD), `res.country` (US, IN, SG, AE), `res.partner.role.master` (customer role).
- `foundation.communication` is in `depends` (Chatterable mixin) — no specific demo records needed.
- Production data already shipped: `crm.sale.stage.csv`, `crm.lost.reason.csv` (lost reasons), `crm.quote.stage.csv`, `workflow_crm_*.xml`, `approval_rules_crm_quote.xml`. The pre-seeded `sales_crm.lost_reason_*` xml-ids are referenced by the lost opportunity.

## Verification

```bash
ede migrate upgrade -t demo_tenant --with-demo=logistics.sales_crm
```

Expected log:
```
demo_load: ~17 created, 0 updated, 0 skipped (scope=logistics.sales_crm)
```

Re-run check (idempotence):
```
demo_load: 0 created, ~17 updated, 0 skipped (scope=logistics.sales_crm)
```

SQL check (`is_demo=True` rows under sales_crm module):
```sql
SELECT model_key, COUNT(*)
  FROM ir_data_reference
 WHERE is_demo = true AND module = 'sales_crm'
 GROUP BY model_key
 ORDER BY model_key;
```

Expected: `res.partner`=7, `crm.inquiry`=3, `crm.lead`=2, `crm.opportunity`=2, `crm.quote`=2. (Versions auto-created by the post-hook are not tagged `is_demo` — they're hook-side-effects, not opt-in demo records.)

Browser walkthrough (the actual demo):
1. Log in as admin → **Sales & CRM** app
2. **Opportunity Pipeline** kanban shows 2 cards — Globex + Stark, both at `Proposal Sent` (the initial stage). User drives Stark forward via the **Start Negotiation** button to populate the Negotiation column.
3. **Inquiries** list — 3 inquiries, all `NEW_PROSPECT`; Globex 001 + Stark 001 have `converted_lead_id` set so Convert-to-Lead button hidden; Wayne is fresh cold (Convert button visible). User drives Wayne forward via Disqualify or Convert-to-Lead to populate other inquiry states.
4. **Leads** list — 2 leads, both `LEAD_QUALIFIED`; Convert-to-Opportunity button hidden on both (already converted). User drives via Drop button to populate dropped state.
5. **Quotes** list — 2 quotes, both `QUOTE_DRAFT`. User clicks Submit on either to advance.
6. Drive the Globex quote end-to-end: open form → Submit → Request Approval (case opens, routed to whoever's bound to `sales_crm.sales_manager` role) → approve via Approvals inbox → Send → Accept. Handover auto-created → visible under **Handovers** menu.

## Authoring order

Within `demo: [...]` list — parents before children, since later files `ref=` earlier records:

1. `demo/demo_partners.xml` — 7 partner records (3 companies + 4 individuals; contacts ref companies via `parent_partner_id`).
2. `demo/demo_pipeline.xml` — inquiry → lead → opportunity; uses forward refs (`converted_lead_id`, `converted_opportunity_id`) resolved by the loader's `_local_refs` cache.
3. `demo/demo_quotes.xml` — quote headers; auto-init hook creates v1 of each as a side effect.

## Recorded e2e tests

The MVP lifecycle is covered end-to-end by Playwright + pytest tests at
[`src/domains/logistics/sales_crm/tests/e2e/`](../../../../src/domains/logistics/sales_crm/tests/e2e/).
They live alongside the module (per the [`authoring-e2e-tests`](../../../../.claude/skills/authoring-e2e-tests/SKILL.md)
module-local rule) so the suite is runnable in isolation:

```bash
./run_tests.sh --e2e -k sales_crm
# or directly:
pytest src/domains/logistics/sales_crm/tests/e2e/ -p ede.foundation.qa_automation.fixtures \
       --browser=chromium -s
```

| Test file | Scenario | Markers |
|---|---|---|
| [`test_single_record_full_lifecycle.py`](../../../../src/domains/logistics/sales_crm/tests/e2e/test_single_record_full_lifecycle.py) | **Canonical end-to-end** — a SINGLE inquiry carried through every stage: Inquiry → Lead → Opportunity → Quote → Acceptance → Handover. Verifies the conversion chain (`converted_lead_id` → `converted_opportunity_id` → `opportunity_id` → `quote_id`) stays intact through every workflow transition + the approval round-trip. | `CRM-MVP-SINGLE-RECORD-LIFECYCLE` |
| [`test_inquiry_to_quote_full_lifecycle.py`](../../../../src/domains/logistics/sales_crm/tests/e2e/test_inquiry_to_quote_full_lifecycle.py) | Per-step diagnostic walk-through (5 methods, fresh record per method). Each method isolates one stage transition so a regression points at the exact failing step. | `CRM-MVP-FULL-LIFECYCLE` |
| [`test_inquiry_disqualified.py`](../../../../src/domains/logistics/sales_crm/tests/e2e/test_inquiry_disqualified.py) | Walk-in prospect requesting hazardous-cargo lane forwarder doesn't serve — Disqualify terminal state. | `CRM-MVP-INQUIRY-DISQUALIFY` |
| [`test_lead_dropped.py`](../../../../src/domains/logistics/sales_crm/tests/e2e/test_lead_dropped.py) | Globex went dark after 3 follow-ups — Drop Lead with reason 'No Decision'. | `CRM-MVP-LEAD-DROP` |
| [`test_opportunity_close_as_lost.py`](../../../../src/domains/logistics/sales_crm/tests/e2e/test_opportunity_close_as_lost.py) | Stark Singapore-Mumbai pilot — Close Lost with reason 'Lost to Competitor'. Exercises the `opp_lost_has_reason` workflow guard. | `CRM-MVP-OPPORTUNITY-CLOSE-LOST` |
| [`test_quote_rejected.py`](../../../../src/domains/logistics/sales_crm/tests/e2e/test_quote_rejected.py) | Globex rejects sent quote on price — `crm.quote.acceptance` row written with `decision_type='rejected'` and `lost_reason_id` set; no handover created. | `CRM-MVP-QUOTE-REJECT` |
| [`test_multi_leg_quote.py`](../../../../src/domains/logistics/sales_crm/tests/e2e/test_multi_leg_quote.py) | Wayne door-to-door Mumbai → Singapore (T/S) → Sydney FCL. Authors 4 route legs (origin-haulage / main-carriage / transshipment / destination-haulage) on a single quote version + 7-line charge profile (US$ 60,990 sell across 19 transit days). Verifies leg-type ordering, transit-day rollup (19d), version totals (buy US$ 51,250 / sell US$ 60,990), route summary round-trip, full lifecycle through accept, and handover's denormalised `special_instruction_text` embedding the route summary. | `CRM-MVP-MULTI-LEG-QUOTE` |

All tests use the shared `CrmFactory` (in `tests/e2e/conftest.py`) for realistic
record creation (USD 65k revenue estimates, Net 30 payment terms, 13%
margin, FCL ocean-freight charge profile — Ocean Freight + THC Origin/
Destination + BAF + Documentation). The session fixture `sales_crm_demo`
loads the demo XML, binds the seeded admin to the `sales_crm.sales_manager`
role so single-step approval routing resolves, and seeds the
`logistics.product.master` charge codes used by every quote line.

Cross-module navigation in these tests follows the dependency arrow: the
lifecycle test drives the Approvals app's `My Approvals` inbox because
sales-crm depends on foundation.approval. The reverse (foundation.approval
tests reaching into sales-crm) is forbidden — peer modules' joints belong
in their own suites.

---

*Back to [demo-usecase index](../../README.md).*

<!-- SYNC-BLOCK: videos -->
## Polished Demo Videos

### Sales-CRM Inquiry to Quote Full Lifecycle
<video controls width="720" preload="metadata">
  <source src="videos/sales-crm-inquiry-to-quote-full-lifecycle.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

<!-- /SYNC-BLOCK -->
