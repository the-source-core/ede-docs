<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Postgres-Only Persistence — Implementation Docs

**Module:** `ede.core` (`adapters/persistence/sqlalchemy/`) + `ede.cli` (`migrate.py`) + `foundation` (`settings.py`)
**Roadmap:** [roadmap/platform/13-postgres-only-persistence.md](../roadmap/platform/13-postgres-only-persistence.md)
**Status:** ✅ Delivered 2026-07-08 · **partly superseded 2026-09-07**
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.
>
> **Partly superseded** by [Platform Change 18](platform-18-runtime-schema-upgrades.md): the migration generator this change made convergent no longer exists. What it delivered that still stands is below.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
The first half of removing SQLite. It made PostgreSQL the default configured engine and took SQLite out of migration generation, deliberately stopping short of the runtime engine — which turned out to be load-bearing for the test suite and was deferred to [Platform Change 14](platform-14-sqlite-runtime-and-test-removal.md).
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
The persistence layer carried SQLite in two dev-only roles: as the default runtime engine, and as the hard-wired reference database that migration autogenerate diffed against. Production never used it — the provisioning path refused anything but PostgreSQL, and e2e forced it.

The second role was the expensive one. SQLite reflects far less than PostgreSQL, so diffing against a SQLite reference produced spurious operations on every run — the entire subject of [Platform Change 12](../roadmap/platform/12-migration-generator-convergence.md). Pointing the diff at a real PostgreSQL database removed the cause rather than filtering the symptom.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
`DATABASE_ENGINE` defaults to `postgresql`. A config-less boot targets PostgreSQL.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
Two pieces outlived the generator they were built for, and both are load-bearing today:

**Deterministic ≤63-char identifier names.** The metadata builder composes foreign-key names and caps them at PostgreSQL's `NAMEDATALEN-1`, hashing rather than truncating when a name would overflow. 279 of 1,513 FK names exceeded 63 characters before this; none do now. The schema comparison depends on the same convention — a truncated name is a name that silently collides.

**Audit FKs excluded by decision.** `created_uid` / `updated_uid` → `res.user` constraints are declared on every model and deliberately never created. That exclusion is now encoded once, in `KnownDivergenceFilter`, and both the drift checker and the synchroniser read it from there — an exception encoded twice is one that will eventually disagree with itself.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none_ | | Infrastructure change; no models. |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `SqlAlchemyMetadataBuilder` | Composes identifier names and caps them at 63 chars; `skip_audit_fks` controls audit-FK emission | [metadata_builder.py](../src/ede/core/adapters/persistence/sqlalchemy/metadata_builder.py) |
| `KnownDivergenceFilter` | Holds the audit-FK exclusion as data, read by both the checker and the synchroniser | [known_divergences.py](../src/ede/core/services/migration/known_divergences.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | | |
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
Not applicable.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: settings -->
| Key | Change |
|---|---|
| `DATABASE_ENGINE` | Default flipped from `sqlite` to `postgresql` |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: irconfig -->
| Key | Purpose |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed -->
_none_
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status -->
| Piece | Status | Notes |
|---|---|---|
| PostgreSQL as the default configured engine | ✅ Delivered | |
| ≤63-char deterministic identifier names | ✅ Delivered | Still load-bearing for the schema comparison |
| Audit FKs excluded by decision | ✅ Delivered | Now encoded in `KnownDivergenceFilter` |
| SQLite out of migration generation | ✅ Delivered · superseded | There is no generator any more |
| SQLite runtime engine + test suite | ➡ Deferred | Delivered by [Platform Change 14](platform-14-sqlite-runtime-and-test-removal.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Capability | Notes |
|---|---|
| PostgreSQL is the sole production engine | Provisioning refuses anything else |
| Deterministic identifier naming | 279 over-length FK names → 0 |
| Audit-FK exclusion as a single recorded decision | Read by the drift checker and the synchroniser alike |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Notes |
|---|---|---|
| Two acceptance items reference `ref.db` / `alembic.ini` | 🟢 Moot | Both artefacts were deleted by Platform Change 18 |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- SYNC-BLOCK: pitfalls -->
- **Hand-writing a constraint name.** The builder composes and caps it. A name you shorten yourself is a name that can collide after PostgreSQL truncates it.
<!-- /SYNC-BLOCK -->

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
The audit foreign keys are declared in code and absent from every database, by decision. `ede migrate check` reports them as suppressed known divergences rather than findings — 440 of them on a tenant built by the old migration files.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
_none_
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Platform Change 14 — Complete SQLite removal](platform-14-sqlite-runtime-and-test-removal.md) — the deferred half.
- [Platform Change 18 — Runtime schema upgrades](platform-18-runtime-schema-upgrades.md) — what replaced the generator.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-09-07. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
