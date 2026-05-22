<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Document Report Engine (DRE) — Implementation Docs

**Module:** `foundation.document` (`src/ede/core/engines/document/` + `src/ede/foundation/document/`)
**Roadmap:** [roadmap/foundation/document/](../roadmap/foundation/document/README.md)
**Status:** 🔴 Not Started (roadmap drafted 2026-05-12; architecture revised 2026-05-21)
**Layer:** Foundation engine — consumes `foundation.dataset` substrate

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
The **Document Report Engine (DRE)** is a **two-stage engine** glued by a serializable intermediate XML.

- **Stage 1** — takes a `.dml` template + caller params + registered datasets/metrics and produces a **resolved `.xml`** with all inheritance, includes, page expansion (`<page for= as=>`), conditionals (`<page if=>`), variable evaluation, datasource binding, and placeholder substitution complete. **No logic constructs remain.** The resolved XML is a first-class artifact — persisted in `ir.document.resolution`, hashable, replayable, format-independent.
- **Stage 2** — three (Phase 2) and eventually four (Phase 3) **independent output engines** read that same resolved XML and emit format-specific bytes: PDF (Phase 1, pixel-perfect via ReportLab Canvas — *not* HTML-to-PDF), HTML (Phase 2, standalone + email modes), DOCX (Phase 2, OOXML via python-docx), Print (Phase 3, consumes PDF). Engines share zero rendering code — only the resolved-XML schema.

The grammar is **page-centric**: `<pages><page>` are explicit, authored units (with `for=` / `as=` / `if=` / `key=` for loops + conditionals). Pagination — i.e. one logical `<page>` overflowing to multiple physical pages — is computed by each Stage 2 engine independently via a contracted two-pass cycle (Pass 1 measure → Pass 2 emit with `{PageNumber}` / `{TotalPages}` / `{LoopIndex}` substituted).

Templates are first-class records (`ir.document.template`) resolved per `(doc_type, company_id, country_id, language_id, priority)` — so jurisdictional + localized variants compose via inheritance without forking.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every domain that ships paper or PDFs to customers needs the same primitive. Without a shared engine, each domain reinvents — typically badly. Quote PDFs, shipping documents, invoices, certificates, contracts — all consume one engine.

Three load-bearing design choices drive the architecture:

1. **Two-stage architecture with a serializable resolved XML.** One Stage 1 run feeds N Stage 2 engines without recomputing data binding. The resolved XML is the perfect audit artifact (customer disputes an invoice → load the resolved XML and see exactly which data was rendered), the natural cache boundary, and the testing-isolation seam (Stage 1 tests need no PDF library; Stage 2 tests need no database).

2. **Page-centric grammar, not body-flow.** Pages are explicit, authored units — `<page id="cover">`, `<page id="content-page" for="$customers" as="customer">`, `<page id="extras" if="$matched">`. Authors retain layout intent; the renderer handles only the physical-pagination overflow within each logical page.

3. **Native pixel-perfect PDF, not HTML-to-PDF.** PDF engine uses ReportLab Canvas at the coordinate level — glyph at (x, y) at exact size. Regulatory documents (customs declarations, certified invoices, government forms) require this fidelity. No browser engine, no HTML/CSS rasterization, no WeasyPrint dependency.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX** — Settings → Technical → Document Templates: form with notebook tabs (General / DML Body / Inheritance / Resolution / Audit). `[Render Preview]` button generates an inline PDF in <2 seconds. `[Inspect Resolved XML]` shows the Stage 1 output — critical for debugging. Phase 3 ships a **React drag-drop document designer**: pick a business model or dataset, see available fields, drag onto pages.
- **Authoring** — Authors write `.dml` files (DML XML), upload via the admin form, or ship under `<app>/demo/templates/`. Phase 3 designer emits DML.
- **Programmatic entry points:**
  - `env.dispatch(Command("ede.document.render", payload={"key": ..., "params": {...}, "format": "pdf"}))` — render to a single format (Phase 1: PDF; Phase 2: + HTML, DOCX; Phase 3: + print).
  - `env.dispatch(Command("ede.document.resolve", payload={"key": ..., "params": {...}}))` — Stage 1 only; returns resolved XML (useful for debugging, format-agnostic snapshots).
  - `env.dispatch(Command("ede.document.batch_render", payload={"key": ..., "params": {...}, "formats": [...]}))` — Phase 3; one Stage 1 → parallel Stage 2 fanout.
  - `@api.on_event("ede.document.rendered")` — react to render completion (delivery dispatcher, audit logger).
  - HTTP: `POST /api/document/<key>/render` · `POST /api/document/<key>/resolve` · `POST /api/document/<key>/preview` · `POST /api/document/<key>/batch_render` (Phase 3).
- **Integration boundary** — PRODUCES rendered bytes + `ir.document.resolution` (cached resolved XML) + `ir.document.render.audit` rows + `ede.document.rendered` events. CONSUMES `foundation.dataset` (datasource binding + shared safe AST evaluator for every dynamic attribute), `foundation.storage` (template body files + rendered output persistence), `foundation.base` (res.organization / res.country / res.user.language_id for resolution + variable scope), `foundation.l10n` (Phase 3 — Babel/CLDR locale resolution).
<!-- /SYNC-BLOCK -->

### Design choices — load-bearing decisions
<!-- SYNC-BLOCK: design-choices -->
DRE makes a small set of deliberate choices that show up everywhere in the engine. (Full reasoning in [roadmap/foundation/document/README.md → Design Choices](../roadmap/foundation/document/README.md#design-choices--load-bearing-decisions).)

1. **Two-stage architecture with serializable resolved XML.** Stage 1 emits `.xml`; Stage 2 consumes it. Resolved XML is a first-class artifact (persisted, hashable, cacheable, replayable).
2. **Records-as-data inheritance.** Templates are `ir.document.template` records; inheritance via `parent_id` + `inherit_mode` (`extend` / `copy`); applicability via `domain` field (EDE search-domain DSL). XPath merger reused from `foundation.presentation` view inheritance — one implementation, two consumers. Same pattern as `@api.extend_model` for models and `<extend ref=>` for views.
3. **Page-centric grammar (`<pages><page>`).** Pages are explicit, authored units. `<page for= as=>` loops; `<page if=>` conditionals; `<for-each>` wrappers for multi-page-per-iteration.
4. **Pixel-perfect PDF via native writer.** ReportLab Canvas at the coordinate level — no HTML-to-PDF intermediary.
5. **Three (then four) independent output engines.** PDF, HTML, DOCX, Print share no rendering code. Adding EPUB / RTF later means writing a new engine, never touching Stage 1.
6. **One expression language across the engine.** Variable formulas, `<if test=>` predicates, `<page if=>` conditionals, `<cell style_if=>`, chart axis bounds, `<barcode value=>` formula values all route through the same safe AST evaluator (`src/ede/core/engines/formula/safe_eval.py`).
7. **Two-pass pagination per Stage 2 engine.** Each engine implements its own Pass 1 (measure) → Pass 2 (emit with render-time tokens substituted).
8. **Auto-key from dict / index.** `<page for= as=>` and `<for-each items= as=>` auto-derive `key` and `{LoopIndex}`: dict iteration → dict key + 1-based positional index; list iteration → 0-based key + 1-based positional index. Explicit `key="$expr"` overrides.
9. **Explicit cycle detection.** Variable + style inheritance graphs validated via Tarjan's algorithm at Stage 1 parse time.
10. **Datasource binding routes through the dataset/metric registry.** No local lookup table.
11. **Babel/CLDR locale (Phase 3) — not hardcoded tables.**
12. **Plugin architecture for custom DML nodes (Phase 3).** `@api.dml_node` + per-engine renderer hooks. Registry-open, grammar-closed.
13. **Authoring-time `<requires>` validation (Phase 3).** Lock-time fail, not render-time.
14. **React drag-drop designer (Phase 3), not a code editor.** Pick model/dataset, drag fields onto pages, emit DML XML. Non-developer authoring.
15. **`ir.document.resolution` caching keyed by `(template_id, template_version, matched_extensions_hash, params_hash, data_hash)`.** Different applied-extension sets get separate cache rows; cache invalidates naturally when any extension's `write_date` changes. Multi-format batch is N parallel Stage 2 reads against one cached resolution.
16. **`<include>` recursion bounded at depth 5.**
17. **`active` flag on `ir.document.template`.** Standard EDE soft-archive.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
                  ╔════════════════════════════════════════════════════════════╗
                  ║                                                            ║
   .dml template  ║   Stage 1: Resolution                                      ║
       +          ║   ─────────────────────                                    ║
   params (data)  ║   - Inheritance + <override>                               ║
       +          ║   - <include> directives (max depth 5)                     ║
   datasets       ║   - Variable evaluation (cycle detection)                  ║
   (registry)     ║   - Datasource binding (foundation.dataset registry)       ║
       │          ║   - Page expansion: <page for= as=> loops + <page if=>     ║
       ▼          ║   - Within-page <if> / <for-each> preprocessing            ║
   ┌──────────┐   ║   - Placeholder substitution ({field}, <field name=>)      ║
   │ DML      │──►║   - Render-time tokens left UNRESOLVED:                    ║
   │ parser + │   ║       {PageNumber}, {TotalPages},                          ║
   │ resolver │   ║       {LogicalPageId}, {LogicalPageIndex},                 ║
   └──────────┘   ║       {PageInLogical}, {LogicalPageCount},                 ║
                  ║       {LoopIndex}, {LoopCount}, {LoopKey}                  ║
                  ║                            │                               ║
                  ║                            ▼                               ║
                  ║                    resolved .xml                           ║
                  ║   (cached in ir.document.resolution, audited, replayable)  ║
                  ║                            │                               ║
                  ╠════════════════════════════│═══════════════════════════════╣
                  ║   Stage 2: Output engines  │   ── each engine independent  ║
                  ║   ───────────────────────  │                               ║
                  ║                            ▼                               ║
                  ║              ┌─────────────┼─────────────┬───────────────┐ ║
                  ║              ▼             ▼             ▼               ▼ ║
                  ║       ┌──────────┐  ┌──────────┐  ┌──────────┐    ┌──────┐ ║
                  ║       │ PDF      │  │ HTML     │  │ DOCX     │    │Print │ ║
                  ║       │ (Phase 1)│  │ (Phase 2)│  │ (Phase 2)│    │(Ph.3)│ ║
                  ║       │          │  │          │  │          │    │      │ ║
                  ║       │ ReportLab│  │ Semantic │  │ OOXML    │    │ lp / │ ║
                  ║       │ Canvas — │  │ HTML +   │  │ via      │    │ Win32│ ║
                  ║       │ coordinate│  │ inline   │  │ python-  │    │ from │ ║
                  ║       │ -level   │  │ CSS      │  │ docx     │    │ PDF  │ ║
                  ║       │          │  │          │  │          │    │ bytes│ ║
                  ║       │ Two-pass │  │ Two-pass │  │ Two-pass │    │      │ ║
                  ║       │ paginatn │  │ paginatn │  │ paginatn │    │      │ ║
                  ║       │ + token  │  │ + token  │  │ + token  │    │      │ ║
                  ║       │ subst    │  │ subst    │  │ subst    │    │      │ ║
                  ║       └─────┬────┘  └─────┬────┘  └─────┬────┘    └───┬──┘ ║
                  ║             ▼             ▼             ▼             ▼    ║
                  ║          .pdf          .html         .docx         to lp   ║
                  ╚════════════════════════════════════════════════════════════╝

Core engine layout:
src/ede/core/engines/document/
├── dml/                 # Stage 1: DML → resolved XML
├── resolved/            # Resolved-XML schema (Stage 1 ↔ Stage 2 contract)
└── output/              # Stage 2: per-format engines
    ├── primitives/      #   shared measurement + two-pass scaffold
    ├── pdf/             #   ReportLab Canvas (Phase 1)
    ├── html/            #   Semantic HTML (Phase 2)
    ├── docx/            #   python-docx OOXML (Phase 2)
    └── print/           #   POSIX lp + Win32 (Phase 3 — consumes PDF)

Foundation shell:
src/ede/foundation/document/
├── models/              # ir.document.template + ir.document.resolution
│                        # + ir.document.render.audit + ir.document.post_process (Ph.2)
├── controllers.py       # HTTP routes
├── services.py          # Render orchestrator (Stage 1 + Stage 2)
├── views/               # Admin form + React designer (Phase 3)
├── data/                # RBAC + menus
└── demo/                # Demo templates (.dml)
```
<!-- /SYNC-BLOCK -->

### Document IR catalog
<!-- SYNC-BLOCK: ir-catalog -->
DML parses into a strongly-typed `DocumentIR` tree (Pydantic). Every supported tag maps to one IR node. Mirrored from [roadmap/foundation/document/README.md](../roadmap/foundation/document/README.md#stage-1--dml-node-catalog-template-side).

**Phase 1 — DML nodes (template-side):**
- Document structure: `Document`, `Meta`, `PaperFormat`, `Margins`, `StyleDef`, `VarDef`
- Page-centric: `PagesNode`, `PageNode` (with `for=`, `as=`, `if=`, `key=` for loops + conditionals), `HeaderNode`, `FooterNode`, `ForEachNode` (page-level wrapper), `IfNode`
- Content: `TextNode`, `FieldNode`, `SpanNode`, `ParagraphNode`, `HeadingNode`, `ImageNode`, `BlockNode` (+ `LabelNode` + `DetailsNode`), `GroupNode`, `SectionNode`, `ComponentNode` (with `keep_together` + scoped `variables`), `TableNode` family (+ `style_if`), `OverrideNode`, `IncludeNode`, `PageBreakNode`

**Phase 2 — DML language additions:**
- `GridLayoutNode` + `GridCellNode`
- `ChartNode` family (+ `AxisNode` + `DataSeriesNode` + `SlicesNode` + `LegendNode` + `DataLabelsNode` + `ColorsNode` + `GridNode`)
- `BarcodeNode`, `QRCodeNode`

**Resolved XML schema (Stage 1 output):** strictly smaller grammar. `<resolved>` root with `<meta>`, `<styles>` (flattened), `<headers>`, `<pages>` (expanded — no `<for-each>`, no `if=` left), `<footers>`. No `<override>`, no `<include>`, no `<if>`, no `<for-each>`, no `<var>`, no `<rows datasource=>`. Logic gone; data + layout intent remain. Render-time tokens (`{PageNumber}`, `{TotalPages}`, `{LoopIndex}`, etc.) survive as literal strings for Stage 2 to substitute.
<!-- /SYNC-BLOCK -->

### Render-time token vocabulary
<!-- SYNC-BLOCK: render-tokens -->
Tokens NOT substituted at Stage 1 — survive as literal text in resolved XML, filled in by Stage 2 Pass 2.

| Token | Meaning |
|---|---|
| `{PageNumber}` | Current physical page (1-based) |
| `{TotalPages}` | Total physical pages (from Pass 1 measurement) |
| `{LogicalPageId}` | `id=` of the current logical `<page>` |
| `{LogicalPageIndex}` | 1-based index of current logical page in surviving `<pages>` |
| `{PageInLogical}` | Current physical page within the current logical page (1..K when overflowing) |
| `{LogicalPageCount}` | Total surviving logical pages (after `if=` filtering) |
| `{LoopIndex}` | Iteration index (1-based) within the current loop |
| `{LoopCount}` | Total iterations of the current loop |
| `{LoopKey}` | The `key=` value — dict key (dict iteration) or 0-based index (list); explicit `key="$expr"` overrides |

`key` and `{LoopIndex}` auto-resolve: dict iteration → dict key; list iteration → 0-based index. `{LoopIndex}` is always 1-based for printed output.
<!-- /SYNC-BLOCK -->

### Closed vocabularies
<!-- SYNC-BLOCK: closed-vocabularies -->
Every dynamic attribute enforces a closed taxonomy — authoring outside the set is a `DmlValidationError` at parse time. Mirrored from [roadmap/foundation/document/README.md](../roadmap/foundation/document/README.md#closed-vocabularies).

| Vocabulary | Values | Phase |
|---|---|---|
| `Orientation` | `portrait` · `landscape` | 1 |
| `PageType` (header/footer on physical pages) | `all` · `first` · `rest` · `last` · `odd` · `even` · `except-first` · `except-last` · `first-and-last` (9) | 1 |
| `PageVisibility` (within overflowing logical page) | `all` · `first` · `last` · `odd` · `even` (5) | 1 |
| `TextAlign` · `VerticalAlign` · `FontWeight` · `FontStyle` · `TextDecoration` · `TextTransform` · `BorderStyle` · `ImagePosition` | (see roadmap) | 1 |
| `OverrideMode` | `replace` (default) · `append` · `prepend` · `remove` | 1 |
| `VarType` | `string` · `number` · `integer` · `decimal` · `date` · `datetime` · `boolean` · `color` · `currency` · `list` (10) | 1 |
| `VarScope` | `document` (default) · `component` | 1 |
| `OutputFormat` | `pdf` (Phase 1) · `html` · `docx` (Phase 2) · `print` (Phase 3) | 1+ |
| `ChartType` | `bar` · `column` · `line` · `area` · `pie` · `doughnut` · `scatter` · `radar` · `stacked-bar` · `stacked-column` (10) | 2 |
| `ChartTheme` | `default` · `minimal` · `corporate` · `print` · `mono` (5) | 2 |
| `AxisType` · `LegendPosition` | (see roadmap) | 2 |
| `BarcodeType` | `code128` · `code39` · `ean13` · `ean8` · `upc-a` · `upc-e` · `itf` · `codabar` · `pdf417` · `datamatrix` (10) | 2 |
| `QRCodeErrorCorrection` | `L` (~7%) · `M` (~15%) · `Q` (~25%) · `H` (~30%) | 2 |
<!-- /SYNC-BLOCK -->

### Inheritance Model — Records-as-Data
<!-- SYNC-BLOCK: inheritance-model -->
Templates inherit via record-relationship — no DML-side `extends=` attribute. The full design lives in [README.md → Inheritance Model](../roadmap/foundation/document/README.md#inheritance-model--records-as-data--xpath); this is the brief mirror.

**Two modes:**

| `inherit_mode` | Semantics | Result template `key` | Original affected? |
|---|---|---|---|
| `extend` | Layer onto parent when `domain` matches | (extensions invisible in print menu — they layer on their parent) | Yes — anyone rendering parent's key sees the layered version when applicable |
| `copy` | Separate template starting from parent's resolved DML | Different from parent | No — original remains intact |

**Domain semantics by mode + view context:**

| Case | Behaviour |
|---|---|
| `domain = NULL` | Global — always applies |
| `extend` + domain set | Applies to parent only when domain matches current target record (or current `res.organization` fallback) |
| `copy` + domain set, **form view** | Copy shown in print menu only when domain matches the form's current record |
| `copy` + domain set, **list / bulk-print** | Domain bypassed — copy always shown if `model_id` + `organization_id` match |

**Two resolver flows:**

1. **Render flow** — `resolve_template_for_render(key, env, target_record)` loads the target by unique key, recursively resolves parent for `copy` mode, discovers `extend`-mode extensions via `parent_id` lookup, filters by domain, applies in priority order via XPath merger.
2. **Print-menu flow** — `list_templates_for_print(model_key, env, target_record, is_multi_record)` returns visible templates: extensions excluded; bases + copies filtered by domain in form view; domain bypassed in list view.

**XPath patch grammar** — reuses `foundation.presentation` view-inheritance machinery. Standard `position=` values: `inside` (default), `after`, `before`, `replace`, `attributes`.

**Lock-time XPath validation** — every `<xpath expr=>` in an extension/copy is verified against the parent's current DML at lock time; parents being re-locked re-validate all active extensions and auto-revert orphans to draft.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none yet — all 🔴 Not Started_ | **Phase 1:** `ir.document.template` — **records-as-data inheritance**: unique `key`, `parent_id` + `inherit_mode` (`extend` / `copy`), `domain` (EDE search-domain DSL for applicability), `priority`, `organization_id` hard scope, `model_id` binding (drives print-menu visibility), `dml` field (loaded via `type="xml"` — full body for base, `<xpath>` patches for extensions/copies), full `(doc_type, company_id, country_id, language_id, priority)` resolution cascade, draft/locked state, active flag. `ir.document.resolution` (cached resolved XML keyed by `(template_id, template_version, matched_extensions_hash, params_hash, data_hash)` — different applied-extension sets get separate cache rows; cache invalidates naturally when any extension's `write_date` changes). `ir.document.render.audit` (append-only — `cache_hit`, `applied_extension_keys`, `target_record_uuid`, output_format, output_size, success/error). **Phase 2:** `ir.document.post_process` (signing/watermark/encrypt/compress chain — PDF-specific). | (planned) `src/ede/foundation/document/models/...` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ | **Stage 1 (Phase 1):** `DmlParser` (XML → Pydantic IR), `RelaxNgValidator`, `TemplateResolver` (records-as-data: discovers applicable extensions via `parent_id` + `inherit_mode` + `domain` matching; layers via XPath merger in priority order), `XPathMerger` (**thin wrapper reusing `foundation.presentation`'s view-inheritance machinery — one impl, two consumers**), `DomainEvaluator` (wraps EDE's standard search-domain evaluator with current-target-record + org-fallback context), `IncludeResolver` (max depth 5), `VariableEvaluator` (value form + scope + `$varName` substitution + Tarjan cycle detection), `StyleCascadeResolver` (inherit + cache), `DataSourceBinder` (routes through `foundation.dataset` registry), `PageExpander` (`<page for= as= key=>` + `<for-each>` wrapper + auto-key), `Preprocessor` (`<if test=>` + content-level `<for-each>` via shared safe AST), `PlaceholderBinder` (non-system tokens), `ResolvedXmlEmitter` (validated against `resolved.rng`), `EnumKernel`. **Stage 2 PDF (Phase 1):** `ResolvedXmlParser`, `UnitResolver`, `PaperSize`, `PdfLayoutEngine` (flow + overflow + headers/footers per `pagetype`), `TwoPassExecutor` (Pass 1 measure → Pass 2 emit with token subst), `PdfEngine` + `PdfWriter` (ReportLab Canvas, coordinate-level). **Foundation shell:** `PrintMenuLister` (form vs list-mode domain semantics → `GET /api/document/_list`). **Phase 2:** `VariableEvaluator` formula-form (substrate AST), `GridIR` parser, `ChartParser` + `ChartSvgEmitter` (10 types · 5 themes), `BarcodeImageEmitter` (10 symbologies + 4 QR EC), `HtmlEngine` (standalone + email), `DocxEngine` (python-docx OOXML), `PostProcessor` chain (`Sign` → `Watermark` → `Encrypt` → `Compress` — PDF-only). **Phase 3:** `PrintEngine` (consumes PDF), `BatchExecutor` (parallel Stage 2 fanout), `DocumentLocaleResolver` (Babel/CLDR), `DmlNodePluginRegistry` (`@api.dml_node`), `RequiresValidator` (lock-time contract), React `DocumentDesigner` (drag-drop authoring → DML XML). | (planned) `src/ede/core/engines/document/...` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.document.render` | HTTP `/api/document/<key>/render`, programmatic dispatch | Resolves template → Stage 1 → Stage 2 (single format) → audit row → emit event → return bytes |
| `ede.document.resolve` | HTTP `/api/document/<key>/resolve`, debugging | Stage 1 only → return resolved XML string |
| `ede.document.preview` | In-browser designer live preview | Same pipeline as `render` but skips audit row + uses draft body |
| `ede.document.batch_render` (Phase 3) | HTTP `/api/document/<key>/batch_render` | One Stage 1 → parallel Stage 2 fanout → return `{format: bytes}` dict |
| `ede.create`/`ede.update`/`ede.delete` on `ir.document.template` | Admin form / React designer | CRUD; locked templates block updates via hook; `<requires>` validation runs at lock-time (Phase 3) |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.document.rendered` | After successful render (Stage 2 complete) | Delivery dispatcher (e.g. email attachment); reporting Phase 3 (snapshot capture) |
| `ede.document.resolved` | After successful Stage 1 (resolution cached) | Audit log; cache-warmth indicators |
| `ede.document.render.failed` | After exception during render | Audit log; alerting |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/document/<key>/render` | Render to single format → bytes (`format` query param: `pdf`/`html`/`docx`/`print`) | `src/ede/foundation/document/controllers.py` |
| `POST /api/document/<key>/resolve` | Stage 1 only → resolved XML string | same |
| `POST /api/document/<key>/preview` | Render a draft body without audit (designer live preview) | same |
| `POST /api/document/<key>/batch_render` (Phase 3) | Multi-format from one Stage 1 → dict of format → bytes | same |
| `GET /api/document/_list?model=<model_key>&record_id=<uuid\|null>&mode=form\|list` | **Print-menu lister** — returns templates available for the model (form mode filters by `domain` against the given record; list mode bypasses `domain`; extensions are always excluded — they layer on their parent) | same |
| `GET /api/document/data_sources` (Phase 3) | List registered business models + metrics + datasets for the React designer | same |
| `GET /api/document/data_sources/<key>` (Phase 3) | Schema for a specific data source (field names + types + display labels) | same |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.ir.document.template.update` | Blocks updates when `state="locked"`; on lock transition, validates every `<xpath expr=>` in the template's `dml` resolves against the parent (for extensions / copies) |
| `post.ir.document.template.update` | When a base template's `dml` changes, re-validate every active extension's xpath expressions against the new parent DML; auto-revert orphaned extensions to `state="draft"` with an audit log entry |
| `pre.ir.document.template.update` (Phase 3) | On `state → "locked"`, runs `RequiresValidator` — rejects if `<requires>` block is incomplete relative to template body |
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

Locked templates still render — locking only blocks edits. Phase 3 adds `<requires>`-based contract validation at the lock transition; a template referencing undeclared fields cannot lock.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES`: `"document"` (added in Phase 1)
- Manifest `depends`: `["base", "presentation", "dataset"]` — `storage` added in Phase 2 (post-processing); `l10n` added in Phase 3 (Babel/CLDR locale resolution)
<!-- /SYNC-BLOCK -->

### Foundation-level settings
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `DOCUMENT_DEFAULT_PAPER_SIZE` | str | `"A4"` | `DOCUMENT_DEFAULT_PAPER_SIZE` | Fallback when template omits `<paperformat>`. |
| `DOCUMENT_DEFAULT_LOCALE` | str | `"en"` | `DOCUMENT_DEFAULT_LOCALE` | Locale fallback for number/date formats. Babel/CLDR integration ships in Phase 3. |
| `DOCUMENT_RENDER_TIMEOUT_SECONDS` | int | `60` | `DOCUMENT_RENDER_TIMEOUT_SECONDS` | Hard timeout per render. |
| `DOCUMENT_STRICT_BINDING_DEFAULT` | bool | `False` | `DOCUMENT_STRICT_BINDING_DEFAULT` | When True, missing placeholders raise instead of rendering empty. |
| `DOCUMENT_RESOLUTION_CACHE_TTL_SECONDS` | int | `3600` | `DOCUMENT_RESOLUTION_CACHE_TTL_SECONDS` | TTL for `ir.document.resolution` rows. |
| `DOCUMENT_OUTPUT_ENGINES_ENABLED` | str | `"pdf"` (P1) · `"pdf,html,docx"` (P2) · `"pdf,html,docx,print"` (P3) | `DOCUMENT_OUTPUT_ENGINES_ENABLED` | Comma-separated enabled engines. |
| `DOCUMENT_PDF_SIGNING_KEY_PATH` (Phase 2) | str | `""` | `DOCUMENT_PDF_SIGNING_KEY_PATH` | Default signing key path for PDF post-processing. |
| `DOCUMENT_CHART_THEME_DEFAULT` (Phase 2) | str | `"default"` | `DOCUMENT_CHART_THEME_DEFAULT` | Chart theme fallback. |
| `DOCUMENT_BARCODE_QUIET_ZONE_MM` (Phase 2) | float | `2.0` | `DOCUMENT_BARCODE_QUIET_ZONE_MM` | Default white margin around barcodes. |
| `DOCUMENT_QR_ERROR_CORRECTION_DEFAULT` (Phase 2) | str | `"M"` | `DOCUMENT_QR_ERROR_CORRECTION_DEFAULT` | QR EC level default. |
| `DOCUMENT_HTML_EMAIL_LITMUS_STRICT` (Phase 2) | bool | `True` | `DOCUMENT_HTML_EMAIL_LITMUS_STRICT` | HTML email mode fails fast on disallowed constructs. |
| `DOCUMENT_DEFAULT_PRINTER` (Phase 3) | str | `""` | `DOCUMENT_DEFAULT_PRINTER` | Default printer name. |
| `DOCUMENT_BABEL_CACHE_SIZE` (Phase 3) | int | `128` | `DOCUMENT_BABEL_CACHE_SIZE` | LRU cache for Babel locale lookups. |
| `DOCUMENT_PLUGIN_PACKS_ENABLED` (Phase 3) | bool | `True` | `DOCUMENT_PLUGIN_PACKS_ENABLED` | Master switch for third-party `@api.dml_node` plugins. |
| `DOCUMENT_DESIGNER_PREVIEW_DEBOUNCE_MS` (Phase 3) | int | `500` | `DOCUMENT_DESIGNER_PREVIEW_DEBOUNCE_MS` | React designer live-preview debounce. |
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
| `demo/demo_document_templates.xml` | 4 Phase 1 demo templates (Default Quote / India Quote variant / Rate Sheet / Sales Acknowledgement) |
| `demo/templates/*.dml` | 4 Phase 1 DML template body files |
| `demo/templates/bill_of_lading.dml` (Phase 2) | Bill of Lading with chart + grid + barcode + QR + post-processing demo |
| `demo/demo_document_post_process.xml` (Phase 2) | Watermark + encrypt chain for B/L |
| `demo/templates/marketplace/*.dml` (Phase 3) | 8 commercial templates |
| `demo/demo_template_localization.xml` (Phase 3) | French / Spanish / German variants of default quote |
| `demo/templates/commercial_invoice.dml` (Phase 3) | reference `<requires>` block example |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Stage 1 Resolution + Stage 2 PDF Engine (20 workstreams — DML parser, RelaxNG, **records-as-data template inheritance** (`parent_id` + `inherit_mode` + `domain` + XPath merger reused from `foundation.presentation`), `<include>`, variables + style cascade + cycle detection, datasource binding, page expansion with `<page for= as= if= key=>` + `<for-each>` wrapper + auto-key, within-page preprocessor, placeholder/`<field>` binding, resolved-XML emitter + schema, **data-loader `type="xml"` + `eval=` attribute support**, closed-vocab enums, resolved-XML parser, measurement primitives, layout engine, two-pass executor, native pixel-perfect PDF via ReportLab Canvas, models incl. `ir.document.resolution` keyed by matched-extensions hash, **print-menu endpoint** (`GET /api/document/_list` with form vs list-mode domain semantics), demo + `l10n_in` HSN extension demo + VIP copy demo + admin form) | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/document/phase-1-implementation.md) |
| Phase 2 | HTML + DOCX engines + Charts + Barcodes + Grid + Post-Processing (9 workstreams — formula-form `<var>`, `<grid columns=>`, 10 chart types × 5 themes (SVG), 10 barcode symbologies + 4 QR EC levels with formula-bindable values, **HTML output engine** (standalone + email), **DOCX output engine** (OOXML via python-docx), PDF chart/barcode/grid integration, post-processing (sign/watermark/encrypt/compress — PDF-only), Bill-of-Lading demo across all 3 formats) | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/document/phase-2-implementation.md) |
| Phase 3 | Print + Multi-Format Batch + Localization + React Designer + Plugins + Validation (9 workstreams — print engine (consumes PDF), multi-format batch (one Stage 1 → parallel Stage 2 fanout), template localization (language_id FK + cascade), **React drag-drop document designer** (visual authoring — pick model/dataset, drop fields onto pages, emits DML), Babel/CLDR locale, `@api.dml_node` plugin architecture with per-engine hooks, `<requires>` authoring-time validation, 8-template marketplace seed) | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/document/phase-3-implementation.md) |
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
| `ir.document.author` | Viewer + CRUD on draft templates + use React designer (Phase 3) |
| `ir.document.admin` | Author + lock/unlock + manage post-processing pipelines (Phase 2) + manage marketplace seed (Phase 3) + register plugin packs (Phase 3) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.dataset`](./foundation-dataset.md) — datasource binding through metric/dataset registry; shared safe AST evaluator (variables + `<if test=>` + `<page if=>` + `style_if` + chart-axis expressions + barcode formula values — **one expression language for the whole engine**).
- [`foundation.storage`](./17-storage-module.md) — Phase 2 persists rendered output + template body files.
- [`foundation.reporting`](./foundation-reporting.md) — Phase 3 of reporting consumes DRE for PDF snapshots.
- [`foundation.email`](./foundation-jobs.md) — Phase 2 HTML engine output consumable as email body.
- [`foundation.presentation`](./foundation-presentation.md) — admin form for managing templates; React designer (Phase 3) is a new view kind that emits DML.
- [`foundation.base`](./foundation-customization.md) — `res.organization` / `res.country` / `res.user.language_id` for resolution + variable scope.
- [`foundation.l10n`](./foundation-l10n.md) — **Phase 3** locale resolution chain (template `<meta language>` → caller params → principal → tenant default → setting); Babel/CLDR formatting.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-21 (**inheritance lock-down** — records-as-data via `ir.document.template` with `parent_id` + `inherit_mode` (`extend` / `copy`) + `domain` field (EDE search-domain DSL) + `model_id` binding; XPath merger reused from `foundation.presentation` view inheritance; print-menu endpoint `GET /api/document/_list` with form-view vs list-mode domain semantics; resolution cache key extended with `matched_extensions_hash` so different applied-extension sets get separate cache rows; data-loader extended to support `type="xml"` + `eval=` field attributes; canonical `l10n_in` HSN-column demo extension shipped. **Prior 2026-05-21 entries:** major architectural restructure — two-stage pipeline, page-centric grammar, native pixel-perfect PDF via ReportLab Canvas, HTML + DOCX as independent Stage 2 engines, React drag-drop designer in Phase 3, 38 workstreams.) To refresh, invoke the `syncing-roadmap-to-docs` skill.*
