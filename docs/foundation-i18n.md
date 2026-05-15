<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Internationalization (i18n) — Implementation Docs

**Module theme:** `foundation.i18n` (theme — code lands in `foundation.base` for the master, with FK retrofits in consumers)
**Roadmap:** [roadmap/foundation/i18n/README.md](../roadmap/foundation/i18n/README.md)
**Status:** ✅ Phase 1 Delivered (Phases 2 & 3 not started)
**Layer:** Foundation theme

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A set of platform reference data and runtime services for internationalization. Phase 1 ships `res.language` — a BCP-47-keyed master with native names, ISO 639-1/2 codes, and text direction — plus a nullable `language_id` Reference on `res.user` and a `DEFAULT_LANGUAGE_CODE` foundation setting. Future phases layer runtime translation tables (`ir.translation`), locale-aware notification template fallback, locale-aware date/number/currency formatters, and UI translation pipelines on top of the language master.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
EDE has no foundation language master today. The notifications engine carries `locale_code` as a free `Char` field with no FK target, `res.user` has no per-user locale preference, and downstream consumers (email templates, document rendering, customer portals) have nowhere to register language metadata. Every i18n feature is blocked behind this — locale-aware template fallback, RTL text direction handling, per-user language preferences. `res.language` is the cheapest unlock for all of these and lives in `foundation.base` because every ERP domain that emits user-visible text needs it.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End user (admin)** — Settings → Localization → Languages renders the BCP-47 catalogue list; admins can create / edit / archive rows.
- **End user (any signed-in user)** — Profile → Preferred Language picks one of the active `res.language` rows; selection persists on `res.user.language_id`.
- **Programmatic consumer** — read `env.principal.user_id` → load `res.user.language_id` → fall back to `settings.DEFAULT_LANGUAGE_CODE` when null. Notification dispatcher (Phase 2) and email template engine (Phase 2) will use this lookup chain; Phase 1 just ships the data + FK so consumers can start reading.
- **Integration boundary** — Phase 1 produces the `res.language` master and the `res.user.language_id` FK; it consumes nothing beyond standard `res.*` plumbing. Phase 2 introduces `ir.translation` and locale-aware template fallback; Phase 3 introduces formatters + UI bundles.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
            ┌──────────────────────────────────────┐
            │  res.language  (foundation.base)     │
            │  ────────────────────────────────    │
            │  code (BCP-47)  · name · native_name │
            │  iso_639_1 · iso_639_2 · direction   │
            │  is_active                           │
            └──────────────┬───────────────────────┘
                           │
       ┌───────────────────┼───────────────────────┐
       ▼                   ▼                       ▼
  res.user.language_id  ir.notification.template  mail.template
  (Phase 1)             .locale_code              .locale_code
                        (Phase 2 — FK validation) (Phase 2)

  Fallback chain at consumer read time:
    user.language_id   →   settings.DEFAULT_LANGUAGE_CODE   →   "en"
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `res.language` | BCP-47 language master — code, name, native_name, ISO 639-1/2, direction, is_active | [src/ede/foundation/base/models/language.py](../src/ede/foundation/base/models/language.py) |
| `ir.translation` *(planned, Phase 2)* | Runtime translation rows keyed by (model_key, field, language_id, source_id) | _Phase 2_ |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ | Phase 1 ships master data + FK only — no new services. Phase 2 introduces a translation lookup helper and dispatcher locale-fallback hook. | |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none yet_ | Phase 1 uses generic CRUD only — no domain-specific commands needed. | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none yet_ | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| Generic CRUD via `ede.create` / `ede.read_one` / `ede.update` / `ede.delete` / `ede.search` on model_key `res.language` | Standard `res.*` access — admin form/list views go through these | platform `CrudKernel` |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
`res.language.is_active` is a soft-archive flag. Inactive rows are hidden from picker UIs by default and excluded from notification template lookup. There is no transition state machine beyond the boolean.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Empty rows fine; missing sections an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry: no new entry. `res.language` ships under existing `foundation.base`.
- `ACTIVE_DOMAINS` entry: n/a (foundation theme).
- Manifest `depends`: no change to `foundation.base`'s manifest.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `DEFAULT_LANGUAGE_CODE` | str | `"en"` | `DEFAULT_LANGUAGE_CODE` | System fallback when a user has no `language_id` set and a downstream consumer (notification dispatcher, email template engine) needs a locale |
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
| [`src/ede/foundation/base/data/res.language.csv`](../src/ede/foundation/base/data/res.language.csv) | 93 BCP-47 rows derived from Odoo 19 `res.lang.csv` — full coverage including 14 Spanish regional variants (es-AR/BO/CL/CO/CR/DO/EC/ES/GT/MX/PA/PE/PY/UY/VE), 6 English variants (en-US/AU/CA/GB/IN/NZ), 4 French variants (fr-FR/BE/CA/CH), 3 Chinese variants (zh-CN/HK/TW), 2 Korean (ko-KP/KR), 2 Portuguese (pt-PT/BR/AO), 2 German (de-DE/CH), both Serbian scripts (sr-Cyrl/latin), Indic (hi, gu, bn, ml, te), Southeast Asian (vi, th, id, ms, km, lo, my, tl), Slavic (ru, uk, pl, cs, sk, sl, hr, bs, mk, bg, be), Nordic (sv, nb, da, fi), and 4 RTL (`ar`, `ar-SY`, `fa-IR`, `he-IL`). Plus African + Caucasian (am, sw, kab, ka, hy). All `is_active=true`. |
| [`src/ede/foundation/base/data/ir.rbac.permission.csv`](../src/ede/foundation/base/data/ir.rbac.permission.csv) | 4 permissions: `res.language.read` (internal user), `res.language.create/update/delete` (system admin) |
| [`src/ede/foundation/base/data/base_menus.xml`](../src/ede/foundation/base/data/base_menus.xml) | `base.action_res_language` + `base.menu_settings_languages` under Settings → Localization |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Foundation Language Master                | ✅ Delivered | [phase-1-implementation.md](../roadmap/foundation/i18n/phase-1-implementation.md) |
| Phase 2 | Locale-Aware Notifications + Translations | 🔴 Not Started | _Phase 2 plan to be written when Phase 1 lands_ |
| Phase 3 | Locale-Aware Formatters & UI Translations | 🔴 Not Started | _Phase 3 plan to be written when Phase 2 lands_ |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| WS-I1 — `res.language` BCP-47 master + 93-row seed (Odoo-derived) | `res.language` | [`models/language.py`](../src/ede/foundation/base/models/language.py) · [`data/res.language.csv`](../src/ede/foundation/base/data/res.language.csv) | [phase-1 WS-I1](../roadmap/foundation/i18n/phase-1-implementation.md) |
| WS-I2 — `res.user.language_id` Reference + `DEFAULT_LANGUAGE_CODE` setting | `res.user` (extended) | [`models/user.py`](../src/ede/foundation/base/models/user.py) · [`settings.py`](../src/ede/foundation/settings.py) | [phase-1 WS-I2](../roadmap/foundation/i18n/phase-1-implementation.md) |
| WS-I3 — admin views + RBAC + Settings menu | n/a | [`views/res_language_views.xml`](../src/ede/foundation/base/views/res_language_views.xml) · [`data/ir.rbac.permission.csv`](../src/ede/foundation/base/data/ir.rbac.permission.csv) · [`data/base_menus.xml`](../src/ede/foundation/base/data/base_menus.xml) | [phase-1 WS-I3](../roadmap/foundation/i18n/phase-1-implementation.md) |
| Migration `a8c3f7e21d59` — additive (new table + new column + index + FK) | n/a | [migration file](../src/ede/foundation/base/migrations/versions/a8c3f7e21d59_foundation_base_res_language_user_lang.py) | phase-1 §E2E |
| Enhancement 01 — retire legacy `res.user.lang` Char; backfill `language_id` from `lang` then drop the column; register accepts `language_id` (preferred) or `lang` (resolved server-side via `res.language.code` lookup, raises on miss); auth `/me` returns `language_id` instead of `lang` | `res.user` (`lang` removed; `language_id` is now the only language field) | [`models/user.py`](../src/ede/foundation/base/models/user.py) · [`auth/api/me.py`](../src/ede/foundation/auth/api/me.py) · [`views/res_user_views.xml`](../src/ede/foundation/base/views/res_user_views.xml) · [migration `f2a4c8b13e95`](../src/ede/foundation/base/migrations/versions/f2a4c8b13e95_foundation_base_drop_res_user_lang_char.py) | [enhancements/01](../roadmap/foundation/i18n/enhancements/01-retire-legacy-lang-char.md) |
| Enhancement 02 — user-timezone-aware DateTime display & input; backend serializer emits `+00:00` UTC marker on naive datetimes (was previously parsed as local browser time by JS); frontend `utils/datetime.ts` built on `Intl.DateTimeFormat` exposes `formatDateTime` / `toInputLocal` / `fromInputLocal` (DST-aware via two-pass offset lookup) / `formatRelative`; `WorkspaceClient` pushes `bootstrap.user.timezone` into an ambient setter so class-based `FieldComponent` subclasses can read it; `DateTimeField` view mode displays in user TZ + edit mode round-trips through user-TZ wall-clock with a `(<IANA>)` suffix label; `ApprovalCaseView` + `UserNotification` adopt the shared util | none (DateTime field type only — wiring change) | [`generic_repo.py`](../src/ede/core/adapters/persistence/sqlalchemy/generic_repo.py) · [`utils/datetime.ts`](../src/frontend/src/utils/datetime.ts) · [`DateTimeField.tsx`](../src/frontend/src/workspace/views/fields/DateTimeField.tsx) · [`WorkspaceClient.tsx`](../src/frontend/src/workspace/components/WorkspaceClient.tsx) · [`UserNotification.tsx`](../src/frontend/src/workspace/components/header/UserNotification.tsx) | [enhancements/02](../roadmap/foundation/i18n/enhancements/02-display-datetimes-in-user-timezone.md) |

> Phase 1 ✅ Delivered (2026-05-09). Backend + frontend wiring complete; full test suite (1173 Python + 343 frontend) green; 9 test files updated to register `Language` ahead of `User` so the `language_id` FK resolves cleanly. Browser walkthrough of Settings → Localization → Languages and the user-form Preferred Language picker confirmed.
> Enhancement 01 ✅ Delivered (2026-05-10). 1368 pytest (+3 new register-with-language_id cases) + 343 vitest green. Forward-only migration; downgrade raises `NotImplementedError` because the original free-form `lang` strings are unrecoverable from the FK after the column drop.
> Enhancement 02 ✅ Delivered (2026-05-10). 1418 pytest (+5 new generic-repo serialization cases) + 380 vitest (+35 new datetime util cases + 4 updated DateTimeField cases) green; `bun run build` clean (2830 modules, 0 errors); manual browser walkthrough confirmed by user. No schema change, no Alembic migration, no auto-backfill — pre-existing data mis-entered through the broken editor stays as-is by deliberate choice (silently shifting per-user offsets would corrupt records that happened to be entered correctly).
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| ~~`res.user.lang` legacy Char remains alongside `language_id`~~ — ✅ closed 2026-05-10 by Enhancement 01: backfilled `language_id` from `lang`, dropped the column, register accepts both `language_id` and (legacy) `lang` payload keys | ✅ Closed | [enhancements/01-retire-legacy-lang-char.md](../roadmap/foundation/i18n/enhancements/01-retire-legacy-lang-char.md) |
| ~~DateTime values render in browser TZ, not user TZ; editor saves naive picker output as raw UTC (silent shift on every edit)~~ — ✅ closed 2026-05-10 by Enhancement 02: backend serializer attaches `+00:00` to naive datetimes; frontend `utils/datetime.ts` (`formatDateTime` / `toInputLocal` / `fromInputLocal` / `formatRelative`) built on `Intl.DateTimeFormat` is DST-aware via two-pass offset lookup; `DateTimeField` view mode displays in user TZ and edit mode pre-fills via `toInputLocal` + emits via `fromInputLocal` so the form pipeline still sees only UTC; `ApprovalCaseView` + `UserNotification` consume the shared util | ✅ Closed | [enhancements/02-display-datetimes-in-user-timezone.md](../roadmap/foundation/i18n/enhancements/02-display-datetimes-in-user-timezone.md) |
| **`ir.notification.template.locale_code` not validated against `res.language`** — bare Char today; FK validation lands in Phase 2 | 🟠 High (Phase 2) | [README.md](../roadmap/foundation/i18n/README.md) |
| **No runtime translations** — no `ir.translation` table; can't translate model field values | 🟠 High (Phase 2) | [README.md](../roadmap/foundation/i18n/README.md) |
| **No notification locale fallback** — dispatcher reads `locale_code='en'` only | 🟠 High (Phase 2) | [Notifications Phase 3](../roadmap/foundation/notifications/phase-3-implementation.md) (depends on i18n Phase 2) |
| **No locale-aware number / currency formatters** — numeric/monetary values render in a single locale (date formatters are being addressed by Enhancement 02 above; number/currency remain Phase 3) | 🟡 Medium (Phase 3) | [README.md](../roadmap/foundation/i18n/README.md) |
| **No RTL UI** — `direction` is recorded but frontend does not flip layout per language | 🟡 Medium (Phase 3) | [README.md](../roadmap/foundation/i18n/README.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- **Don't bypass `res.language` with bare `Char` locale fields.** When adding a new locale-aware field to a future model, point it at `res.language.code` (or `res.language` Reference) — not a Char with implicit BCP-47 semantics. Bare Char fields are exactly the gap this module exists to close.
- **Don't add language to `foundation.i18n`'s package** — code lives in `foundation.base/models/language.py`. The roadmap is the *theme* home, not the *code* home. The roadmap README documents this on purpose so future contributors don't try to "fix" the placement.
- **Don't query `language_id` without falling back to `DEFAULT_LANGUAGE_CODE`.** Users created before Phase 1 have `language_id=null` and that's the supported steady state — the foundation setting is the always-present default, not an emergency fallback.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- **Phase 1 schema additions** — `res_language` table; `res_user.language_id` column + FK + index. Both additive; no destructive changes to existing data.
- **Phase 1 data load** — seed `res.language.csv` runs idempotently via `noupdate=0` so adding seed rows in later phases re-applies cleanly.
- **No in-place data migration on `ir.notification.template.locale_code`** in Phase 1. Phase 2 introduces validation that the value matches a `res.language.code`; until then, the engine falls back to `'en'` when the value is unknown.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `rbac.role_internal_user` | `res.language.read` (so the user-form picker works) |
| `rbac.role_system_admin` | `res.language.create` / `update` / `delete` (manage the catalogue) |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Notifications](foundation-notifications.md) — primary near-term consumer; Phase 3 of notifications depends on Phase 2 of i18n
- [Foundation Email](../src/ede/foundation/email/) — Phase 2 candidate for locale variants on `mail.template`
- [Foundation Model Naming](foundation-model-naming.md) — `res.*` vs `ir.*` conventions
- [CLAUDE.md Model Placement Test](../CLAUDE.md) — why language belongs in foundation.base
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-10 (Enhancement 02 — user-timezone-aware DateTime display & input — flipped 🟡 → ✅ Delivered after manual browser walkthrough pass: gap row struck through with "✅ closed 2026-05-10 by Enhancement 02" annotation; new Built Capabilities row added covering generic_repo serializer + utils/datetime.ts + DateTimeField view/edit conversion + ApprovalCaseView/UserNotification adoption; 1418 pytest + 380 vitest green; bun run build clean). Earlier 2026-05-10 sync: Enhancement 02 added as 🟡 In Progress with consolidated Phase 3 "locale-aware formatters" gap (narrowed to numbers/currencies since dates ship ahead of Phase 3). Earlier 2026-05-10 sync: Enhancement 01 — retire legacy `res.user.lang` Char — flipped 🔴 → ✅ Delivered. To refresh, invoke the syncing-roadmap-to-docs skill.*
