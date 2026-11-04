# Demo Use Case — NVOCC Agency Commercial (BRD N2)

> **Module:** `logistics_nvocc_agency.agency_commercial` (+ the base-tier models it
> depends on: `logistics_base.sales_crm`, `logistics_base.commercial_terms`)
> **Demo file:** `logistics_nvocc_agency/agency_reports/demo/demo_agency_commercial.xml`
> **Loads with:** `ede migrate upgrade -t <tenant> --with-demo=all`
> **Scenario:** the one defined in [docs/demo-usecase/README.md](../../README.md)

---

## Why the demo file lives in `agency_reports`

`agency_commercial` ships **no models of its own** — every class in it is an
`@api.extend_model`. Its demo records are quotes, rates and contracts owned by
`logistics_base`, tagged with a principal owned by `agency_masters`, and read
back through registers owned by `agency_reports`. Putting the file in the module
that loads **last** is the only ordering under which every reference resolves,
and it is where the Phase 1 agency demo already lives.

---

## The story

Acme Forwarding's Mumbai desk acts as local shipping agency for **Maersk Demo
Line** (`PRN-MAERSK`, seeded by the Phase 1 demo). A garment exporter,
**Vasudha Garments**, asks for a price on four 40ft high-cubes and one reefer,
Nhava Sheva → Singapore, for its autumn programme.

The desk raises an offer that carries — for the first time in this platform —
**who it is made to, what it will move in, and what free days it gives away.**

| Step | What the reviewer sees |
|---|---|
| Open **Agency → Commercial → Quotes** | One quote, tagged to Maersk Demo Line. The Principal field is read-only. |
| Open the version's **Parties** tab | Six rows: shipper, consignee, notify, billing party, origin agent, destination agent. The customer is the header; the principal is the header. Neither is duplicated as a row. |
| Open the **Equipment** tab | 4 × 45GP promised *available*; 1 × 22RT *subject to confirmation*. There is no reefer checkbox — the reefer is a reefer because its equipment type says so. |
| Open the **Free Time** tab | Detention 7 days (standard). Demurrage 9 days against a standard of 5 — **beyond standard**, and flagged. |
| Look at the version's **Lane** block | POR / POL / POD / final destination and the service type, all read-only. Nobody typed them: they are derived from the single route leg. |
| Press **Submit** | **Refused.** See below — this is the demo's centrepiece. |

---

## The two refusals the demo is built around

### 1. BR-N2-04 — the quote outlives its rate

The quote is valid to **2026-12-31**. The Maersk principal tariff behind it
expires **2026-11-30**. With `nvocc.quote_validity_vs_rate_validity` at its
default `block`, Submit is refused and the message names the offending rate and
its expiry date.

That refusal is the **single genuine engine gap** BRD N2 exposed. Before this
phase, `quote_expiry_date` and `pricing.rate.valid_to` were two unrelated fields
and a quote valid past its own cost was a loss discovered at invoicing.

**To drive the demo through**, do one of:

- shorten the quote's expiry to 2026-11-30 (the correct commercial fix), or
- set `nvocc.quote_validity_vs_rate_validity` to `warn` under
  **Settings → Agency → Agency Commercial**, or
- set it to `allow_with_approval` and let the approval chain decide.

All three are worth doing once — the three modes behave visibly differently.

### 2. N2-25 — a suspended principal cannot hand a quote to booking

Suspend `PRN-MAERSK` (Agency → Setup → Principals → Suspend, reason required),
then try **Accept** on a sent quote. It is refused, and the refusal is written to
`nvocc.principal.usage.refusal` where the Phase 1 KPI counts it.

---

## What the demo deliberately does NOT do

- **Nothing arrives pre-accepted.** `crm.quote.status`, `pricing.rate.status`
  and `pricing.contract.status` are all `workflow=True`; the ORM guard forbids
  seeding them, and a fixture that bypassed the lifecycle would prove none of the
  controls above. Every record lands in its initial stage.
- **No margin engine, approval engine or charge master is demonstrated as new** —
  all three ship already. The demo shows the principal *dimension* on them.
- **The beyond-standard free-time term loads without an approval**, because a
  data load has no acting user to route one to. Create the same term from the UI
  and it is refused (BR-N2-13). That asymmetry is deliberate and is the reason
  the guard carves out `env.uid is None`.

---

## Registers the demo populates

| Register / KPI | What it shows after the demo loads |
|---|---|
| Quote Register | One principal-tagged quote |
| Quote Margin Register | One version with cost, sell, margin, the resolved rate basis and the lane |
| Free Time Giveaway Register | Two terms, one flagged beyond standard |
| KPI — Quote Outcomes by Principal | One row for Maersk: 1 raised, 0 accepted |
| KPI — Margin Health by Principal | Average margin over one version — the version count beside it is why that number should not be trusted yet |
| KPI — Free Time Given by Type | 7 detention days, 9 demurrage days, 1 beyond standard |
| Manual Override Audit | Empty until the reviewer overrides a line price — do it and watch the row appear with the reason |

---

## Cost visibility (N2-26)

The demo binds the admin user to `agency_masters.role_system_admin`, so cost and
margin are visible. To see the matrix work, add
`agency_masters.role_viewer` to **Settings → Sales & CRM → Quotation → Cost
Visibility → Roles Denied Cost & Margin**, log in as a user holding only that
role, and reopen the quote: buy amount, margin and the whole Margin Summary block
are **absent from the payload**, not merely hidden in the browser.

One gap is stated rather than hidden: the quote form's scrolling summary strip
still shows the Cost and Margin KPIs to every role, because the view grammar
allows no gate attribute on `<kpi>`. Closing it needs a change to
`src/ede/core/services/presentation/dsl/schemas/`, which is a platform change.

---

## Prerequisites

Load with `--with-demo=all`. The file references `logistics_base.masters` **demo**
rows (`masters.demo_port_innsa`, `masters.demo_port_sgsin`,
`masters.demo_commodity_fak`) and the Phase 1 agency demo's principals
(`agency_reports.demo_principal_maersk`). Referencing them keeps this on the one
unifying scenario rather than forking a second set of Nhava Shevas.
