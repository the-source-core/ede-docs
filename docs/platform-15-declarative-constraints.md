<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Declarative Constraints (DB-driven + code-driven) — Implementation Docs

**Module:** `ede.core.kernel` (`constraints.*`) + `ede.core.api` (`@api.constrains`) + `ede.core.registry` + `ede.core.adapters.persistence.sqlalchemy` + `ede.core.bus.command_bus`
**Roadmap:** [roadmap/platform/15-declarative-constraints.md](../roadmap/platform/15-declarative-constraints.md)
**Status:** 🔴 Not Started
**Layer:** Platform-wide

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
Two parallel, first-class ways for a `DomainModel` to declare an invariant, replacing the rejected single `@api.constraints(..., sql=True)` mode-flag:

- **`constraints.*`** — a new kernel namespace (imported exactly like `fields`) of class-body descriptors for **database-enforced** constraints: `constraints.Check("min_qty <= max_qty", message=...)` and `constraints.Unique("code", "org_id", ...)`. Declared beside the fields they constrain, scanned by the metadata builder, emitted into a generated migration, enforced by the database against *any* writer.
- **`@api.constrains(*fields)`** — a decorator for **code-enforced** business rules (conditional, cross-field, cross-record), sugar over the existing `pre.{model}.{op}` hook machinery.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Today EDE has only single-field flags (`required=` / `unique=` / `regex=`) and hand-written lifecycle hooks. There is **no composite uniqueness** and **no table-level CHECK** the database itself guarantees — so an invariant cannot be protected against raw SQL, the data loader, or a future non-ORM writer. And ordinary business rules cost boilerplate: the same rule is written twice (`pre.create` + `pre.update`) with a manual merge of live record + incoming payload each time. This change gives each concern an honest declaration surface: `constraints.*` for database guarantees, `@api.constrains` for one-declaration code rules.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Model author (DB constraint):** `from ede.core.kernel import constraints`, then `qty_bounds = constraints.Check("min_qty <= max_qty", message="min_qty must be ≤ max_qty")` in the class body. Run `ede migrate generate` to emit the named constraint; a violation surfaces the declared `message`, not a raw driver error.
- **Model author (code rule):** decorate a method with `@api.constrains("stage", "partner_id")`; it runs on create and update with the record pre-merged, `self.env` injected, raise to veto.
- **Downstream module:** a named constraint is addressable by attribute name, symmetric with how `@api.extend_model` addresses fields (override / reference by name).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
Two independent tracks (either may ship first):

```
DB-driven (Phase 1):
  constraints.Check / .Unique  (class body)
    -> ConstraintSpec (name = attribute name)
      -> metadata builder scan (alongside FieldSpec)
        -> ede migrate generate  -> named CHECK / unique index
          -> IntegrityError (constraint name) -> ConstraintSpec.message -> ValidationError

Code-driven (Phase 2):
  @api.constrains("a", "b")  (method)
    -> registry expands into  pre.{model}.create + pre.{model}.update  hooks
      -> wrapper builds merged record + gates on trigger fields
        -> method(self, record); raise vetoes inside the txn
```

The attribute name is the constraint identity: it drives rename-safe migration diffing, the `IntegrityError`→`message` join, and cross-module addressability. A bare (unassigned) `constraints.Check(...)` in a class body is evaluated and discarded — the assignment is mechanically required for the `__dict__` scan to find it, exactly as with `fields`.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none new_ | Constraints are model *metadata*, not models. Every `DomainModel` can declare them. | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `constraints.Check` / `constraints.Unique` / `ConstraintSpec` | Class-body descriptors → constraint spec (name from attribute) | `src/ede/core/kernel/constraints.py` (new) |
| Class-scan in `register_model()` | Collect `ConstraintSpec`s from `__dict__` alongside `FieldSpec`s | [src/ede/core/registry.py](../src/ede/core/registry.py) · [src/ede/core/kernel/model.py](../src/ede/core/kernel/model.py) |
| Metadata builder | Emit `CheckConstraint` / named unique `Index`; normalize + length-validate names | [src/ede/core/adapters/persistence/sqlalchemy/metadata_builder.py](../src/ede/core/adapters/persistence/sqlalchemy/metadata_builder.py) |
| Error translation | `IntegrityError` (constraint name) → `ConstraintSpec.message` → `ValidationError` | [src/ede/core/bus/command_bus.py](../src/ede/core/bus/command_bus.py) |
| `@api.constrains` expansion | Expand decorated method into merged-record pre-create + pre-update hooks with trigger-field gating | [src/ede/core/api.py](../src/ede/core/api.py) · [src/ede/core/registry.py](../src/ede/core/registry.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none new_ | `@api.constrains` rides the existing `ede.create` / `ede.update` pre-hook path; DB constraints fire in the database. | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none new_ | | |
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
| `pre.{model}.create` + `pre.{model}.update` | Generated by `@api.constrains` expansion — merged record, trigger-field gate, raise to veto. |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
_none_
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> This is a kernel authoring surface — it introduces no runtime configuration knobs. The "configuration" is the declaration API itself (`constraints.*` and `@api.constrains`).

### Activation
<!-- SYNC-BLOCK: activation -->
- _None_ — kernel capability available to every model once shipped; no `ACTIVE_MODULES` entry, no manifest `depends`.
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
| _none_ | (Kernel capability — ships no records.) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | DB-driven constraints (`constraints.*` namespace) | 🔴 Not Started | [roadmap/platform/15-declarative-constraints.md](../roadmap/platform/15-declarative-constraints.md) |
| Phase 2 | Code-driven constraints (`@api.constrains`) | 🔴 Not Started | [roadmap/platform/15-declarative-constraints.md](../roadmap/platform/15-declarative-constraints.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from the roadmap's phase acceptance criteria.

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| `constraints.Check` (table-level SQL CHECK) — not yet built. | 🔴 Not Started | [Phase 1](../roadmap/platform/15-declarative-constraints.md) |
| `constraints.Unique` (composite uniqueness) — not yet built. | 🔴 Not Started | [Phase 1](../roadmap/platform/15-declarative-constraints.md) |
| Metadata-builder scan + `ede migrate generate` emission (named, ≤63-char, SQLite batch-rebuild) — not yet built. | 🔴 Not Started | [Phase 1](../roadmap/platform/15-declarative-constraints.md) |
| `IntegrityError` → friendly-`message` translation — not yet built. | 🔴 Not Started | [Phase 1](../roadmap/platform/15-declarative-constraints.md) |
| `@api.constrains(*fields)` decorator + pre-hook expansion (merged record, trigger-field gate) — not yet built. | 🔴 Not Started | [Phase 2](../roadmap/platform/15-declarative-constraints.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _Section reserved — will be populated once the capability lands and integration learnings emerge._

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- **Existing models:** no migration until a model adopts a constraint. Adding a `constraints.Check`/`Unique` generates one additive migration for that app (the named object); no field-column change.
- **`@api.constrains`:** pure code — no schema, no migration.
- **SQLite:** CHECK/unique adds on an existing table go through Alembic batch mode (table rebuild) per the `writing-alembic-migrations` skill; the constraint is always real on PostgreSQL.
- All migrations are **generated** via `ede migrate generate` — never hand-authored.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ | (Kernel authoring capability — no RBAC surface.) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- Hook-driven validation (`@api.on_hook`) — the mechanism `@api.constrains` is sugar over.
- `fields.*` — the sibling namespace for column/relation declarations (single-field `required=`/`unique=`).
- `writing-alembic-migrations` skill — governs cross-dialect FK/index/constraint object declaration.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-07-10. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
