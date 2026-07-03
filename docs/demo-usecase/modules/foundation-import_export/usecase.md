# `foundation.import_export` — Demo Use-Case

**Module:** `ede.foundation.import_export`
**App key:** `foundation.import_export` (activated in `ACTIVE_MODULES` as `import_export`; the demo loader / `--with-demo` uses the full `foundation.import_export` key)
**Demo manifest entries** (Phase 1): `demo/demo_io_template_partner_quick.xml`, `demo/demo_io_run_committed.xml`
**Status:** 🟡 Authored — Phase 1 demo data ships; awaiting user-witnessed walkthrough before the module row flips ✅.

---

## Use-case

The import/export substrate is engine code, but its **headline value — "let a non-developer author a template and bulk-load a spreadsheet, no code, no deploy"** — is only believable with demo data waiting at first login. On the Acme Forwarding demo tenant, the story is:

> The Acme Forwarding **admin** wants to bootstrap the address book by importing a contact list. Rather than typing each partner in by hand, they open **Settings → Imports & Exports → Templates**, find the ready-made **"Demo — Quick Partner Import"** template (an *admin-source* template, so its columns are fully editable), tweak it if they like, then head to **Contacts**, click **Import**, drop in a CSV, walk the preview screen, and commit — new `res.partner` rows appear. The **Import Runs** list already shows one previously-committed run so the history screen is not empty on a cold tenant.

This demonstrates the three things Phase 1 promises end-to-end:

1. **Admin-UI template authoring** (Mode C) — a template that exists purely as data, editable in the browser, no `@api.io_template` decorator and no XML redeploy.
2. **The parse → validate → preview → commit pipeline** — reachable from the auto-injected List-view **Import** button.
3. **Run history + archive** — the committed run in the Runs list, with per-row counts.

The engine's other authoring modes are already exercised by the first adopter: `pricing.rate.fcl.upload` (Mode A — `@api.io_template` in `src/domains/logistics/pricing/`) ships with the pricing demo data, so `--with-demo=all` shows both an admin-source and a code-source template side by side.

## Records produced (Phase 1)

### `demo/demo_io_template_partner_quick.xml`

| External ID | Model | Notes |
|---|---|---|
| `import_export.demo_io_template_partner_quick` | `ir.io.template` | `code="demo.res_partner.quick_import"`; `target_model_id → base.model_res_partner`; `source="admin"` (fully editable columns); `on_duplicate="skip"`; `file_format="both"`. |
| `import_export.demo_io_template_partner_quick_col_a` | `ir.io.template.column` | Column **A → `name`** (char, required). |
| `import_export.demo_io_template_partner_quick_col_b` | `ir.io.template.column` | Column **B → `email`** (char; `validator_csv="regex:^.+@.+\..+$"`; `is_dedupe_key=true`). |
| `import_export.demo_io_template_partner_quick_col_c` | `ir.io.template.column` | Column **C → `phone`** (char, optional). |

### `demo/demo_io_run_committed.xml`

| External ID | Model | Notes |
|---|---|---|
| `import_export.demo_io_run_partner_committed` | `ir.io.run` | One already-committed run of the template above; `status="committed"`; `uploaded_by_uid → base.admin_user`; 5 rows total / 5 valid / 5 committed. Datetime fields use literal ISO-8601 strings (the loader coerces `datetime`-typed columns; `eval=` accepts literals only, not `datetime.utcnow()` calls). |

## Out of scope

- **Export templates** (`direction="export"`) and the Download-Template button — `foundation.import_export` Phase 2.
- **Async / background runs** for large files — Phase 2 (`foundation.jobs`).
- **A seeded error report + invalid-row run** — the malformed-row path is covered by the acceptance walkthrough test and the live browser demo (upload a CSV with one broken row); it is not seeded because error-report bytes belong to `storage.document`, which demo XML does not populate.

## Dependencies

- `foundation.base` — `base.model_res_partner` (the `ir.model` target row) and `base.admin_user` (run uploader). Both are non-demo seed rows, so the demo loads even without `--with-demo=foundation.base`.
- No cross-module demo refs — this module's demo is self-contained.

## Verification

```
ede migrate upgrade -t <tenant> --with-demo=foundation.import_export
```

Then, in the React webclient:

1. **Settings → Imports & Exports → Templates** — "Demo — Quick Partner Import" (source **admin**) visible; if pricing demo is also loaded, `pricing.rate.fcl.upload` (source **code**) appears too.
2. Open the demo template → **Columns** tab is **editable** (admin source); add/remove a column and save.
3. **Settings → Imports & Exports → Import Runs** — one committed run visible (`contacts_sample.csv`, 5/5 committed).
4. **Settings → Imports & Exports → Validators** — the 8 built-in validators listed (read-only catalogue).
5. Navigate to **Contacts** (`res.partner` list) → **Import** toolbar button present → pick a small CSV (include one malformed-email row) → preview shows valid/invalid split → **Commit** → new contacts appear.

## Authoring order

1. `foundation.import_export` Phase 1 engine ships (done).
2. `demo/demo_io_template_partner_quick.xml` + `demo/demo_io_run_committed.xml` authored + declared under the manifest `demo:` key (this WS).
3. Smoke-test `--with-demo=import_export`, then the user-witnessed walkthrough flips the module row ✅.

---

*Back to [demo-usecase index](../../README.md).*
