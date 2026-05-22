# `foundation.document` — Demo Use-Case

**Module:** `ede.foundation.document`
**App key:** `foundation.document`
**Demo manifest entries** (planned, Phase 1): `demo/demo_document_templates.xml`, `demo/templates/*.dml`
**Status:** 🔴 Not authored (module itself is 🔴 Not Started — see [roadmap/foundation/document/](../../../../roadmap/foundation/document/))

---

## Use-case

The document engine ships **4 demo templates** demonstrating the four load-bearing features:

1. **Datasource binding** — Default Quote PDF sources line items via the registered `crm.quote.lines` dataset; `<rows datasource="crm.quote.lines">` resolves through the `foundation.dataset` registry.
2. **Template inheritance** — India Quote variant `extends="crm.quote.default"`, overrides only `terms_block` with a GST clause. Proves the `extends` + `<override>` semantics on real demo data.
3. **Multi-page table flow** — Rate Sheet renders the `pricing.rate` matrix across multiple pages with header rows repeating per page. Proves the layout engine handles natural pagination.
4. **Minimal template** — Sales Acknowledgement is a one-page template that proves the engine on a trivial case (style cascade + variables + placeholders) without inheritance or datasource binding.

## The unifying scenario fit

- Default Quote PDF renders against `demo_quote_globex_001` (Acme → Globex quote on lane INMUM → SGSIN).
- India Quote variant adds GST terms for India-domiciled customer quotes.
- Rate Sheet renders `pricing.rate` records (already in the Acme demo) grouped by lane.
- Sales Acknowledgement renders a minimal acknowledgement of `demo_inquiry_globex_001`.

## Records produced (planned, Phase 1)

### `demo/demo_document_templates.xml`

| External ID | Model | Key | doc_type | parent_template_id |
|---|---|---|---|---|
| `document.demo_quote_default` | `ir.document.template` | `crm.quote.default` | `Quote` | _none_ |
| `document.demo_quote_india` | `ir.document.template` | `crm.quote.india` | `Quote` | `document.demo_quote_default` |
| `document.demo_rate_sheet` | `ir.document.template` | `pricing.rate_sheet` | `RateSheet` | _none_ |
| `document.demo_sales_ack` | `ir.document.template` | `crm.sales.acknowledgement` | `SalesAcknowledgement` | _none_ |

### `demo/templates/*.dml`

Four DML XML template body files, each authored by hand:

| File | Notes |
|---|---|
| `quote_default.dml` | Full quote with header/footer/parties/line-items table/totals/terms component; 6 named styles inheriting from `base`; system fields `{PageNumber}` + `{TotalPages}`. |
| `quote_india.dml` | Extends `quote_default`; overrides `terms_block` with GST clause + India footer; otherwise inherits everything. |
| `rate_sheet.dml` | Single large table with `<rows datasource="pricing.rates">`; multi-page flow expected on demo data. |
| `sales_acknowledgement.dml` | One page; minimal style cascade + 3 placeholders. |

## Out of scope

- Charts in PDFs — Phase 2 (DML chapter 10).
- HTML output mode — Phase 2.
- Post-processing (signing / watermarking / encryption) — Phase 2.
- Multi-language template variants — Phase 3.
- 8 marketplace templates (Commercial Invoice / B/L variants / etc.) — Phase 3.

## Dependencies

- `foundation.dataset` Phase 1 must ship (datasource binding routes through it).
- `foundation.base` + `crm.quote` (+ lines + version) demo data.
- `logistics.pricing` rate demo data.

## Verification (once Phase 1 lands)

```
ede migrate upgrade -t demo --with-demo=foundation.document
```

Then:
1. Open `Settings → Technical → Document Templates` — 4 templates visible.
2. Click `crm.quote.default` → "Render Preview" returns a 1-page PDF inline.
3. Click `crm.quote.india` → "Render Preview" returns the same layout but with India GST terms — proves the override.
4. Click `pricing.rate_sheet` → "Render Preview" returns a multi-page PDF — proves table flow + repeating header.
5. `POST /api/document/crm.quote.default/render` returns valid PDF bytes ≥ a known minimum size.

## Authoring order

1. `foundation.dataset` Phase 1 ships (datasource registry).
2. `foundation.document` Phase 1 ships (parser + renderer).
3. Demo templates authored last:
   - `quote_default.dml` first.
   - `quote_india.dml` after the default is rendering correctly (since it extends).
   - `rate_sheet.dml` + `sales_acknowledgement.dml` in parallel.
4. Smoke-test with `--with-demo=foundation.document`; visual inspection of all 4 PDFs.

---

*Back to [demo-usecase index](../../README.md).*
