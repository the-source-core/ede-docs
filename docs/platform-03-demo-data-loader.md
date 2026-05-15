<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Demo Data Loader (`ede migrate upgrade --with-demo`) — Implementation Docs

**Module:** `ede.core.cli` + `ede.core.services.data_loader` + `ede.foundation.base.ir.data.reference`
**Roadmap:** [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md)
**Status:** ✅ Delivered (2026-05-12)
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A per-invocation CLI flag — `ede migrate upgrade -t <tenant> --with-demo=<value>` — that loads each app's `demo/*.xml` files after schema upgrade and main data load. Demo records register in `ir.data.reference` with `is_demo=True` so they are excluded from orphan cleanup and (in a future PR) can be purged independently of production seed data. Fully opt-in: omitted flag → no demo load, production migrations unaffected.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
The existing manifest `data: [...]` channel is for production reference seeds (countries, UOMs, RBAC roles). There is no parallel channel for demoable sample records — sample leads, demo organizations, dummy shipments — used in sales demos, internal training, and PR-review environments. Today these are either hand-loaded fixtures (drift-prone) or copy-pasted into production seed files (pollutes prod tenants). This adds a clean, opt-in, taggable demo channel.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Platform operator (CLI):**
  - `ede migrate upgrade -t <tenant> --with-demo=all` — load every active app's demo files
  - `ede migrate upgrade -t <tenant> --with-demo=foundation.base` — load demo for one app
  - `ede migrate upgrade -t <tenant> --with-demo=foundation.base,logistics.masters` — comma-separated app_keys
  - Omit the flag → no demo loaded (default; preserves current behaviour byte-for-byte)
- **App author:** declare `demo: ["demo/sample_users.xml", ...]` in `__manifest__.py`; create the listed XML files under `<app_root>/demo/` using the same record DSL as production `data/*.xml`. Cross-references to manifest-data records work via `ref="base.user_admin"` because the loader's in-session ref cache survives across the data → demo passes and apps load in dependency order.
- **Tagged in DB:** every record created during a demo pass writes `is_demo=True` to its `ir.data.reference` row; orphan cleanup filters those out so demo records survive any flag-less migrate.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
ede migrate upgrade -t <tenant> [--with-demo=<scope>]
  ├─ boot_runtime_and_environment        # registers AppSpec + demo_files
  ├─ alembic upgrade heads                # schema upgrade
  ├─ RegistrySync (optional)
  ├─ DataLoader.load_all_apps             # manifest data/*  → ir.data.reference (is_demo=False)
  ├─ DataLoader.load_demo_for_apps        # (only if --with-demo) demo/* → ir.data.reference (is_demo=True)
  │                                       # _is_demo flag on the loader; shared _local_refs cache across passes
  └─ OrphanCleanup.run                    # filters [is_demo=False] before computing orphans
```
Two passes share the same `DataLoader` instance, so demo XML can `ref=` records created by the immediately-preceding data pass without a DB roundtrip.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.data.reference` (extended) | External-ID registry. Gains `is_demo: Boolean(default=False, index=True)` to distinguish demo loads from production seeds. | [src/ede/foundation/base/models/data_reference.py](../src/ede/foundation/base/models/data_reference.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `AppSpec` (extended) | New `demo_files: Tuple[str, ...]` field carries the manifest-declared demo paths. | [src/ede/core/registry.py](../src/ede/core/registry.py) |
| `ModuleLoader._resolve_manifest_path_list` | Generalised helper used for both `data:` and `demo:` manifest keys. | [src/ede/core/loader.py](../src/ede/core/loader.py) |
| `DataLoader.load_demo_for_apps(registry, app_keys)` | Public entry point for the demo pass. Filters apps when `app_keys` is supplied; sets `self._is_demo=True` via try/finally so `_register_ref` writes `is_demo=True`. | [src/ede/core/services/data_loader/loader.py](../src/ede/core/services/data_loader/loader.py) |
| `DataLoader._load_app_demo` / `_collect_demo_files` | Per-app demo iteration; enforces `<app_root>/demo/` subdir constraint. | [src/ede/core/services/data_loader/loader.py](../src/ede/core/services/data_loader/loader.py) |
| `OrphanCleanup._read_all_refs` | Filters `[["is_demo", "=", False]]` so a flag-less migrate never classifies demo rows as orphans. | [src/ede/core/services/data_loader/orphan_cleanup.py](../src/ede/core/services/data_loader/orphan_cleanup.py) |
| `migrate_upgrade` CLI | Adds `--with-demo VALUE` Click option; validates value (`all` or comma-separated app_keys) against the live registry after boot; invokes the demo pass between main data load and orphan cleanup. | [src/ede/cli/commands/migrate.py](../src/ede/cli/commands/migrate.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | | (Feature surfaces as a CLI flag; no new EDE command names dispatched.) |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ | | (Demo records emit the standard `ede.record.created` / `ede.record.updated` events via the underlying CRUD path, same as data-load records.) |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none_ | | (CLI-only feature.) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | (Demo records flow through the standard CRUD hooks of their target model; no new hook keys introduced.) |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- _None_ — `--with-demo` is a per-invocation CLI flag, not a settings entry. Production migrations remain unaffected unless the flag is explicitly passed.
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
| _none_ | (This change makes demo seeds possible; it does not add any.) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Single delivery | `--with-demo` CLI flag, `demo:` manifest key, `is_demo` tag on `ir.data.reference`, orphan-cleanup filter | ✅ Delivered (2026-05-12) | [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| `--with-demo=all|<app_keys>` CLI flag on `ede migrate upgrade` | _none (CLI surface)_ | [src/ede/cli/commands/migrate.py](../src/ede/cli/commands/migrate.py) | [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md) |
| `demo: [...]` manifest key + `AppSpec.demo_files` | _none_ | [src/ede/core/loader.py](../src/ede/core/loader.py), [src/ede/core/registry.py](../src/ede/core/registry.py) | [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md) |
| `DataLoader.load_demo_for_apps` (per-app `demo/*.xml` pass, shared ref cache, `is_demo=True` tagging) | _none_ | [src/ede/core/services/data_loader/loader.py](../src/ede/core/services/data_loader/loader.py) | [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md) |
| `ir.data.reference.is_demo` Boolean + index | `ir.data.reference` | [src/ede/foundation/base/models/data_reference.py](../src/ede/foundation/base/models/data_reference.py), [03658b1fa089_add_ir_data_reference_is_demo.py](../src/ede/foundation/base/migrations/versions/03658b1fa089_add_ir_data_reference_is_demo.py) | [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md) |
| `OrphanCleanup` filters `is_demo=True` rows so flag-less migrates never reclassify demo rows as orphans | `ir.data.reference` | [src/ede/core/services/data_loader/orphan_cleanup.py](../src/ede/core/services/data_loader/orphan_cleanup.py) | [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap.

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| `ede demo purge` command — filed as a future follow-up, made trivial by the `is_demo` index this feature introduces. | 🟢 Backlog | [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md) ("Out of Scope") |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _Section reserved — will be populated once the feature is in use and integration learnings emerge._

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- New Alembic revision in `foundation.base` adds `ir_data_reference.is_demo` (Boolean) + named index `ix_ir_data_reference_is_demo`. `server_default=sa.false()` backfills existing rows uniformly to `False`; the default is then dropped to match the project convention. Forward-compatible: existing tenants get the column on the next `ede migrate upgrade`.
- Pre-existing `ir.data.reference` rows are NOT retroactively reclassified. If a tenant had previously hand-loaded demo-flavoured data via the regular `data:` key, those rows remain treated as production seeds (orphan-cleanup eligible) — same as today.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ | (Feature is CLI-only and runs with the same privileges as `ede migrate upgrade` itself.) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Roadmap Tracker](../roadmap/roadmap-tracker.md) — cross-module status index.
- `ir.data.reference` is owned by [`ede.foundation.base`](foundation-model-naming.md).
- Orphan cleanup is documented inline at [src/ede/core/services/data_loader/orphan_cleanup.py](../src/ede/core/services/data_loader/orphan_cleanup.py).
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-12 (flipped 🔴 → ✅ — 1531 pytest + 445 vitest green; 19 new pytest tests under `src/tests/core/test_loader_demo_manifest_key.py`, `src/tests/data_loader/test_demo_load.py`, and `TestOrphanCleanupSkipsDemoRows` in `src/tests/data_loader/test_orphan_cleanup.py`). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
