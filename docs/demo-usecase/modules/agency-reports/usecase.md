# `logistics_nvocc_agency.agency_reports` — Demo Use-Case

**Module:** NVOCC / Agency — Registers, KPIs & Principal Reporting (`src/domains/logistics_nvocc_agency/agency_reports/`)
**Manifest key:** `demo: ["demo/demo_agency_reports_n9.xml"]`
**Load:** `ede migrate upgrade -t <tenant> --with-demo=all` (or `--with-demo=logistics_nvocc_agency.agency_masters,logistics_nvocc_agency.agency_reports`)
**Prerequisite demos:** `agency_masters` — it ships the Maersk principal (`agency_masters.demo_principal_maersk`) that the BRD N9 records hang off. **`--with-demo` takes app keys literally and does not expand along `depends`**: a declared dependency loads a module's code and its `data`, never its `demo`. So `agency_masters` has to be in the selection too, or the principal `ref=` cannot resolve. What the dependency *does* fix is the order the selected apps load in, so the principal exists before this file reads it.

> **Corrected 2026-11-05.** This module previously declared three demo files, two of
> which created records it does not own: BRD N1's principals / agents / numbering
> rules / EDI partners, and BRD N2's quote chain. Each now ships from the module
> that owns the models — `agency_masters`, `agency_network`, `agency_numbering`,
> `agency_integration`, `agency_commercial` — so `--with-demo=<that app>` produces
> its own records instead of nothing. This module keeps only the BRD N9 file.
**BRD:** N1 (§18 registers, §19 KPIs) · **N9 — Principal Reporting & Agency Finance Integration**
**Roadmap:** [roadmap/logistics/nvocc-agency/phase-9/README.md](../../../../roadmap/logistics/nvocc-agency/phase-9/README.md)

> ⚠ **The BRD N9 half of this demo cannot be loaded today.** Its five tables have no
> migration — `ede migrate generate` is blocked by a pre-existing platform defect, recorded
> in full under [Known blocker](../../../../roadmap/logistics/nvocc-agency/phase-9/README.md#known-blocker--the-alembic-migration-is-not-generated).
> The file is authored, declared and reviewed; the `--with-demo` smoke test is the one step
> that has not run. This page describes what it will show, and says plainly that it has not
> yet been seen.

---

## The story this data tells

Acme Forwarding Ltd. acts for **Maersk Demo Line** out of Mumbai. August 2026 is over, the
operational work is done, and somebody has to tell the principal what happened and then
declare the month final.

They cannot. Two things stand in the way, and the point of the demo is that the system
**names both of them** rather than saying "cannot close":

* a charge worth **USD 1,840** was marked billing-ready on the 14th and **never reached
  accounting**. Nobody would have noticed until August closed USD 1,840 short. It has **no
  owner** — which is the state that turns an exception into a log line, and the reason this
  module exists;
* accounting **rejected** a second hand-off with a real reason ("Tax code not mapped for
  charge DEM at branch INMUM"). Somebody is on it, but it is not fixed.

Neither is a defect in the fixture. They are the two failure modes BRD N9's reconciliation
layer is built to surface, and the demo puts one of each on the screen.

## Records shipped

| Model | Records | Illustrates |
|---|---|---|
| `ir.rbac.role.binding` | 4 — demo only | The N9 verbs are gated on **action** resources, not model CRUD. Without these every button in the walkthrough refuses — correctly, and unhelpfully for a demo |
| `nvocc.reconciliation.exception` | 2 — one unowned `missing_revenue`, one owned `finance_rejection` | N9-15, and the two failure modes: revenue nobody noticed, and a rejection somebody has to act on |
| `nvocc.reporting.period.closure` | 1 — August 2026, `open`, no checklist | N9-21. The gate, before it has run |

**Nothing else is seeded, and every omission is deliberate.**

* **No closure checklist rows.** They are the gate's *output*. Seeding eleven rows would
  teach the reviewer that the checklist is a fixture rather than a control. Press *Run
  Closure Checks* and watch them appear — three of them `not_applicable` with their carried
  dependency named.
* **No report submission.** A submission stamps `content_hash` from a real
  `storage.document.version`, and a version is real bytes on a real storage backend. An XML
  fixture cannot produce bytes, and an invented hash would break the one property the model
  exists for — that the hash proves what was sent.
* **No resolved exception.** The interesting state is `open`. A resolved one demonstrates
  nothing.
* **No finance hand-off row.** `nvocc.charge.finance.sync` belongs to BRD N7, whose demo
  seeds no calculation for the same reason as the first bullet. So the `finance_rejection`
  exception names its subject **by reference** rather than by foreign key — which is a real
  state, not a shortcut: an exception raised by hand exists before anybody links the attempt.

## Walkthrough

1. **Agency → Reports → Open Reconciliation Exceptions.** Two rows. `MSK-CHG-0007` has no
   owner. Assign it — the workflow guard refuses the transition without one.

2. **Agency → Reports → Period Closure → 2026-08 → Run Closure Checks.** Eleven rows plus
   the exception summary. Read three of them:
   * `bl_manifest_reviewed` → `not_applicable`, *"BRD N5 is not delivered for this domain"*;
   * `report_generated` → **fail**, *"No report has been generated and submitted"*;
   * `open_exceptions_resolved` → **fail**, naming *both* exceptions.

3. **Close Period.** Refused, with every blocker listed individually. **That refusal is the
   deliverable.** "Cannot close" produces a support ticket; this produces a fix.

4. **Generate a report** (`nvocc.principal.report.generate`, `monthly_statement`, `2026-08`).
   A `storage.document` appears at version 1. Run it again: version 2, and version 1 stays
   byte-retrievable. That is BR-N9-07, delivered by the document engine — this module builds
   no versioning of its own.

5. **Submit it** (`nvocc.principal.report.submit`). `version_number` and `content_hash` are
   stamped as snapshots. Now try to edit the version number on the form: refused (BR-N9-08).

6. **Resolve one exception** with a note; **waive** the other on the checklist with a reason
   (BR-N9-12 permits carrying forward — with a name attached). Close again: it succeeds.

7. **Reopen.** It demands a written reason and, by default, Management approval — because by
   now the principal holds the report.

8. **Agency → Dashboards → Reconciliation & Finance Integration.** Read the description
   first: it states that **3 of 4** reconciliation directions are covered and names the one
   that is not. A board that showed three and implied four would read green while revenue
   went missing.

## What a reviewer should NOT expect to see

Stated here so the gaps read as decisions rather than as bugs:

* **No BL, manifest, delivery-order or port-call figures anywhere.** BRDs N5, N6 and N8 are
  not delivered in this domain, and `agency_voyage` is not in `ACTIVE_MODULES`. Columns of
  zeroes would read as "nothing happened" rather than "not measured".
* **Reconciliation covers D&D charges only.** `nvocc.charge.finance.sync.calculation_id` is
  required, so charges from other sources have no hand-off ledger yet. Every register and
  metric says so in its own description.
* **No invoice, payment, journal, GL or receivable anywhere.** EDE owns the charge data, not
  the general ledger, and a registry test asserts it.

---

*Part of the unifying demo scenario — see [docs/demo-usecase/README.md](../../README.md).*
