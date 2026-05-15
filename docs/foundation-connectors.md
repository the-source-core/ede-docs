<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Connectors — Implementation Docs

**Module:** `foundation.connectors` (`src/ede/foundation/connectors/`)
**Roadmap:** [roadmap/foundation/connectors/README.md](../roadmap/foundation/connectors/README.md)
**Status:** ✅ Delivered (baseline — pre-roadmap)
**Layer:** Foundation engine — platform substrate

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A service-agnostic connector framework that owns the central plug-board for every external integration in the platform. It does not know what SMTP, S3, or Drive *are* — it only knows that an integration is one record with a credentials/params bag, a lifecycle, and a "test connection" verb. Consumer modules bring the protocol knowledge.

- `ir.connector` row stores one configured integration instance — category (`storage` / `email` / `messaging`), provider type, owning organization, enabled flag, default flag, current `draft → connected | error` status, last test result.
- `ir.connector.param` rows store key/value params per connector — dot-notation keys (e.g. `installed.client_id`), text values OR uploaded files (certificates, JSON key files), with a sequence for display order.
- Consumer modules declare the connector kinds they accept (via the kernel-level `connector_registry`) and pull params at runtime through `ConnectorService` / `ir.connector.get_config`.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every integration needs a place to store credentials + runtime params and a lifecycle (create / test / enable / disable). Building that per-integration is waste; building it once and letting `foundation.email` / `foundation.storage` / future modules register connector types against it is leverage.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX entry point:** Settings → Integrations → Connectors. One admin page lists every registered integration (storage, email, future messaging) in one place. Admin creates an `ir.connector`, pastes the provider JSON (e.g. Google `credentials.json`) into the import dialog, runs Test Connection, then Activate.
- **Programmatic entry point for consumer modules:** register a `ConnectorProvider` subclass against the kernel-level `connector_registry` keyed by `(category, provider_type)`. At runtime, call `ConnectorService` helpers or dispatch `Command("ir.connector.get_config", ...)` to obtain the assembled config dict to feed to the provider's `from_config()`.
- **Integration boundary** — produces: connector records, param rows, an admin UI, an HTTP API, a `test_connection` lifecycle verb. Consumes: nothing domain-specific. Does NOT produce: protocol implementations (those live in the consumer module).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Consumer module]                    foundation.connectors
  foundation.email                     ┌──────────────────────────┐
  foundation.storage ────registers────▶│ kernel connector_registry│
  (registers SMTP / S3 / GCS / ...)    │  (category, type) → cls  │
                                       └────────────┬─────────────┘
                                                    │
   admin creates instance ─▶ ir.connector  ─owns──▶ ir.connector.param
                                  │
                  test_connection │
                  activate        │
                  deactivate      ▼
                          ConnectorService
                                  │
                       provider_cls.from_config(get_config())
                       provider_cls.test_connection()
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.connector` | One configured integration instance — name, category, provider_type, organization, is_enabled, is_default, status, last test result | [src/ede/foundation/connectors/models/connector.py](../src/ede/foundation/connectors/models/connector.py) |
| `ir.connector.param` | Key/value config parameter (text or file) for an `ir.connector` — dot-notation keys, sequence ordering | [src/ede/foundation/connectors/models/connector_param.py](../src/ede/foundation/connectors/models/connector_param.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `ConnectorService` | Stateless helper: instantiates a `ConnectorProvider` from an `ir.connector` record, calls `test_connection()`, builds config dicts from param rows, used by `StorageRouter` and future routers | [src/ede/foundation/connectors/services/connector_service.py](../src/ede/foundation/connectors/services/connector_service.py) |
| `flatten_json` / `unflatten_params` | Pure JSON flatten/unflatten helpers — nested provider config ↔ dot-notation param rows | [src/ede/foundation/connectors/services/config_utils.py](../src/ede/foundation/connectors/services/config_utils.py) |
| `ConnectorController` | HTTP route group under `/api/connectors` (auth-gated admin endpoints) | [src/ede/foundation/connectors/api/connectors.py](../src/ede/foundation/connectors/api/connectors.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ir.connector.test_connection` | Admin clicks "Test" / route | Builds config from params, calls `ConnectorProvider.test_connection()`, updates `status`, `status_message`, `last_tested_at` |
| `ir.connector.activate` | Admin clicks "Activate" | Sets `is_enabled=True` (requires `status=connected`) |
| `ir.connector.deactivate` | Admin clicks "Deactivate" | Sets `is_enabled=False` |
| `ir.connector.import_config` | Admin pastes / uploads provider JSON | Flattens JSON via provider's `normalize_import_json` (if any) → `flatten_json` → replaces existing `ir.connector.param` rows |
| `ir.connector.get_config` | Consumer module (e.g. `StorageRouter`) needs runtime config | Reads `ir.connector.param` rows, reads file params via `storage.document`, returns nested dict ready for `provider.from_config()` |
| `ede.create` / `ede.update` / `ede.delete` / `ede.search` / `ede.read_one` | Standard CRUD on connector + param | Generic CRUD via `CrudKernel` |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.record.created` / `.updated` / `.deleted` on `ir.connector` | Standard CRUD on a connector | Any module that wants to observe connector lifecycle |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `/api/connectors/*` (auth-gated admin surface — list providers, test, activate, deactivate, import config) | Admin operations on `ir.connector` from the React webclient | `ConnectorController` ([api/connectors.py](../src/ede/foundation/connectors/api/connectors.py)) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.ir.connector.create` | Enforces single-default-per-(category, organization) — clears `is_default` on any existing default in the same category+org before insert |
| `pre.ir.connector.update` | Same invariant on update — clears `is_default` on other connectors in the same category+org when this one is flipped to default |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
```
draft ──test_connection (ok)────▶ connected ──activate──▶ enabled (is_enabled=True)
  │                                   │                       │
  │                                   │                       └─deactivate──▶ disabled (is_enabled=False, status preserved)
  │                                   │
  └──test_connection (fail)──▶ error ─┘  (status_message captured)
```
Status field values: `draft` (default), `connected`, `error`. The `is_enabled` flag is independent of status but `activate` requires `status=connected`.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `connectors`
- Manifest `depends`: `foundation.base`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
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
| [data/ir.rbac.permission.csv](../src/ede/foundation/connectors/data/ir.rbac.permission.csv) | RBAC permission rows for connector administration |
| [data/connectors_menus.xml](../src/ede/foundation/connectors/data/connectors_menus.xml) | Settings → Integrations → Connectors menu entry + action |
| [data/connectors_google_drive.xml](../src/ede/foundation/connectors/data/connectors_google_drive.xml) | Example Google Drive connector starter row (referenced by `foundation.storage`) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Baseline (pre-roadmap) | Connector framework, lifecycle commands, admin UI + HTTP API, RBAC seed | ✅ Delivered | [roadmap/foundation/connectors/README.md](../roadmap/foundation/connectors/README.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Connector record management with single-default-per-(category, org) invariant | `ir.connector` | [models/connector.py](../src/ede/foundation/connectors/models/connector.py) | [roadmap baseline](../roadmap/foundation/connectors/README.md) |
| Param storage (text + file value types, dot-notation keys, sequence ordering) | `ir.connector.param` | [models/connector_param.py](../src/ede/foundation/connectors/models/connector_param.py) | [roadmap baseline](../roadmap/foundation/connectors/README.md) |
| Lifecycle verbs — `test_connection`, `activate`, `deactivate`, `import_config`, `get_config` | `ir.connector` handlers | [models/connector.py](../src/ede/foundation/connectors/models/connector.py) | [roadmap baseline](../roadmap/foundation/connectors/README.md) |
| JSON-blob import — flatten nested provider config into param rows; provider may `normalize_import_json` first | `ir.connector.param` | [services/config_utils.py](../src/ede/foundation/connectors/services/config_utils.py), [models/connector.py](../src/ede/foundation/connectors/models/connector.py) | [roadmap baseline](../roadmap/foundation/connectors/README.md) |
| Coordination helper between `ir.connector` records and `ConnectorProvider` classes registered in the kernel `connector_registry` | n/a (service) | [services/connector_service.py](../src/ede/foundation/connectors/services/connector_service.py) | [roadmap baseline](../roadmap/foundation/connectors/README.md) |
| Admin HTTP API under `/api/connectors/*` (auth-gated) | n/a (controller) | [api/connectors.py](../src/ede/foundation/connectors/api/connectors.py) | [roadmap baseline](../roadmap/foundation/connectors/README.md) |
| Admin UI — Settings → Integrations → Connectors (list + form views) | `ir.connector` | [views/ir_connector_views.xml](../src/ede/foundation/connectors/views/ir_connector_views.xml), [data/connectors_menus.xml](../src/ede/foundation/connectors/data/connectors_menus.xml) | [roadmap baseline](../roadmap/foundation/connectors/README.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| _none yet — secret rotation, OAuth flow capture, and webhook-signature verification belong in `foundation.security` Phase TBD or in the consuming connector_ | n/a | [roadmap baseline](../roadmap/foundation/connectors/README.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- Trying to add a new connector kind (SMTP, S3, Drive, …) **inside** `foundation.connectors`. Don't. Register a `ConnectorProvider` subclass against the kernel `connector_registry` from the **consumer** module (e.g. `foundation.email`, `foundation.storage`). This module stays category-agnostic.
- Reading credentials directly off `ir.connector.config` (the legacy JSON field). Always go through `ir.connector.get_config` / `ConnectorService` so file params, provider `normalize_import_json`, and param-row precedence are honored.
- Forgetting that `activate` requires `status=connected` — call `test_connection` first.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Initial schema + a follow-up migration ship under [migrations/versions/](../src/ede/foundation/connectors/migrations/versions/). Apply via `ede migrate upgrade -t <tenant>`.
- Module is registered explicitly under `ACTIVE_MODULES = [..., "connectors", ...]` in `src/ede/foundation/settings.py`. No auto-discovery.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Administrator (or equivalent) | Full CRUD on `ir.connector` + `ir.connector.param`, plus the lifecycle commands (`test_connection`, `activate`, `deactivate`, `import_config`). Seeded via [data/ir.rbac.permission.csv](../src/ede/foundation/connectors/data/ir.rbac.permission.csv). |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [foundation.email](foundation-i18n.md) — registers SMTP / SendGrid / Mailgun / Gmail / Outlook connector kinds. _(roadmap doc TBD)_
- [Foundation Storage Engine](#) — registers S3 / GCS / Azure Blob / Google Drive connector kinds. _(roadmap doc TBD)_
- [Foundation Security & Authorization](foundation-security.md) — owns secret rotation / OAuth flow capture / webhook-signature verification (future).
- [Platform execution rules](platform-00-execution-rules.md)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-14. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
