# EDE Framework — Connector Framework

## Overview

The Connector Framework is a service-agnostic architecture for managing pluggable external service
integrations. It lets admins configure external providers (cloud storage, email, messaging) through
the UI without any code changes.

**Three concepts:**

| Concept | What it is | Example |
|---------|-----------|---------|
| **Service category** | A capability type | `storage`, `email`, `messaging` |
| **Connector provider** | A specific external implementation | `google_drive`, `aws_s3`, `gmail` |
| **Connector instance** (`ir.connector`) | Admin-configured DB record | "Acme's Google Drive" |

**Built-in adapters** (local filesystem, SMTP) are always on — they are **not** connectors and require
no admin configuration.

---

## Architecture

```
src/ede/core/connectors/          ← Framework kernel (pure interfaces, no domain imports)
src/ede/foundation/connectors/    ← ir.connector model + admin UI
```

---

## Layer 1: `src/ede/core/connectors/` — Framework Kernel

### `interfaces.py` — `ConnectorProvider` ABC

All connector provider implementations extend this base class.

```python
from ede.core.connectors.interfaces import ConnectorProvider
from ede.core.connectors.dto import ConnectionTestResult

class MyStorageProvider(ConnectorProvider):
    category     = "storage"
    provider_type = "my_service"
    display_name  = "My Service"

    @classmethod
    def from_config(cls, config: dict) -> "MyStorageProvider":
        """Instantiate from the assembled config dict (built from ir.connector.param rows)."""
        return cls(api_key=config["api_key"])

    def test_connection(self, connector_uuid: str = "") -> ConnectionTestResult:
        """Verify the service is reachable. Called by admin UI 'Test' button."""
        try:
            self._client.ping()
            return ConnectionTestResult(success=True, message="Connected")
        except Exception as exc:
            return ConnectionTestResult(success=False, message=str(exc))
```

**Required class attributes:**

| Attribute | Type | Purpose |
|-----------|------|---------|
| `category` | `str` | Service category key (`"storage"`, `"email"`, `"messaging"`) |
| `provider_type` | `str` | Unique provider identifier (`"google_drive"`, `"aws_s3"`) |
| `display_name` | `str` | Human-readable name shown in the UI |

**Optional methods** (override only if needed):

| Method | Signature | Purpose |
|--------|-----------|---------|
| `config_schema()` | `cls → list[dict]` | Declares expected params. Used by admin UI to pre-populate `ir.connector.param` rows. Each dict: `{key, label, required, secret}`. |
| `exchange_oauth_code()` | `cls, code, config → dict` | Exchanges an OAuth authorization code for tokens. Returns flat `{key: value}` pairs to upsert as params. Default: `{}` (no OAuth). |
| `normalize_import_json()` | `cls, json_data → dict` | Called before `import_config` flattens a JSON blob. Lets providers unwrap their own wrapper format (e.g. Google's `installed/web` wrapper). Default: identity. |

```python
class GoogleDriveStorageProvider(ConnectorProvider):
    category      = "storage"
    provider_type = "google_drive"
    display_name  = "Google Drive"

    @classmethod
    def config_schema(cls) -> list[dict]:
        return [
            {"key": "client_id",     "label": "Client ID",     "required": True,  "secret": False},
            {"key": "client_secret", "label": "Client Secret", "required": True,  "secret": True},
            {"key": "refresh_token", "label": "Refresh Token", "required": False, "secret": True},
        ]

    @classmethod
    def exchange_oauth_code(cls, code: str, config: dict) -> dict:
        """Exchange OAuth code for refresh_token and return as param key→value."""
        # ... call Google token endpoint ...
        return {"refresh_token": "<token from Google>"}

    @classmethod
    def normalize_import_json(cls, json_data: dict) -> dict:
        """Unwrap Google's installed/web wrapper if present."""
        return json_data.get("installed") or json_data.get("web") or json_data
```

---

### `registry.py` — `ConnectorRegistry`

Global singleton mapping `(category, provider_type)` → provider class.

```python
from ede.core.connectors.registry import connector_registry

# Register a provider (usually in the module's __init__.py)
connector_registry.register(MyStorageProvider)

# Look up a provider class
cls = connector_registry.get("storage", "my_service")
provider = cls.from_config(connector_record["config"])

# List all providers for a category
providers = connector_registry.list_for_category("storage")

# Check if registered
if connector_registry.is_registered("email", "gmail"):
    ...
```

**Registration pattern** — call `register()` when the module is imported:

```python
# src/ede/foundation/storage/connectors/__init__.py
from ede.core.connectors.registry import connector_registry
from .google_drive import GoogleDriveStorageProvider

connector_registry.register(GoogleDriveStorageProvider)
```

---

### `dto.py` — `ConnectionTestResult`

Returned by `ConnectorProvider.test_connection()`.

```python
from ede.core.connectors.dto import ConnectionTestResult

result = ConnectionTestResult(
    success=True,
    message="Connected successfully",
    details={"latency_ms": 42}          # optional extra info
)
```

---

### `errors.py` — Error Taxonomy

```
ConnectorError
├── ConnectorNotFoundError     — (category, provider_type) not in registry
├── ConnectorConfigError       — missing/invalid config fields
├── ConnectionFailedError      — network or auth error during test_connection
└── ConnectorNotSupportedError — operation not supported by this provider
```

---

## Layer 2: `src/ede/foundation/connectors/` — Admin Module

Activated via `ACTIVE_MODULES`. Must appear before modules that depend on it:

```python
# src/ede/foundation/settings.py
ACTIVE_MODULES = ["base", "auth", "presentation", "connectors", "storage"]
```

### `ir.connector` Model

Admin-configured connector instance for a specific organization.

| Field | Type | Notes |
|-------|------|-------|
| `name` | `Char(255)` | Display name. Required. |
| `category` | `Enum` | `storage` / `email` / `messaging`. Required. |
| `provider_type` | `Char(64)` | Provider identifier. Required. |
| `config` | `JSON` (legacy) | Kept for backward compatibility. Use `param_ids` instead. |
| `param_ids` | `One2Many(ir.connector.param)` | Primary config source. Key-value params assembled into a nested dict for `from_config()`. |
| `organization_id` | `Reference(res.organization)` | Owning organization. |
| `is_enabled` | `Boolean` | Admin toggle. Default: `False`. |
| `is_default` | `Boolean` | Default connector for this category + org. Default: `False`. |
| `status` | `Enum` | `draft` / `connected` / `error`. Default: `draft`. |
| `status_message` | `Char(512)` | Last test result message. |
| `last_tested_at` | `DateTime` | Timestamp of last `test_connection` call. |

**Business rules:**
- Only one `ir.connector` per `(category, organization_id)` may have `is_default=True`. Enforced by
  `pre.ir.connector.create` and `pre.ir.connector.update` hooks (existing default is cleared).

**Domain commands:**

| Command | Payload | Behaviour |
|---------|---------|-----------|
| `ir.connector.test_connection` | `{}` | Calls provider `test_connection()`, updates `status` + `status_message` + `last_tested_at` |
| `ir.connector.activate` | `{}` | Sets `is_enabled=True` (requires `status=connected`) |
| `ir.connector.deactivate` | `{}` | Sets `is_enabled=False` |
| `ir.connector.import_config` | `{"json_data": {...}}` or file upload payload | Flattens JSON into `ir.connector.param` rows (replaces existing). Provider's `normalize_import_json()` runs first if defined. Also accepts `{"file_content": "<base64>", "file_name": "...", "file_mime": "..."}` from a `special="file_upload"` button. |
| `ir.connector.get_config` | `{}` | Assembles config dict from `param_ids` rows in sequence order. Falls back to legacy `config` JSON field if no params exist. Returns `{"success": true, "config": {...}}`. |

---

### `ir.connector.param` Model

Key-value configuration parameter for an `ir.connector` record. The full set of params is assembled
into the nested dict passed to `ConnectorProvider.from_config()`.

| Field | Type | Notes |
|-------|------|-------|
| `connector_id` | `Reference(ir.connector, cascade)` | Parent connector. Required. |
| `key` | `Char(255)` | Dot-notation config key, e.g. `installed.client_id`. Required. |
| `value` | `Char(multi_line)` | Text value (used when `value_type=text`). |
| `file_id` | `Reference(storage.document, set null)` | Uploaded file reference (used when `value_type=file`). |
| `value_type` | `Enum` | `text` (default) / `file`. |
| `sequence` | `Integer` | Processing and display order. Default: `10`. |

**Config assembly** — when `ir.connector.get_config` is called:
1. Params are fetched ordered by `sequence asc`.
2. `text` params: value string used directly.
3. `file` params: file content read from `StorageRouter` as UTF-8 string.
4. The flat `[{key, value}, ...]` list is passed through `unflatten_params()` to reconstruct the
   nested dict (dot-notation keys become nested dicts).
5. Falls back to the legacy `config` JSON field when no params exist.

---

### `ConnectorService`

`src/ede/foundation/connectors/services/connector_service.py`

Stateless helper used by controllers and other services.

```python
from ede.foundation.connectors.services.connector_service import ConnectorService

# Test a connector record and update its status fields
result = ConnectorService.test_connection(connector_record)  # → ConnectionTestResult

# Instantiate the provider from a record (uses ConnectorRegistry)
provider = ConnectorService.instantiate_provider(connector_record)

# Get the active default connector for a category + org (returns record dict or None)
record = ConnectorService.get_default_for_category("storage", org_id, env)

# List registered provider types for a category
types = ConnectorService.list_available_provider_types("storage")
# → [{"provider_type": "google_drive", "display_name": "Google Drive", "category": "storage"}, ...]
```

---

### REST API

All endpoints require `auth="user"` and are prefixed `/api/connectors`.

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/connectors` | List connectors (filter by `category`, `organization_id`) |
| `POST` | `/api/connectors` | Create a connector |
| `GET` | `/api/connectors/{id}` | Get a connector |
| `PUT` | `/api/connectors/{id}` | Update a connector |
| `DELETE` | `/api/connectors/{id}` | Delete a connector |
| `POST` | `/api/connectors/{id}/test` | Test connection (updates status) |
| `POST` | `/api/connectors/{id}/activate` | Activate connector |
| `POST` | `/api/connectors/{id}/deactivate` | Deactivate connector |
| `POST` | `/api/connectors/{id}/import-config` | Body: `{"json_data": {...}}`. Flattens JSON into `ir.connector.param` rows. |
| `GET` | `/api/connectors/{id}/config` | Returns assembled config dict from params. |
| `GET` | `/api/connectors/oauth/callback/{provider_type}` | OAuth code exchange callback (see below). |
| `GET` | `/api/connectors/types` | List registered provider types (`?category=storage`) |

---

### OAuth Callback Flow

For providers that use OAuth (e.g. Google Drive), the framework provides a generic callback endpoint
at `GET /api/connectors/oauth/callback/{provider_type}`.

**End-to-end flow:**

1. Provider's `config_schema()` declares `client_id` and `client_secret` as params.
2. Admin creates the connector record and enters the OAuth app credentials as params.
3. The admin UI (or a custom page) redirects the user to the provider's OAuth consent URL —
   this is provider-specific and not a framework concern.
4. After consent, the provider redirects to:
   ```
   GET /api/connectors/oauth/callback/{provider_type}?code=<auth_code>&state=<connector_uuid>:<nonce>
   ```
5. The endpoint resolves the connector record from the `state` parameter (format: `{connector_uuid}:{nonce}`).
6. It calls `provider_cls.exchange_oauth_code(code, config)` which performs the token exchange.
7. The returned `{key: value}` dict is upserted as `ir.connector.param` rows.
8. `ir.connector.test_connection` is automatically called to verify the new tokens.
9. The response includes an `action` field to redirect the browser back to the connector form.

**Implementing `exchange_oauth_code`:**

```python
@classmethod
def exchange_oauth_code(cls, code: str, config: dict) -> dict:
    """Exchange authorization code for tokens via Google's token endpoint."""
    import requests
    resp = requests.post("https://oauth2.googleapis.com/token", data={
        "code": code,
        "client_id": config["client_id"],
        "client_secret": config["client_secret"],
        "redirect_uri": config.get("redirect_uri", ""),
        "grant_type": "authorization_code",
    })
    resp.raise_for_status()
    data = resp.json()
    # Return only the keys you want stored as params
    return {"refresh_token": data["refresh_token"]}
```

Providers that do not use OAuth leave the default (returns `{}`).

---

## Adding a New Provider

1. Create the provider class extending `ConnectorProvider` (or a category-specific subclass):

```python
# src/ede/foundation/storage/connectors/aws_s3.py
from ede.core.connectors.interfaces import ConnectorProvider
from ede.core.connectors.dto import ConnectionTestResult
from ede.core.services.storage.interfaces import StorageBackend

class AwsS3StorageProvider(ConnectorProvider):
    category      = "storage"
    provider_type = "aws_s3"
    display_name  = "Amazon S3"

    def __init__(self, bucket: str, access_key: str, secret_key: str, region: str) -> None:
        self._bucket = bucket
        self._access_key = access_key
        self._secret_key = secret_key
        self._region = region

    @classmethod
    def from_config(cls, config: dict) -> "AwsS3StorageProvider":
        return cls(
            bucket=config["bucket"],
            access_key=config["access_key"],
            secret_key=config["secret_key"],
            region=config.get("region", "us-east-1"),
        )

    def test_connection(self) -> ConnectionTestResult:
        try:
            import boto3
            s3 = boto3.client("s3", aws_access_key_id=self._access_key,
                              aws_secret_access_key=self._secret_key, region_name=self._region)
            s3.head_bucket(Bucket=self._bucket)
            return ConnectionTestResult(success=True, message="Connected")
        except Exception as exc:
            return ConnectionTestResult(success=False, message=str(exc))

    def as_storage_backend(self) -> StorageBackend:
        """Required for storage category providers."""
        from .aws_s3_backend import AwsS3Backend
        return AwsS3Backend(bucket=self._bucket, ...)
```

2. Register it on module import:

```python
# src/ede/foundation/storage/connectors/__init__.py
from ede.core.connectors.registry import connector_registry
from .aws_s3 import AwsS3StorageProvider

connector_registry.register(AwsS3StorageProvider)
```

3. No changes to `ir.connector` model or `ConnectorService` — the new provider is immediately
   available in the admin UI under Settings → Integrations → Connectors.

---

## Adding a New Service Category

1. Define a category-specific provider subclass (optional but recommended for type safety):

```python
# src/ede/core/connectors/interfaces.py  (or in the new module)
class EmailConnectorProvider(ConnectorProvider):
    category = "email"

    @abstractmethod
    def send_message(self, to: str, subject: str, body: str) -> None: ...
```

2. Add the category key to `ir.connector.category` enum field.

3. Implement providers (`GmailProvider`, `OutlookProvider`) and register them.

4. Create the consuming foundation module (e.g., `foundation.email`) that calls
   `ConnectorService.get_default_for_category("email", org_id, env)` to route at runtime.

---

## Menu Placement

Connector management lives under **Settings → Integrations** (admin-only).

```xml
<!-- src/ede/foundation/connectors/data/connectors_menus.xml -->
<ede_data>
  <record model="ir.menu" id="menu_connectors">
    <field name="name">Connectors</field>
    <field name="parent_id" ref="foundation_settings.menu_settings_integrations"/>
    <field name="action_id" ref="action_ir_connector_list"/>
    <field name="sequence">40</field>
  </record>
</ede_data>
```
