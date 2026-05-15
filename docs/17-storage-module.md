# EDE Framework — Storage Module (`foundation.storage`)

## Overview

`foundation.storage` provides document management with a **built-in local filesystem adapter**
and a **pluggable connector backend** for cloud storage (Google Drive, AWS S3, etc.).

**Key principle:** Local filesystem is always available — zero admin configuration. Cloud backends
are configured as `ir.connector` instances in Settings → Integrations → Connectors.

```
src/ede/core/services/storage/    ← Abstract contracts (StorageBackend ABC, DTOs, errors)
src/ede/core/adapters/storage/    ← Built-in local filesystem adapter
src/ede/foundation/storage/       ← Foundation module: models, API, StorageRouter
```

Activation (requires `"connectors"` before `"storage"`):

```python
# src/ede/foundation/settings.py
ACTIVE_MODULES = ["base", "auth", "presentation", "connectors", "storage"]
```

---

## Core Contracts (`src/ede/core/services/storage/`)

### `StorageBackend` ABC

All storage backends implement this interface.

```python
from ede.core.services.storage.interfaces import StorageBackend
from ede.core.services.storage.dto import ObjectRef, StreamResult
from ede.core.services.storage.capabilities import StorageCapabilities

class MyBackend(StorageBackend):
    @property
    def capabilities(self) -> StorageCapabilities:
        return StorageCapabilities(directories=True, atomic_put=True, integrity_checksum=True)

    def put(self, key: str, content: bytes, metadata: dict | None = None) -> ObjectRef:
        ...

    def get(self, key: str) -> StreamResult:
        ...

    def delete(self, key: str) -> None:
        ...

    def exists(self, key: str) -> bool:
        ...

    def list(self, prefix: str = "", *, max_results: int = 1000) -> list[ObjectRef]:
        ...
```

**`StorageCapabilities`** — frozen dataclass declaring what the backend supports:

| Field | Type | Meaning |
|-------|------|---------|
| `directories` | `bool` | Supports directory-based key organisation |
| `atomic_put` | `bool` | Write is atomic (no partial reads) |
| `resumable_upload` | `bool` | Supports resumable/chunked uploads |
| `integrity_checksum` | `bool` | Returns SHA-256 checksum on write |
| `native_sharing` | `bool` | Can generate shareable links |

---

### `ObjectRef` — Immutable File Reference

Returned by `put()` and `list()`.

```python
from ede.core.services.storage.dto import ObjectRef

ref = ObjectRef(
    backend_key="local",
    key="system/shared/storage.document/uuid/v1/report.pdf",
    object_id="",           # native backend ID (e.g. S3 ETag, GDrive file ID)
    checksum="sha256:...",
    size=102400,
    mime_type="application/pdf",
)
```

---

### `StreamResult`

Returned by `get()`.

```python
result = backend.get("path/to/file")
content = result.stream.read()         # file-like object
result.stream.close()
```

---

### `KeyBuilder`

Utility for building consistent, safe logical storage keys.

```python
from ede.core.services.storage.key_builder import KeyBuilder

kb = KeyBuilder()

# Build a full key — segments joined with "/"
key = kb.join("system", "org-uuid", "storage.document", "doc-uuid", "v1", "report.pdf")
# → "system/org-uuid/storage.document/doc-uuid/v1/report.pdf"

# Build a prefix for listing
prefix = kb.prefix("system", "org-uuid", "storage.document", "doc-uuid")
# → "system/org-uuid/storage.document/doc-uuid/"
```

**Segment rules:** Empty strings, `.`, `..`, and segments containing `/` are rejected.

**Standard key pattern:** `{tenant_id}/{org_uuid}/storage.document/{doc_uuid}/v{N}/{filename}`

---

### `DocumentService`

Ergonomic wrapper around `StorageBackend` + `KeyBuilder`.

```python
from ede.core.services.storage.document_service import DocumentService
from ede.core.services.storage.key_builder import KeyBuilder

svc = DocumentService(backend=my_backend, key_builder=KeyBuilder())

# Upload
ref = svc.upload(key="tenant/org/storage.document/uuid/v1/file.pdf",
                 content=file_bytes, metadata={"mime_type": "application/pdf"})

# Download
stream_result = svc.download(key="tenant/org/storage.document/uuid/v1/file.pdf")
content = stream_result.stream.read()

# Delete
svc.delete(key="tenant/org/storage.document/uuid/v1/file.pdf")

# List under a prefix
refs = svc.list_under("tenant/org/storage.document/uuid/")
```

---

### `StorageError` Taxonomy

```
StorageError
├── StorageNotFoundError      — key does not exist
├── StorageAuthError          — authentication failure
├── StoragePermissionError    — access denied
├── StorageConflictError      — key already exists (when overwrite not allowed)
├── StorageTransientError     — temporary backend failure (safe to retry)
├── StorageNotSupportedError  — operation not supported (e.g. directory listing on S3)
└── StorageIntegrityError     — checksum mismatch after write
```

All errors carry an optional `StorageErrorContext` with `backend_key`, `key`, and `operation`:

```python
try:
    svc.download(key="missing/file")
except StorageNotFoundError as exc:
    print(exc.context.key)       # "missing/file"
    print(exc.context.operation) # "get"
```

---

## Built-in Local Filesystem Adapter

`src/ede/core/adapters/storage/local_fs.py`

Always available. No admin configuration. Configured via settings:

```python
# src/ede/foundation/settings.py
STORAGE_LOCAL_ROOT: str = "/var/ede/storage"  # root directory for all stored files
```

**Capabilities:** `directories=True`, `atomic_put=True`, `integrity_checksum=True`

**Atomic write:** Uses `tempfile.mkstemp` + `os.replace()` — no partial reads possible.

**Checksum:** SHA-256 computed during write, returned in `ObjectRef.checksum`.

**Path traversal protection:** Rejects keys containing `..` or starting with `/`.

```python
from ede.core.adapters.storage.local_fs import LocalFilesystemBackend
from ede.foundation.settings import settings

# Factory (reads STORAGE_LOCAL_ROOT from settings)
backend = LocalFilesystemBackend.from_settings(settings)

# Or directly
backend = LocalFilesystemBackend(root="/var/ede/storage")
```

---

## `StorageRouter`

`src/ede/foundation/storage/services/storage_router.py`

Determines which backend to use at runtime. Stateless — created once per request.

**Upload routing:**
1. Is there an active default `ir.connector` for `category="storage"` in this org? → use it
2. Otherwise → `LocalFilesystemBackend` (always available)

**Download routing:**
- `connector_id` is set on the document/version record → load that connector → use its backend
- `connector_id` is null/empty → `LocalFilesystemBackend`

```python
from ede.foundation.storage.services.storage_router import StorageRouter

router = StorageRouter()

# For upload — returns (backend, backend_key_str, connector_uuid_or_None)
backend, backend_key, connector_uuid = router.get_backend_for_upload(org_id, env)

# For download/delete — returns the correct backend for an existing document
backend = router.get_backend_for_document(document_record, env)

# For a specific version record
backend = router.get_backend_for_version(version_record, env)
```

---

## Models

### `storage.document`

Represents a stored document and its current version.

| Field | Type | Notes |
|-------|------|-------|
| `name` | `Char(255)` | Human-readable name. Required. |
| `original_filename` | `Char(255)` | Original upload filename. Required. |
| `content_type` | `Char(120)` | MIME type. |
| `file_size` | `Integer` | File size in bytes. |
| `checksum` | `Char(64)` | SHA-256 checksum. |
| `storage_backend` | `Enum` | `local` / `google_drive` / `aws_s3` / `azure_blob` / `gcs`. |
| `connector_id` | `Reference(ir.connector)` | `null` = local FS. |
| `storage_key` | `Char(1024)` | Logical key for the current version. |
| `storage_object_id` | `Char(1024)` | Backend-native ID (e.g. S3 ETag). |
| `owner_id` | `Reference(res.user)` | Uploader. |
| `organization_id` | `Reference(res.organization)` | Owning org. |
| `description` | `Char(multi_line)` | Optional description. |
| `tags` | `Char(512)` | Comma-separated tags. |
| `is_active` | `Boolean` | Soft-delete flag. Default: `True`. |
| `version_count` | `Integer` | Total versions stored. Default: `1`. |
| `current_version` | `Integer` | Current version number. Default: `1`. |
| `versions` | `One2Many(storage.document_version)` | All version records. |

**Pre-delete hook** (`pre.storage.document.delete`): Attempts to delete all stored version objects
from the backend. Errors are logged but do **not** veto the deletion.

---

### `storage.document_version`

Records each uploaded version of a document.

| Field | Type | Notes |
|-------|------|-------|
| `document_id` | `Reference(storage.document, cascade)` | Parent document. Required. |
| `version_number` | `Integer` | Monotonic version counter. Required. |
| `original_filename` | `Char(255)` | Filename for this version. Required. |
| `content_type` | `Char(120)` | MIME type. |
| `file_size` | `Integer` | File size in bytes. |
| `checksum` | `Char(64)` | SHA-256 checksum. |
| `storage_backend` | `Enum` | Same enum as `storage.document`. |
| `connector_id` | `Reference(ir.connector)` | Backend used for this version. |
| `storage_key` | `Char(1024)` | Logical storage key. Required. |
| `storage_object_id` | `Char(1024)` | Backend-native ID. |
| `change_note` | `Char(512)` | Optional note about this version. |
| `uploaded_by` | `Reference(res.user)` | Who uploaded this version. |

---

## REST API

### `DocumentController` — `/api/storage`

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/storage/documents` | Upload a new document |
| `GET` | `/api/storage/documents` | List documents |
| `GET` | `/api/storage/documents/{id}` | Get document metadata |
| `GET` | `/api/storage/documents/{id}/download` | Download file (binary) |
| `DELETE` | `/api/storage/documents/{id}` | Delete document + all stored versions |
| `PUT` | `/api/storage/documents/{id}` | Update metadata (name, description, tags) |

### `VersionController` — `/api/storage`

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/storage/documents/{id}/versions` | List all versions |
| `POST` | `/api/storage/documents/{id}/versions` | Upload a new version |
| `GET` | `/api/storage/documents/{id}/versions/{n}/download` | Download a specific version |

---

## Upload Modes

Both the initial upload and version upload accept two content modes:

### Binary body (preferred)

```bash
curl -X POST http://localhost:8000/api/storage/documents \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/pdf" \
  --data-binary @invoice.pdf \
  -G -d "filename=invoice.pdf" -d "name=Q1 Invoice"
```

Any non-JSON `Content-Type` sends bytes as `raw_body` to the handler.

### Base64 JSON

```bash
curl -X POST http://localhost:8000/api/storage/documents \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Q1 Invoice","filename":"invoice.pdf","content_b64":"<base64>","content_type":"application/pdf"}'
```

**File size limit:** Enforced in both upload endpoints. Default: 50 MB.

```python
# src/ede/foundation/settings.py
STORAGE_MAX_FILE_SIZE_BYTES: int = 52428800   # 50 MB
```

---

## Settings

```python
# src/ede/foundation/settings.py

STORAGE_LOCAL_ROOT: str = "/var/ede/storage"
# Root directory for the built-in local filesystem backend.
# Created automatically on first upload if it does not exist.

STORAGE_MAX_FILE_SIZE_BYTES: int = 52428800
# Maximum allowed file size in bytes (50 MB). Enforced on upload and version upload.
```

---

## Binary Download

Download endpoints use `request_type="http"` and return a `body_base64` payload:

```python
@api.route("/documents/{document_id}/download", methods=["GET"], auth="user", request_type="http")
def download(self, document_id: str) -> Dict[str, Any]:
    ...
    return {
        "status_code": 200,
        "headers": {"Content-Disposition": f'attachment; filename="{filename}"'},
        "content_type": content_type,
        "body_base64": base64.b64encode(content).decode("ascii"),
    }
```

The `FastApiHttpAdapter._handle_request_dispatcher_http` dispatcher decodes `body_base64` and
returns a proper binary `Response` with the correct headers.

---

## Menu Placement

Documents lives under **Applications** menu as a top-level app entry.

```xml
<!-- src/ede/foundation/storage/data/storage_menus.xml -->
<ede_data>
  <record model="ir.menu" id="menu_storage_root">
    <field name="name">Documents</field>
    <field name="category">application</field>
    <field name="sequence">20</field>
  </record>
</ede_data>
```

---

## Adding a Cloud Storage Provider

See [16-connector-framework.md](16-connector-framework.md) for the full pattern.

**Storage-specific contract:** A storage category connector provider must implement
`as_storage_backend() -> StorageBackend` in addition to the base `ConnectorProvider` interface.
`StorageRouter` calls this method to obtain the backend instance.

```python
class GoogleDriveStorageProvider(ConnectorProvider):
    category      = "storage"
    provider_type = "google_drive"
    display_name  = "Google Drive"

    def as_storage_backend(self) -> StorageBackend:
        return GoogleDriveBackend(credentials=self._credentials, folder_id=self._folder_id)
```

---

## Running Migrations

No migrations are generated automatically. After activation, run:

```bash
ede migrate generate -m "storage_initial" --app storage --config ede.conf
ede migrate generate -m "connectors_initial" --app connectors --config ede.conf
ede migrate upgrade --config ede.conf
```
