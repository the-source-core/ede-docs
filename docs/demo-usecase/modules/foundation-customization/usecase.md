# `foundation.customization` — Demo Use-Case

**Module:** `ede.foundation.customization`
**App key:** `foundation.customization`
**Demo manifest entries** (target): `demo/demo_property_definitions.xml`
**Status:** 🔴 Not yet authored

---

## Use-case

Showcase the **per-tenant custom-properties** capability by attaching one user-defined property to a model that opts in (`pricing.rate` is the first adopter), then having the `logistics.pricing` demo file populate that property on its rate record.

A tester opening the demo rate sees a custom "Vendor Code" field rendered in the "Other Information" tab with a real value — proves the round-trip is wired and renders in the React webclient.

## Records produced

### `demo/demo_property_definitions.xml`

| External ID | Model | Notes |
|---|---|---|
| `customization.demo_prop_vendor_code` | `ir.model.property.definition` | `model_id=ref(ir_model.pricing_rate)`, `key="vendor_code"`, `property_type="char"`, `label="Vendor Code"`, `help="Internal code used by Ops to track the originating vendor for this rate."` |

The actual *value* of `vendor_code` on the demo rate is set in `logistics.pricing/demo/demo_rates.xml` via `<field name="properties">` (JSON dict syntax handled by the data loader for `custom_properties=True` models).

## Out of scope

- New `ir.model*` rows — those mirror the runtime registry automatically.
- `Selection`-type property with seeded options — Phase 2 enhancement (after the customization module ships a second adopter).
- Per-tenant override demo (different tenants seeing different property definitions) — Phase 2.

## Dependencies

- `foundation.customization` production data (registry-sync runs before demo pass, so `ir_model.pricing_rate` resolves)
- `logistics.pricing` demo (the rate that will carry the property value)

## Verification

```
ede migrate upgrade -t demo --with-demo=all
```

Open the demo rate in the React webclient → "Other Information" tab shows `Vendor Code: ACME-OPS-001`. Querying `pricing_rate.properties` returns `{"vendor_code": "ACME-OPS-001"}`.

## Authoring order

1. Customization demo loads first (foundation comes before logistics in `sorted_app_specs`) — defines `customization.demo_prop_vendor_code`.
2. Pricing demo loads later — its `demo_rates.xml` sets `properties` to `{"vendor_code": "ACME-OPS-001"}` on the demo rate. The customization validator hook (already running in production) coerces the value.

---

*Back to [demo-usecase index](../../README.md).*
