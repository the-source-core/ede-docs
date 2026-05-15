# Document Storage

Upload, version, and retrieve files against any record. The default backend is the local filesystem; swap to Google Drive (or any S3-compatible store) via the [Connectors](connectors.md) framework — no code changes in your domain.

```python
# Attach a file to a blog post
doc = env.models["storage.document"].create({
    "name": "cover.png",
    "owner_model": "blog.post",
    "owner_record_uuid": post.id,
    "content_type": "image/png",
    "bytes": open("/path/to/cover.png", "rb").read(),
})

# Later, fetch its bytes back
content = doc.read_bytes()
```

EDE writes the bytes through `StorageRouter` to whichever backend the active organization has configured — local FS, Google Drive, or any custom backend you register.

---

## What you get

-   **`storage.document`** + **`storage.document_version`** models — every upload becomes a versioned record.
-   **`storage.folder`** model — optional organisational tree if you want hierarchy.
-   **`StorageBackend`** ABC — pluggable backend contract.
-   **`LocalFilesystemBackend`** — ships by default; writes under the configured data dir.
-   **Google Drive backend** — register a connector of kind `gdrive` and point the storage router at it.
-   **`DocumentService`** — high-level service: `upload()`, `read()`, `new_version()`, `delete()`, `list_versions()`.
-   **`StorageRouter`** — resolves which backend to use for a given org / model.
-   **`KeyBuilder`** — deterministic per-record key paths so storage layout is predictable.
-   **HTTP API** — `POST /api/storage/upload`, `GET /api/storage/download/{record_uuid}`, plus listing endpoints.
-   **Documents app** — a stand-alone Documents app in the web client app-switcher, plus an inline attachments panel on any record that opts in.

## How to use it

### Upload a file from an HTTP route

```python
@api.route_config(prefix="/api/blog")
class BlogController(RouteController):

    @api.route("/post/{post_uuid}/attach", methods=["POST"])
    async def attach(self, request, env, post_uuid: str):
        body = await request.body()
        doc = env.models["storage.document"].create({
            "name": request.headers["x-filename"],
            "owner_model": "blog.post",
            "owner_record_uuid": post_uuid,
            "content_type": request.headers["content-type"],
            "bytes": body,
        })
        return {"record_uuid": doc.id, "name": doc.name}
```

### Add a new version of an existing document

```python
from ede.foundation.storage.services.storage_router import StorageRouter

doc = env.models["storage.document"].browse(doc_uuid)
StorageRouter.from_env(env).new_version(doc, new_bytes, note="re-rendered")
```

The previous version is preserved; downloading the document always returns the latest version unless you specify a `version_number`.

### List documents attached to a record

```python
docs = env.models["storage.document"].search([
    ("owner_model", "=", "blog.post"),
    ("owner_record_uuid", "=", post.id),
])
```

### Swap the backend

Storage backends are selected via the connector framework. Create a connector record of kind `gdrive` (or your custom kind) on the organization, mark it the default storage backend, and `StorageRouter` will route new uploads there. Existing documents stay where they were written.

```python
env.models["ir.connector"].create({
    "name": "Company Google Drive",
    "category": "storage",
    "kind": "gdrive",
    "organization_id": org.id,
    "is_default": True,
    "config": {"folder_id": "0AHj…", "credentials_id": gdrive_creds.id},
})
```

Existing records continue to read from the local FS; new uploads land on Drive.

### Show attachments on a record's form view

Add the chatter / attachments slot to the form view DSL:

```xml
<chatter attachments="true"/>
```

The React web client renders an upload box + thumbnail list automatically.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `STORAGE_LOCAL_ROOT` | `<data_dir>/storage` | Where `LocalFilesystemBackend` writes files. |
| `STORAGE_MAX_UPLOAD_BYTES` | `25 * 1024 * 1024` | Single-upload size limit at the HTTP layer. |
| `STORAGE_DEFAULT_BACKEND_KIND` | `local` | Backend used when no organization-level connector is set as default. |

## How it composes with other features

-   **[Connectors](connectors.md)** — third-party backends (Google Drive, S3) plug in here.
-   **[Communication (Chatter)](communication.md)** — attaches files to chatter messages on any model.

## Reference

| Concept | Where it lives |
|---|---|
| `StorageBackend` ABC | `src/ede/core/services/storage/interfaces.py` |
| `LocalFilesystemBackend` | `src/ede/core/adapters/storage/local_fs.py` |
| `DocumentService` | `src/ede/core/services/storage/document_service.py` |
| `KeyBuilder` | `src/ede/core/services/storage/key_builder.py` |
| `StorageRouter` | `src/ede/foundation/storage/services/storage_router.py` |
| Models | `src/ede/foundation/storage/models/` |
| Full reference | [Storage Module architecture guide](../17-storage-module.md) |
