<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Storage — Implementation Docs

**Module:** `foundation.storage` (`src/ede/foundation/storage/`)
**Roadmap:** [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md)
**Status:** ✅ Delivered (baseline — pre-roadmap)
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **record-centric document store** that gives any consumer a uniform `storage.document` handle while abstracting away which backend (local filesystem, S3, GCS, Azure Blob, Google Drive, …) actually holds the bytes. Three models — `storage.folder`, `storage.document`, `storage.document_version` — plus a `StorageRouter` service and pluggable connectors (via `foundation.connectors`).
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every domain that handles physical documents (logistics shipments with B/L scans, finance invoices with PDFs, HR with contract scans, CRM with proposals, communication chatter with attachments, …) needs object-store-backed persistence with versioning, folder hierarchy, and pluggable backends. Build the substrate once in foundation; every domain consumes it instead of reinventing blob storage.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End user** — the "Documents" app appears in the app-switcher (seeded via [data/storage_menus.xml](../src/ede/foundation/storage/data/storage_menus.xml)). Users navigate folders, upload documents, and browse version history through the document/folder list + form views.
- **Programmatic consumer (Python)** — use the generic CRUD pipeline (`env.models["storage.document"]`, `env.models["storage.folder"]`, `env.models["storage.document_version"]`); the upload path goes through the HTTP API, which performs the backend write via `StorageRouter` and then creates the rows.
- **HTTP consumer** — `/api/storage/folders/*`, `/api/storage/documents/*`, `/api/storage/documents/{id}/versions/*` cover folder CRUD, document upload/list/get/delete, version listing + per-version download.
- **Integration boundary** — produces: a stable `storage.document` UUID + signed metadata that any module can reference. Consumes: an `ir.connector` (category `storage`) when one is registered, otherwise the built-in local FS backend.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Consumer module]                                foundation.storage
  HTTP / Command                                  ┌──────────────────────┐
  upload bytes + metadata ──────────────────────> │  StorageRouter       │
                                                  │  (chooses backend    │
                                                  │   per organization)  │
                                                  └─────────┬────────────┘
                                          ┌─────────────────┼────────────────────┐
                                          ▼                 ▼                    ▼
                              LocalFilesystemBackend   ir.connector         (future kinds)
                              (default, zero config)   (category=storage)
                                                       Google Drive, S3, …
                                          │                 │
                                          └────────┬────────┘
                                                   ▼
                                       writes new storage.document_version
                                       (append-only); updates
                                       storage.document.current_version
```
Bytes flow through the chosen backend; metadata flows through the three storage models. Old versions are never mutated — a new upload appends a row, never replaces one.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `storage.folder` | Hierarchical filing surface; self-referential `parent_id`; carries owner + organization + icon/color metadata. | [models/folder.py](../src/ede/foundation/storage/models/folder.py) |
| `storage.document` | One logical document with metadata (name, MIME, size, SHA-256 checksum, current-version pointer, storage backend pointer for current version). | [models/document.py](../src/ede/foundation/storage/models/document.py) |
| `storage.document_version` | Immutable per-upload version row carrying `version_number`, `storage_backend` (Enum), `connector_id` (Reference → `ir.connector`), `storage_key`, `storage_object_id`, `file_size`, `checksum`, `change_note`, `uploaded_by`. | [models/document_version.py](../src/ede/foundation/storage/models/document_version.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `StorageRouter` | Resolves the active default storage connector for an organization; falls back to `LocalFilesystemBackend` when no connector is registered. Writes a new version blob and emits the storage pointer (`storage_key` / `storage_object_id`) for the caller to persist. | [services/storage_router.py](../src/ede/foundation/storage/services/storage_router.py) |
| Google Drive connector | First cloud backend implementation, registered through `foundation.connectors`. | [connectors/google_drive.py](../src/ede/foundation/storage/connectors/google_drive.py) |
| `LocalFilesystemBackend` (kernel) | Built-in default backend; reads `STORAGE_LOCAL_ROOT` (auto-detects per-tenant under `~/.local/share/ede/<db>/storage` when empty). | `src/ede/core/adapters/storage/local_fs.py` |
| `DocumentService` (kernel) | Lower-level helper for orchestrating upload / download against any `StorageBackend`. | `src/ede/core/services/storage/document_service.py` |
| `KeyBuilder` (kernel) | Deterministic logical-key construction for stored objects. | `src/ede/core/services/storage/key_builder.py` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ede.create` (`storage.folder`) | `POST /api/storage/folders` | Inserts a folder row; supports `parent_id` for nesting. |
| `ede.update` (`storage.folder`) | `PUT /api/storage/folders/{id}` | Mutates folder metadata. |
| `ede.delete` (`storage.folder`) | `DELETE /api/storage/folders/{id}` | Deletes folder (cascade behavior governed by `on_delete` on `parent_id`). |
| `ede.search` / `ede.read_one` (`storage.folder`) | `GET /api/storage/folders[/{id}]` | List / fetch folders. |
| `ede.create` (`storage.document`) | `POST /api/storage/documents` (multipart upload) | After the `StorageRouter` writes the bytes, creates a `storage.document` and its first `storage.document_version` row. |
| `ede.update` (`storage.document`) | `PUT /api/storage/documents/{id}` | Mutates document metadata (name, folder, owner). |
| `ede.delete` (`storage.document`) | `DELETE /api/storage/documents/{id}` | Pre-delete hook removes storage objects across all versions before the row is destroyed (see Hooks). |
| `ede.search` / `ede.read_one` (`storage.document`) | `GET /api/storage/documents[/{id}]` | List / fetch documents. |
| `ede.create` (`storage.document_version`) | `POST /api/storage/documents/{id}/versions` | Appends a new version (after backend write) and pins it as the document's `current_version`. |
| `ede.search` (`storage.document_version`) | `GET /api/storage/documents/{id}/versions` and download path | List version history; the download endpoint searches for the version row, then streams bytes through the backend. |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.record.created` (`storage.document`) | Generic CRUD `ede.create` succeeds. | Future: virus-scan job, thumbnail/preview job, communication attachment indexer. |
| `ede.record.created` (`storage.document_version`) | Each new upload appends a version. | Future: retention pruner, audit trail. |
| `ede.record.updated` / `ede.record.deleted` | Standard CRUD lifecycle. | Future consumers. |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `GET /api/storage/folders` | List folders. | [api/folders.py](../src/ede/foundation/storage/api/folders.py) |
| `POST /api/storage/folders` | Create folder. | [api/folders.py](../src/ede/foundation/storage/api/folders.py) |
| `GET /api/storage/folders/{folder_id}` | Get folder. | [api/folders.py](../src/ede/foundation/storage/api/folders.py) |
| `PUT /api/storage/folders/{folder_id}` | Update folder. | [api/folders.py](../src/ede/foundation/storage/api/folders.py) |
| `DELETE /api/storage/folders/{folder_id}` | Delete folder. | [api/folders.py](../src/ede/foundation/storage/api/folders.py) |
| `POST /api/storage/documents` | Multipart upload — backend write + create document + first version. | [api/documents.py](../src/ede/foundation/storage/api/documents.py) |
| `GET /api/storage/documents` | List documents. | [api/documents.py](../src/ede/foundation/storage/api/documents.py) |
| `GET /api/storage/documents/{document_id}` | Get document metadata. | [api/documents.py](../src/ede/foundation/storage/api/documents.py) |
| `GET /api/storage/documents/{document_id}/versions` | List version history. | [api/versions.py](../src/ede/foundation/storage/api/versions.py) |
| `POST /api/storage/documents/{document_id}/versions` | Append a new version. | [api/versions.py](../src/ede/foundation/storage/api/versions.py) |
| `GET /api/storage/documents/{document_id}/versions/{version_number}/download` | Stream a specific version's bytes from its backend. | [api/versions.py](../src/ede/foundation/storage/api/versions.py) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.storage.document.delete` | `Document.delete_storage_objects` — iterates the document's `storage.document_version` rows and asks the matching backend to delete each stored object before the parent row is removed. Prevents orphaned bytes on cloud backends. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none — storage rows are CRUD-only; the only state-like progression is `storage.document.current_version` advancing as new `storage.document_version` rows append._
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `storage`
- `ACTIVE_DOMAINS` entry: _not applicable — foundation app_
- Manifest `depends`: `foundation.base`, `foundation.connectors`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `STORAGE_LOCAL_ROOT` | `str` | `""` (auto-detect `~/.local/share/ede/<db_name>/storage`) | `EDE_STORAGE_LOCAL_ROOT` | Root directory for the built-in local filesystem backend. Empty resolves per-tenant under the user's XDG data dir. |
| `STORAGE_MAX_FILE_SIZE_BYTES` | `int` | `52428800` (50 MB) | `EDE_STORAGE_MAX_FILE_SIZE_BYTES` | Hard cap on single-file upload size; enforced at the API boundary (Phase 2 hardening). |
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
| [data/ir.rbac.permission.csv](../src/ede/foundation/storage/data/ir.rbac.permission.csv) | RBAC permission rows for `storage.folder`, `storage.document`, `storage.document_version` (owner-scoped + system-admin variants). |
| [data/storage_menus.xml](../src/ede/foundation/storage/data/storage_menus.xml) | "Documents" app-switcher entry + folder/document leaf menus and their `ir.action` rows. |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 0 | Baseline (pre-roadmap) — models, migrations, HTTP API, `StorageRouter`, Google Drive connector, RBAC seed, Documents menu | ✅ Delivered | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Enh 01 · P1 | Frontend Drive client action — functional MVP (browse / multi-file upload / download / delete / new folder) | ✅ Delivered (browser-verified 2026-06-10) | [Storage Enhancement 01](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md) |
| Enh 01 · P2 | Frontend Drive — three-pane mockup UX parity (FolderTreeNav · DocumentGrid · DocumentPreviewPane, thumbnails, version list, filter rail) | ✅ Delivered (browser-verified 2026-06-10) | [Storage Enhancement 01](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md) |
| Enh 01 · P3 | Frontend Drive — advanced (PDF preview, cross-folder search, move/copy, tags + quota; backend `copy`/`tags`/`usage` routes) | 🟡 Core delivered; residuals pending (Surface B · SSE refresh · RBAC write-gating · FE tests + docs) | [Storage Enhancement 01](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md) |
| Enh 01 · P4 | Frontend Drive — "Documents" section (`foundation.document` Attachable virtual tree in the left rail) | ✅ Delivered (browser-verified 2026-07-10) | [Storage Enhancement 01](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Folder hierarchy with owner + organization scoping | `storage.folder` | [models/folder.py](../src/ede/foundation/storage/models/folder.py), [views/storage_folder_views.xml](../src/ede/foundation/storage/views/storage_folder_views.xml) | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Document metadata (name, MIME, size, SHA-256, current-version pointer) | `storage.document` | [models/document.py](../src/ede/foundation/storage/models/document.py), [views/storage_document_views.xml](../src/ede/foundation/storage/views/storage_document_views.xml), [api/documents.py](../src/ede/foundation/storage/api/documents.py) | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Immutable version history per upload | `storage.document_version` | [models/document_version.py](../src/ede/foundation/storage/models/document_version.py), [api/versions.py](../src/ede/foundation/storage/api/versions.py) | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Backend abstraction (local FS default, connector-routed for cloud) | — | [services/storage_router.py](../src/ede/foundation/storage/services/storage_router.py), [connectors/google_drive.py](../src/ede/foundation/storage/connectors/google_drive.py) | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| HTTP API for upload / download / list / version pin | — | [api/documents.py](../src/ede/foundation/storage/api/documents.py), [api/folders.py](../src/ede/foundation/storage/api/folders.py), [api/versions.py](../src/ede/foundation/storage/api/versions.py) | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Documents app in the app-switcher | — | [data/storage_menus.xml](../src/ede/foundation/storage/data/storage_menus.xml) | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Pre-delete cleanup hook removes backend objects before row destruction | `storage.document` | [models/document.py](../src/ede/foundation/storage/models/document.py) (`@api.on_hook("pre.storage.document.delete")`) | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| RBAC permission seed (owner-scoped + system-admin variants) | — | [data/ir.rbac.permission.csv](../src/ede/foundation/storage/data/ir.rbac.permission.csv) | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Frontend Drive — functional MVP (P1): `storage.drive` client action at `/wc/<org>/documents`; browse, multi-file upload, download, delete, new folder over the Phase-0 API; zero backend changes | — | [frontend/client-actions/storage-drive.tsx](../src/ede/foundation/storage/frontend/client-actions/storage-drive.tsx), [frontend/components/DriveBrowser.tsx](../src/ede/foundation/storage/frontend/components/DriveBrowser.tsx), [frontend/services/StorageService.ts](../src/ede/foundation/storage/frontend/services/StorageService.ts), [frontend/components/useStorage.ts](../src/ede/foundation/storage/frontend/components/useStorage.ts) | [Storage Enhancement 01 · P1](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md) |
| Frontend Drive — three-pane mockup UX parity (P2): FolderTreeNav (left) · DocumentGrid grid/list + size slider (center) · DocumentPreviewPane with image/text preview, metadata, version history (right); thumbnails, star toggle, multi-select bar, file-type + Library filters; Tailwind theme tokens only | — | [frontend/components/FolderTreeNav.tsx](../src/ede/foundation/storage/frontend/components/FolderTreeNav.tsx), [frontend/components/DocumentGrid.tsx](../src/ede/foundation/storage/frontend/components/DocumentGrid.tsx), [frontend/components/DocumentPreviewPane.tsx](../src/ede/foundation/storage/frontend/components/DocumentPreviewPane.tsx), [frontend/components/Breadcrumb.tsx](../src/ede/foundation/storage/frontend/components/Breadcrumb.tsx) | [Storage Enhancement 01 · P2](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md) |
| Frontend Drive — "Documents" section (P4): `foundation.document` Attachable virtual tree in the left rail — one folder per Attachable model (Leads · Inquiries · Opportunities · Quotations · Handovers, per-model icons + right-aligned counts) → all records as folders (empty included, capped 500 most-recent) → attached files; upload-to-record inside a record folder; consumes the shipped `/api/document/dms/virtual/*` (all-records contract added to `virtual_tree.py`) | — | [frontend/services/DocumentTreeService.ts](../src/ede/foundation/storage/frontend/services/DocumentTreeService.ts), [frontend/components/useDocumentTree.ts](../src/ede/foundation/storage/frontend/components/useDocumentTree.ts), [frontend/components/FolderTreeNav.tsx](../src/ede/foundation/storage/frontend/components/FolderTreeNav.tsx), [frontend/components/DriveBrowser.tsx](../src/ede/foundation/storage/frontend/components/DriveBrowser.tsx), [document/api/virtual_tree.py](../src/ede/foundation/document/api/virtual_tree.py) | [Storage Enhancement 01 · P4](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Virus scan on upload — no AV bridge; uploaded bytes hit the backend unscanned. | 🟠 High | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Thumbnail / preview generation — no resized image or PDF-page preview produced. | 🟠 High | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Signed-URL download — current download streams through the API server; no short-lived pre-signed URL for direct backend GET. | 🟠 High | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Folder-level RBAC — owner/organization scoping exists on rows, but no per-folder ACL. Belongs in `foundation.security` Phase 6 (record-level scopes on folder tree). | 🟠 High | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Document retention policy — no auto-purge of versions older than N days; storage grows unbounded. | 🟢 Low | [roadmap/foundation/storage/README.md](../roadmap/foundation/storage/README.md) |
| Drive P3 residuals — **Surface B** (`open_record_documents` record-scoped dialog; deferred until a first consumer model places the button), **SSE refresh** (storage CRUD events not forwarded on the client SSE channel; grid can't auto-refetch on others' uploads), **RBAC write-gating** (hiding Upload/Delete/Move/Copy without `storage.document.write` needs a frontend permission API for client-action components), **FE component tests + docs sync**. Core P3 features (PDF preview via native blob `<iframe>`, cross-folder search, drag/bulk move + copy, tag facet, quota widget, backend `copy`/`tags`/`usage` routes) are delivered build-green. | 🟡 | [Storage Enhancement 01 · P3](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- The actual model key is `storage.document_version` (single underscore between `document` and `version`), **not** `storage.document.version`. The roadmap header copy may read "document.version" loosely; the registered key uses an underscore.
- Backend selection is **per organization**, not global. `StorageRouter` looks up the active default `ir.connector` (category `storage`) for the current `Env.tenant_id` / organization scope and falls back to local FS only when none is registered.
- Old `storage.document_version` rows are immutable. Never mutate `storage_key` or `storage_object_id` on an existing version — append a new one. The `pre.storage.document.delete` hook depends on this invariant when cleaning up backend objects.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Baseline migrations under `src/ede/foundation/storage/migrations/versions/` create the three tables.
- No schema changes pending; future virus-scan / thumbnail / retention work will introduce new tables or columns under their own roadmap phases.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `rbac.role_internal_user` | Read own documents (`owner_id == $principal.user_id`), upload documents, update own document metadata, delete own documents, read own version history, full folder CRUD (except folder delete). |
| `rbac.role_system_admin` | Read / update / delete any document, read full version history, delete folders. |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Connectors](foundation-connectors.md) — backend plug-board; new cloud-storage kinds (S3, GCS, Azure Blob, …) register here.
- [Foundation Communication](foundation-communication.md) — Phase 3 plans to attach `storage.document` rows to chatter messages and activity completions.
- [Storage Module (legacy hand-authored guide)](17-storage-module.md) — kernel-side `StorageBackend` ABC, `LocalFilesystemBackend`, `DocumentService`, `KeyBuilder` reference.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-07-10 (Storage Enhancement 01 — **Frontend Drive Phase 4 ✅ Delivered, browser-verified**. The "Documents" section — the `foundation.document` Attachable virtual tree — now renders in the Drive's left rail: one folder per Attachable model (Leads · Inquiries · Opportunities · Quotations · Handovers) with per-model icons + right-aligned counts, drilling a category shows all RBAC-readable records as folders (empty included, capped 500 most-recent), and a record folder lists its attached files with an upload-to-record action. Consumes the shipped `/api/document/dms/virtual/*` — the only backend touch was the all-records contract in `document/api/virtual_tree.py` (`_fetch_all_consumer_records` + `_RECORDS_FOLDER_CAP=500`). Shipped in commits `543240cf` + `625c2116` (+ polish). Enhancement stays 🟡 overall: P3 residuals still pending (Surface B · SSE refresh · RBAC write-gating · FE tests + docs). Previous sync: 2026-06-10 (Storage Enhancement 01 — **Frontend Drive Phases 1–2 ✅ Delivered, browser-verified**. P1 functional MVP (browse / multi-file upload / download / delete / new folder at `/wc/<org>/documents`) + P2 three-pane mockup UX parity (FolderTreeNav · DocumentGrid · DocumentPreviewPane, thumbnails, version list, file-type/Library filter rail) confirmed working end-to-end in the browser. Enhancement stays 🟡 overall: P3 core delivered build-green with residuals pending (Surface B record-scoped dialog · SSE push refresh · RBAC write-gating · FE component tests + docs); P4 "Documents" Attachable virtual tree 🔴 Not Started. Implementation notes vs the original draft: the Drive resolves at `/wc/<org>/documents` (seeded action path `documents`, not `storage-drive`); PDF preview ships via native blob `<iframe>` (no pdfjs dep); signed-URL preview dropped as unnecessary; all styling is Tailwind theme tokens per CLAUDE.md (no `_storage.scss`). No new configuration introduced — frontend wiring over the Phase-0 substrate; demo-data exempt. Previous sync: 2026-06-04 (Storage Enhancement 01 — **Frontend Drive (DMS) as a Client Action — 🔴 Not Started** authored at [`roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md`](../roadmap/foundation/storage/enhancements/01-frontend-drive-client-action.md). Closes the substrate↔frontend gap that's been open since `foundation.storage` Phase 0 shipped 8 months ago — the new `src/frontend` workspace client has zero DMS surface even though the backend is complete. **Pattern**: two surfaces, one plugin — menu-attached `open_storage_drive` opens a full-page Drive at `/wc/<org>/storage-drive` (ClientActionPage host, URL-pushed, refresh-safe); button-attached `open_record_documents` opens a Radix Dialog scoped to a record's documents (ClientActionDialogHost, ephemeral, single-instance). Same plugin lifecycle (`onResolve` → `onMount` → `onCommand` / `onRefresh` → `onClose`) per Enhancement 08's contract; layout XML lives as `<view type="clientaction">` rows registered through the existing presentation view-registry. **Components**: `DriveBrowser` (top orchestrator) composes `FolderTreeNav` (left rail, lazy-load on expand) + `DocumentGrid` (main area, list/grid toggle) + `DocumentPreviewPane` (right rail, MIME-routed renderer). `PdfPreview` wraps `pdfjs-dist` behind `React.lazy()` so the ~600KB PDF chunk only loads when the user opens a PDF in the preview pane (zero impact on first paint, same lazy-bulk discipline as Enhancement 08's R3F equipment viewer). `UploadDropZone` (page-level drag-over overlay) + `UploadDialog` (modal alternative) share the same upload queue + per-file progress + retry-on-failure flow. `VersionHistoryDialog` lists `storage.document_version` rows per document with download-per-version. `SearchBox` (cmdk-driven across-folder search) maps to a new `storage.document.search` backend command. **Backend additions** (small): preview-signed-URL endpoint + per-connector `sign_download_url(storage_key, ttl_seconds)` extension method; `storage.document.search` command + route; `storage.document.move` + `storage.document.copy` commands. **Cross-module composition**: any module wanting a "Documents" button on its form view drops a one-line `<extend ref="<their_form>"><xpath>...<button name="btn_documents" command="open_record_documents" context='{"scoped_to_record": true}'/></xpath></extend>` patch — no patching the storage module. **Tests planned**: 46 cases (24 pytest covering new commands + RBAC gates + connector preview-URL behavior; 22 vitest covering DriveBrowser composition + upload queue + TanStack cache invalidation). New developer guide planned at `docs/foundation-storage-frontend-guide.md`. **Depends on**: [foundation.presentation Enhancement 08 — Client-Action Lifecycle + Registry](../roadmap/foundation/presentation/enhancements/08-client-action-lifecycle-and-registry.md) (the substrate this enhancement is the second adopter of). Implementation queued behind Enhancement 08. Known Gaps gains the matching 🔴 row. Previous sync: 2026-05-14. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
