# Checkpoint — foundation.customization Phase 4 (2026-06-12, end of session)

Session-handoff checkpoint: pull on the next machine, read this, resume at
**Next up**. Everything below is committed on `main` up to `630d3b29`.

---

## Where things stand

| Slice | Status | Evidence |
|---|---|---|
| 4A — anchored `<property>` + property-bag registry | 🟡 | Substrate delivered (`033e2537`, 2026-06-11); live usage now ships via 4B demo (Vendor Code on the rate form). Open: see follow-ups ledger. |
| 4B — `ir.application.view` DB-backed views | ✅ Delivered 2026-06-12 | `458ffaed` (+ `f78becd5` registry self-mirror, `630d3b29` frontend). Migrations `a6aef4be0cf2` + `f3ad3bee60fd` applied on tenant `dharmangsoni`; demo loaded; browser walkthrough user-confirmed. 3779 pytest + 560 vitest green. |
| 4C — AI-assisted customization | 🔴 Next | [Feature file](../../../roadmap/foundation/customization/phase-4/03-ai-driven-view-customization.md) · design spec §6. |

This session's commits: `458ffaed` (Phase 4B — model/hooks/ViewSync/ViewComposer/cutover/grammar/admin/demo/migrations + server-side action domain + server-grouped action_records), `f78becd5` (RegistrySync mirrors its own `ir.model*` rows), `630d3b29` (`widget="code"` CodeMirror + server-grouped ListView).

## New-machine setup (office laptop)

1. `git pull`
2. `cd src/frontend && bun install` — new deps: `@uiw/react-codemirror`, `@codemirror/lang-xml`, `@codemirror/lang-json`
3. `ede migrate upgrade -t <your-tenant>` — applies the two new revisions if that machine's tenant DB hasn't seen them, creates the 3 mirror `ir.model` rows (kills the view-sync warnings), re-syncs views (stamps `module_key`)
4. Sanity: `./run_tests.sh` → 3779 pytest; `cd src/frontend && bun run build && bun run test` → 560 vitest
5. The throwaway dev server from this session (port 8002) lives on the old machine only — nothing to carry over

## Next up — Phase 4C (queued by user: 4C → Phase 2 → Phase 3)

Per [03-ai-driven-view-customization.md](../../../roadmap/foundation/customization/phase-4/03-ai-driven-view-customization.md) + spec §6. Roadmap entry already exists and is user-approved; implementation can start directly.

Scope recap:
- **Customization skill pack** (`src/ede/foundation/assistant/`): expose host `model_key`, current `view_key`, existing property schema; proposer tool `propose_custom_field` emitting the structured op (read-only contract preserved — BR-AI-01).
- **Frontend confirm card** (`src/frontend/src/managers/assistant/` — note: the skill doc's `src/frontend/src/assistant/` path is stale post-revamp): render the ✨ op; on confirm dispatch TWO `ede.create`s through the command bus (property definition + selections, then the `ir.application.view` extension row `mode=extension owner=user` with the `<xpath>` placing `<property name="properties:<key>"/>`), then re-compose/refetch (BR-AI-02/03).
- **`scope=organization`** resolves `organization_id` from the acting principal (BR-AI-04).
- xpath safety is DONE (4B shipped BR-AV-04 dry-run + compose-time graceful skip) — only its 4C-named tests remain.
- Tests per the feature file: proposal-shape + zero-mutating-dispatch, write path + RBAC denial, xpath guard, vitest confirm card, e2e extension of `test_customization_property.py`.

Implementation note: a 3-agent exploration of the assistant surfaces (backend pack registry / frontend op rendering / write-path plumbing) was started this session and **stopped before producing results** — redo that mapping first on resume (`register_skill_pack`, `@api.ai_tool`, view-intent op transport, where ops render with the ✨ marker, how the frontend dispatches `ede.create` + surfaces RBAC errors).

**Gotcha for the confirm card:** the frontend needs the open form's `view_key` + its `ir.application.view` parent uuid to build the extension row — check whether `presentation.load_action` exposes the resolved view_key per view type to the client; if not, that's a small 4B-side addition 4C needs first.

## Then — Phase 2, then Phase 3

- **Phase 2 — JSONB-path search domains** ([roadmap](../../../roadmap/foundation/customization/phase-2/README.md)): domain compiler `properties.<key>` leaf, PG GIN index, auto property filters in SearchPanel. Unblocks 4A's "search property filtering" caveat.
- **Phase 3 — runtime-DDL manual fields**: deferred-by-design; **confirm scope with the user before starting** (their queue note said so).

## Open follow-ups ledger (not blocking 4C)

- **4A leftovers** (tracked in [01-property-element-and-bag-registry.md](../../../roadmap/foundation/customization/phase-4/01-property-element-and-bag-registry.md)): list/kanban cell value-extraction from `record[bag][key]`; `test_property_fielddef_synthesis.py`; frontend property vitest.
- **`ir.action.view.view_id` Char drop** — follow-up revision once all deployments have run the sync backfill; fold in the missed NOT NULL→nullable relaxation (autogen gap).
- **Kanban server-grouping** — kanban still client-buckets its columns from the flattened rows; adopting the grouped payload per column is a natural follow-up.
- **Composer/strip-pass perf** — compose results are generation-cached; the predicate strip-pass remains per-request and uncached (deferred from presentation Enh 04).
- **PROGRESS.md** — this session's work (4A status sync, 4B full build, bug fixes, code widget) is not yet logged; user hasn't said to log it.

---

*Roadmap truth lives in [phase-4/README.md](../../../roadmap/foundation/customization/phase-4/README.md) + [roadmap-tracker.md](../../../roadmap/roadmap-tracker.md); this file is a session handoff, not a status site.*
