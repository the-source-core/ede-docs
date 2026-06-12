# `foundation.customization` — Demo Use-Case

**Module:** customization substrate (code home: `ede.foundation.base` — `ir.model.property.*` + `ir.application.view`)
**Demo manifest entries:** `logistics.pricing → demo/demo_customization.xml` (property definition + anchored placement on the first-adopter host) · `foundation.base → demo/demo_application_view.xml` (organization-form extension row)
**Status:** ✅ Verified 2026-06-12 — loaded via `--with-demo` on tenant `dharmangsoni`; browser walkthrough confirmed (Vendor Code renders on the demo rate, Internal Notes on org forms)

---

## Use-case

Showcase the two customization capabilities end to end:

1. **Per-tenant custom properties (Phase 1 + 4A).** Attach one user-defined property to the opt-in host model (`pricing.rate`, `custom_properties=True`), anchor it into the rate form at a tenant-chosen position via a `<property name="properties:vendor_code"/>` element carried by an `ir.application.view` extension row, and have the pricing demo populate the value. A tester opening the demo rate sees a custom "Vendor Code" field rendered in an "Other Information" section with a real value — proving the define → place → save → render round-trip.
2. **DB-backed view extensions (Phase 4B).** A runtime-style (`owner=user`) `ir.application.view` extension row adds an "Internal Notes" section to the organization form with zero file changes — proving the composer applies DB extension rows on top of the file-synced primary.

## Records produced

### `logistics.pricing → demo/demo_customization.xml`

| External ID | Model | Notes |
|---|---|---|
| `pricing.demo_prop_vendor_code` | `ir.model.property.definition` | `model_id=ref(pricing.model_pricing_rate)`, `key="vendor_code"`, `property_type="char"`, `label="Vendor Code"` |
| `pricing.demo_appview_rate_form_other_info` | `ir.application.view` | `mode=extension`, `owner=user`, `parent_id=ref(pricing.view_pricing_rate_form_view)`, arch = `<xpath expr="//section[@string='Validity']" position="after">` adding an "Other Information" section with `<property name="properties:vendor_code"/>` |

The *value* of `vendor_code` on the demo rate is set in `logistics.pricing/demo/demo_rates.xml` (`<field name="properties" eval='{"vendor_code": "ACME-OPS-001"}'/>` on `pricing.demo_rate_buy_sea_maersk_mbai_sg`). `demo_customization.xml` is listed **before** `demo_rates.xml` because `PropertiesValidator` silently drops unknown keys on create.

### `foundation.base → demo/demo_application_view.xml`

| External ID | Model | Notes |
|---|---|---|
| `base.demo_appview_org_form_internal_notes` | `ir.application.view` | `mode=extension`, `owner=user`, `parent_id=ref(base.view_res_organization_form_view)`, arch adds an "Internal Notes" section (existing `notes` field) after "General" |

## Out of scope

- New `ir.model*` rows — those mirror the runtime registry automatically (RegistrySync).
- New **primary** `ir.application.view` rows — primaries are file-owned and mirrored by the view-sync loader; demo rows are extensions only (BR-AV-02).
- `Selection`-type property with seeded options — later enhancement (after a second adopter ships).
- Organization-scoped / role-gated extension demo — Phase 4C territory (AI propose → confirm flow shows scoping interactively).

## Dependencies

- `ir_application_view` migration applied (`ede migrate generate --app base` → `ede migrate upgrade`).
- Registry-sync + view-sync phases (both run automatically inside `ede migrate upgrade`, before the data/demo passes — so `pricing.model_pricing_rate` and `pricing.view_pricing_rate_form_view` resolve).
- `logistics.pricing` demo (the rate that carries the property value).

## Verification

```
ede migrate upgrade -t demo --with-demo=foundation.base,logistics.pricing
```

- Open the demo rate `→` an "Other Information" section after "Validity" shows `Vendor Code: ACME-OPS-001`; querying `pricing_rate.properties` returns `{"vendor_code": "ACME-OPS-001"}`.
- Open any organization form `→` an "Internal Notes" section appears after "General".
- Settings → Customizations → Application Views lists the file-synced primaries (`owner=system`) plus the two demo extension rows (`owner=user`).

## Authoring order

1. View-sync mirrors every file-authored view into `ir.application.view` (primaries + file extensions, `owner=system`).
2. `foundation.base` demo loads — org-form extension row (`owner=user`); the pre-create hook dry-runs its xpath against the synced parent arch (BR-AV-04).
3. `logistics.pricing` demo loads — property definition first, then the rate-form extension row, then `demo_rates.xml` stamps the property value (validated by `PropertiesValidator`).

## Phase 4C — AI-assisted customization (propose → confirm → render → save)

On any customizable screen the **View Customization** assistant pack (`assistant.customization`, seeded in `data/seed_skill_packs.xml`) lets a user add a custom field by asking — e.g. *"Add a Risk Band selection field to this form"*. The flow:

1. The user asks the in-app assistant. The `propose_custom_field` proposer tool (read-only, BR-AI-01) resolves the host `ir.model` uuid + the acting org and returns a confirm-gated `ActionButton`.
2. The webclient renders a **confirm card** (`CustomFieldConfirmCard`) summarising the field + placement + scope. Nothing is written yet.
3. On **Add field**, the client dispatches two `ede.create`s through the command bus (RBAC-checked, BR-AI-02): the `ir.model.property.definition` (+ selection rows), then an `ir.application.view` extension (`owner=user`, `mode=extension`) placing `<property name="properties:<key>"/>` at the proposed xpath. The view re-composes and the field renders in place; values save into the `properties` JSONB.

### Recorded e2e tests

- The deterministic admin surface is covered by [`src/tests/e2e/foundation/base/test_customization_property.py`](../../../../src/tests/e2e/foundation/base/test_customization_property.py) (property definition reachable in Settings → Customizations).
- The placed-property **render + save** outcome is browser-confirmed via the 4B demo (Vendor Code on the rate form, see Verification above).
- **Deferred:** a full chat-driven `propose → confirm` browser e2e needs a deterministic AI-provider stub (the LLM turn is non-deterministic). The proposal contract is covered at the unit level (`test_customization_proposals.py`, BR-AI-01/04), the write path by `test_customization_write_path.py` (BR-AI-02/03), and the confirm card by `CustomFieldConfirmCard.test.tsx`. Tracked as a `foundation.qa-automation` follow-up (AI-provider stub harness).

---

*Back to [demo-usecase index](../../README.md).*
