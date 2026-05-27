<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Import / Export Engine — Implementation Docs

**Module:** `foundation.import_export` (`src/ede/foundation/import_export/`)
**Roadmap:** [roadmap/foundation/import_export/](../roadmap/foundation/import_export/README.md)
**Status:** 🔴 Not Started — drafted 2026-05-27
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **template-driven, declarative, runtime-resolved import + export engine** that turns "user uploads an Excel" into a four-step pipeline (parse → validate → preview → commit) configured by a single record (`ir.io.template`) and invoked from any UI entry point — list-view toolbar button, form-view command button, settings admin page, REST API, or scheduled job / email attachment. It does not know what a "rate" or a "partner" or a "shipment" is — it only knows that a template binds a target model + a column mapping + a validator vocabulary, and that the framework can resolve any of those against the live `Registry`.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Without this engine, every consumer reimplements the same eight concerns: file picker, Excel/CSV parser, per-cell type coercion, lookup-by-code resolver, dedupe-key matcher, partial-success transaction policy, error-report formatter, and audit trail of uploaded files. Eight concerns × the dozen+ modules that will need bulk upload = ~100 hand-rolled half-baked implementations that drift in policy. The platform owns it once; consumers add one decorator or one XML row and get all eight for free. First consumer (re-scoping its planned own-built Excel upload onto this engine): `logistics.pricing` Phase 1 Feature 05 (Rate Sheet Import).
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user entry points** — auto-injected "Import" button on any ListView / KanbanView whose model has ≥1 template (opt out via `<list import="disabled">`, pin via `<list import-template="...">`); `<button special="io_import" template_code="..."/>` DSL hook on FormViews for child-record imports; Settings → Data Management → Imports (new top-level Settings group) for admin browsing.
- **Programmatic entry points** — `Command("io.run.upload" | "io.run.preview" | "io.run.commit" | "io.run.cancel")` via the bus · `@api.on_event("io.run.committed" | "io.run.failed")` for downstream reactions · `register_io_validator(code, fn)` to ship a custom validator from a consumer module · `@api.io_template(...)` decorator (or `<record model="ir.io.template">` XML, or admin UI form) as the three coexisting authoring modes.
- **Integration boundary** — produces `ir.io.run` rows + target-model records on commit; consumes `foundation.storage` (uploaded file + error-report archive), `foundation.approval` (optional approval gate), `foundation.notifications` (uploader + role-list notifications), `foundation.communication` (optional chatter post on created records), `foundation.jobs` (Phase 2 — async), `foundation.presentation` (auto-injected button, reuses `FileUploadSpecial`, preview rendered by standard FormView), `foundation.dataset` (Phase 2 safe AST evaluator for the `formula` validator).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
Four-stage pipeline behind a single `IoService` class:

```
User clicks "Import"  →  IoService.upload()
                            ├─ resolve template (by code)
                            ├─ archive file → storage.document
                            ├─ ir.io.run row (status=parsing)
                            ├─ Parser (Excel via openpyxl / CSV via stdlib)
                            │     → ir.io.row inserts with raw_values
                            ├─ Validator chain per column per row
                            │     → parsed_values + errors + action
                            └─ status = preview
User reviews preview  →  IoService.commit()
                            ├─ (optional) approval gate via foundation.approval
                            ├─ env.transaction():
                            │     for each valid row → target.create / .write
                            ├─ emit io.run.committed
                            ├─ notify uploader + notify-list
                            └─ status = committed
```

Phase 2 retrofits an async branch (auto-routes through `foundation.jobs` when row count > `IO_MAX_SYNC_ROWS`). Phase 3 adds multi-sheet templates, scheduled SFTP sources, email-attachment ingestion, per-row inline edit on preview, run rollback, UPDATE diff preview, template versioning, JSON export/import, retention sweeper.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Phase |
|---|---|---|
| `ir.io.template` | Declarative template — code, label, target_model, parent_model (for child imports), direction, file_format, on_duplicate, approval_required, source (code/xml/admin) | 1 |
| `ir.io.template.column` | One column mapping — excel_col letter, header_label, field_name, data_type, validators, dedupe-key flag | 1 |
| `ir.io.run` | One upload / export attempt — template, file_document (storage.document), status, row counts, error_summary, approval_case | 1 |
| `ir.io.row` | Per-row state — raw_values_json, parsed_values_json, errors_json, action, target_record_uuid | 1 |
| `ir.io.validator` | Registry mirror of named validators (8 built-ins + consumer-registered) | 1 |
| `ir.io.template.version` | Append-only snapshot of a template's columns at the moment it changed; runs pin to their version | 3 |
| `ir.io.scheduled.source` | SFTP / Drive / S3 folder watch — connector + template + cron + after_import_action | 3 |
| `ir.io.email.binding` | Maps `imports+<local-part>@<tenant>` email address → template_code | 3 |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `IoService` | Public engine entry: `upload / preview / commit / cancel / export / download_blank` | `src/ede/foundation/import_export/services/io_service.py` |
| `ExcelParser` | openpyxl-driven `.xlsx` reader (Phase 1: read; Phase 2: also write for export) | `services/parsers/excel_parser.py` |
| `CsvParser` | stdlib csv with delimiter sniffer + UTF-8 BOM + latin-1 fallback | `services/parsers/csv_parser.py` |
| `validator_registry` | `register_io_validator(code, fn)` API + 8 built-ins (`required`/`regex`/`range`/`enum`/`iso_4217_currency`/`lookup`/`date`/`formula`) | `services/validators/registry.py` + `services/validators/builtins.py` |
| `template_registry` | `@api.io_template(...)` decorator registry + boot-time sync to `ir.io.template` rows | `services/template_registry.py` + `services/template_sync.py` |
| `pipeline.upload_stage / validate_stage / preview_stage / commit_stage` | The four engine stages | `services/pipeline/*.py` |
| `pipeline.export_stage` (P2) | Reverse — filter → fetch → format → file → archive | `services/pipeline/export_stage.py` |
| `pipeline.rollback_stage` (P3) | Reverses a committed run inside the retention window | `services/pipeline/rollback_stage.py` |
| `formatters.excel_writer / csv_writer` (P2) | Output side of the engine (export + download-blank) | `services/formatters/` |
| `scheduled_sources.sweep_scheduled_sources` (P3) | `@api.scheduled_job` master sweeper for SFTP / Drive / S3 folder watchers | `services/scheduled_sources.py` |
| `sweeper.run_retention_sweep` (P3) | Nightly `@api.scheduled_job` purging runs past `IO_RUN_RETENTION_DAYS` | `services/sweeper.py` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `io.run.upload` (P1) | Upload from any UI or API | Parse + validate → run lands in `status=preview` |
| `io.run.preview` (P1) | UI lists rows / paginates | Returns run + paginated rows |
| `io.run.commit` (P1) | User clicks Commit | Transactionally writes valid rows to target model; optional approval gate |
| `io.run.cancel` (P1) | User abandons | Status → `cancelled`; rows preserved for audit |
| `io.run.export` (P2) | UI Export button / API | Generates output file + archives → user downloads |
| `io.template.download_blank` (P2) | UI Download Template button | Generates blank Excel with header + data-validation dropdowns |
| `io.row.revalidate` (P3) | UI inline cell edit | Re-runs validators on one row; updates errors / action / approval-case lifecycle |
| `io.run.rollback` (P3) | UI / API | Reverses a committed run (deletes CREATEs / restores `before_values_json` on UPDATEs) |
| `io.template.export_json` / `import_json` (P3) | CLI | Portable JSON of template + columns for cross-tenant move |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `io.run.committed` (P1) | After successful commit | `notification.send` (uploader + notify-list); consumer-side `@api.on_event` for downstream recompute |
| `io.run.failed` (P1) | Parse / validate / commit failure terminal | `notification.send` |
| `io.run.async_started` (P2) | Async branch enqueued | `notification.send` "processing in background" |
| `io.run.exported` (P2) | Export run completes | Logs / audit downstream |
| `io.run.rolled_back` (P3) | Rollback success | Audit log |
| `io.run.rollback_failed` (P3) | Rollback aborted (blocked rows) | Audit + notification |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Phase |
|---|---|---|
| `POST /api/io/run/upload` | Upload file + template_code (multipart) | 1 |
| `GET /api/io/run/{uuid}` | Run + paginated rows (preview) | 1 |
| `POST /api/io/run/{uuid}/commit` | Commit | 1 |
| `POST /api/io/run/{uuid}/cancel` | Cancel | 1 |
| `GET /api/io/run/{uuid}/error-report` | Stream generated error CSV | 1 |
| `GET /api/io/templates/for-model?model_key=...&direction=...` | List templates targeting a model (used by ListView auto-injection) | 1 (P2 adds `direction` filter) |
| `GET /api/io/run/{uuid}/progress` | Lightweight polling endpoint for async runs | 2 |
| `POST /api/io/run/export` | Programmatic export trigger | 2 |
| `GET /api/io/run/{uuid}/export-file` | Stream export file | 2 |
| `GET /api/io/templates/{code}/blank` | Stream blank-template Excel | 2 |
| `POST /api/io/row/{uuid}/revalidate` | Re-run validators after inline edit | 3 |
| `POST /api/io/run/{uuid}/rollback` | Rollback a committed run | 3 |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior | Phase |
|---|---|---|
| `pre.ir.io.template.update` | `code` + `source` immutable | 1 |
| `pre.ir.io.template.column.{create,update,delete}` | Reject runtime structural edits when parent template `source='code'` (outside reconciler context) | 1 |
| `post.ir.io.template.create / update` | Auto-snapshot to `ir.io.template.version` (Phase 3 versioning) | 3 |
| `post.ir.io.template.column.{create,update,delete}` | Auto-snapshot to `ir.io.template.version` | 3 |
| `@api.on_event("ir.approval.case.approved")` | When subject is `ir.io.run` (whole-run mode) or `ir.io.row` (per-row mode) → auto-dispatch commit | 1+2 |
| `@api.on_event("ir.approval.case.rejected")` | Cancel run / mark row rejected | 1+2 |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
```
ir.io.run.status:
  uploading
      ├─► processing_async (P2)  ─► parsing
      └─► parsing
           ├─► failed
           └─► validating
                ├─► failed
                └─► preview
                     ├─► cancelled
                     ├─► approved_pending (when approval_required) ─► committing
                     ├─► committing
                     │       ├─► failed
                     │       └─► committed ─► rolling_back (P3) ─► rolled_back / rollback_failed
                     └─► (per-row mode) preview stays here while individual rows
                          flip awaiting_approval → create / rejected; status
                          flips committed only when all rows resolved

ir.io.run.direction = 'export' (P2):
  exporting ─► export_complete / failed
```
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `import_export` (placed after `presentation`)
- Manifest `depends` (Phase 1): `["base", "storage", "approval", "notifications", "presentation"]`. Phase 2 adds `"jobs"`. Phase 3 adds `"email"` and (transitively) `"connectors"`.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Phase | Purpose |
|---|---|---|---|---|---|
| `IO_ENABLED` | `bool` | `True` | `EDE_IO_ENABLED` | 1 | Hard kill-switch for the entire engine |
| `IO_MAX_SYNC_ROWS` | `int` | `5000` | `EDE_IO_MAX_SYNC_ROWS` | 1 | Sync ceiling — Phase 1 rejects; Phase 2 auto-routes async |
| `IO_PREVIEW_ROW_LIMIT` | `int` | `100` | `EDE_IO_PREVIEW_ROW_LIMIT` | 1 | Preview-page page size |
| `IO_ARCHIVE_UPLOADS` | `bool` | `True` | `EDE_IO_ARCHIVE_UPLOADS` | 1 | Archive uploaded files to `storage.document` |
| `IO_RUN_RETENTION_DAYS` | `int` | `365` | `EDE_IO_RUN_RETENTION_DAYS` | 1 (column) / 3 (sweeper) | Run-row retention; Phase 3 wires the purge sweeper |
| `IO_ASYNC_ENABLED` | `bool` | `True` | `EDE_IO_ASYNC_ENABLED` | 2 | Master switch — when False, files > sync limit reject |
| `IO_MAX_ASYNC_ROWS` | `int` | `500000` | `EDE_IO_MAX_ASYNC_ROWS` | 2 | Hard ceiling for async runs |
| `IO_PROGRESS_REPORT_EVERY_N_ROWS` | `int` | `100` | `EDE_IO_PROGRESS_REPORT_EVERY_N_ROWS` | 2 | Tick `env.job_progress` every N rows |
| `IO_RUN_ROLLBACK_WINDOW_DAYS` | `int` | `30` | `EDE_IO_RUN_ROLLBACK_WINDOW_DAYS` | 3 | Commits older than this cannot be rolled back |
| `IO_SCHEDULED_SOURCES_ENABLED` | `bool` | `True` | `EDE_IO_SCHEDULED_SOURCES_ENABLED` | 3 | Master toggle for SFTP / folder watchers |
| `IO_EMAIL_INBOUND_ENABLED` | `bool` | `True` | `EDE_IO_EMAIL_INBOUND_ENABLED` | 3 | Master toggle for email-attachment imports |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Phase | Purpose |
|---|---|---|---|---|---|
| `io.default_archive_uploads` | org | bool | `true` | 1 | Default `ir.io.template.archive_uploads` for admin-created templates |
| `io.default_approval_required` | org | bool | `false` | 1 | Default `ir.io.template.approval_required` for admin-created templates |
| `io.default_on_duplicate` | org | Enum (`error`/`skip`/`update`) | `error` | 1 | Default `ir.io.template.on_duplicate` for admin-created templates |
| `io.default_approval_mode` | org | Enum (`whole_run`/`per_row`) | `whole_run` | 2 | Default for new approval-required templates |
| `io.default_notify_on_completion` | org | char | `""` | 2 | Default `notify_on_completion_role_codes_csv` |
| `io.scheduled_source.default_cron` | org | char | `0 6 * * *` | 3 | Default cron for new watchers |
| `io.email_binding.default_auto_create_partner` | org | bool | `false` | 3 | Default for new email bindings |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields | Phase |
|---|---|---|---|
| Settings → Data Management → Imports | `data/import_export_settings.xml` | `io.default_archive_uploads`, `io.default_approval_required`, `io.default_on_duplicate` | 1 |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds | Phase |
|---|---|---|
| `data/ir.rbac.permission.csv` | 13 permission rows (template:read/manage · template.column:read/manage · run:read/manage · row:read/write · validator:read/manage · io.run.upload · io.run.commit · io.run.cancel · io.skip_approval; P3 adds `io.run.rollback`) | 1 (P3 adds rollback) |
| `data/import_export_validators.xml` | 8 built-in `ir.io.validator` rows | 1 |
| `data/import_export_menus.xml` | Settings → Data Management top-level group + Imports sub-menu (Templates / Runs / Validators) | 1 |
| `data/io_notification_templates.xml` | 6 `ir.notification.template` rows (3 events × web/email) | 2 |
| `data/io_sweep_notification_templates.xml` | 1 web template for retention-sweep summary | 3 |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Import Engine Core + First Adopter | 🔴 Not Started | [phase-1/](../roadmap/foundation/import_export/phase-1/README.md) |
| Phase 2 | Async + Approval Polish + Export Sibling | 🔴 Not Started | [phase-2/](../roadmap/foundation/import_export/phase-2/README.md) |
| Phase 3 | Power Features + External Ingestion | 🔴 Not Started | [phase-3/](../roadmap/foundation/import_export/phase-3/README.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries). The entire module is currently scoped-but-not-started; every Phase-1 WS is a gap.

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Module skeleton + manifest + settings + activation | 🔴 | [phase-1/WS-01](../roadmap/foundation/import_export/phase-1/WS-01-module-skeleton.md) |
| 5 core models + Alembic | 🔴 | [phase-1/WS-02](../roadmap/foundation/import_export/phase-1/WS-02-core-models.md) |
| Excel + CSV parsers | 🔴 | [phase-1/WS-03](../roadmap/foundation/import_export/phase-1/WS-03-parsers.md) |
| 8 built-in validators + registration API | 🔴 | [phase-1/WS-04](../roadmap/foundation/import_export/phase-1/WS-04-validator-registry.md) |
| `@api.io_template` decorator + XML + admin authoring | 🔴 | [phase-1/WS-05](../roadmap/foundation/import_export/phase-1/WS-05-decorator-and-xml.md) |
| Pipeline (upload / validate / preview / commit) + 4 commands + 2 events | 🔴 | [phase-1/WS-06](../roadmap/foundation/import_export/phase-1/WS-06-import-engine.md) |
| 6 HTTP routes + 13 RBAC permissions | 🔴 | [phase-1/WS-07](../roadmap/foundation/import_export/phase-1/WS-07-controllers-and-rbac.md) |
| Frontend (auto-injected button + preview view + admin) | 🔴 | [phase-1/WS-08](../roadmap/foundation/import_export/phase-1/WS-08-frontend.md) |
| First adopter `pricing.rate.fcl.upload` + pricing re-scope | 🔴 | [phase-1/WS-09](../roadmap/foundation/import_export/phase-1/WS-09-first-adopter-pricing.md) |
| Phase 1 acceptance + demo + docs | 🔴 | [phase-1/WS-10](../roadmap/foundation/import_export/phase-1/WS-10-acceptance-demo-docs.md) |
| Phase 2 — async via `foundation.jobs` | 🔴 | [phase-2/WS-11](../roadmap/foundation/import_export/phase-2/WS-11-async-routing.md) |
| Phase 2 — progress ticks + polling endpoint | 🔴 | [phase-2/WS-12](../roadmap/foundation/import_export/phase-2/WS-12-job-progress.md) |
| Phase 2 — notifications (async start / completion / notify-list) | 🔴 | [phase-2/WS-13](../roadmap/foundation/import_export/phase-2/WS-13-notifications-async.md) |
| Phase 2 — per-row approval mode | 🔴 | [phase-2/WS-14](../roadmap/foundation/import_export/phase-2/WS-14-per-row-approval.md) |
| Phase 2 — export direction + pipeline | 🔴 | [phase-2/WS-15](../roadmap/foundation/import_export/phase-2/WS-15-export-pipeline.md) |
| Phase 2 — Download Blank Template | 🔴 | [phase-2/WS-16](../roadmap/foundation/import_export/phase-2/WS-16-download-template.md) |
| Phase 2 — export frontend (toolbar + selection) | 🔴 | [phase-2/WS-17](../roadmap/foundation/import_export/phase-2/WS-17-export-frontend.md) |
| Phase 2 — acceptance + docs | 🔴 | [phase-2/WS-18](../roadmap/foundation/import_export/phase-2/WS-18-acceptance.md) |
| Phase 3 — multi-sheet templates | 🔴 | [phase-3/WS-19](../roadmap/foundation/import_export/phase-3/WS-19-multi-sheet.md) |
| Phase 3 — scheduled imports (SFTP / Drive / S3) | 🔴 | [phase-3/WS-20](../roadmap/foundation/import_export/phase-3/WS-20-scheduled-imports.md) |
| Phase 3 — email-attachment imports | 🔴 | [phase-3/WS-21](../roadmap/foundation/import_export/phase-3/WS-21-email-attachment-imports.md) |
| Phase 3 — per-row inline edit on preview | 🔴 | [phase-3/WS-22](../roadmap/foundation/import_export/phase-3/WS-22-per-row-edit.md) |
| Phase 3 — run rollback | 🔴 | [phase-3/WS-23](../roadmap/foundation/import_export/phase-3/WS-23-run-rollback.md) |
| Phase 3 — UPDATE diff preview | 🔴 | [phase-3/WS-24](../roadmap/foundation/import_export/phase-3/WS-24-update-diff.md) |
| Phase 3 — template versioning + JSON export/import | 🔴 | [phase-3/WS-25](../roadmap/foundation/import_export/phase-3/WS-25-template-versioning.md) |
| Phase 3 — run retention sweeper | 🔴 | [phase-3/WS-26](../roadmap/foundation/import_export/phase-3/WS-26-run-sweeper.md) |
| Phase 3 — acceptance + module closeout | 🔴 | [phase-3/WS-27](../roadmap/foundation/import_export/phase-3/WS-27-acceptance.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _Will be populated as integration learnings emerge once Phase 1 lands. Anticipated pitfalls (from the roadmap discussion): (1) confusing the manifest-driven `data_loader` with this engine's runtime user-driven path — they share the CSV row-parsing primitive but diverge sharply at orchestration. (2) Forgetting `parent_uuid` on parent-bound templates — engine raises but error message could be clearer. (3) Trying to edit columns on a `source='code'` template via the admin form — locked by hook; edit the decorator in source instead._

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1 lands 5 new tables (`ir_io_template` · `ir_io_template_column` · `ir_io_run` · `ir_io_row` · `ir_io_validator`) via Alembic. All cleanly additive — no existing-table changes.
- Phase 1 adds 8 RBAC permission rows + 3 default role bindings. Existing roles unaffected.
- Phase 1 adds 3 settings via `FoundationSettings` and 3 `ir.config` keys — all with safe defaults.
- Phase 2 adds 4 additive columns on `ir.io.run` (`job_run_id`, `progress_pct`, `progress_msg`, `direction`) + 5 columns on `ir.io.template` (`approval_mode`, `notify_on_completion_role_codes_csv`, `export_default_domain_json`, `export_sort_order`) + 1 on `ir.io.row` (`approval_case_id`). All nullable.
- Phase 2 adds `openpyxl` already-shipped runtime dep usage for write-side (export).
- Phase 3 adds 4 new tables (`ir.io.template.version`, `ir.io.scheduled.source`, `ir.io.email.binding`, plus storage for `before_values_json` + `before_values_preview_json` columns on `ir.io.row` and `rollback_of_run_id` / `rolled_back_by_run_id` on `ir.io.run`).
- Phase 3 retention sweeper purges runs past `IO_RUN_RETENTION_DAYS`. Existing tenants with old runs: set `IO_RUN_RETENTION_DAYS=0` to disable until ready to migrate.
- `logistics.pricing` Phase 1 Feature 05 (Rate Sheet Import) is re-scoped from "build a custom upload" (~6-8d) to "declare an `ir.io.template`" (~1-2d) the moment this module's Phase 1 ships.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `role_internal_user` | `ir.io.template:read` · `ir.io.run:read` · `ir.io.row:read` · `ir.io.validator:read` (+ optionally `io.run.upload` / `io.run.commit` / `io.run.cancel` granted per-org by admin) |
| `role_system_admin` | All permissions, incl. `io.skip_approval` (Phase 1) + `io.run.rollback` (Phase 3) |
| Per-template `required_role_codes_csv` | Layered on top of base permission — a template can require additional role membership to be invoked |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.storage`](foundation-storage.md) — uploaded file + error-report archive target
- [`foundation.approval`](foundation-approval.md) — optional approval gate between preview and commit
- [`foundation.notifications`](foundation-notifications.md) — "your import is complete" dispatch
- [`foundation.jobs`](foundation-jobs.md) — Phase 2 async-import executor
- [`foundation.presentation`](foundation-presentation.md) — auto-injected Import button + preview FormView + `FileUploadSpecial` reuse
- [`foundation.dataset`](foundation-dataset.md) — safe AST evaluator reused for `formula` validator
- [`foundation.email`](foundation-email.md) — Phase 3 email-attachment imports
- [`foundation.connectors`](foundation-connectors.md) — Phase 3 SFTP / Drive / S3 / Azure Blob connector kinds
- [`foundation.communication`](foundation-communication.md) — optional chatter post on import success
- [`logistics.pricing`](../src/domains/logistics/docs/pricing.md) — first adopter (Phase 1 Feature 05 re-scoped)
- [Foundation Model Naming](foundation-model-naming.md) — `ir.*` engine namespace convention
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-27. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
