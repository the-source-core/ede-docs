# `logistics.documentation` — Demo Use-Case

**Module:** `domains.logistics.documentation`
**App key:** `logistics.documentation`
**Demo manifest entries** (target): `demo/demo_documentation.xml`
**Status:** 🟡 In Progress (Phase 1 implementation, 2026-06-25) — 84 module tests green; postgres `--with-demo` smoke owed in the team DB env (the documentation demo references the Globex shipment/house from `logistics.shipments`, whose full `--with-demo` load is postgres-only — upstream booking/pricing/sales-crm demos ship date strings the sqlite adapter rejects, pre-existing).

---

## Use-case

The documentation layer for the unifying scenario's **Globex Inc.** ocean export
(Mumbai → Singapore, FCL). Once the Globex shipment and its house exist
(`shipments.demo_shipment_globex_001` / `shipments.demo_house_001`), the doc team
prepares the transport documents on the **one polymorphic** `logistics.document`
envelope: a draft **House B/L** on the house — carrying its type-specific field
values (`bl_type=original`, `place_of_receipt=Mumbai ICD`, 3 originals) — plus a
draft **Shipping Instruction** on the master. Alongside the documents, the demo
seeds the **document checklist** the master must satisfy — the auto-generated plan
of required documents (HBL + SI) the DOC-13 gate reads from — so a fresh tenant
sees the checklist-driven "what must exist" view immediately, with both items
still `required` (0% complete) ready to be satisfied as the documents are issued.

It illustrates the no-re-typing handoff (parties / subject pulled from the
shipment, not re-keyed), the polymorphic envelope (HBL on a house, SI on the
master, both the same model class), and the checklist → gate spine.

## Records produced

### `demo/demo_documentation.xml`

| External ID | Model | Notes |
|---|---|---|
| `documentation.demo_doc_hbl` | `logistics.document` | House B/L on `shipments.demo_house_001`; `origin=generated`, `status=draft`, `visibility=customer`, 3 originals |
| `documentation.demo_doc_hbl_fv_bltype` | `logistics.document.field.value` | `bl_type = original` |
| `documentation.demo_doc_hbl_fv_por` | `logistics.document.field.value` | `place_of_receipt = Mumbai ICD` |
| `documentation.demo_doc_si` | `logistics.document` | Shipping Instruction on the Globex master; `status=draft`, `visibility=internal` |
| `documentation.demo_checklist_globex` | `logistics.document.checklist` | Master checklist (ocean / export / master), `completion_percent=0.00` |
| `documentation.demo_checklist_item_hbl` | `logistics.document.checklist.item` | HBL required (mandatory), sourced from `documentation.rule_ocean_export_hbl` |
| `documentation.demo_checklist_item_si` | `logistics.document.checklist.item` | SI required (mandatory), sourced from `documentation.rule_ocean_export_si` |

## Production seeds (not demo)

The reference masters ship as **production seed** (`data/`), not demo records:

- `data/logistics.document.release.type.master.csv` — 6 release types (original / telex / surrender / express / electronic / DO-release).
- `data/logistics.document.checklist.rule.csv` — baseline checklist rules per mode × direction × hierarchy.
- `data/logistics.document.leg.matrix.csv` — leg-mode × leg-role → document-type matrix.
- `data/logistics.document.gate.point.csv` — 8 downstream gate points.
- `data/logistics.document.template.master.csv` — per-type DRE template bindings.
- `data/document_field_schemas.xml` — per-type field schemas + the operational document types (SI / VGM / DO / POD / pre-alert / arrival-notice).

## Out of scope

- Issued / released documents — the demo stays at `draft` so the issue → release →
  surrender command surface is exercised by tests + the UI, not pre-seeded.
- Parties / charges on the demo documents — sourced from the shipment at issue;
  not pre-seeded to keep the demo free of partner-role coupling.
- External review, full release control, customs filing, EDI, e-signature (Phase 2).
- eBL, customer portal, reports/KPIs, AI/OCR (Phase 3).

## Dependencies

- `logistics.shipments` demo (`shipments.demo_shipment_globex_001`, `shipments.demo_house_001`) — the document subjects.
- `logistics.masters` (`masters.document_type_hbl`, `masters.transport_mode_sea`) — document type + mode.
- This module's own production seed (`documentation.document_type_si`, `documentation.rule_ocean_export_hbl`, `documentation.rule_ocean_export_si`) — the SI type + the checklist rules the items cite.
