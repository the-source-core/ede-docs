<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Complete SQLite Removal (Runtime + Test Suite) — Implementation Docs

**Module:** `ede.core` (`adapters/persistence/sqlalchemy/`) + `foundation` (`settings.py`) + `src/tests/**` + CI
**Roadmap:** [roadmap/platform/14-sqlite-runtime-and-test-removal.md](../roadmap/platform/14-sqlite-runtime-and-test-removal.md)
**Status:** ✅ Delivered 2026-09-06
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state*. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
The second and final half of the SQLite removal. [Platform Change 13](platform-13-postgres-only-persistence.md) took SQLite out of migration generation and made PostgreSQL the default engine, but deferred everything the SQLite unit-test double depended on. This removed that dependency: the test suite moved to PostgreSQL, then the SQLite runtime engine was deleted outright.

`grep -rIn "sqlite" src/ede` now returns **nothing**.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Two engines meant every persistence feature had to be expressed twice, or expressed in whatever both could do. The cost was invisible until it wasn't: a dialect branch in `generic_repo` produced different grouping behaviour on each engine, and the tests — running on SQLite — could not see it.

It was also a prerequisite, not an end in itself. [Platform Change 18](platform-18-runtime-schema-upgrades.md) had to be written against one dialect: a synchroniser that also needed a SQLite strategy would have been work written, tested, reviewed and thrown away. The decisive capability it unlocks is altering a table's rules after creation — which is what makes converging an *existing* database possible rather than merely creating a new one.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- `./run_tests.sh` provisions PostgreSQL for the default run — a live server is now required to run the suite.
- CI runs a `postgres:16` service.
- A non-`postgresql` `DATABASE_ENGINE` fails fast with a clear error rather than degrading quietly.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
Test fixtures share one PostgreSQL harness (`src/tests/_postgres.py`) rather than each building a throwaway SQLite file: a shared provider, per-test schema reset, and suite-wide connection disposal. Sharing matters — the first cut opened a provider per test and exhausted the server's connection limit.
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
| `PostgresTestSettings` + harness helpers | Shared PostgreSQL fixtures: database creation, schema reset, engine lifecycle | `src/tests/_postgres.py` |
| `SqlAlchemyProvider` | PostgreSQL only; pins every connection's session to `timezone=utc` | [sqlalchemy_provider.py](../src/ede/core/adapters/persistence/sqlalchemy/sqlalchemy_provider.py) |
| `generic_repo` | Single PostgreSQL path — no `_SQLITE_STRFTIME_MAP`, no quarter synthesis, no JSONB text fallback | [generic_repo.py](../src/ede/core/adapters/persistence/sqlalchemy/generic_repo.py) |
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
Not applicable — this removes configuration rather than adding it.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: settings -->
| Key | Change |
|---|---|
| `SQLITE_TENANT_DATABASE_DIR` | **Removed** |
| `DATABASE_ENGINE` | `postgresql` only; any other value fails fast |
| `EDE_TEST_POSTGRES_{HOST,PORT,USER,PASSWORD}` | New — where the test harness provisions its databases |
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
| Chapter | Status | Notes |
|---|---|---|
| 1 — PostgreSQL test harness + UTC session pin | ✅ Delivered | |
| 2 — 107 fixture files onto PostgreSQL | ✅ Delivered | 94 suites migrated in waves; the heterogeneous tail by hand |
| 3–5 — Delete the runtime engine, settings and dialect branches; CI on `postgres:16` | ✅ Delivered | |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Capability | Notes |
|---|---|
| One engine, end to end | `grep -rIn "sqlite" src/ede` returns nothing |
| Shared PostgreSQL test harness | `src/tests/_postgres.py`; suite-wide connection disposal |
| Sessions pinned to UTC | Engines connect with `-c timezone=utc` |
| CI on PostgreSQL | `postgres:16` service with a health check |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Notes |
|---|---|---|
| Running the suite needs a live PostgreSQL | 🟡 Accepted | A workflow change for every contributor, accepted when the change was scoped |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- SYNC-BLOCK: pitfalls -->
- **Opening a provider per test.** It exhausts the server's connection limit. Use the shared harness.
- **Assuming a tz-aware value round-trips.** It does now, because the session is pinned to UTC — that pin is load-bearing, not cosmetic.
<!-- /SYNC-BLOCK -->

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
Four real product bugs surfaced during the move and were fixed rather than papered over:

- **Timezone shift** — tz-aware writes into `timestamp without time zone` shifted by the session timezone. Engines now pin `timezone=utc`.
- **`read_group` date truncation** — SELECT and GROUP BY were built separately, producing `$1` vs `$2` and a broken grouping. GROUP BY now reuses SELECT's expression objects.
- **Approval recall decisions silently dropped** — `""` written into a nullable FK, with the exception swallowed.
- **A `seed_user` helper that lied** — it returned a `record_uuid` the repository had not used, leaving dangling join rows.

These were invisible on SQLite. Finding them is part of what the change bought.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
_none_
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Platform Change 13 — Postgres-only persistence](platform-13-postgres-only-persistence.md) — the first half.
- [Platform Change 18 — Runtime schema upgrades](platform-18-runtime-schema-upgrades.md) — what this unblocked.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-09-07. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
