# Showcase tenant `djlogistics` — DJ Logistics, Jan–Jun 2026

> **Scope change, 2026-11-24 (user decision):** the dataset covers **January–June 2026**, with the
> snapshot as at **30 Jun 2026**. The monthly plan in §8–§9 is unchanged for January–June. The
> July–October columns are not generated, and the 10-month totals below are the original design.
>
> - **Jan–Jun volume:** 5,462 TEU — India 3,976 (2,655 import + 1,321 export), Dubai 1,486
>   (1,157 AE→US + 329 IN→AE forwarding) — on 3,444 boxes and 1,825 shipments.
> - **Moved:**
>   - disputes and exceptions (§10.7) move to May–June;
>   - the held close is Lantern's **May** close, so 29 periods are closed and June is still open;
>   - the October accrual becomes a June accrual;
>   - the forward pipeline covers 1–15 July.
> - **Phase changes:**
>   - Phases 2 and 3 run in parallel;
>   - each continues past January without a review stop when January verifies;
>   - the commands no shipped role can run yet are granted through tenant-only roles (security
>     Enhancement 05).

**Status:** ✅ Approved 2026-11-23. Phase 1 generated on tenant `djlogistics`; Phases 2–3 in progress.
**Product:** MaargSetu (EDE platform) — `logistics_base`, `logistics_freight_forwarding`, `logistics_nvocc_agency`
**Tenant:** `djlogistics` (database `db_ede_djlogistics`), created **without** `--with-demo`
**Built by:** a re-runnable generator script that drives the platform's real commands under a simulated clock. These are not module demo files.
**Snapshot:** as at **30 Jun 2026 23:59** local time (originally 31 Oct — see the scope change above). The group went live on **1 Jan 2026**.
**Reader:** the person approving the build. Engineers who build the generator should read sections 13–16 first.

---

## 1. Summary

DJ Logistics is a fictional group with three entities:

- a holding company and HQ in **Houston (US)**;
- an **India** branch, which is the liner/NVOCC agent for **6 fictional principals**;
- a **Dubai** branch, which does freight forwarding.

The tenant holds ten months of real work: **10,000 TEU**, **USD 5,000,000** of freight-forwarding sell, and **USD 2,000,000** of agency billings. Every record was produced by the command a user would have run, executed as the user who would have run it, on the date they would have run it. So every number came out of a numbering rule, every approval has a named decider, and every stage was reached through its guard.

| | US HQ | India (agency) | Dubai (forwarding) | Group |
|---|---|---|---|---|
| Users | 8 | 24 | 18 | **50** |
| Principals / carriers | — | 6 principals | 5 carriers | |
| Business parties | 4 US agents | 71 | 49 | **124** + 6 principals |
| TEU (Jan–Oct) | — | 7,300 | 2,700 | **10,000** |
| Containers / shipments | — | 4,718 / 2,451 | 1,587 / 890 | **6,305 / 3,341** |
| Booked revenue (USD) | — | 2,000,000 | 5,000,000 | **7,000,000** |
| Gross margin | — | 67.1% (gross billings) | 13.2% | |

**Lanes:** China → India 4,900 TEU (agency imports) · India → Dubai 3,000 TEU (2,400 agency exports + 600 forwarding imports) · Dubai → US 2,100 TEU (forwarding exports).

**What it costs to build:** about **235,000 commands**. That is an estimated **12–16 hours** of single-process runtime, delivered in four reviewed phases, resumable at any step (§15.8).

**Three changes to the brief**, each explained where it applies:

1. **Charges, finance hand-offs and month-ends cannot be a separate Phase 4.** Agency period close checks ignore the period they are closing (new defect D5), and dates cannot run backwards over work already moved on. So each month's close must happen *inside* the month-by-month run that produced the work. Phase 4 becomes group consolidation and verification (§16).
2. **Every container number makes exactly one trip.** The D&D date feed and release readiness both read a box's *whole* movement history (D6). A reused box would be billed against its previous trip.
3. **Disputes and reconciliation exceptions appear only in September–October, and none is resolved.** Resolving one leaves it counted as open forever (D4), and an open exception blocks that month's close permanently. One principal's September close is therefore held on purpose — a genuine state.

The generator needs one dev-only Python package (`time-machine`) to run the clock (open decision 1). Everything else uses code already in the repository.

---

## 2. What this is, and what it is not

| | `--with-demo` module data | This showcase |
|---|---|---|
| Source | XML/CSV in `<app>/demo/`, loaded by the data loader | A generator script that dispatches commands |
| States | Initial stages only (`workflow=True` cannot be seeded) | Every state the lifecycle reaches, at the date it reached it |
| Scenario | Acme Forwarding (unifying scenario) | DJ Logistics (stand-alone — none of the Acme or demo-carrier records are loaded) |
| Volume | Tens of records per module | About 450,000 business rows |
| Tenant | Any | `djlogistics` only. The generator refuses any other tenant |
| Code / schema change | Per feature | **None.** No model, field, migration or `src/` edit |

The Model Placement Test (CLAUDE.md Q1–Q5) is not triggered. The generator declares nothing; it writes data through existing commands.

---

## 3. The group

```
                    DJ Logistics Inc.  (DJL-US)   Houston, TX · America/Chicago
                    holding company + group HQ
                    = base.default_organization, renamed
                           │ parent_id
            ┌──────────────┴──────────────┐
DJ Logistics India Pvt Ltd (DJL-IN)    DJ Logistics FZE (DJL-AE)
Navi Mumbai + Mundra desk               Jebel Ali Free Zone
Asia/Kolkata                            Asia/Dubai
NVOCC / liner agency for 6 principals   Freight forwarding (ocean FCL)
```

- `res.organization` rows, with `parent_id` pointing at DJL-US.
- Each org gets its root team from the `ensure_org_default_team` hook, which runs on create.
- **Re-using `base.default_organization`** as DJL-US keeps every default-org fallback pointing at a real entity. The update is `ede.update`: name, code `DJL-US`, country US, timezone, and an email on `djlogistics.example`.
- **Scoping:** most agency models are `company_scope="strict"`, which is an exact match on the active org — it does not cascade down the tree. So HQ staff see India or Dubai rows only after switching into that org. This is the platform's behaviour, and the showcase displays it rather than working around it.

**Teams** (`res.team`, with 8 new `res.team.type` rows: MGMT, SALES, PRICING, CS, OPS, DOCS, FIN, IT):

| Org | Teams |
|---|---|
| DJL-US | US-EXEC · US-FIN · US-COMM · US-IT · US-CARE |
| DJL-IN | IN-MGMT · IN-PRIN (principal desk & commercial) · IN-CS · IN-EQC (equipment control) · IN-DOCS · IN-REL (import & release) · IN-CHG (D&D & charges) · IN-VOPS (vessel ops, Nhava Sheva) · IN-MUN (Mundra desk) · IN-FIN |
| DJL-AE | AE-MGMT · AE-SALES · AE-PRICE · AE-CS · AE-OPS · AE-DOCS · AE-FIN |

---

## 4. The 50 users

**How each user is created:**

1. `res.user.register` with name, email, a password from `DJL_DEMO_PASSWORD` and `default_organization_id`. The password variable is never committed.
2. `ede.update` to set `allowed_organization_ids`, `team_ids` and `primary_team_id`.
3. `ir.rbac.role.binding.assign` for each role.

**Binding scope:**

- Approver roles are bound **GLOBAL**. The approval engine resolves ROLE approvers from GLOBAL bindings only, because `requester_branch_id` is hard-coded to `""`.
- All other roles are bound **BRANCH** to the user's org.
- A binding's scope does not narrow its permissions (`principal_enricher.py`); the strict company scope is what keeps the data apart.

**Contact details:** every email is `firstname.lastname@djlogistics.example`. Phone numbers are obviously synthetic: US `+1 555 01xx`, India `+91 22 0000 01xx`, Dubai `+971 4 000 01xx`.

### 4.1 US HQ — DJ Logistics Inc. (8)

| # | Name | Title | Team | Roles | Allowed orgs |
|---|---|---|---|---|---|
| U1 | Daniel Whitcombe | Group Chief Executive | US-EXEC | `logistics_viewer`, `agency_management`, `shipment.management`, `profitability_viewer` | US · IN · AE |
| U2 | Rebecca Lindqvist | Group Chief Financial Officer | US-FIN | `profitability_viewer`, `agency_auditor`, `shipment.finance` | US · IN · AE |
| U3 | Marcus Ellery | Group Financial Controller | US-FIN | `profitability_approver`, `finance_sync_operator`, `agency_finance_admin` | US · IN · AE |
| U4 | Priyanka Raman | Group Systems Administrator — runs Phase 1 | US-IT | `system_admin` (GLOBAL) | US · IN · AE |
| U5 | Jordan Castellano | Commercial Director, Global Accounts | US-COMM | `sales_crm.sales_admin`, `pricing.sales_manager` | US · IN · AE |
| U6 | Alicia Brennan | Group Procurement & Carrier Pricing Manager — approves carrier buy rates | US-COMM | `pricing.manager` | US · AE |
| U7 | Theo Nakamura | Compliance & Internal Audit Manager | US-FIN | `agency_auditor`, `logistics_viewer`, `doc.viewer` | US · IN · AE |
| U8 | Sandra Okafor | US Customer Care Lead — US arrival and release milestones for Dubai exports | US-CARE | `shipment.customer_service`, `tracking_updater` | US · AE |

U1, U2 and U7 also hold `view` entitlements on all 6 principals.

### 4.2 India — DJ Logistics India Pvt Ltd (24)

Entitlements are `nvocc.principal.entitlement` rows scoped to DJL-IN, all with `can_switch=true`. Principal short codes are defined in §5.

| # | Name | Title | Team | Roles | Entitlements (principal → level) |
|---|---|---|---|---|---|
| I1 | Vikram Deshpande | Country Manager, India | IN-MGMT | `agency_management` (GLOBAL), `agency_regional_admin` | all 6 → view |
| I2 | Ananya Kulkarni | Principal Relations Manager — approves principals and numbering rules | IN-PRIN | `agency_principal_admin` (GLOBAL) | all 6 → full |
| I3 | Rohan Mehta | Commercial Manager — approves free-time extensions | IN-PRIN | `agency_commercial_manager` (GLOBAL) | all 6 → full |
| I4 | Sneha Iyer | Customer Service Lead — Imports | IN-CS | `agency_customer_service`, `booking_clerk` | KBL SWS LRC CSN MAL → booking |
| I5 | Farhan Shaikh | Customer Service Executive — Imports | IN-CS | `agency_customer_service`, `booking_clerk` | KBL SWS LRC → booking |
| I6 | Meera Pillai | Customer Service Executive — Exports | IN-CS | `agency_customer_service`, `booking_clerk` | KBL ICL CSN MAL → booking |
| I7 | Arjun Nair | Booking & CRO Executive | IN-CS | `agency_customer_service`, `booking_clerk` | KBL ICL CSN MAL → booking |
| I8 | Kavita Joshi | Equipment Control Manager — approves CROs | IN-EQC | `agency_container_control` (GLOBAL) | all 6 → container |
| I9 | Siddharth Rao | Equipment Control Executive | IN-EQC | `logistics_coordinator`, `agency_viewer` | all 6 → container |
| I10 | Nikhil Bhatt | Documentation Manager — approves B/L release and amendments | IN-DOCS | `agency_documentation_manager` (GLOBAL) | all 6 → full |
| I11 | Pooja Menon | Export Documentation Executive | IN-DOCS | `agency_documentation_executive` | KBL ICL → full |
| I12 | Imran Qureshi | Export Documentation Executive | IN-DOCS | `agency_documentation_executive` | CSN MAL → full |
| I13 | Lakshmi Sundaram | Import Documentation Executive | IN-DOCS | `agency_documentation_executive` | KBL SWS LRC → full |
| I14 | Tanvi Agarwal | Import Documentation Executive | IN-DOCS | `agency_documentation_executive` | CSN MAL → full |
| I15 | Rajesh Patil | Import & Release Manager | IN-REL | `agency_release_manager` (GLOBAL) | all 6 → full |
| I16 | Deepika Rane | Delivery Order Executive | IN-REL | `agency_do_executive` | KBL SWS LRC → full |
| I17 | Sameer Gokhale | Delivery Order Executive | IN-REL | `agency_do_executive` | CSN MAL → full |
| I18 | Harish Venkatesh | D&D Executive | IN-CHG | `agency_dnd_executive` | KBL SWS LRC → finance |
| I19 | Neha Chauhan | D&D Executive | IN-CHG | `agency_dnd_executive` | CSN MAL → finance |
| I20 | Prakash Shetty | Operations Admin, Port & Feeder — approves D&D reviews, adjustments and CRO amendments | IN-VOPS | `agency_operations_admin` (GLOBAL), `agency_port_operations` | all 6 → full |
| I21 | Aditya Kamath | Vessel Operations Executive — Nhava Sheva | IN-VOPS | `agency_port_operations`, `agency_feeder_operations`, `logistics_coordinator` | all 6 → container |
| I22 | Manish Solanki | Mundra Desk Executive — vessel ops and DOs at INMUN | IN-MUN | `agency_port_operations`, `agency_do_executive`, `logistics_coordinator` | all 6 → full |
| I23 | Shreya Banerjee | Finance Coordinator — finance hand-off and posting | IN-FIN | `agency_finance_coordinator`, `agency_finance_admin`, `proforma_ar` | all 6 → finance |
| I24 | Karthik Subramanian | Principal Reporting Executive — monthly statements and period close | IN-FIN | `agency_principal_reporting_executive` | all 6 → finance |

### 4.3 Dubai — DJ Logistics FZE (18)

| # | Name | Title | Team | Roles · team role on AE-SALES |
|---|---|---|---|---|
| A1 | Hamad Al Rashidi | Branch General Manager | AE-MGMT | `shipment.management`, `sales_crm.sales_manager` · **COUNTRY_HEAD** |
| A2 | Natalia Sousa | Sales Manager | AE-SALES | `sales_crm.sales_manager` · **SALES_MANAGER** |
| A3 | Ravi Krishnan | Senior Sales Executive — industrial (aluminium, steel, building materials) | AE-SALES | `sales_crm.sales_rep` |
| A4 | Layla Haddad | Sales Executive — re-export and consumer goods | AE-SALES | `sales_crm.sales_rep` |
| A5 | Tomasz Wrobel | Sales Executive — nominated India imports | AE-SALES | `sales_crm.sales_rep` |
| A6 | Faisal Qasim | Pricing Manager | AE-PRICE | `pricing.manager`, `sales_crm.pricing_desk` · **PRICING_APPROVER** |
| A7 | Grace Mwangi | Pricing Executive | AE-PRICE | `pricing.executive`, `sales_crm.pricing_desk` |
| A8 | Ahmed Farouk | Customer Service Lead — accepts handovers | AE-CS | `booking_manager` |
| A9 | Joanne D'Silva | Customer Service Executive | AE-CS | `booking_clerk` |
| A10 | Bilal Siddiqui | Customer Service Executive | AE-CS | `booking_clerk` |
| A11 | Elena Petrova | Operations Manager — closes shipments | AE-OPS | `shipment.ops_manager`, `ops_manager`, `tracking_admin` |
| A12 | Sunil Thapa | Operations Executive — exports | AE-OPS | `shipment.ops_user`, `ops_executor`, `tracking_updater` |
| A13 | Mariam Yousef | Operations Executive — imports and clearance | AE-OPS | `shipment.ops_user`, `tracking_updater` |
| A14 | Kevin Oduya | Operations Executive — haulage and gate | AE-OPS | `ops_dispatcher`, `tracking_updater` |
| A15 | Fatima Noor | Documentation Executive | AE-DOCS | `doc.user`, `shipment.documentation_user` |
| A16 | Rizwan Akhtar | Documentation Executive | AE-DOCS | `doc.user`, `shipment.documentation_user` |
| A17 | Helen Abboud | Finance Manager — ticks "Credit check cleared" | AE-FIN | `profitability_approver`, `proforma_ar`, `proforma_ap` · **FINANCE_APPROVER** |
| A18 | Ajay Varghese | Accounts Executive | AE-FIN | `profitability_editor`, `proforma_ar`, `proforma_ap` |

### 4.4 Who decides each approval

| Approval (policy) | Approver type | Decided by |
|---|---|---|
| Quote, standard (`crm.quote.standard`) | TEAM_ROLE SALES_MANAGER on AE-SALES | A2 |
| Quote, low margin / sell below buy | + PRICING_APPROVER | A6 |
| Quote, payment-term change | FINANCE_APPROVER | A17 |
| Quote, country-head step | COUNTRY_HEAD | A1 |
| Principal master; numbering rule | ROLE `agency_principal_admin` | I2 |
| Container release order | ROLE `agency_container_control` | I8 |
| CRO amendment; D&D review; charge adjustment | ROLE `agency_operations_admin` | I20 |
| B/L release; B/L amendment | ROLE `agency_documentation_manager` | I10 |
| DO release exception or amendment | ROLE `agency_release_manager` | I15 |
| Free-time extension (waivers are not used — D3) | ROLE `agency_commercial_manager` | I3 |
| D&D tariff and carrier buy rates (`pricing`) | pricing policy: standard margin auto-approves | U6 / A6 when routed |

`ir.approval.case.decide` accepts only the user holding the open task (`approval_service.decide_case`). The generator therefore reads the case's pending task and decides **as that user**. It never decides as an administrator.

---

## 5. The six principals (India)

Principals are created in `draft`. Each goes `nvocc.principal.submit` → I2 decides → `active`.

- Every principal has `effective_date` **2025-12-15** (the agency agreements' signing date).
- Approval happens on 2025-12-18, so the effective-date guard is genuinely true when the approval completes. D1 never gets a chance to skip it.

| Code | Name | `principal_type` | Doc prefix | Box owner code | Lanes | Own service (rotation, weekly) | Import TEU | Export TEU |
|---|---|---|---|---|---|---|---|---|
| `PRN-KESTREL` | Kestrel Bay Line | shipping_line | KBL | QKBU | CN→IN, IN→AE | **CIX** CNSHA–CNNGB–CNSHK–INNSA–INMUN · **KGX** INMUN–INNSA–AEJEA | 1,470 (30%) | 600 (25%) |
| `PRN-SAFFRON` | Saffron Wake Shipping | shipping_line | SWS | QSWU | CN→IN | **SWX** CNNGB–CNSHA–INNSA | 980 (20%) | — |
| `PRN-INDIGO` | Indigo Current Lines | feeder_operator | ICL | QICU | IN→AE | **IGF** INMUN–INNSA–AEJEA feeder | — | 720 (30%) |
| `PRN-LANTERN` | Lantern Reef Container Line | shipping_line | LRC | QLRU | CN→IN (South China) | **LPX** CNSZX–CNSHK–INMUN–INNSA | 1,078 (22%) | — |
| `PRN-CINNABAR` | Cinnabar Sea NVOCC | nvocc | CSN | QCSU | CN→IN, IN→AE | slots on CIX and IGF | 882 (18%) | 600 (25%) |
| `PRN-MONSOON` | Monsoon Arc Logistics | nvocc | MAL | QMAU | CN→IN, IN→AE | slots on SWX and IGF | 490 (10%) | 480 (20%) |
| | | | | | | **Total** | **4,900** | **2,400** |

**Planned revenue by principal (USD):**

| Principal | Revenue |
|---|---|
| Kestrel | 581,276 |
| Saffron | 325,103 |
| Indigo | 112,345 |
| Lantern | 357,614 |
| Cinnabar | 386,214 |
| Monsoon | 237,448 |
| **Total** | **2,000,000** |

These are lane revenue × share; the planner reprints the exact split.

**Notes:**

- **Owner codes use the Q-series**, so they read as synthetic.
- **IMO numbers on the principals' vessels are 7 digits with a deliberately wrong IMO check digit.** The platform does not validate IMO check digits, and a number that fails the checksum cannot belong to a real ship.
- **Container numbers carry valid ISO 6346 check digits.** The generator turns on `logistics.equipment.check_digit_validation` to prove it.
- **Numbering rules:** 21 `nvocc.numbering.rule` rows, each with its own `ir.sequence` — one per (principal × document type the principal uses: `cro`, `bl`, `manifest`, `delivery_order`). Formats are `KBLB-2026-000001` (B/L), `KBLC-…` (CRO), `KBLM-…` (manifest) and `KBLD-…` (DO). Each rule goes submit → I2 → `active`.
- **NVOCC principals get their own voyages.** `nvocc.voyage` requires `principal_id`, so Cinnabar and Monsoon get their own `nvocc.voyage` and port-call records on the same vessel and date as the operating line's sailing. Slot sharing is visible, not hidden.

---

## 6. Business parties

All names are invented. Every party has an `@<name>.example` email, a `+… 0000`-style phone, and IEC `0000000000` for Indian traders. Credit hold uses `res.partner.credit_hold` (contributed by `agency_booking`). Roles are `res.partner.role` rows. No boolean flags are used (CLAUDE.md Q4).

**Customers by lane:**

| Lane / side | Code range | Count | Parties (industry) |
|---|---|---|---|
| CN→IN · Indian consignees (agency import customers) | `IN-CNE-01…14` | 14 | Vasundhara Electronics (consumer electronics), Shrijan Solar Modules (solar), Konark Machine Tools (machinery), Aranya Specialty Chemicals (chemicals; 1% DG class 8), Navrang Textile Machinery, Trivedi Auto Components, Sahyadri Home Furnishings, Kumbh Toys & Gifts, Ojas Mobile Accessories, Bharatvarsh Steel Fittings, Pratham Packaging Films, Ganga Sanitaryware Imports, Medhavi Lab Instruments, Rasayan Polymers (resins via INMUN) |
| CN→IN · Chinese shippers | `CN-SHP-01…14` | 14 | One per consignee — e.g. Ningbo Qinghe Machine Works, Shenzhen Lanting Electronics, Shanghai Mingrui Chemical, Xiamen Haiyun Solar |
| IN→AE · Indian exporters (agency export customers) | `IN-EXP-01…12` | 12 | Gomti Basmati Exports (rice), Malabar Spice Traders (spices), Shivneri Fresh Foods (frozen — reefer 45RT), Kaveri Cotton Mills (textiles), Jaipur Handloom House (home textiles), Morbi Vitrified Tiles (tiles), Nashik Engineering Castings, Ludhiana Fastener Works, Vapi Pharma Packaging (non-DG), Vidarbha Agro Commodities (sugar, pulses), Saurashtra Salt & Minerals, Ratnagiri Cashew Processors |
| IN→AE · UAE consignees | `AE-CNE-01…12` | 12 | Dunes Gate General Trading, Creekside Foodstuff Trading, Marina Bay Textiles, Sand Pearl Building Materials, Oasis Spice Souk Trading, Falcon Crest Hardware, Twin Towers Tiles Trading, Al Barari Rice Distributors, Sabkha Steel Supplies, Coral Reef Cashew Trading, Desert Lantern Home Textiles, Mirage Frozen Foods |
| AE→US · Dubai shippers (forwarding customers) | `AE-SHP-01…12` | 12 | Jebel Profile Aluminium Industries (aluminium extrusions), Gulf Arc Building Products, Desert Rose Dates Company, Hatta Steel Wire Industries, Creek Re-Export Electronics, Mirdif Marble & Granite, Palm Coast Lubricants (non-DG), Liwa Glass Containers, Ras Al Khor Furniture Industries, Hessa Paper Products, Qudra Polymers, Warsan Cable Trays |
| AE→US · US consignees | `US-CNE-01…12` | 12 | Brazos Valley Building Supply (HOU), Cape Fear Aluminum Distributors (SAV), Hudson Gate Imports (NYC), Bayou Industrial Wire (HOU), Golden Mesa Home Goods (LAX), Peachtree Specialty Foods (SAV), Lone Prairie Glassworks (HOU), Harborline Electronics Wholesale (NYC), Sierra Pine Furniture (LAX), Gulf Coast Paperworks (HOU), Atlantic Marble Importers (NYC), Palmetto Lubricants Supply (SAV) |
| IN→AE · UAE importers nominating Dubai (forwarding customers) | `AE-IMP-01…06` | 6 | Barsha Wholesale Foods, Qusais Industrial Supplies, Deira Fabric House, Karama Kitchenware Trading, Oasis Silicon Electronics, Hamriya Engineering Spares |
| IN→AE · Indian shippers for those imports | `IN-SHP-01…06` | 6 | Pune Precision Gears, Tirupur Knitwear Exports, Moradabad Brassware Exports, Kolhapur Jaggery & Foods, Bhiwandi Kitchen Steel, Coimbatore Pump Works |

**Vendors and agents:**

| Entity | Role (`res.partner.role.master` code) | Parties |
|---|---|---|
| India | `terminal_operator` | Sheva Bay Container Terminal, Nhava East Box Terminal, Mundra West Container Terminal |
| India | `cfs_operator` | Uran Gateway CFS, Dronagiri Logistics Park CFS, Mundra Coastal CFS |
| India | `depot_operator` | Panvel Empty Yard, Jasai Container Depot, Mundra Box Park |
| India | `transporter` | Raigad Roadlines, Konkan Haulage Co., Kutch Trailer Services, Western Ghats Carriers |
| India | `cha` | Tidemark Customs House Agents, Bandar Clearing & Forwarding Agency, Seaview CHA Services |
| India | `surveyor` / `repair_vendor` / `port_vendor` | Coastline Marine Surveyors / Panvel Container Repairs / Sheva Marine Agency Services (pilotage, towage) |
| Dubai | `carrier` (plus `logistics.carrier`, approved via `logistics.carrier.approve` when governance is on) | Northwind Atlas Lines, Bluefin Meridian Shipping, Starboard Union Shipping (AE→US); Coral Harbour Lines, Quayside Global Lines (IN→AE) |
| Dubai | `transporter` | Sand Dune Transport, Al Khail Trucking, Jebel Express Haulage |
| Dubai | `cha_customs_broker` | Creek Customs Clearing, Gateway Declarations |
| Dubai | `terminal_operator` / `depot_operator` | Jebel Ali South Container Terminal / Jebel Ali Empty Park |
| Dubai | `origin_agent` (India side of nominated imports) | Sahyog Freight Services |
| US | `destination_agent` / `overseas_agent` | Empire Harbor Agency (NYC), Coastal Plain Customs & Delivery (SAV), Lone Star Destination Services (HOU), Pacific Wharf Agents (LAX) |

**Totals:**

- 124 business parties plus the 6 principals, which are delegated to `res.partner`.
- About 130 contacts (child partners), about 180 addresses (billing, office, warehouse) and about 230 role rows.
- Payment terms: NET_30 for most customers, NET_15 for UAE consignees, ADVANCE for 3 new exporters. All come from the seeded `res.payment.term.csv`.
- Credit-hold stories (each lasts 2–5 days, and never spans a month-end close — D5):
  - Ludhiana Fastener Works, April;
  - Saurashtra Salt & Minerals, August;
  - Desert Lantern Home Textiles, June. It is only a consignee, so there is no refusal; the hold only shows on its record.

---

## 7. Masters

| Master | How it is obtained | Rows |
|---|---|---|
| Ports — `logistics.unlocode.master` | **`ede.create`** (see the note below) | 11: CNSHA, CNNGB, CNSZX, CNSHK, INNSA, INMUN, AEJEA, USNYC, USSAV, USHOU, USLAX |
| Facilities — `logistics.facility.master` (TERMINAL / CFS / DEPOT / EMPTY_PARK, with `unlocode_id` and `timezone_id`) | `ede.create` | 12: the 9 India ones in §6 plus 3 at Jebel Ali |
| Vessels — `logistics.vessel` (container / feeder category) | `ede.create` | 22: 6 each on CIX, SWX and LPX, plus 2 each on KGX and IGF. Forwarding bookings carry carrier vessel and voyage as text from the provider confirmation |
| Services — `nvocc.service` | `ede.create` | 5: CIX, SWX, LPX, KGX, IGF |
| Equipment types | **Seeded** by `masters/data` — verified, not created | 22GP, 42GP, 45GP, 45RT |
| Movement event types, free-time types, event maps, role master, payment terms, currencies, rate types | **Seeded** — verified only | — |
| Commodities — `logistics.commodity.master` (`hs_reference`, `dg_flag`, `un_number`, `temperature_control_flag`) | `ede.create` (nothing is seeded in production) | 30 (ELEC, SOLAR, MACH, CHEM, CHEM-DG8, …, ALU, BLDG, DATES, MARBLE) |
| Charge products — `logistics.product.master`, `product_type=charge`; codes align with `logistics.charge` where a charge code exists | `ede.create` (production seeds only DISCOUNT/ADJ, plus DEM/DET/STO/DND from `agency_charges`) | 18. Agency: THCD, THCO, DOF, BLF, IDF (import docs/IGM), EDF (export docs/EGM/VGM), CLN, SEAL, AGCOM, AGFEE. Forwarding: OFR, BAF, TRK, DLV, DOC, CUSE, CUSI, AMS, DTHC |
| FX — `res.exchange.rate`, type MONTHLY | `ede.create`, **progressively** — each month's rows are created at 00:05 UTC on the 1st, in simulated time (D11) | 66: 11 months × 6 directed pairs |
| Free-time terms — `logistics.free.time.term` | `ede.create` | 20: DEM 5 / DET 14 per principal, plus DET 21 on 4 contract customers (`customer_contract` beats the standard) |
| D&D tariffs — `pricing.rate` → `.line` → `.charge.slab` | `ede.create` → `pricing.rate.submit` → approve | 6 (one per principal) |
| Carrier buy rates — `pricing.rate` per carrier × POD × month | created on the 25th of the previous month → submit → approve | 160 |
| Key-account contracts — `pricing.contract` | `ede.create`, active | 8 |
| Customs EDI partner — `nvocc.integration.partner` | create → `begin_testing` → environment=production → `activate`. `endpoint_ref` is a connector reference with no credential, so nothing is sent — same as the shipped demo | 1 |

**Ports are created, not synced.** The memory rule says "UN/LOCODE is sync-not-seed". But no sync exists today: OneMaster is retired, `foundation.servora_sync` is 🔴 not started, and no connector syncs UN/LOCODE. Creating the 11 rows through `ede.create` is how the e2e `CrmFactory` and the masters demo obtain ports, and they are rows a user would key in. They are not an XML seed, so the rule's intent (never ship UN/LOCODE as seed data) is kept. If a Servora sync lands later, it adopts these rows by `code`. **This needs your confirmation — open decision 4.**

**FX series (synthetic, not market data):**

| | Dec 25 | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct |
|---|---|---|---|---|---|---|---|---|---|---|---|
| USD→INR | 85.40 | 85.60 | 85.90 | 86.30 | 86.00 | 85.70 | 86.20 | 86.80 | 87.40 | 87.90 | 88.30 |

- USD→AED is the peg, 3.6725, every month.
- AED→INR is the cross (USD→INR ÷ 3.6725).
- Both directions of every pair are seeded, because `agency_charges/services/fx.py stamp_fx` does not try the inverse.

**D&D tariff:** USD, valid 2026-01-01 to 2026-12-31, imports at INNSA and INMUN. Free days are calendar days, from the seeded event maps.

| Clock | Container | Band 1 | Band 2 |
|---|---|---|---|
| Demurrage (discharge → full gate-out) | 20' | days 1–5 @ 40 | 6+ @ 75 |
| Demurrage | 40' | 1–5 @ 80 | 6+ @ 150 |
| Detention (full gate-out → empty return) | 20' | 1–7 @ 35 | 8+ @ 70 |
| Detention | 40' | 1–7 @ 70 | 8+ @ 140 |

**Tenant config** (`ir.config` writes, each one documented in the run log):

| Key | Value | Why |
|---|---|---|
| `profitability.base_currency` | USD | Group reporting currency |
| `nvocc.charge_fx_rate_type` | MONTHLY | Matches the seeded rates |
| `logistics.equipment.check_digit_validation` | true | Every number is valid, so this proves it |
| `nvocc.finance_connector_required` | false | No finance connector exists. This is the product's documented "manual export process" mode |
| `nvocc.port_call_closure_requires_documents` | false | The shipped voyage demo sets the same value. With the checklist engine unreachable, it refuses every closure (D13) |
| `nvocc.principal_mandatory` | true (default kept) | Every create runs under a switched principal |
| `nvocc.reporting.closure_blocks_on_exceptions` | true (default kept) | The control stays on; see D4 |

---

## 8. Volume plan

**How the months are assigned:**

- Imports count in the month of vessel arrival (ATA) in India.
- Agency exports count in the month of sailing (ATD) from India.
- Forwarding exports count in the month of departure (ETD) from Jebel Ali.
- Forwarding imports count in the month of arrival (ATA) at Jebel Ali.

**Seasonality:**

- January ramps up from go-live.
- Chinese New Year (17 Feb 2026) cuts departures from China, and that dip shows in **March** arrivals in India.
- Ramadan (≈18 Feb – 19 Mar) and Eid soften India→Dubai in March, after a food rush before Ramadan in February.
- The UAE summer dips in June.
- The peak runs August–October: Diwali (8 Nov) drives Indian imports, and Q4 stock-up drives Dubai→US.

**Equipment mix:**

- 20' dry is 22GP.
- 40' boxes split 35% 42GP and 65% 45GP (high cube).
- 3% of export boxes are 45RT reefers (Shivneri Fresh Foods).
- 1% of import boxes are DG class 8 (Aranya Specialty Chemicals).

**Allocation:** months are allocated by largest remainder, so every row sums exactly.

| Lane · measure | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|---|
| **CN→IN agency import — TEU** | 421 | 491 | 363 | 451 | 470 | 459 | 490 | 539 | 579 | 637 | **4,900** |
| 20' | 105 | 123 | 91 | 113 | 118 | 115 | 122 | 135 | 145 | 159 | 1,226 |
| 40' | 158 | 184 | 136 | 169 | 176 | 172 | 184 | 202 | 217 | 239 | 1,837 |
| boxes | 263 | 307 | 227 | 282 | 294 | 287 | 306 | 337 | 362 | 398 | 3,063 |
| B/Ls (shipments) | 131 | 154 | 113 | 141 | 147 | 144 | 153 | 169 | 181 | 199 | 1,532 |
| **IN→AE agency export — TEU** | 212 | 264 | 197 | 216 | 226 | 206 | 216 | 258 | 283 | 322 | **2,400** |
| 20' | 80 | 100 | 75 | 82 | 86 | 78 | 82 | 98 | 107 | 122 | 910 |
| 40' | 66 | 82 | 61 | 67 | 70 | 64 | 67 | 80 | 88 | 100 | 745 |
| boxes | 146 | 182 | 136 | 149 | 156 | 142 | 149 | 178 | 195 | 222 | 1,655 |
| bookings (shipments) | 81 | 101 | 75 | 83 | 87 | 79 | 83 | 99 | 108 | 123 | 919 |
| **IN→AE forwarding import — TEU** | 53 | 65 | 50 | 54 | 56 | 51 | 54 | 65 | 71 | 81 | **600** |
| 20' | 15 | 19 | 14 | 16 | 16 | 15 | 16 | 19 | 21 | 23 | 174 |
| 40' | 19 | 23 | 18 | 19 | 20 | 18 | 19 | 23 | 25 | 29 | 213 |
| boxes | 34 | 42 | 32 | 35 | 36 | 33 | 35 | 42 | 46 | 52 | 387 |
| shipments | 23 | 28 | 21 | 23 | 24 | 22 | 23 | 28 | 31 | 35 | 258 |
| **AE→US forwarding export — TEU** | 168 | 184 | 194 | 198 | 203 | 210 | 222 | 235 | 243 | 243 | **2,100** |
| 20' | 24 | 26 | 28 | 28 | 29 | 30 | 32 | 33 | 35 | 35 | 300 |
| 40' | 72 | 79 | 83 | 85 | 87 | 90 | 95 | 101 | 104 | 104 | 900 |
| boxes | 96 | 105 | 111 | 113 | 116 | 120 | 127 | 134 | 139 | 139 | 1,200 |
| shipments | 51 | 55 | 58 | 60 | 61 | 63 | 67 | 71 | 73 | 73 | 632 |
| **All lanes — TEU** | **854** | **1,004** | **804** | **919** | **955** | **926** | **982** | **1,097** | **1,176** | **1,283** | **10,000** |
| All lanes — boxes | 539 | 636 | 506 | 579 | 602 | 582 | 617 | 691 | 742 | 811 | 6,305 |
| All lanes — shipments | 286 | 338 | 267 | 307 | 319 | 308 | 326 | 367 | 393 | 430 | 3,341 |

**TEU by lane:** CN→IN 4,900 · IN→AE 3,000 · AE→US 2,100 = **10,000**. **By entity:** India 7,300 · Dubai 2,700.

**Boxes per shipment:**

| Lane | Boxes per shipment |
|---|---|
| Agency import B/L | 2.0 |
| Agency export booking | 1.8 |
| AE→US forwarding | 1.9 |
| IN→AE forwarding | 1.5 |

**Pipeline beyond the snapshot** (about 170 shipments). These are excluded from every figure above by their anchor date. They exist so 31 October looks like a live day:

- import B/Ls for vessels arriving 1–7 Nov (pre-arrival documentation done);
- export bookings for sailings 1–12 Nov (CRO sent, some empties out);
- forwarding bookings for departures 1–15 Nov.

---

## 9. Money plan

### 9.1 What "booked" means

| Product | Booked revenue = |
|---|---|
| Forwarding (Dubai) | Σ `logistics.shipment.profitability.total_revenue_amount`, in USD (base currency), over Dubai shipments with ETD (exports) or ATA (imports) in Jan–Oct, excluding cancelled. All forwarding sell lines are quoted in **USD**; AED local charges are converted at the 3.6725 peg on the quote, so the figure is **exact to the cent**. |
| Agency (India) | (a) Σ profitability `total_revenue_amount` (USD) over India shipments anchored in Jan–Oct. This covers local charges billed to customers in **INR** (converted at that month's rate) and commission plus handling fees billed to principals in **USD**. **Plus** (b) Σ `nvocc.dnd.calculation.net_amount` (USD) in every live state from `calculated` to `posted`, including `disputed`, plus `estimated` accruals on clocks still running at 31 Oct. Superseded and cancelled calculations are excluded. |

**These are gross agency billings.** The D&D (USD 733k) and THC (USD 667k) are collected for the principal and the terminal. The platform has no payable ledger — N9-11 is reference only — so no principal payable is modelled. On the plan, agency-retained revenue (commission, handling, document fees, and the THC, cleaning and seal margins) is about **USD 0.61M**. Open decision 5 asks whether you want the headline on that basis instead.

### 9.2 Rate card (USD unless stated; base rates before calibration)

**Agency import (CN→IN), per box:**

- THCD INR 9,800 (20') / INR 14,600 (40'). Cost: 93% (terminal).
- Cleaning INR 1,100 / INR 1,600. Cost: 70%.
- Commission 5% of principal ocean freight (USD 640 / 1,060 × monthly index).
- Handling fee USD 22 / 32.

**Agency import, per B/L:** DO fee INR 6,200 and import documentation/IGM fee INR 3,400.

**Agency import D&D:** per the §7 tariff. On plan, 25% of boxes overrun demurrage (average 6 days) and 32% overrun detention (average 8 days).

**Agency export (IN→AE), per box:**

- THCO INR 9,400 / INR 14,000. Cost: 93%.
- Seal INR 450. Cost: 40%.
- Commission 4% of freight (USD 430 / 720 × index).
- Handling USD 18 / 26.

**Agency export, per booking:** B/L fee INR 5,200 and export documentation/EGM/VGM fee INR 1,700.

**Forwarding AE→US, per box:**

- OFR by port of discharge (20' / 40'): NYC and SAV 1,750 / 2,800 · HOU 1,950 / 3,050 · LAX 2,300 / 3,600. Cost: 88.5% (carrier buy rate).
- BAF 175 / 320. Pass-through.
- THCO AED 870 / 1,160. Cost: 96%.
- Pickup trucking AED 950 / 1,250 on 75% of shipments. Cost: 82%.

**Forwarding AE→US, per shipment:** DOC AED 350, B/L AED 275, export customs AED 320, AMS/ISF USD 45.

**Forwarding IN→AE (FOB, consignee-nominated), per box:**

- OFR 470 / 760. Cost: 88%.
- BAF 55 / 90.
- DTHC AED 905 / 1,210. Cost: 97%.
- Delivery trucking AED 820 / 1,120 on 90% of shipments. Cost: 84%.

**Forwarding IN→AE, per shipment:** DO AED 525, import clearance AED 450, documentation AED 250.

**Ocean-freight market index:**

| Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct |
|---|---|---|---|---|---|---|---|---|---|
| 1.00 | 1.02 | 0.97 | 0.96 | 0.98 | 1.01 | 1.04 | 1.07 | 1.06 | 1.03 |

**Calibration** makes the plan sum exactly. It applies one factor per product: forwarding OFR × **0.9776**, and agency handling fee × **1.0093** (both values from today's plan). Unit rates are then rounded to whole USD. The rounding residue is spread +/−1 USD across the last October boxes of each lane, so the plan totals are exact before a single command runs.

### 9.3 Planned revenue, cost and margin (USD)

| Lane (product) | Jan | Feb | Mar | Apr | May | Jun | Jul | Aug | Sep | Oct | **Total** | Direct cost | GM |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| CN→IN agency import | 140,114 | 163,608 | 120,127 | 149,404 | 156,203 | 152,603 | 162,741 | 179,075 | 191,698 | 209,943 | **1,625,516** | 451,111 | 72.2% |
| IN→AE agency export | 33,338 | 41,505 | 30,689 | 33,763 | 35,517 | 32,278 | 33,779 | 40,236 | 43,825 | 49,554 | **374,484** | 207,042 | 44.7% |
| **Agency (India)** | 173,452 | 205,113 | 150,816 | 183,167 | 191,720 | 184,881 | 196,520 | 219,311 | 235,523 | 259,497 | **2,000,000** | 658,153 | 67.1% |
| AE→US forwarding export | 350,744 | 389,351 | 396,019 | 401,094 | 417,425 | 441,410 | 477,089 | 515,319 | 529,528 | 518,389 | **4,436,368** | 3,859,616 | 13.0% |
| IN→AE forwarding import | 49,401 | 61,181 | 45,741 | 49,504 | 51,689 | 47,834 | 51,222 | 62,472 | 68,156 | 76,432 | **563,632** | 479,064 | 15.0% |
| **Forwarding (Dubai)** | 400,145 | 450,532 | 441,760 | 450,598 | 469,114 | 489,244 | 528,311 | 577,791 | 597,684 | 594,821 | **5,000,000** | 4,338,680 | 13.2% |
| **Group** | 573,597 | 655,645 | 592,576 | 633,765 | 660,834 | 674,125 | 724,831 | 797,102 | 833,207 | 854,318 | **7,000,000** | | |

**Revenue components (USD):**

| Lane | Components |
|---|---|
| Agency import | THC 447,847 · cleaning 49,447 · commission 139,025 · handling 86,553 · B/L-level fees 169,604 · D&D 733,040 |
| Agency export | THC 218,932 · seal 8,589 · commission 37,751 · handling 36,082 · B/L-level fees 73,129 |
| AE→US forwarding | OFR 3,261,508 · BAF 340,500 · THC 355,344 · trucking 287,951 · per-shipment fees 191,065 |
| IN→AE forwarding | OFR 242,348 · BAF 28,740 · DTHC 113,057 · trucking 93,428 · per-shipment fees 86,059 |

Per TEU, these come to agency import ≈ 332, agency export ≈ 156, AE→US ≈ 2,113, IN→AE forwarding ≈ 939.

**How exact the totals will be:**

- **Forwarding** is exact to the cent: USD lines × integer quantities, with the AED peg.
- **Agency** is accepted within **±0.25% (±USD 5,000)**. Two amounts are not the planner's own: the engine computes D&D, and INR lines convert at the monthly rate the profitability engine resolves. Phase 2's first month compares the engine's D&D with the planner's prediction line by line before the run continues (§16).

---

## 10. Lifecycles

**Notation:**

- Day offsets are relative to the anchor (§8), in local business hours (07:00–21:00), so no event crosses a UTC month boundary.
- "→" means one command dispatch by the named persona, followed by an event drain.
- `+T` marks DO commands that are followed by `env.workflow.transition(<code>)` (D7).
- Every root record carries a deterministic business reference, which is also the idempotency key (§15.6).

### 10.1 Agency import, China → India (per B/L; anchor A = ATA at INNSA or INMUN)

| When | Persona | Commands | Result |
|---|---|---|---|
| weekly, A−14 | I21 / I22 | `ede.create nvocc.voyage` (+ rotation lines, a `nvocc.port.call` per India port, a discharge `nvocc.load.discharge.list` with a line per box) → `nvocc.voyage.activate` | voyage `active`, port call `planned` |
| A−9 | I4 / I5 | `nvocc.principal.switch` (cached per user and principal) → `ede.create logistics.booking`. The booking carries: IMPORT shipment type; CN POL, IN POD; `primary_provider_booking_ref` = origin booking number; parties shipper, consignee and notify; cargo line with commodity, packages and weight; principal stamped by `PrincipalScope`. Then `logistics.booking.confirm` | `confirmed` — this records the origin office's booking, as Enhancement 01 does, but confirmed rather than left in draft (D5) |
| A−9 | I4 | `logistics.shipment.create_from_booking` (`hierarchy_type=direct`, D14) → `ede.create logistics.shipment.attachment` → `logistics.shipment.mark_ready` → `set_operational_status in_transit` | shipment `ready_for_execution`; profitability header created by event |
| A−9 | I9 | per box: `ede.create logistics.equipment` (a fresh Q-series number with a valid check digit — D6) → `equipment-operations.allocate.usage` | usage open |
| A−8 | I13 / I14 | `nvocc.bl.create_from_booking` (+ container, cargo and party lines) → `submit_for_review` → `send_to_customer` → `record_customer_decision(approved, shipper contact)` → `approve` | `approved` once I10 decides |
| A−8 | I10 | `ir.approval.case.decide APPROVE` | |
| A−8 | I13 / I14 | `nvocc.bl.issue` → `nvocc.bl.telex_release` (originals surrendered at origin) | `telex_released`, numbered by the principal's B/L rule |
| A−2 | I13 / I14 | per port call: IGM `nvocc.manifest` (import) → `assemble` → `mark_ready` → `submit` (customs EDI partner) → (A−1) `record_acceptance` | IGM `accepted` |
| A−2 | I21 / I22 | `nvocc.load_discharge_list.issue` | |
| A | I21 / I22 | `nvocc.port_call.arrive` → `berth` → `start_operations`. The voyage commences by itself | `operations_started` |
| A (07:00–A+1) | I21 / I22 | per box: `equipment-operations.log.movement.event DISCHARGED` (port_call_id, voyage_id, terminal) | |
| A+1 | I21 / I22 | `complete_operations` → `nvocc.port_call.reconcile` → `sail` | `sailed`; 0.5% of boxes are short-landed → `nvocc.discrepancy.assign` → discharged later → `resolve` |
| A | I4 | `ede.create logistics.shipment.charge` (THCD and CLN per container type, AGCOM, AGFEE, DOF, IDF) → `logistics.profitability.recompute` → `set_operational_status arrived` | revenue booked |
| A | I16 / I17 / I22 | `nvocc.do.create_from_bl` — the B/L is already released, so the document envelope is copied at create (avoids D8) | DO `draft` |
| A+2…A+6 | I16 / I17 | `nvocc.customs.release.record cleared` (release date, Bill of Entry ref). In 5% of cases: `exam` → `hold` (with reason) → `cleared` → I15 `logistics.document.release_hold` (clearing customs does not lift a hold, BR-N6-15) | `cleared` |
| clear+0 | I16 / I17 | `nvocc.do.check_readiness` (must be READY) → `submit_for_validation` +T → `mark_ready` +T → `issue` +T → `log_communication` (to the consignee's transporter) → `mark_sent` +T → `await_release` +T | `release_pending` |
| clear+0 | I23 | `logistics.proforma.document.create_from_profitability` (local-charges invoice to the consignee, INR) | |
| +0…+3 | I16 / I17 | per line: `nvocc.do.record_cargo_release` +T (`partial_release` / `release`). This logs GATE_OUT_FULL through the Phase 4 command, which stops demurrage | `released` |
| +3…+24 | I9 | `log.movement.event EMPTY_RETURN` at the depot | |
| return | I16 / I17 | `nvocc.do.complete` +T → I13 `nvocc.bl.close` | DO `completed`, B/L `closed` |
| return+1 | I18 / I19 | `nvocc.charge.calculate`, demurrage, then detention (booking, shipment, equipment_ids, USD, organization) | `calculated` — zero or an amount |
| return+1 | I18 / I19 | non-zero only: `nvocc.charge.submit_for_review` | `pending_review` |
| return+2 | I20 | decide APPROVE | `approved` |
| return+2 | I23 | `nvocc.charge.mark_billing_ready` → `nvocc.charge.send_to_finance` | `sent_to_finance` (same day — D5) |
| return+4 | I23 | `nvocc.charge.mark_posted` (synthetic journal reference) | `posted`, within the reconciliation SLA |
| return+2 | I4 | `logistics.shipment.capture_pod` (generated one-page gate-pass PDF) → `set_operational_status closed` | shipment `closed` |
| weekly | I21 / I13 | `nvocc.port_call.close` · `nvocc.voyage.complete` → `close` · IGM `nvocc.manifest.close` | closed |

**Variants, at planned rates:**

- 5% customs examination with a hold (above).
- 2% free-time extension: `nvocc.freetime.extension.submit` → I3 decides. `approved_days` is ≥1 before submit (D1). It is approved before empty return, so the first calculation already includes it (D2). 6 late ones use the Enhancement 01 workaround, `nvocc.charge.recalculate` with a reason, which supersedes the earlier calculation.
- 1% `nvocc.charge.place_hold finance_review` → `release_hold`, released before any close.
- 1% DG cargo: UN number on the cargo line.

### 10.2 Agency export, India → Dubai (per booking; anchor D = ATD)

| When | Persona | Commands | Result |
|---|---|---|---|
| weekly, D−10 | I21 / I22 | voyage + port calls + load list (lines added as boxes are allocated) → `activate`; `nvocc.voyage.booking.link` per booking | |
| D−12 | I6 / I7 | switch → `ede.create logistics.booking` (EXPORT; INNSA/INMUN → AEJEA; parties; cargo line — reefers carry the set-point, DG carries the UN number) → `logistics.booking.confirm` | `confirmed`. For 6 bookings the confirm is **refused** on credit hold, quoting the reason (a planned refusal, asserted); finance clears the hold and the confirm passes |
| D−12 | I6 | shipment `create_from_booking` → attachment → `mark_ready` | |
| D−11 | I7 | `nvocc.cro.create_from_booking` (pickup facility = empty depot; reefers type the set-point into `special_conditions`) → `nvocc.cro.submit` | `pending_approval` |
| D−11 | I8 | decide APPROVE | `approved` |
| D−11 | I7 | `nvocc.cro.issue` → `ede.create nvocc.cro.communication` → `nvocc.cro.send` | `sent` |
| D−8…D−5 | I9 | per box: create equipment → allocate usage → `nvocc.cro.record_release` (empty release / gate-out) → `logistics.equipment.seal.application` | `fully_released` |
| D−4…D−2 | I9 | `log.movement.event GATE_IN_FULL` at the terminal | |
| D−4 | I11 / I12 | `ede.create nvocc.shipping.instruction` (+ cargo, containers, parties) → `validate`. In 8% of cases one blocking finding (VGM mismatch, seal mismatch, missing HS, DG or reefer gap) → `ede.update` the line → `validate` again. Then `nvocc.bl.create_from_booking` → `apply_to_bl` | SI `applied` |
| D−2 | I21 / I22 | `load_discharge_list.issue` | |
| D−1…D | I21 / I22 | `arrive` → `berth` → `start_operations` → per box `LOADED` → `complete_operations` → `reconcile` → `sail` | `sailed`. 1% short-loaded → `nvocc.discrepancy.roll_over` → loads next week |
| D−2 | I11 / I12 | B/L `submit_for_review` → `send_to_customer` → `record_customer_decision(approved)` → `approve`; then I10 decides | |
| D…D+1 | I11 / I12 | `nvocc.bl.issue` → `release` (OBL, 60%) or `telex_release` (on payment, 40%) | released |
| D+1 | I11 / I12 | EGM: `assemble` → `mark_ready` → `submit` → (D+2) `record_acceptance` → `close` | `closed` |
| D | I6 | charges THCO, SEAL, BLF, EDF, AGCOM, AGFEE → `recompute` → `in_transit` → I23 proforma invoice to the shipper | |
| D+5 | I6 / I7 | `set_operational_status arrived` (the principal's Jebel Ali agent reports arrival) → `nvocc.cro.complete` | the agency's scope ends |
| D+30 | I11 / I12 | `nvocc.bl.close` | `closed` |

**Variant:** 2% post-issue B/L amendment — `nvocc.bl.amendment` create → `submit` → I10 `record_decision(approved)` → `apply`. These are header fields only (consignee address, marks).

### 10.3 Agency month-end (per principal; business day 3 of the following month, 17:00 IST)

| Step | Persona | Commands |
|---|---|---|
| 1st of the month, 09:00 | I24 | `ede.create nvocc.reporting.period.closure` (`period_code` YYYY-MM, `open`) |
| BD3 16:00 | generator | **Pre-close assertion (D5).** For this principal: no booking in draft, pending or requested; no D&D calculation in draft or estimated; every billing-ready calculation has a sync row; no sync needs attention. A failure halts the run and is reported, never waived |
| BD3 16:30 | I24 | `nvocc.reporting.period.validate {closure_id}`, dispatched with the id (D10) |
| BD3 16:40 | I24 | `nvocc.principal.report.generate monthly_statement` → `nvocc.principal.report.submit` (synthetic recipient `agency.accounts@<principal>.example`) |
| BD3 17:00 | I24 | `validate` → `nvocc.reporting.period.close` |
| BD3 17:30 | I23 | Principal commission statement: `logistics.proforma.document.create_from_charges`, selecting the month's AGCOM and AGFEE lines |

- **September, PRN-LANTERN:** a dispute raised on 24 Sep (§10.6) becomes an `open_dispute` reconciliation exception, so the close is **refused**. That is a planned refusal; it is recorded and the closure stays `open`. The other 5 principals close September.
- **October:** closures are created on 1 Oct and **validated** at 31 Oct 18:00, never closed. The checklist then shows the live blockers: estimated D&D, open exceptions, no statement yet.

### 10.4 Daily jobs, in simulated time

**Jobs:** `nvocc.cro.expiry` · DO expiry · `nvocc.voyage.operations.sweep` · `nvocc.charge.recalculation.sweep` · `nvocc.reconciliation.sweep`.

**When:** at their cron times (02:00–03:30 IST), dispatched the way `foundation.jobs` runs them.

**Planner guarantees:**

- CRO and DO validity always covers the planned release.
- Hand-offs are posted inside the SLA.

**Failure rule:** if the reconciliation sweep raises an exception the plan did not schedule, the generator treats it as a generator bug and halts. It means real money was left unreconciled.

**Year-end accrual run:** at 31 Oct 18:00, after every September close is done, I18 and I19 run `nvocc.charge.calculate` on every clock still running. That leaves about 200 `estimated` calculations — the October accrual.

### 10.5 Forwarding export, Dubai → US (per shipment; anchor E = ETD Jebel Ali)

| When | Persona | Commands | Result |
|---|---|---|---|
| once per account (Jan–Sep) | A3 / A4 / A5 | `ede.create crm.lead` (shipment type, mode, load type, service mode, lanes, commodity, estimate, branch, team AE-SALES) → `crm.lead.convert_to_opportunity` → `crm.opportunity.close_as_won` (`expected_close_date`) | 40 leads: 30 won, 10 dropped. About 55 opportunities in total, including lane extensions |
| 25th of each month | A7 → U6 | carrier buy rates for next month (`pricing.rate` + lines) → `submit` → approve | approved |
| E−18 | A3 / A4 | `crm.opportunity.create_quote` → `ede.update crm.quote` (incoterm CFR/CIF, `payment_terms_days`, expiry E+15) → `crm.quote.create_lines_from_rate` (OFR, BAF) + `ede.create crm.quote.line` (local charges, USD) → `crm.quote.submit` → `crm.quote.request_approval` | `QUOTE_UNDER_APPROVAL` |
| E−18 | A2 (+A6 on low margin, 15%) | decide APPROVE | `QUOTE_APPROVED` |
| E−17 | A3 / A4 | `crm.quote.send` | `QUOTE_SENT` |
| E−17 | A17 | `workflow.requirement.complete` ("Credit check cleared", on every sent quote — D12) | |
| E−14 | A3 / A4 | `crm.quote.accept` (`decided_by_contact_id`) | `QUOTE_ACCEPTED`; acceptance and handover created |
| E−13 | A8 | handover: `ede.create crm.handover.party` (consignee, billing party) · document items → received · operational owner → `confirm_documents` → `submit` → `start_review` → `accept` | handover `accepted` — through the real commands, never the direct status write scene 04 used |
| E−13 | A9 / A10 | `logistics.booking.create_from_handover` (POL/POD overrides) → line quantities and weights → feasibility findings → link → `logistics.booking.feasibility.check.commit` → `request_provider` → provider request create → `send` | `requested` |
| E−11 | A9 / A10 | provider request `confirm` (carrier booking number, vessel, voyage, ETD/ETA, SI/VGM/gate-in cut-offs, container numbers) → `logistics.booking.confirm` | `confirmed` |
| E−5 | A12 | `logistics.booking.mark_ready` (registers carrier equipment usages) → `logistics.shipment.create_from_booking (direct)` → attachment → `mark_ready` | profitability header + tracking activation |
| E−5 | A18 | `logistics.proforma.document.create_from_profitability` (customer invoice, USD) | |
| E−4…E | A14 / A12 | `tracking.milestone.manual_update`: pickup, gate-in, loaded, departed → `set_operational_status in_transit` | |
| E−1…E+1 | A15 / A16 | `logistics.shipment.generate_hbl` / `logistics.document` → `prepare_from_subject` → `issue` → `release {approved}` (on payment) | HBL released |
| E+T (27–35 days) | U8 | arrival milestone → `set_operational_status arrived` | |
| arrival +2…+7 | A12 | release milestone → `logistics.shipment.capture_pod` (destination agent's signed release, generated PDF) | `delivered` |
| E+3…+12 | A18 | vendor proforma bills: `create_from_charges` for carrier, trucker, terminal and broker | |
| POD+5…+20 | A11 | `set_operational_status closed` → `crm.handover.complete` → `tracking.activation.close` | `closed` |

- **Lost and expired quotes (300):** same path to `QUOTE_SENT` and credit check, then `crm.quote.mark_lost` (lost reason) at E−10, or `mark_expired` at expiry.
- **Quote revisions:** 10% of quotes get a version 2 after a customer counter, *if* the revision command is confirmed in the Phase 3 dry run (risk R5).

### 10.6 Forwarding import, India → Dubai (per shipment; anchor I = ATA Jebel Ali)

This is the same CRM and booking path as §10.5, with these differences:

- shipment type IMPORT, incoterm FOB, POL INNSA or INMUN, POD AEJEA;
- the customer is the UAE importer; the carrier is Coral Harbour or Quayside; the origin agent is Sahyog Freight Services.

| When | Event |
|---|---|
| I−25 | Quote |
| I−20 | Booking confirmed |
| I−8 | Origin gate-in |
| I−6 | Loaded and departed |
| I | Arrived |
| I+0…+1 | DO collected |
| I+1…+3 | `CUSI` clearance milestone |
| I+2…+5 | Delivered — POD, then vendor bills |
| POD+5…+20 | Closed |

### 10.7 Disputes and exceptions (September–October only — D4)

| What | Count | Path | State at 31 Oct |
|---|---|---|---|
| Demurrage or detention disputes | 14 (2 in late Sep on PRN-LANTERN, 12 in Oct) | `nvocc.charge.dispute.open` (reason, disputed amount, customer reference) → 9 `start_review` | all open (`open` / `under_review`); calculation `disputed` |
| Finance rejection | 1 (Oct, PRN-KESTREL) | `nvocc.charge.mark_rejected` ("Tax code not mapped…") → the sweep raises a `finance_rejection` exception → `nvocc.reconciliation.exception.assign` | owned, under review |
| Hand-off awaiting response | 3 (Oct, PRN-CINNABAR) | left un-posted past the SLA → the sweep raises them | open |
| Customs holds | 77 across the year | §10.1 | all lifted |

---

## 11. State mix at 31 Oct 2026

From a seeded Monte Carlo over the plan. The generator prints the exact figures at plan time.

| Record type | State at snapshot | Count |
|---|---|---|
| Agency import B/Ls (1,532) | `closed` (DO completed, empties back) | ≈1,403 |
| | `telex_released`, DO open | ≈129 |
| Agency import DOs | `completed` | ≈1,403 |
| | `released`, empty still out (detention running) | ≈78 |
| | `issued` / `release_pending` | ≈10 |
| | `draft` / `pending_validation` (customs pending or DO not yet issued) | ≈41 |
| Customs entries | `cleared` | ≈1,500 |
| | `pending` / `exam` / `hold` | ≈32 |
| D&D calculations (≈3,270) | `posted` | ≈1,280 |
| | `calculated`, zero charge | ≈1,700 |
| | `estimated` (October accrual) | ≈200 |
| | `disputed` | 14 |
| | `pending_review` → `sent_to_finance` | ≈60 |
| | `superseded` | 6 |
| Finance syncs | `posted` | ≈1,280 |
| | `sent`, not yet posted | ≈20 |
| | rejected / overdue | 4 |
| Agency export bookings (919) | `confirmed` | all |
| CROs | `completed` | ≈915 |
| | `fully_released` | ≈4 |
| Agency export shipments | `arrived` | ≈902 |
| | `in_transit` | ≈13 |
| | sailed, B/L not yet issued | ≈4 |
| Agency export B/Ls | `closed` (sailed by 1 Oct) | ≈810 |
| | `released` / `telex_released` | ≈109 |
| EGMs | `closed` | all but ≈4 (`submitted`) |
| Port calls (≈688) | `closed` | ≈650 |
| | `sailed` (last 2 weeks) | ≈38 |
| | plus Nov calls `planned` | ≈10 |
| Period closures (60) | Jan–Sep `closed` | 53 |
| | Sep PRN-LANTERN `open` (held by its dispute) | 1 |
| | Oct `open`, validated with live blockers | 6 |
| Principal statements | Jan–Sep submitted (v1; 3 months also have a v2 correction) | 54 |
| Forwarding quotes (≈1,225) | `ACCEPTED` | 890 |
| | `LOST` | ≈245 |
| | `EXPIRED` | ≈55 |
| | `SENT`, awaiting customer (credit pending on ≈5) | ≈25 |
| | `DRAFT` / `UNDER_APPROVAL` | ≈10 |
| AE→US shipments (632) | `closed` | ≈521 |
| | `delivered` (POD), bills pending | ≈29 |
| | `arrived` | ≈11 |
| | `in_transit` | ≈71 |
| IN→AE forwarding shipments (258) | `closed` | ≈236 |
| | `delivered` | ≈17 |
| | `arrived`, clearance pending | ≈5 |
| Profitability headers | `estimated` — the stage has no command off `estimated` (D15) | all |
| Pipeline (beyond 31 Oct) | import B/Ls pre-arrival (telex-released, IGM not yet filed) | ≈45 |
| | export bookings for 1–12 Nov (CRO sent, ≈30 with empties out) | ≈42 |
| | forwarding bookings for 1–15 Nov | ≈35 |
| | forwarding imports on water | ≈15 |

---

## 12. Product defects — what the generator does about each

The **Source** column says where each defect is recorded: CHANGELOG 0.3.0 *Known open*, Enhancement 01, or **new** — found while writing this spec, from code reading, not reproduced.

| # | Defect | Source | Generator response |
|---|---|---|---|
| D1 | Completing an approval skips the next transition's guard | 0.3.0, Enh 01 | **Avoid.** The planner makes every guard condition true before submit: principal `effective_date` ≤ approval date, extension `approved_days` ≥ 1, CRO amendments described. A self-check re-evaluates each guard's condition after completion and fails the run if it is false |
| D2 | Approving a free-time extension does not re-run the affected charges | 0.3.0, Enh 01 | **Avoid** for 90% of extensions: they are approved before the cycle completes, so the first calculation includes them. The 6 late ones use the Enh 01 workaround — `nvocc.charge.recalculate` with a reason |
| D3 | Applying any charge waiver is refused (the amount reaches the guard as a string) | 0.3.0, Enh 01 | **Avoid.** No waivers. Disputes stay open (D4) |
| D4 | Stored values computed from a workflow status go stale: a resolved dispute or exception still reads `is_open`, which also blocks period close | 0.3.0, Enh 01 | **Avoid by timing.** Disputes and exceptions are raised only in Sep–Oct, and none is resolved. PRN-LANTERN's September close is held, as a genuine state. `closure_blocks_on_exceptions` is not switched off (open decision 6) |
| D5 | **Period-close checks are not bounded by the period.** `_check_bookings` and `_check_dnd_charges` read **all** of the principal's bookings and calculations, so one draft booking for next month blocks this month's close | **new** — `agency_reports/services/closure.py` | **Schedule around it.** Agency bookings are confirmed in the same step as creation. D&D is calculated only on completed cycles until the September close (≈5 Oct). Hand-off happens the same day as billing-ready. The pre-close assertion (§10.3) runs before every close |
| D6 | **The D&D date feed reads a container's whole movement history** (first gate-out and empty return, last discharge) — the same root as the known "release readiness reads whole history" | 0.3.0 (readiness) + **new** (`equipment_operations/services/dd_feed.py`) | **Avoid.** One trip per container number. 6,305 trips means 6,305 boxes |
| D7 | DO commands run their checks but never move the stage | 0.3.0, stage_chain | **Workaround:** after each DO command, `env.workflow.transition(<code>)` — the stage_chain recipe. The guard still runs, and nothing writes the stage directly. Each one is logged |
| D8 | A DO copies the B/L's document link only at create | 0.3.0 | **Avoid.** DOs are created with `nvocc.do.create_from_bl` after the B/L is released |
| D9 | Closing a period discards waivers recorded on its checklist | 0.3.0 | **Avoid.** No waivers; every check genuinely passes |
| D10 | The Run Closure Checks button sends no closure id | 0.3.0 | Not a generator issue: `validate` is dispatched with `closure_id` |
| D11 | **Profitability FX takes the latest-dated rate regardless of the transaction date** (`_latest_rate_row` orders by `rate_date desc`) | **new** — `profitability/services/currency.py` | **Workaround:** FX rows are created progressively on the 1st of each simulated month. **After generation, a user who recomputes an old India shipment re-converts it at October's rate.** This should be fixed in the product |
| D12 | The quote's SENT checklist blocks *every* outbound transition, including `mark_lost` and `mark_expired`, until "Credit check cleared" is ticked | brief + `workflow_crm_quote.xml` | **Process choice:** finance clears credit on every sent quote within a day. Quotes still awaiting credit at 31 Oct stay `SENT` |
| D13 | Port-call closure with "requires documents" on refuses every closure (checklist engine unreachable) | voyage demo | Same as the shipped demo: the config is set to `false`, and the change is logged |
| D14 | `house_master_linked` moves every charge to the house, and profitability reads only master charges, so revenue would be zero | **new** (inferred, `shipment.py` × `revenue.py`) | **Avoid.** FCL shipments are `direct` |
| D15 | The profitability header has no command that moves it off `estimated` | **new** (gap) | Left as is: every header reads `estimated`. Documented |
| D16 | The N9 agency-fee view labels categories (`agency`, `agency_fee`, `commission`) that the profitability category enum does not contain | **new** (gap) | KPIs select AGCOM/AGFEE by **product code**. The view will show nothing — listed as a finding |
| D17 | Some list columns render `[object Object]` | 0.3.0 | Cosmetic; ignored |
| D18 | Event-handler failures are swallowed into the dead-letter queue (stage_chain prints them and continues) | stage_chain | **Stricter:** a dead-lettered event halts the run |

The generator never uses `sudo()` or `with_company_test(False)` for a business write — its only system writes are the ones the product makes as system too: the scheduler's `ir.job.run` rows (`steps/jobs.py`) and event handlers (`boot.py` drain). It never writes a `workflow=True` field, and never writes a status with `ede.update`. Reads use the model proxy directly, as CLAUDE.md requires.

---

## 13. Record-count estimates

| Area | Model → approximate rows |
|---|---|
| Group | `res.organization` 3 · `res.team` 22 (+3 root) · `res.team.type` 8 · `res.user` 50 · `ir.rbac.role.binding` ≈140 · `res.team.role.assignment` 4 · `nvocc.principal.entitlement` ≈95 |
| Parties | `res.partner` ≈265 (124 business, 6 principals, ≈130 contacts, plus user and org shadows) · `res.partner.role` ≈230 · `res.partner.address` ≈180 |
| Masters | ports 11 · facilities 12 · vessels 22 · services 5 · commodities 30 · products 18 · FX 66 · free-time terms 20 · `pricing.rate` 166 (+≈600 lines, ≈72 slabs) · contracts 8 · numbering rules 21 (+21 `ir.sequence`) · integration partner 1 |
| CRM | `crm.lead` 40 · `crm.opportunity` ≈55 · `crm.quote` ≈1,225 (+ versions ≈1,225, lines ≈9,800) · acceptances 890 · `crm.handover` 890 (+ parties ≈1,780, document items ≈3,560) · `ir.workflow.requirement.state` ≈1,220 |
| Booking and shipment | `logistics.booking` ≈3,490 (+ parties ≈10,500, lines ≈3,500, charges ≈7,100, provider requests 890, feasibility checks 890) · `logistics.shipment` ≈3,420 (+ charges ≈24,000, attachments ≈3,420, PODs ≈3,300) |
| Money | `logistics.shipment.profitability` ≈3,420 (+ charge lines ≈24,000, currency lines ≈13,000) · `logistics.proforma.document` ≈5,350 (+ lines ≈25,000) |
| Equipment | `logistics.equipment` ≈6,650 · `logistics.equipment.usage` ≈6,650 · `logistics.equipment.movement.event` ≈14,500 · seal applications ≈1,700 · tracking milestones ≈6,200 |
| Agency voyage | `nvocc.voyage` ≈430 (+ rotation lines ≈1,300) · `nvocc.port.call` ≈690 · load/discharge lists ≈690 (+ lines ≈4,720) · booking links 919 · discrepancies ≈35 |
| Agency documents | CRO 919 (+ lines 919, communications 919) · SI 919 (+ cargo, containers and parties ≈5,400) · `nvocc.bl` ≈2,540 (+ lines ≈14,600, amendments ≈20) · `nvocc.manifest` ≈690 (+ lines ≈2,450) |
| Agency release | `nvocc.customs.release` ≈1,580 · `nvocc.delivery.order` ≈1,532 (+ lines ≈3,060, parties ≈4,600, communications ≈1,532) |
| Agency charges | `nvocc.dnd.calculation` ≈3,270 (+ lines ≈6,500) · finance syncs ≈1,300 · disputes 14 · extensions ≈60 · holds ≈30 · reconciliation exceptions ≈6 |
| Agency reporting | period closures 60 (+ checks ≈720) · report submissions ≈57 · `storage.document` ≈3,360 (PODs, gate passes, statements) |
| Engines | `ir.approval.case` ≈6,000 (+ tasks and decisions ≈13,000) · `ir.workflow.instance` ≈25,000 · `ir.workflow.event_log` ≈120,000 |
| **Total** | **≈450,000 business rows**, plus logs. Estimated database size 1.5–3 GB |

---

## 14. Throughput — measured and estimated

**Measured** (read-only, on the existing `fcldemo` tenant, 2026-11-23):

| What | Result |
|---|---|
| In-process boot (stage_chain recipe) | **2.6 s** |
| `env.ref` / small search + read | **2–3 ms** |
| Command cadence during the 19 Nov stage_chain run, from `ir_workflow_event_log` timestamps | 40–60 ms for a submit/approve pair; 80–300 ms for DO, manifest and closure commands; event drain included |
| Data loader | 611 demo records in ≈7 s (≈11 ms per plain create) |

**Commands per unit** (§10, including drains, approvals, list lines and amortised voyages, manifests and closes):

| Unit | Commands each | Units | Commands |
|---|---|---|---|
| Agency import B/L | ≈72 | 1,532 | ≈110,000 |
| Agency export booking | ≈61 | 919 | ≈56,000 |
| AE→US shipment | ≈63 | 632 | ≈40,000 |
| IN→AE forwarding shipment | ≈60 | 258 | ≈15,500 |
| Lost / expired quotes | ≈9 | 300 | ≈2,700 |
| Pipeline · month-ends · daily jobs · Phase 1 masters | | | ≈10,000 |
| **Total** | | | **≈235,000** |

**Runtime estimate:**

- A mean of 0.15–0.25 s per command (writes are heavier than the measured cadence, and tables grow to tens of thousands of rows).
- About 10,000 heavy calls (calculate, readiness, assemble, validate, report generate) at ≈1 s each.

| Phase | Estimate |
|---|---|
| Phase 1 | ≈10 min |
| Phase 2 (India) | **7–11 h** |
| Phase 3 (Dubai) | **2.5–4 h** |
| Phase 4 | ≈15 min |
| **Total** | **≈12–16 h**, single process |

Phase 2's January run is the calibration: the generator re-projects the remaining runtime from real per-command timings before you commit to the full run. Phases 2 and 3 *could* run as two parallel processes, because the orgs are independent and the clock is per-process. That is left out by default because you review the phases one after another.

---

## 15. Generator architecture

### 15.1 Where it lives

```
scripts/showcase/dj_logistics/          # beside scripts/projects_sync.py; not module demo data, not /tmp
    generate.py        # CLI: plan · run · verify · status
    boot.py            # stage_chain's boot(), with the tenant as a parameter + persona envs + strict drain
    clock.py           # time-machine wrapper; refuses to move backwards inside a stream
    plan/              # deterministic planner: parties, users, lanes, timelines, money; selfcheck asserts
    steps/             # masters.py · agency_import.py · agency_export.py · agency_month_end.py · ff_export.py · ff_import.py · jobs.py
    verify.py          # the §17 KPIs, through ORM reads
    .state/<tenant>/   # gitignored: ledger.jsonl · plan.json · run logs · pg_dump snapshots
```

**Boot:**

- It copies `boot()` from `demo-recording/scenes/10-agency-ocean-fcl/scratch/stage_chain.py`: the `ede.cli.main` import order, `boot_runtime_and_environment`, the `PrefixTenantDatabaseIdentityResolver`, the eager Celery executor and the `EventDispatcher` drain.
- It puts the repository's `src` first on `sys.path`, so the active worktree wins (memory: worktree editable-install).
- That boot code would then exist in two places. Lifting it into `src/ede` is a framework change and needs your approval (open decision 9).

### 15.2 Personas

Each user gets an env built as `base.with_principal({user_id, email, tenant_id, identity, active_organization_id}).with_active_organization_id(org)`, exactly like stage_chain's admin env.

- Writes go only through `env.dispatch(Command(name, payload=…, model_key, model_id))`.
- Reads go through `env.models[...]`.
- The persona for every step is fixed in the plan (§10), and approvals are decided by the case's task assignee.
- U4 (system administrator) is the only persona in Phase 1 bootstrap, and nobody uses the seeded `admin@example.com`. Its password is reset from `DJL_DEMO_PASSWORD` in step 1.

### 15.3 The clock

All of the platform's timestamps are taken in Python:

- `generic_repo._now_utc` for `created_at_utc` and `updated_at_utc`;
- about 270 `datetime.now` / `date.today` calls across the domains;
- the workflow formula's `today()`.

None comes from the database server. **`time-machine`** therefore moves the whole process to each step's business time:

- `travel(t0, tick=True)` once, then `move_to(t)` per step;
- it is C-level, so `datetime.now`, `date.today` and `time.time` all follow, while `time.monotonic` does not, so timeouts still work.

This is what makes the date-sensitive guards true *honestly*: quote validity and the accept guard, rate `valid_to`, CRO and DO validity, chronology on movement events and port-call actuals, `effective_date_reached`, `utcnow()` stamps such as `issued_at` and `closed_at`, and numbering years.

- **Fallback:** `freezegun`, which is slower but pure Python.
- **Check:** `max(created_at_utc)` on every table must be ≤ 2026-11-01 (§17).

### 15.4 Plan and seed

- `generate.py plan --seed 20260101` builds the whole plan in memory: parties, users, every shipment with its timeline, every amount, every planned refusal.
- It prints the §8 and §9 tables and writes `plan.json` with its SHA-256.
- The same seed always produces the same plan.
- `run` refuses to resume against a ledger whose plan hash differs, unless `--replan-from YYYY-MM-DD` is given.
- **Plan selfcheck** (asserts; the run will not start if one fails):
  - TEU, USD, box and shipment totals equal §8 and §9;
  - each box has one usage;
  - every guard the plan relies on is true at its step time;
  - no booking stays unconfirmed across a close moment.

### 15.5 Execution

- Each stream is ordered by step time: India for Phase 2, Dubai for Phase 3.
- Each step is one business action. Multi-command steps run inside `env.transaction()`, and events are drained after commit.
- **Planned refusals are asserted.** These are the credit-hold confirms, first SI validations, the September Lantern close and others. If one passes, the run fails.
- **Everything else is fail-fast.** On an unplanned refusal or a dead-lettered event, the run stops and reports the step key, persona, payload and error. `--continue-on-refusal` exists only for exploratory dry runs.

### 15.6 Idempotency, checkpoints and resume

- **No `ir.data.reference` rows.** Orphan cleanup on a later `ede migrate upgrade` *deletes* records whose external ids are no longer declared. Generator-owned xml ids would be data loss waiting to happen.
- **Root records carry deterministic business references** in real fields: the origin booking number in `primary_provider_booking_ref`, enquiry numbers in the lead name, customer PO numbers. A step finds its root by that reference, then checks state before acting, as stage_chain does. A finished step is a no-op.
- **`ledger.jsonl`** records, per step: key, start, end, simulated time, record ids, outcome and duration. It is the resume cursor and the timing source. Deleting it only makes resume slower — it re-verifies by state.
- **Snapshots:** `pg_dump -Fc` at the end of every phase, and optionally at each month end. `pg_restore` rolls a bad month back.
- **Resume:** re-running the same command continues from the cursor and re-verifies the last simulated day.

### 15.7 Guards and safety

- The generator writes only to a tenant named `djlogistics*`.
- It refuses a tenant that holds any `is_demo` reference rows.
- Do not run `ede serve --with-worker` against the tenant while generation runs. The live scheduler would run the same jobs at the real date.

### 15.8 How you run it

```bash
# Phase 0 — once, by you
ede migrate upgrade -t djlogistics --config ede.conf           # no --with-demo
.venv/bin/pip install time-machine                             # dev-only; open decision 1
export DJL_DEMO_PASSWORD='…'                                   # never committed

.venv/bin/python scripts/showcase/dj_logistics/generate.py plan            # prints §8/§9, touches no DB
.venv/bin/python scripts/showcase/dj_logistics/generate.py run --phase 1
.venv/bin/python scripts/showcase/dj_logistics/generate.py verify --phase 1
.venv/bin/python scripts/showcase/dj_logistics/generate.py run --phase 2 --until 2026-01-31   # calibration month
.venv/bin/python scripts/showcase/dj_logistics/generate.py run --phase 2                      # resumes to 31 Oct
.venv/bin/python scripts/showcase/dj_logistics/generate.py status                             # cursor, rate, ETA
```

---

## 16. Phases and review gates

| Phase | Simulated clock | Contents | Review gate (`verify --phase N`) |
|---|---|---|---|
| **1 — Group and masters** | 2025-12-01 → 12-31 | orgs, teams, users, bindings, team roles · ports, facilities, vessels, services, commodities, products, Dec/Jan FX · parties, roles, addresses, contacts · principals → active, entitlements, sequences, numbering rules → active · customs EDI partner · D&D tariffs, free-time terms · January carrier rates · config · **RBAC smoke test:** each persona dispatches each of its command types once, in a transaction that is rolled back. A refusal is fixed by changing a binding (data), never with sudo | §17.1 counts; RBAC smoke report |
| **2 — India agency, Jan → Oct** | 2026-01-01 → 10-31 | §10.1–10.4 and §10.7: imports, exports, voyages, D&D, finance hand-off, month-ends, daily jobs, the October accrual. **Pause after January:** D&D engine vs plan line by line (day-count convention), runtime re-projection, then resume | §17.2: TEU 7,300 by month; agency USD 2.0M ±0.25%; state mix; 53 closures closed |
| **3 — Dubai forwarding, Jan → Oct** | 2026-01-01 → 10-31 (Dubai stream) | §10.5–10.6: leads, quotes, approvals, handovers, bookings, shipments, HBLs, tracking, proformas, PODs, closures, carrier rates | §17.3: TEU 2,700; USD 5,000,000.00 exactly; GM ≈13.2%; quote funnel |
| **4 — Group consolidation** | none (reads only) | group KPIs by lane, entity and month; integrity checks (§17.4); the run report with every workaround applied (D7, D11, D13) and every planned refusal | The final report |

**Why the content moved from the brief:**

- **Charges, hand-offs and month-ends sit in Phases 2 and 3**, at the moment each happened:
  - D5 means a January close needs a January-shaped database;
  - D4 means an exception cannot be raised and closed later;
  - the clock cannot run backwards over records that have already moved on.
- **Running the India and Dubai streams one after the other is safe.** They share only parties, FX and engine tables. The Dubai stream never reads INR rates, and the AED peg is constant.

---

## 17. Verification queries and KPIs

`verify.py` runs these through ORM reads. The SQL equivalents below use column names confirmed against `fcldemo` (`logistics_booking.expected_etd`, `logistics_shipment.etd/eta/principal_id`, `logistics_shipment_profitability.total_revenue_amount`, `nvocc_dnd_calculation.net_amount/status`). Tables and columns marked † are confirmed in the Phase 1 dry run.

### 17.1 After Phase 1

```sql
SELECT o.code, count(u.*) FROM res_user u JOIN res_organization o ON o.record_uuid = u.default_organization_id
 GROUP BY 1;                                                         -- DJL-US 8 · DJL-IN 24 · DJL-AE 18 (+ seeded admin)
SELECT principal_code, status FROM nvocc_principal ORDER BY 1;       -- 6 rows, all 'active'
SELECT count(*) FROM nvocc_numbering_rule WHERE status = 'active';   -- 21
SELECT count(*) FROM pricing_rate WHERE status = 'approved';         -- 6 D&D tariffs + January carrier rates
```

Also checked:

- every user has a primary team and at least one binding;
- every India user's entitlements match §4.2;
- the 11 ports exist;
- FX rows exist in both directions for December and January.

### 17.2 After Phase 2 (India)

```sql
-- TEU handled: imports by discharge, exports by load (one trip per box, D6)
SELECT mt.code, date_trunc('month', ev.event_at) AS month,
       sum(CASE WHEN et.code LIKE '2%' THEN 1 ELSE 2 END) AS teu, count(*) AS boxes
  FROM logistics_equipment_movement_event ev
  JOIN logistics_movement_event_type_master mt ON mt.record_uuid = ev.event_type_id AND mt.code IN ('DISCHARGED','LOADED')
  JOIN logistics_equipment eq ON eq.record_uuid = ev.equipment_id
  JOIN logistics_equipment_type_master et ON et.record_uuid = eq.equipment_type_id
  JOIN res_organization o ON o.record_uuid = ev.organization_id AND o.code = 'DJL-IN'
 WHERE ev.event_at >= '2026-01-01' AND ev.event_at < '2026-11-01'
 GROUP BY 1, 2 ORDER BY 1, 2;                                        -- must equal §8 row by row; Σ 4,900 + 2,400

-- Agency booked revenue (USD): (a) shipment billings + (b) D&D. Anchor: imports by ETA/ATA, exports by ETD/ATD
SELECT sum(p.total_revenue_amount) FROM logistics_shipment_profitability p
  JOIN logistics_shipment s ON s.record_uuid = p.shipment_id
  JOIN logistics_booking b ON b.record_uuid = s.source_booking_id
  JOIN logistics_shipment_type_master t ON t.record_uuid = b.shipment_type_id          -- †
  JOIN res_organization o ON o.record_uuid = s.organization_id AND o.code = 'DJL-IN'
 WHERE (CASE WHEN t.code = 'IMPORT' THEN s.eta ELSE s.etd END) >= '2026-01-01'
   AND (CASE WHEN t.code = 'IMPORT' THEN s.eta ELSE s.etd END) <  '2026-11-01'
   AND s.status <> 'cancelled';
SELECT status, count(*), sum(net_amount) FROM nvocc_dnd_calculation
 WHERE status NOT IN ('draft','superseded','cancelled') GROUP BY 1;  -- (a)+(b) = 2,000,000 ± 5,000

SELECT principal_id, period_code, status FROM nvocc_reporting_period_closure ORDER BY 2, 1;  -- 53 closed, 7 open
```

Also checked:

- state mix by `status` for `nvocc_bl`, `nvocc_delivery_order`, `nvocc_customs_release`, `nvocc_container_release_order`, `nvocc_manifest`, `nvocc_port_call`, `nvocc_dnd_calculation` and the finance syncs, compared with §11;
- no INR profitability currency line with rate 1 (catches the silent 1:1 fallback in `resolve_rate`);
- per-principal TEU and revenue compared with §5.

### 17.3 After Phase 3 (Dubai)

```sql
SELECT sum(p.total_revenue_amount), sum(p.gross_profit_amount) / sum(p.total_revenue_amount) AS gm
  FROM logistics_shipment_profitability p
  JOIN logistics_shipment s ON s.record_uuid = p.shipment_id
  JOIN logistics_booking b ON b.record_uuid = s.source_booking_id
  JOIN logistics_shipment_type_master t ON t.record_uuid = b.shipment_type_id          -- †
  JOIN res_organization o ON o.record_uuid = s.organization_id AND o.code = 'DJL-AE'
 WHERE (CASE WHEN t.code = 'IMPORT' THEN s.eta ELSE s.etd END) >= '2026-01-01'
   AND (CASE WHEN t.code = 'IMPORT' THEN s.eta ELSE s.etd END) <  '2026-11-01'
   AND s.status <> 'cancelled';                                      -- 5,000,000.00 exactly; GM ≈ 13.2%
SELECT status, count(*) FROM crm_quote GROUP BY 1;                   -- funnel per §11
```

Also checked:

- **Cross-check:** Σ `logistics_shipment_charge.sell_amount` equals the profitability revenue, with each charge counted once.
- **TEU:** Dubai TEU from booking equipment allocations† → usage → equipment type gives 2,700.
- **PODs:** every `closed` shipment has a POD.
- **Approvals:** every approval decided by the task assignee — no administrator decisions.

### 17.4 Phase 4 — group and integrity

- **Group totals:** TEU 10,000 · USD 7,000,000 (5.0M exact + 2.0M ±0.25%) · by lane, entity and month, matching §8 and §9.
- **No real-date leakage:** `max(created_at_utc)` ≤ 2026-11-01 on every business table.
- **Who did the work:** nothing created by the seeded admin; nothing created by U4 after Phase 1.
- **No impossible states:**
  - no `completed` DO without a released B/L and returned empties;
  - no `estimated` D&D dated before 2026-10-05;
  - no unplanned `guard_failed` rows in `ir_workflow_event_log`.
- **Workarounds:** every DO stage move matches a logged workaround (D7).

---

## 18. Risks

| # | Risk | Mitigation |
|---|---|---|
| R1 | **Paths never driven live.** These are: the import B/L issue → IGM → DO chain (Enh 01 states it was not driven live), the real forwarding handover commands (scene 04 bypassed them), feasibility linking, `create_lines_from_rate`, quote revision, POD capture with a storage document, shipment close (BR-SH-21), and `nvocc.discrepancy.resolve` against stale-compute behaviour | Phase 1's RBAC smoke and Phase 2/3's first simulated week act as the dry run. A path that fails is reported to you with root cause and options, never patched in `src/` without approval |
| R2 | **RBAC gaps** — a shipped role lacks a permission a persona needs | Phase 1 smoke test. Bindings are fixed as data. A command no shipped role can authorise is reported, because real users would hit it too |
| R3 | **Volume slows the run.** Closure checks, the reconciliation sweep (`search([])` over all finance syncs) and readiness scan whole tables; per-command cost grows | January calibration, then re-projection. The ledger gives per-command timings; snapshots allow a pause |
| R4 | **`time-machine` and Python 3.14** compatibility is unverified | Check it at install; fall back to `freezegun` |
| R5 | No quote-revision command is confirmed | Without one, the 10% of quotes with a version 2 are dropped; nothing else changes |
| R6 | **The snapshot ages when served.** At the real date (late November), aging, SLA and validity figures compute against the real today. A worker would run cron jobs in real time — expiring pipeline CROs, re-estimating D&D, escalating October approvals | Serve without `--with-worker` (open decision 2). The README for the tenant says so |
| R7 | **Generator-created masters keep plain codes** (THCO, OFR, …). A future production seed with the same natural key would be adopted and overwritten by the loader | Acceptable for a showcase tenant; noted |
| R8 | **Box owner codes or invented names may coincide with real registrations** | Q-series owner codes; IMO numbers fail their checksum; every contact detail is synthetic |
| R9 | **D&D share of agency billings is high** (37%). That is defensible for Indian imports with CFS dwell, but a reader may question it | It is a tunable assumption (§9.2). Lower the incidence and the calibration raises fees instead |

---

## 19. Open decisions (each has a default; nothing waits on them)

| # | Decision | Default |
|---|---|---|
| 1 | Add `time-machine` as a dev-only package for the generator (not a runtime dependency of EDE) | **Yes.** Without a simulated clock, quote acceptance, validity, chronology and every `created_at` would sit at the real date and the data would not be honest |
| 2 | How the tenant is served after generation | `ede serve` **without** `--with-worker`. The snapshot stays as at 31 Oct; age-based figures read against the real today |
| 3 | Restructure Phase 4 (charges, finance and month-ends inline in Phases 2–3) | **Yes** (§16) — forced by D4, D5 and chronology |
| 4 | Ports created through `ede.create`, given no UN/LOCODE sync exists | **Yes** — 11 rows keyed by users, not a seed file |
| 5 | Agency headline: gross billings (USD 2.0M including D&D and THC) or agency-retained revenue (≈USD 0.61M on this plan) | **Gross billings**, with retained revenue reported beside it |
| 6 | Keep `nvocc.reporting.closure_blocks_on_exceptions` on | **On.** The Lantern September close is held. Turning it off would let disputes appear across the whole year, at the cost of switching off the control the showcase demonstrates |
| 7 | Fix D3, D4, D5 and D6 before generation, so the data can show resolved disputes, waivers, box reuse and earlier exceptions | **Generate first, fix later.** After the fixes, the dispute and exception slice can be regenerated from the last September snapshot |
| 8 | Agency revenue exact, or within tolerance | **±0.25%.** Exact is possible by re-solving October's handling fee after September's actuals — say if you want it |
| 9 | Copy stage_chain's boot into the generator, or lift it into `src/ede` as a shared in-process boot | **Copy** (scripts only, no framework change). Lifting it is a framework change that needs your approval |
| 10 | Tax (GST, VAT) on proformas | **Not applied** (`apply_taxes` is an explicit command and the tax rates are not configured) |
| 11 | Generated PDFs for PODs, gate passes and statements (about 3,400 one-page files in storage) | **Yes.** `capture_pod` requires a document, and statements need real bytes for their hash |
| 12 | Dubai as destination for India's *agency* exports (same boxes) | **No.** It would double-allocate the same containers. Dubai's India imports are separate nominated cargo |

---

## 20. Out of scope

- Transshipment routings, LCL and consolidation, air, and US-side door delivery.
- A next-sailing roll-over beyond the 1% short-load case.
- Export-side D&D.
- Holiday calendars: no holiday master exists, so calendar days are counted.
- Accounting postings beyond the product's hand-off records (no finance domain).
- Fixing any product defect (§12). Each finding is reported for a `[FIX]` item.
- Localized names and multi-language content.

*This spec is hand-authored. The numbers in §8, §9 and §11 come from the planner prototype at the seed above; the generator re-derives and prints them on every `plan` run.*

---

## 21. Generation log (2026-11-24)

**Where it stands:**
- **Phase 1 (group and masters):** done.
- **Phase 3 (Dubai forwarding, Jan–Jun):** done; every month verified.
- **Phase 2 (India agency):** January is verified exactly. February–June is paused at 4 Feb, the first month-end close, waiting for the run to be allowed to continue in the build session.
- **Phase 4 (group checks):** runs after Phase 2 (`phase4.py`).

### 21.1 Results so far

| Stream | Measure | Plan | Actual |
|---|---|---|---|
| Dubai, Jan–Jun | Shipments | 489 (348 AE→US + 141 IN→AE) | 489 |
| Dubai, Jan–Jun | TEU | 1,486 | 1,486 |
| Dubai, Jan–Jun | Revenue (USD) | 2,701,393.00 | 2,701,393.00, exact for every lane-month |
| Dubai, Jan–Jun | Gross margin | — | 13.34% (13.14% AE→US, 14.95% IN→AE) |
| Dubai, Jan–Jun | Quotes | 726 | 528 accepted · 146 lost · 31 expired · 11 sent · 10 approved |
| India, Jan | Import TEU / boxes / B/Ls | 421 / 263 / 131 | 421 / 263 / 131 |
| India, Jan | Export TEU / boxes / bookings | 212 / 146 / 81 | 212 / 146 / 81 |
| India, Jan | Revenue to date (USD) | 125,269.98 | 125,269.95 |
| India, Jan | D&D lines, engine against plan | 186 | 186 identical (calendar days, start day excluded) |

Planned India revenue for Jan–Jun is USD 1,216,574.79: billings 691,449.79 plus D&D 525,125.00.

### 21.2 Workarounds that live in this tenant only

- **Generator event drain:** the generator drains events with a positive timeout and runs handlers as system.
- **Tenant-only roles:** roles named `djl_sc_*` (`SHOWCASE_GRANTS` in `plan/group.py`) grant the domain commands that no shipped role can run yet (security Enhancement 05), plus a few create and update grants on child models.
- **Booking settings:** agency bookings use three tenant settings so a booking needs no handover or quote.

### 21.3 Product defects found by the run

**Fixed in `src` on 2026-11-24.** Each fix has a regression test.

| # | Defect | Where |
|---|---|---|
| 1 | Consolidation eligibility read display values, so every shipment event failed | `consolidation/models/eligibility.py` |
| 2 | Worker event handlers ran with no principal, so every permission-checked handler write dead-lettered | `cli/commands/worker.py`, `server.py` |
| 3 | Record-rule reads could not see rows created in the same transaction (blocked AR proformas) | `authorization_service._fetch_record_dict` |
| 4 | Permission domains were judged against an empty row, so "own" grants always denied and `!=` failed open | `authorization_service.require` |
| 5 | The RBAC hook ignored a target id carried in the payload | `permission_registry._make_hook` |
| 6 | Any internal user could update any `res.user` (password hash, organisations, bindings) or create users | `res.user` guards in `user.py` |
| 7 | The user-management console was open to any user who could read users | `user_management_routes.py` |
| 8 | Any user could create an out-of-office window for someone else and reroute their approvals | `base` permission CSV |
| 9 | A list-form `values` skipped the organisation guards and RBAC | `command_bus.dispatch` |
| 10 | Deleting a storage document removed files before the permission was decided | `storage/models/document.py` |
| 11 | Four agency sweep jobs rejected the runner's `payload` argument, so they never ran | agency `services/*` |
| 12 | `logistics.profitability.recompute` always raised | `shipment_profitability.py` |
| 13 | Non-admins got uuid fallback document numbers (9 numbering helpers) | `foundation/base/services/sequence_counter.py` |
| 14 | Quote approval rules were numbered against the engine's order, so low-margin quotes never reached pricing | `approval_rules_crm_quote.xml` |

**Logged, not fixed:**
- Security [Enhancement 05](../../../../roadmap/foundation/security/enhancements/05-per-command-permissions.md): per-command permissions.
- Security [Enhancement 06](../../../../roadmap/foundation/security/enhancements/06-writes-that-skip-the-command-pipeline.md): writes that skip the pipeline, with the review follow-ups.
- §12 D1–D6, still as described there.
- Also found by the run:
  - The billing gate goes stale after an approval (D4 again).
  - Charges added after "ready" never reach profitability.
  - Manifest assembly does not filter by principal.
  - Port-call numbers must be unique across the whole tenant.
  - No shipped role may create `communication.message`, so field tracking fails for non-admins.
  - There is no `dnd_charge` numbering rule.
  - `pricing.rate` record rules hide tariffs from the D&D roles.
  - Preparing a house bill from a direct shipment raises.
  - A sell price copied from a buy-rate snapshot counts as a manual override.
  - The handover checklist keys `ocean` while the mode code is `sea`.
  - Booking-from-handover copies cargo using the wrong field names.
  - The provider-request role match reads the wrong field.
  - The quote cost treats quantity 0 as 1.
  - An invoiced charge blocks vendor-bill selection.
  - `notify` and `notify_party` role codes disagree.
  - Issuing an HBL number does not set the bill to issued.
  - Auto-approved cases record the submitter as the decider.

**Deviation caused by defect 14:** all 726 Dubai quotes were decided by A2. The corrected rule order applies to quotes created after the tenant is next migrated.
