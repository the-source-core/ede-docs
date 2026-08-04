# `foundation.tax` — Demo Use-Case

**Module:** `ede.foundation.tax`
**App key:** `foundation.tax`
**Demo manifest entries:** `demo/demo_tax.xml`
**Status:** ✅ Delivered (2026-07-24) · **rewritten 2026-07-31** for D-10 (determination retired) + D-11 (taxes are a set)

---

## Use-case

Acme Forwarding Ltd. (Mumbai HQ) bills freight on the flagship lane **INMUM → SGSIN**
(Mumbai, India → Singapore). Its operational-tax configuration is India-jurisdiction
`res.tax.*` data that the pure engine runs on.

The organising idea is that **a charge line carries one tax because that is what the
service is** — freight is an 18% GST supply, so the catalogue tax is `IN-GST-18`. What
varies per document is not the service; it is the **jurisdiction**. So the same catalogue
tax has to become three different things:

- **Export of services out of India** (the flagship international-freight lane) →
  **zero-rated**: prints an international-transport exemption reason and computes no tax.
- **A supply inside the entity's own state** → **CGST 9% + SGST 9%**, and the taxed party
  must see them as **two separate taxes** on the document.
- **Anything else** (inter-state) → a single **IGST 18%**.

That remapping is what `res.tax.fiscal.position` does, and it is the module's **only**
answer to "which tax applies" (D-10). Which position applies is configured data, not code:
an ordered, first-match-wins chain, each position guarded by a formula expression over the
resolution context (`direction`, `place_of_supply_state`, `entity_state`) and narrowed to
India by the indexed `country_id` pre-filter — with the catch-all correctly placed **last**
(seq 90). Deriving the place of supply is the calling document's job; this module never
guesses it (D-8).

### Why the demo carries two look-alike mechanisms

A destination set and `res.tax.component` can both express "9% + 9%", so the demo draws the
boundary explicitly and shows each doing only its own job:

| Mechanism | Means | Demo record |
|---|---|---|
| A destination **code set** | Independent, **co-applied** levies on the same net. Each is a separate tax to the payer. | `IN-GST-18` → [`IN-CGST-9`, `IN-SGST-9`] |
| `res.tax.component` | **Tax-on-tax within one legal levy** — a component whose base is the *preceding total*, which no set of co-applied codes can express (TAX-06). | `IN-GST-28-CESS` (GST 28% on net, then Compensation Cess 12% on net + that 28%) |

`IN-GST-28-CESS` is a catalogue code **no position remaps**, so it does double duty: it keeps
the compound engine path demo-covered, and it demonstrates the pass-through rule — a source
tax with no map row is charged exactly as configured.

The demo is deliberately self-contained: it references only `foundation.base` seeds
(`base.country_in`) and this module's own production seeds (the generic rounding rule and
the international-transport exemption reason), never a logistics record. That keeps the
platform module's demo loadable on its own — `--with-demo=foundation.tax` produces a usable
Indian tax configuration with no domain module present.

## Records produced

### `demo/demo_tax.xml`

**Tax codes + rates**

| External ID | Model | Notes |
|---|---|---|
| `tax.demo_code_in_gst18` | `res.tax.code` | `IN-GST-18` · the **catalogue / mapping-source** code · taxable · India · rounding = seed `tax.rounding_line_2dp_half_up`. |
| `tax.demo_rate_in_gst18` | `res.tax.rate` | Percentage 18 · effective from 2017-07-01 (GST rollout) · open-ended. Resolves on its own when no position applies. |
| `tax.demo_code_in_cgst9` | `res.tax.code` | `IN-CGST-9` · the central half of an intra-state supply · a **destination** code. |
| `tax.demo_rate_in_cgst9` | `res.tax.rate` | Percentage 9 · effective from 2017-07-01. |
| `tax.demo_code_in_sgst9` | `res.tax.code` | `IN-SGST-9` · the state half · co-applied with `IN-CGST-9`. |
| `tax.demo_rate_in_sgst9` | `res.tax.rate` | Percentage 9 · effective from 2017-07-01. |
| `tax.demo_code_in_igst18` | `res.tax.code` | `IN-IGST-18` · inter-state · single-rate path (the service promotes it into a one-component spec). |
| `tax.demo_rate_in_igst18` | `res.tax.rate` | Percentage 18 · effective from 2017-07-01 · open-ended. |
| `tax.demo_code_in_export_zero` | `res.tax.code` | `IN-EXPORT-ZERO` · **zero_rated** · exemption reason = seed `tax.exemption_intl_transport`. Computes no tax; prints the reason (TAX-05). |
| `tax.demo_code_in_gst28_cess` | `res.tax.code` | `IN-GST-28-CESS` · **compound** · rate comes from the two components below, not a flat rate. |
| `tax.demo_component_in_gst28` | `res.tax.component` | GST 28% · sequence 10 · base `net`. |
| `tax.demo_component_in_cess12` | `res.tax.component` | Compensation Cess 12% · sequence 20 · base **`preceding_total`** — the tax-on-tax path (TAX-06). On 1,000: 280 + 153.60 = **433.60**. |

**Fiscal positions — ordered, first match wins, catch-all last**

| External ID | Model | Notes |
|---|---|---|
| `tax.demo_fiscal_position_in_export` | `res.tax.fiscal.position` | `IN-EXPORT` · seq 10 · guard `direction == 'export'` · India. Highest priority so it shadows the domestic positions. |
| `tax.demo_fiscal_position_in_intra` | `res.tax.fiscal.position` | `IN-INTRA` · seq 20 · guard `place_of_supply_state == entity_state` · India. |
| `tax.demo_fiscal_position_in_inter` | `res.tax.fiscal.position` | `IN-INTER` · seq 90 · **empty guard** (catch-all, correctly placed last) · India. |

**The mapping — one row per (position, source code)**

| External ID | Model | Notes |
|---|---|---|
| `tax.demo_fp_map_export_gst18` | `res.tax.fiscal.position.map` | Under `IN-EXPORT`: `IN-GST-18` → [`IN-EXPORT-ZERO`]. |
| `tax.demo_fp_map_intra_gst18` | `res.tax.fiscal.position.map` | Under `IN-INTRA`: `IN-GST-18` → [`IN-CGST-9`, `IN-SGST-9`] — a **set**, two visible taxes. |
| `tax.demo_fp_map_inter_gst18` | `res.tax.fiscal.position.map` | Under `IN-INTER`: `IN-GST-18` → [`IN-IGST-18`]. |

**18 records** — 6 codes, 4 rates, 2 components, 3 fiscal positions, 3 map rows.

> The production seed also ships one non-demo position, `tax.tax_fiscal_position_domestic`
> (`DOMESTIC`, identity / no remapping, manual-only, sequence 100). It is deliberately **not**
> in the auto-apply chain: an empty guard there would match every document and shadow every
> jurisdiction position above it.

## Out of scope

- **Billing-document tax lines / snapshots** — owned by the consuming domain; this
  module ships only the reusable masters.
- **Country localization packs** — the demo is a single illustrative India setup, not a
  statutory rate pack (that is a future `foundation.l10n` scope; see phase OQ-2).
- **State-restricted positions** — the demo leaves `state_ids` empty (country-scoped only),
  so it does not depend on which `res.state` rows a tenant has seeded. The pre-filter itself
  is covered by unit tests.
- **Organization-scoped positions** — the demo leaves `organization_id` empty, so it does not
  forward-ref the not-yet-shipped `base.demo_org_*` record.
- **New rounding rules / exemption reasons** — the demo reuses the production seeds
  (`data/res.tax.rounding.rule.csv`, `data/res.tax.exemption.reason.csv`) rather than
  re-authoring them as demo rows.

## Dependencies

- `foundation.base` production seeds — `res.country` (`base.country_in`). Loads first
  (foundation before domains, data pass before demo pass).
- `foundation.tax` production seeds — `data/res.tax.rounding.rule.csv`
  (`tax.rounding_line_2dp_half_up`) and `data/res.tax.exemption.reason.csv`
  (`tax.exemption_intl_transport`), loaded in the same upgrade's data pass before the
  demo pass.

## Verification

```bash
ede migrate upgrade -t <tenant> --with-demo=foundation.tax
# first run:  demo_load: 14 created, 4 updated, 0 skipped (scope=foundation.tax)
#             (4 updated = the codes/rate that already existed before the D-11 rewrite)
# re-run:     demo_load: 0 created, 18 updated, 0 skipped (idempotent)
```

Post-load query — the mapping, as configured:

```sql
SELECT p.code AS position, sc.code AS source,
       COALESCE(string_agg(dc.code, ' + ' ORDER BY dc.code), '(empty = out of scope)') AS destination
  FROM res_tax_fiscal_position_map m
  JOIN res_tax_fiscal_position p ON p.record_uuid = m.fiscal_position_id
  JOIN res_tax_code sc           ON sc.record_uuid = m.source_tax_code_id
  LEFT JOIN res_tax_fiscal_position_map_res_tax_code_rel r ON r.res_tax_fiscal_position_map_id = m.record_uuid
  LEFT JOIN res_tax_code dc      ON dc.record_uuid = r.res_tax_code_id
 GROUP BY p.code, sc.code, p.sequence
 ORDER BY p.sequence;
--  IN-EXPORT | IN-GST-18 | IN-EXPORT-ZERO
--  IN-INTRA  | IN-GST-18 | IN-CGST-9 + IN-SGST-9
--  IN-INTER  | IN-GST-18 | IN-IGST-18
```

Engine sanity (any tenant with the demo loaded):

```python
from decimal import Decimal

from ede.foundation.tax.services.fiscal_position import map_tax_codes, resolve_fiscal_position
from ede.foundation.tax.services.tax_service import compute_tax_for

def apply(amount, context, sources):
    """What a consuming billing line does: resolve, remap, compute each code."""
    position = resolve_fiscal_position(env, context=context)
    return [compute_tax_for(env, amount, code) for code in map_tax_codes(env, position, sources)]

# export lane → zero-rated, prints a reason
[r.tax_code for r in apply(Decimal("10000"), {"direction": "export"}, ["IN-GST-18"])]
# -> ["IN-EXPORT-ZERO"]   tax_amount 0, exemption_reason_code "INTL-TRANSPORT"

# intra-state → two separate taxes, 900 + 900
[(r.tax_code, r.tax_amount) for r in
 apply(Decimal("10000"), {"place_of_supply_state": "MH", "entity_state": "MH"}, ["IN-GST-18"])]
# -> [("IN-CGST-9", 900.00), ("IN-SGST-9", 900.00)]

# inter-state → one tax, same total
[(r.tax_code, r.tax_amount) for r in
 apply(Decimal("10000"), {"place_of_supply_state": "KA", "entity_state": "MH"}, ["IN-GST-18"])]
# -> [("IN-IGST-18", 1800.00)]

# compound, no remap (pass-through) → tax-on-tax
compute_tax_for(env, Decimal("1000"), "IN-GST-28-CESS").tax_amount   # -> Decimal("433.60")
```

## Authoring order

Within `demo/demo_tax.xml`, records that are `ref`d later must appear earlier:

1. All `res.tax.code` rows, each followed by its rate(s) / component(s).
2. The three `res.tax.fiscal.position` rows.
3. The three `res.tax.fiscal.position.map` rows (each refs a position **and** the source +
   destination codes declared above). A destination set is written as `<ref>` children of the
   `dest_tax_code_ids` field, which the loader turns into idempotent link commands.

Manifest `demo: ["demo/demo_tax.xml"]`.

---

*Back to [demo-usecase index](../../README.md).*
