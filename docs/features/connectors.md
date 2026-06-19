# Connectors

One canonical shape for every external system — storage backends, email transports, messaging, and your own integrations. A provider class registers against the kernel registry; `ir.connector` records configure instances per organization; consumers resolve the active connector for a category at runtime.

```python
from ede.core.connectors import StorageConnectorProvider, connector_registry
from ede.core.connectors.dto import ConnectionTestResult


class MyStorageProvider(StorageConnectorProvider):
    category = "storage"
    provider_type = "my_storage"
    display_name = "My Storage"

    def __init__(self, config: dict):
        self._config = dict(config)

    @classmethod
    def from_config(cls, config: dict) -> "MyStorageProvider":
        return cls(config or {})

    def test_connection(self, connector_uuid: str = "") -> ConnectionTestResult:
        return ConnectionTestResult(success=True, message="Connected")

    def as_storage_backend(self):
        return MyStorageBackend(self._config)


connector_registry.register(MyStorageProvider)
```

Once registered, administrators create an `ir.connector` for the provider, test it, and enable it; consumer modules call `ConnectorService.get_default_for_category(...)` to pick the right backend per organization.

---

## What you get

-   **`ir.connector`** — one record per configured integration. Fields: `name`, `category`, `provider_type`, `organization_id`, `is_enabled`, `is_default`, `status` (`draft` / `connected` / `error`), `status_message`, `last_tested_at`, plus `param_ids`.
-   **`ir.connector.param`** — typed key-value config rows: `key`, `value`, `value_type` (`text` / `file`), `is_secret`, `sequence`. Secret params are Fernet-encrypted at rest and masked in the UI.
-   **`ConnectorProvider`** ABC + category bases — `StorageConnectorProvider` (implements `as_storage_backend()`), `EmailConnectorProvider` (implements `send_message()`), `AiConnectorProvider`.
-   **`connector_registry`** — import-time registry keyed by `(category, provider_type)`.
-   **`connector_category_registry`** — runtime category registry; categories are declared, not hardcoded, so a module can contribute its own (e.g. a `tracking` category).
-   **`connector_provider_catalog`** — declare a provider line-up up front (each entry `available` or `upcoming`) independent of whether its engine has landed.
-   **`ConnectorService`** — static helpers: `get_default_for_category`, `test_connection`, `instantiate_provider`, `list_provider_catalog`, `config_schema_for`.
-   **Lifecycle commands** — `ir.connector.test_connection`, `ir.connector.activate`, `ir.connector.deactivate`, `ir.connector.import_config`, `ir.connector.get_config`.
-   **HTTP API** — `/api/connectors/*` CRUD + test / activate / deactivate / import-config / config + catalogue + an OAuth callback.
-   **Single-active categories** — a category marked single-active (e.g. `storage`) allows only one enabled connector at a time; activating another deactivates its siblings.

## How to use it

### Implement a provider

Subclass the category base, set the three class attributes, and implement `from_config` + `test_connection`. Declare `config_schema()` so the engine knows which params to encrypt and the admin form knows what to render:

```python
from ede.core.connectors import EmailConnectorProvider, connector_registry
from ede.core.connectors.dto import ConnectionTestResult
from ede.core.connectors.errors import ConnectorConfigError


class AcmeMailProvider(EmailConnectorProvider):
    category = "email"
    provider_type = "acme_mail"
    display_name = "Acme Mail"
    icon = "mail"
    description = "Send transactional email through Acme Mail."

    def __init__(self, config: dict):
        self._config = dict(config)

    @classmethod
    def config_schema(cls) -> list[dict]:
        return [
            {"key": "api_key", "label": "API Key", "required": True, "secret": True},
            {"key": "sender",  "label": "Sender",  "required": True, "secret": False},
        ]

    @classmethod
    def from_config(cls, config: dict) -> "AcmeMailProvider":
        config = config or {}
        if not config.get("api_key"):
            raise ConnectorConfigError("Acme Mail missing api_key",
                                       category="email", provider_type="acme_mail")
        return cls(config)

    def test_connection(self, connector_uuid: str = "") -> ConnectionTestResult:
        # Never raise — return a failed result instead.
        return ConnectionTestResult(success=True, message="Connected to Acme Mail")

    def send_message(self, message):
        ...
```

`test_connection` must never raise — catch everything and return `ConnectionTestResult(success=False, message=...)`.

### Register at import time

Each consumer module owns its providers and imports them so registration fires when the app activates:

```python
# my_app/connectors/__init__.py
from . import acme_mail   # noqa: F401 — registers AcmeMailProvider
```

There is no auto-discovery — the import chain that runs when the app is in `ACTIVE_MODULES` / `ACTIVE_DOMAINS` is what triggers `connector_registry.register(...)`.

### Configure a connector instance

Author the connector and its params as data. The provider's secret params are encrypted automatically:

```xml
<ede>
    <record id="connector_acme_mail" model="ir.connector">
        <field name="name">Acme Mail (Production)</field>
        <field name="category">email</field>
        <field name="provider_type">acme_mail</field>
        <field name="organization_id" ref="res.organization.acme"/>
        <field name="is_enabled">true</field>
        <field name="is_default">true</field>
        <field name="status">draft</field>
    </record>

    <record id="connector_acme_mail_api_key" model="ir.connector.param">
        <field name="connector_id" ref="connector_acme_mail"/>
        <field name="key">api_key</field>
        <field name="value">$ENV:ACME_MAIL_API_KEY</field>
        <field name="is_secret">true</field>
        <field name="sequence">10</field>
    </record>
    <record id="connector_acme_mail_sender" model="ir.connector.param">
        <field name="connector_id" ref="connector_acme_mail"/>
        <field name="key">sender</field>
        <field name="value">noreply@acme.example</field>
        <field name="sequence">20</field>
    </record>
</ede>
```

### Resolve and use a connector at runtime

Consumers ask the service for the active default in a category, then instantiate the provider from its assembled config:

```python
from ede.core.connectors import connector_registry
from ede.foundation.connectors.services.connector_service import ConnectorService

record = ConnectorService.get_default_for_category("email", org_id, env)
if record is None:
    raise RuntimeError("No email connector configured for this organization")

provider_cls = connector_registry.get(record["category"], record["provider_type"])
provider = provider_cls.from_config(record["config"])
provider.send_message(message)
```

`get_default_for_category` returns the enabled default `ir.connector` for that `(category, organization)` with its config assembled from the param rows (secrets decrypted), or `None`. This is exactly how [Email](email.md) and [Document Storage](storage.md) pick their backend.

### Test a connection

```python
record = env.models["ir.connector"].browse(connector_uuid).read()[0]
result = ConnectorService.test_connection(record)
print(result.success, result.message)
```

The same path backs the test action in the admin UI and updates the connector's `status` / `status_message` / `last_tested_at`.

## Categories

Categories are registered, not hardcoded. The baseline ships:

| Category | Single-active | Notes |
|---|---|---|
| `storage` | Yes | One enabled file-storage backend at a time; falls back to the built-in local backend. |
| `email` | No | Multiple email connectors may coexist; one is the org default. |
| `messaging` | No | Messaging integrations. |

Contribute a new category from your module at import time:

```python
from ede.core.connectors import ConnectorCategory, connector_category_registry

connector_category_registry.register(
    ConnectorCategory(key="tracking", label="Tracking", icon="map-pin", single_active=False)
)
```

## How it composes with other features

-   **[Document Storage](storage.md)** — storage backends are `category="storage"` providers; resolved per organization, with a built-in local fallback.
-   **[Email — Outbound Queue](email.md)** — email transports are `category="email"` providers; the router resolves the org default.
-   **[Multi-Tenant Gateway](gateway.md)** — connectors are `Env`-scoped, so each tenant configures its own.

## Reference

| Concept | Where it lives |
|---|---|
| Provider ABC + category bases | `src/ede/core/connectors/interfaces.py`, `storage.py`, `email.py`, `ai.py` |
| `connector_registry` / categories / catalogue | `src/ede/core/connectors/registry.py`, `categories.py`, `catalog.py` |
| `ConnectionTestResult` + errors | `src/ede/core/connectors/dto.py`, `errors.py` |
| `ir.connector`, `ir.connector.param` | `src/ede/foundation/connectors/models/` |
| `ConnectorService` | `src/ede/foundation/connectors/services/connector_service.py` |
| HTTP API | `src/ede/foundation/connectors/api/` (prefix `/api/connectors`) |
| Architecture | [Connector Framework](../16-connector-framework.md) |
