<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Document Engine (Intellex DML) — Implementation Docs

**Module:** `foundation.document` (`src/ede/core/engines/document/` + `src/ede/foundation/document/`)
**Roadmap:** [roadmap/foundation/document/](../roadmap/foundation/document/README.md)
**Status:** 🔴 Not Started (roadmap drafted 2026-05-12)
**Layer:** Foundation engine — consumes `foundation.dataset` substrate

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **template engine** that consumes **Intellex DML v1.1** XML, binds it to data (datasets / metrics from `foundation.dataset` + caller-supplied params), and renders PDF / HTML / print output. Templates are first-class records (`ir.document.template`) resolved per `(doc_type, company_id, country_id, priority, language)` — so jurisdictional + localized variants compose via inheritance without forking.

Intellex DML is the declarative, XML-based document language (16 chapters: structure, body tags, tables, styles, dynamic data binding, page visibility, inheritance, charts, variables). The module implements the EDE conforming engine for that language.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every domain that ships paper or PDFs to customers needs the same primitive: a declarative template language + data binding + style cascade + page layout + PDF/HTML output. Without a shared engine, each domain reinvents — typically badly. Quote PDFs, shipping documents, invoices, certificates, contracts — all consume the same engine.

Three load-bearing design choices:
1. **Same substrate, different renderer** — `<rows datasource="X">` resolves through the same `foundation.dataset` registry that powers reports and dashboards. A "Quote PDF" and a "Quote Dashboard" share the `quote.lines` dataset.
2. **Shared safe AST evaluator** — DML `<var formula="$subtotal * $taxRate"/>` reuses the substrate's safe AST evaluator (Phase 2 of dataset). One implementation, two consumers, one security model to harden.
3. **`(doc_type, company, country, priority, language)` resolution cascade** — purpose-built for legal documents with jurisdictional variants. Inheritance via `extends` + `<override>` keeps variants surgical.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX** — Settings → Technical → Document Templates: form with notebook tabs (General / DML Body / Inheritance / Resolution / Audit). `[Render Preview]` button generates an inline PDF in <2 seconds for demo templates. Phase 3 adds an in-browser DML editor with live preview.
- **Authoring** — Authors write `.idml` files (Intellex DML XML), upload via the admin form or ship under `<app>/demo/templates/`.
- **Programmatic entry points:**
  - `env.dispatch(Command("ede.document.render", payload={"key": ..., "params": {...}, "format": "pdf"}))` — render a template by key.
  - `@api.on_event("ede.document.rendered")` — react to render completion (e.g. delivery dispatcher, audit logger).
  - HTTP: `POST /api/document/<key>/render` (Phase 1 PDF; Phase 2 + HTML / EML; Phase 3 + print + batch).
- **Integration boundary** — PRODUCES rendered bytes + `ir.document.render.audit` rows + `ede.document.rendered` events. CONSUMES `foundation.dataset` (datasource binding), `foundation.dataset` Phase 2 (safe AST eval for formulas), `foundation.storage` (template body files + rendered output persistence), `foundation.base` (res.organization / res.country / res.user.language_id for resolution + variable scope).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Template author]                        [foundation.document]                       [Consumer]
─────────────────                        ─────────────────────                       ──────────
quote_default.idml      ──►   ir.document.template (doc_type, jurisdiction)   ──►   env.dispatch(
quote_india.idml                    │                                                 ede.document.render(...))
   extends quote_default            │
   <override target="terms_block">  │
     ...GST clause...               ▼
   </override>             DocumentEngine pipeline (Phase 1):
                              1. DmlParser → DocumentIR
                              2. RelaxNG validator
                              3. InheritanceResolver (extends + <override>)
                              4. VariableEvaluator (value form; Phase 2: formula via shared AST eval)
                              5. StyleCascadeResolver
                              6. DataSourceBinder (resolves <rows datasource="X">)
                              7. PlaceholderBinder ({field} + <field format="...">)
                              8. RenderPlanBuilder (layout + visibility rules)
                              9. PdfRenderer (WeasyPrint)
                                          │
                                          ▼
                                     bytes returned to consumer
                                          │
Phase 2 adds:                             ▼
  HtmlRenderer (standalone + email)   ir.document.render.audit (append-only)
  ChartRenderer (SVG embedded)
  PostProcessor (sign / watermark / encrypt / compress)
Phase 3 adds:
  PrintRenderer (POSIX lp / Win32)
  BatchRenderer (pdf+html+eml in one call)
  Locale-aware resolution cascade
  Marketplace template seed

Core engine:        src/ede/core/engines/document/
  dml/parser.py        — XML → DocumentIR
  dml/schema/          — RelaxNG schemas for chapters 1-7 (Phase 1) + 10 (Phase 2) + 11 (Phase 2)
  dml/inheritance.py   — extends + <override> resolver
  dml/variables.py     — variable evaluator (formula support via foundation.dataset Phase 2)
  dml/style.py         — named-style cascade resolver
  dml/binding.py       — datasource + placeholder binding
  dml/charts/          — Phase 2: chart parser + IR types
  render/plan.py       — RenderPlan builder
  render/layout.py     — pagination + page_visibility rules
  render/pdf.py        — WeasyPrint
  render/html.py       — Phase 2
  render/print.py      — Phase 3
  render/batch.py      — Phase 3
  post/                — Phase 2: sign / watermark / encrypt / compress

Foundation shell:   src/ede/foundation/document/
  models/              — ir.document.template + ir.document.render.audit
                         + Phase 2: ir.document.post_process
  controllers.py       — HTTP wiring
  views/               — admin form + Monaco editor (Phase 3)
  data/                — RBAC seed + menus
  demo/templates/      — 4 Phase 1 demo .idml files
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none yet — all 🔴 Not Started_ | Phase 1: `ir.document.template` (template body + `(doc_type, company_id, country_id, priority)` resolution + `parent_template_id` inheritance + draft/locked state), `ir.document.render.audit` (append-only render log). Phase 2: `ir.document.post_process` (signing / watermark / encrypt / compress pipeline). Phase 3 adds `language_id` to template for localization. | (planned) `src/ede/foundation/document/models/...` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ | Phase 1: `DmlParser`, `RelaxNgValidator`, `InheritanceResolver`, `VariableEvaluator`, `StyleCascadeResolver`, `DataSourceBinder`, `PlaceholderBinder`, `RenderPlanBuilder`, `LayoutEngine`, `PdfRenderer`. Phase 2: `HtmlRenderer`, `ChartRenderer`, `PostProcessor` (chains `Sign` → `Watermark` → `Encrypt` → `Compress`). Phase 3: `PrintRenderer`, `BatchRenderer`. | (planned) `src/ede/core/engines/document/...` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.document.render` | HTTP `/api/document/<key>/render`, programmatic dispatch | Resolves template → parses → binds → renders → writes audit row → returns bytes |
| `ede.document.preview` (Phase 3) | In-browser editor live preview | Same pipeline but skips audit row + uses draft body |
| `ede.create`/`ede.update`/`ede.delete` on `ir.document.template` | Admin form | CRUD; locked templates block updates via hook |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.document.rendered` | After successful render | Delivery dispatcher (e.g. email attachment); reporting Phase 3 (snapshot capture) |
| `ede.document.render.failed` | After exception during render | Audit log; alerting |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/document/<key>/render` | Render a template by key → returns bytes (format query param: `pdf`/`html`/`eml`) | `src/ede/foundation/document/controllers.py` |
| `POST /api/document/<key>/preview` (Phase 3) | Render a draft body without audit | same |
| `GET /api/document/_list` | List templates visible to caller | same |
| `POST /api/document/<key>/batch_render` (Phase 3) | Render multiple formats in one call | same |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.ir.document.template.update` | Blocks updates when `state="locked"` |
| `pre.ir.document.render.audit.update` | Always raises — append-only |
| `pre.ir.document.render.audit.delete` | Always raises — append-only |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
`ir.document.template.state`:

```
draft  ──(action_lock; template must parse + validate cleanly)──►  locked
locked ──(action_unlock; bumps revision)─────────────────────────►  draft
```

Locked templates still render — locking only blocks edits.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES`: `"document"` (added in Phase 1)
- Manifest `depends`: `["base", "presentation", "dataset"]` — `storage` added in Phase 2
<!-- /SYNC-BLOCK -->

### Foundation-level settings
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `DOCUMENT_DEFAULT_PAPER_SIZE` | str | `"A4"` | `DOCUMENT_DEFAULT_PAPER_SIZE` | Fallback when template omits `<paperformat>`. |
| `DOCUMENT_DEFAULT_LOCALE` | str | `"en"` | `DOCUMENT_DEFAULT_LOCALE` | Locale fallback for number/date formats. |
| `DOCUMENT_RENDER_TIMEOUT_SECONDS` | int | `60` | `DOCUMENT_RENDER_TIMEOUT_SECONDS` | Hard timeout per render. |
| `DOCUMENT_STRICT_BINDING_DEFAULT` | bool | `False` | `DOCUMENT_STRICT_BINDING_DEFAULT` | When True, missing placeholders raise instead of rendering empty. |
| `DOCUMENT_PDF_SIGNING_KEY_PATH` (Phase 2) | str | `""` | `DOCUMENT_PDF_SIGNING_KEY_PATH` | Default signing key path. |
| `DOCUMENT_CHART_THEME_DEFAULT` (Phase 2) | str | `"default"` | `DOCUMENT_CHART_THEME_DEFAULT` | Chart theme fallback. |
| `DOCUMENT_DEFAULT_PRINTER` (Phase 3) | str | `""` | `DOCUMENT_DEFAULT_PRINTER` | Default printer name. |
| `DOCUMENT_EDITOR_DEBOUNCE_MS` (Phase 3) | int | `500` | `DOCUMENT_EDITOR_DEBOUNCE_MS` | In-browser editor preview debounce. |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `data/document_rbac.csv` | 3 RBAC roles — `ir.document.viewer`, `ir.document.author`, `ir.document.admin` |
| `data/document_menus.xml` | `Settings → Technical → Document Templates` |
| `demo/demo_document_templates.xml` | 4 Phase 1 demo templates (default Quote / India Quote variant / Rate Sheet / Sales Acknowledgement) |
| `demo/templates/*.idml` | 4 Phase 1 DML template body files |
| `demo/templates/bill_of_lading.idml` (Phase 2) | Bill of Lading with chart + post-processing demo |
| `demo/demo_document_post_process.xml` (Phase 2) | Watermark + encrypt chain for B/L |
| `demo/templates/marketplace/*.idml` (Phase 3) | 8 commercial templates (commercial invoice / packing list / cert of origin / 2 B/L variants / purchase order / delivery note / contract) |
| `demo/demo_template_localization.xml` (Phase 3) | French / Spanish / German variants of default quote |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | DML Parser + PDF Renderer | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/document/phase-1-implementation.md) |
| Phase 2 | HTML + Charts + Post-Processing | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/document/phase-2-implementation.md) |
| Phase 3 | Print + Multi-Format + Localization + Editor | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/document/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Entire module not yet built | 🔴 Not Started | [roadmap/foundation/document/](../roadmap/foundation/document/README.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
*Will be populated as the module is built and first consumers integrate.*

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
*Pre-build — no migrations yet.*
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `ir.document.viewer` | Read `ir.document.template`; call `POST /api/document/<key>/render` |
| `ir.document.author` | Viewer + CRUD on draft templates |
| `ir.document.admin` | Author + lock/unlock + manage post-processing pipelines (Phase 2) + manage marketplace seed (Phase 3) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.dataset`](./foundation-dataset.md) — datasource binding through metric/dataset registry; shared safe AST evaluator (Phase 2).
- [`foundation.storage`](./17-storage-module.md) — Phase 2 persists rendered output + template body files.
- [`foundation.reporting`](./foundation-reporting.md) — Phase 3 of reporting consumes this engine for PDF snapshots.
- [`foundation.email`](./foundation-jobs.md) — when shipped, consumes Phase 2 HTML output as email body.
- [`foundation.presentation`](./foundation-presentation.md) — admin form for managing templates.
- [`foundation.base`](./foundation-customization.md) — `res.organization` / `res.country` / `res.user.language_id` for resolution + variable scope.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-12. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
