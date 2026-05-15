# Connectors

One contract for every external system — SMTP, S3, Google Drive, Gmail OAuth2, custom. Connector kinds register themselves against the kernel registry; user-visible records (`ir.connector`) configure instances per organization.

```python
# Register a new connector kind in your app's __init__.py
from ede.core.connectors.registry import connector_registry
from .connectors.s3 import S3ConnectorProvider

connector_registry.register(S3ConnectorProvider)
```

Once registered, administrators create `ir.connector` records via the Settings UI, test the connection, and mark a default per (category, organization). Consumer modules (Email, Storage, future ones) call `ConnectorService.resolve(...)` to pick the right backend.

---

## What you get

-   **`ConnectorProvider`** ABC — implement four methods (`test_connection`, `activate`, `deactivate`, `import_config`).
-   **`ConnectorRegistry`** — module-level singleton; connector *kinds* register here at import time.
-   **`ir.connector`** model — user-visible records, one per configured backend.
-   **`ir.connector.param`** model — typed parameters per connector (credentials, endpoints, OAuth tokens).
-   **`ConnectorService`** — lifecycle verbs: `test_connection()`, `activate()`, `deactivate()`, `import_config()`, `get_config()`.
-   **HTTP API** — `/api/connectors/*` admin endpoints.
-   **Settings UI** — Integrations → Connectors, with per-kind config forms.
-   **Single-default invariant** — at most one `is_default=True` connector per (category, organization).

## How to use it

### Define a new connector kind

```python
from ede.core.connectors.interfaces import ConnectorProvider


class S3ConnectorProvider(ConnectorProvider):
    kind = "s3"
    category = "storage"
    label = "Amazon S3"

    parameter_schema = [
        {"name": "bucket", "type": "string", "required": True},
        {"name": "access_key_id", "type": "string", "required": True, "secret": True},
        {"name": "secret_access_key", "type": "string", "required": True, "secret": True},
        {"name": "region", "type": "string", "required": True},
    ]

    def test_connection(self, config) -> tuple[bool, str]:
        client = boto3.client("s3", **config)
        try:
            client.head_bucket(Bucket=config["bucket"])
            return True, "Connected"
        except ClientError as exc:
            return False, str(exc)
```

Register it once at app load:

```python
# my_app/__init__.py
from ede.core.connectors.registry import connector_registry
from .connectors.s3 import S3ConnectorProvider

connector_registry.register(S3ConnectorProvider)
```

### Configure an instance (data file)

```xml
<record id="connector_acme_s3" model="ir.connector">
    <field name="name">Acme S3</field>
    <field name="category">storage</field>
    <field name="kind">s3</field>
    <field name="organization_id" ref="res.organization.acme"/>
    <field name="is_default">true</field>
    <field name="config" eval="{
        'bucket': 'acme-prod',
        'region': 'us-east-1',
        'access_key_id': '$ENV:S3_KEY',
        'secret_access_key': '$ENV:S3_SECRET',
    }"/>
</record>
```

### Resolve a backend at runtime

```python
from ede.foundation.connectors.services.connector_service import ConnectorService

backend = ConnectorService.from_env(env).resolve(
    category="storage",
    organization_id=env.active_organization_id,
)
```

The service returns the default connector for that (category, org) — or `None` if no default is configured.

### Test a connection from the UI

The "Test Connection" button in the Settings UI calls `ConnectorService.test_connection()` synchronously and surfaces the (ok, message) tuple in a toast.

## How it composes with other features

-   **[Document Storage](storage.md)** — registers backend kinds against the connectors framework.
-   **[Email](email.md)** — SMTP / SendGrid / Mailgun / Gmail OAuth2 are connector kinds.
-   **[Multi-Tenant Gateway](gateway.md)** — every tenant has its own connectors; resolution is `Env`-scoped.

## Reference

| Concept | Where it lives |
|---|---|
| `ConnectorProvider` ABC | `src/ede/core/connectors/interfaces.py` |
| `ConnectorRegistry` | `src/ede/core/connectors/registry.py` |
| `ir.connector` model | `src/ede/foundation/connectors/models/connector.py` |
| `ConnectorService` | `src/ede/foundation/connectors/services/connector_service.py` |
| Full reference | [Connector Framework architecture guide](../16-connector-framework.md) |
