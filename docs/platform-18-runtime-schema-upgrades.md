<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Runtime Schema Upgrades (Remove Alembic) — Implementation Docs

**Module:** `ede.core.services.migration` (drift checker · synchroniser · upgrade-script runner · migration-history guard) + `ede.cli` (`migrate.py`) + `foundation.base` (`ir.app`, `AppStateSync`) + `foundation.gateway` (SaaS provisioning)
**Roadmap:** [roadmap/platform/18-runtime-schema-upgrades.md](../roadmap/platform/18-runtime-schema-upgrades.md)
**Status:** ✅ Delivered 2026-09-07
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.
>
> **Developer guide:** [Runtime Schema Upgrades](runtime-schema-upgrades.md) — read that first if you are changing a model. This page is the delivery record.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
The replacement for Alembic. There are no migration files: `ede migrate upgrade` compares what the models declare against what a tenant's database actually has, and applies the difference. Adding, widening or indexing a field is a one-step change — edit the model.

Two changes a comparison can never infer still ship as files, and only these two: a **rename** and a **data backfill**. They live under `<app_root>/upgrades/<version>/` as `before_*.py` / `after_*.py` and run when the tenant's recorded version crosses that version.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
The migration graph was one Alembic environment over ~39 per-app `version_locations`. A revision's `down_revision` records **who authored what, and when** — not which app owns the table it touches. With 25 unmerged heads and 51 cross-app edges, the order Alembic interleaved those branches depended on which domains were active, so activating one domain could move another domain's revision ahead of the table it referenced. A fresh `ede migrate upgrade` failed with `UndefinedTable: relation "ir_model" does not exist`.

Ordering was the symptom. The cause is that files describe **history**, and history is not what a database needs in order to be brought up to date — the current models are.

The measurement that settled it: a database built by the synchroniser reports **0 drift findings at every severity**; the same tenant built by the 234 migration files reports **42 type mismatches, 35 nullability divergences and 859 missing indexes**. Autogenerate had never emitted a type change, so every `VARCHAR(32)` the models had since widened to `TEXT` stayed narrow in every database the files built. The files were **936 facts** behind the models.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- `ede migrate check -t <tenant>` — reports how the database differs from the models. Writes nothing; safe against production.
- `ede migrate upgrade -t <tenant>` — runs the four phases (before scripts → schema → data → after scripts, then records the version).
- Programmatic: `MigrationRunner.upgrade_tenant(...)`, used by `DbProvisionerService` for SaaS provisioning.
- Model authors interact with it by **not** interacting with it: changing a field is the whole change.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
ede migrate upgrade -t <tenant>
  ├─ Phase 1  before_*.py           # only for apps whose recorded version moved
  ├─ Phase 2  SchemaSynchronizer    # pass 1: tables + columns
  │                                 # pass 2: foreign keys + indexes + NOT NULL
  ├─ Phase 3  RegistrySync · ViewSync · DataLoader · OrphanCleanup
  └─ Phase 4  after_*.py  →  AppStateSync.commit()   # version recorded LAST
```

**Why two passes.** A foreign key cannot be created until the table it points at exists. Applying each one as its column is created makes processing order decide whether the upgrade works — and order depends on which modules a customer activated. Deferring every connection until all structure exists removes the question: by the time any foreign key is applied, every table it could reference is already there. The ordering bug class stops being possible rather than being managed.

**Why the version is recorded last.** A failure anywhere earlier leaves the app at its old version, so the transition retries. Recording it sooner would mark a half-finished upgrade as done, and the script that should have run would never run again.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.app` | One row per activated app: `app_key`, `installed_version`, `state`, `installed_at_utc`. Answers "has this app moved since this tenant was last upgraded?" | [ir_app.py](../src/ede/foundation/base/models/ir_app.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `SchemaShapeReader` | Describes a schema in comparable terms, from the models or from a live database | [schema_shape.py](../src/ede/core/services/migration/schema_shape.py) |
| `SchemaDriftChecker` | Compares the two shapes and classifies every difference by kind + severity | [schema_drift.py](../src/ede/core/services/migration/schema_drift.py) |
| `KnownDivergenceFilter` | Suppresses differences that are accepted by decision (the audit FKs, `alembic_version`). A **blocking** finding is never suppressible | [known_divergences.py](../src/ede/core/services/migration/known_divergences.py) |
| `SchemaSynchronizer` | Applies the difference in two passes, in one transaction, under a PostgreSQL advisory lock | [schema_synchronizer.py](../src/ede/core/services/migration/schema_synchronizer.py) |
| `UpgradeScriptRunner` | Discovers and runs `before_*.py` / `after_*.py` for the versions a tenant crosses | [upgrade_scripts.py](../src/ede/core/services/migration/upgrade_scripts.py) |
| `AppUpgradePlanner` / `AppStateSync` | Reads recorded versions before anything is touched; writes them last | [app_state_sync.py](../src/ede/foundation/base/services/app_state_sync.py) |
| `assert_migration_history_complete` | Refuses a database that never finished the migration era | [final_migration_state.py](../src/ede/core/services/migration/final_migration_state.py) |
| `MigrationRunner` | Programmatic single-tenant convergence, used by the gateway | [runner.py](../src/ede/core/services/migration/runner.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | | This is CLI + service surface; it dispatches no domain commands. |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
Always on — it is the schema mechanism. No `ACTIVE_MODULES` entry.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: settings -->
| Key | Default | Notes |
|---|---|---|
| _none_ | | There is one mechanism, so there is nothing to select. The staged rollout's `SCHEMA_UPGRADE_MODE` was removed at cutover. |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: irconfig -->
| Key | Purpose |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed -->
_none_ — `ir.app` rows are written by `AppStateSync` during the upgrade, not seeded.
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status -->
| Stage | Status | Evidence |
|---|---|---|
| 0 — Read-only drift checker + `ede migrate check` | ✅ Delivered | **Gate A**: a migration-built database reported 0 blocking drift |
| 1 — `ir.app` per-tenant module state | ✅ Delivered | 38 apps recorded on a live tenant |
| 2 — Synchroniser, behind a flag, running after Alembic | ✅ Delivered | **Gate B**: 0 tables, 0 columns, 0 FKs, 0 relaxations |
| 3 — Version-keyed `before` / `after` upgrade scripts | ✅ Delivered | 0.2.0 and 0.3.0 ran, 0.1.0 did not; a failure left the version unrecorded and retried |
| 4 — Cutover | ✅ Delivered | A database built entirely by the synchroniser: 0 findings at every severity |
| 5 — Removal | ✅ Delivered | 37,398 lines; a tenant provisioned with no migration files in the tree |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Capability | Notes |
|---|---|
| Schema derived from the models | Creates, adds, tightens and relaxes. **Never** drops a column, table or constraint; **never** changes a type |
| Two-pass application | Structure, then connections — cross-module foreign keys cannot be applied too early |
| Version-keyed upgrade scripts | `before_*.py` (raw connection, old shape still standing) and `after_*.py` (connection + ORM) |
| Per-tenant module state | `ir.app`, written last so a failure retries |
| Read-only reporting | `ede migrate check`, with severity filtering and known-divergence suppression |
| Concurrency safety | One transaction, PostgreSQL advisory lock per database |
| Migration-history guard | Refuses a tenant that never finished replaying the old files |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Notes |
|---|---|---|
| Retired columns and indexes accumulate forever | 🟡 Accepted | The price of safe rollback. Removal, if ever wanted, is an explicit human-run step |
| A type the models changed is never converted | 🟡 Accepted | A conversion can lose data. A tenant migrated from the old files keeps its `VARCHAR(32)` where the models now say `TEXT` — expect permanent `type_mismatch` findings there |
| No per-change reviewable artefact | 🟡 Accepted knowingly | `ede migrate check` is the replacement: it prints what would change before anything runs |
| `final_migration_state.py` is temporary | 🟢 By design | Deletable once no database predating the cutover remains anywhere |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- SYNC-BLOCK: pitfalls -->
- **Forgetting the manifest version bump.** An `upgrades/1.1.0/` script is only found if the app's `version` moves past `1.1.0`. This is the mistake everyone makes once.
- **Assuming a `before` script gets `env`.** It does not, deliberately: the ORM at that moment is built from the *new* models and would describe a database that does not exist yet.
- **Writing a non-idempotent `before` script.** They commit in their own transaction, so a later failure re-runs them.
- **Expecting a dropped field to drop its column.** It does not. The column keeps its data and loses only its NOT NULL.
<!-- /SYNC-BLOCK -->

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- A database with no `alembic_version` table was built by this mechanism and proceeds without comment.
- A database whose `alembic_version` matches the frozen final head set replayed everything and proceeds.
- Anything else is **refused**, naming the revisions not reached. Comparison reproduces shape and never the data those migrations carried, so converging such a tenant would give it the right shape and the wrong contents.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
_none_ — CLI and provisioning surface; no end-user permissions.
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Runtime Schema Upgrades](runtime-schema-upgrades.md) — the developer guide.
- [Platform Change 14 — SQLite removal](platform-14-sqlite-runtime-and-test-removal.md) — the prerequisite: one dialect, so no lowest-common-denominator compromises.
- [`foundation.gateway`](foundation-gateway.md) — `MigrationRunner` is how SaaS tenants are provisioned.
- [Model & View Extension SDK](foundation-base-extensions.md) — an extension's column is attributed to the module that declares it.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-09-07. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
