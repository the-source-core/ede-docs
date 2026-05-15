# `foundation.i18n` — Demo Use-Case

**Module:** `ede.foundation.i18n` (theme)
**App key:** `foundation.base` (i18n models currently live in base — see roadmap)
**Demo manifest entries** (target): `demo/demo_languages_extra.xml`
**Status:** 🔴 Not yet authored

---

## Use-case

Production seeds ship a small core of languages (`res.language.csv`). Demo extends that set with a handful of additional locales relevant to the freight scenario (Hindi, Mandarin, Arabic, French, Spanish), and assigns one of the demo users to a non-English preferred language so the React webclient demo can show the language picker working.

## Records produced

### `demo/demo_languages_extra.xml`

| External ID | Model | Notes |
|---|---|---|
| `base.demo_language_hi` | `res.language` | code=`hi`, native_name="हिन्दी", direction=ltr |
| `base.demo_language_zh_cn` | `res.language` | code=`zh-CN`, native_name="简体中文" |
| `base.demo_language_ar` | `res.language` | code=`ar`, native_name="العربية", direction=rtl |
| `base.demo_language_fr` | `res.language` | code=`fr`, native_name="Français" |
| `base.demo_language_es` | `res.language` | code=`es`, native_name="Español" |

### Demo user preference

One user (e.g. `base.demo_user_ops_manager`) gets `language_id=ref(base.demo_language_hi)` so a tester can switch to that user and see UI strings flip locale. The assignment lives in `foundation.base/demo/demo_users.xml` (because that's where the user is declared), not here.

## Out of scope

- Translated UI string catalogues — production concern, ships via `res.lang.*` static assets, not demo data.
- Per-language `res.country.local_name` overrides — Phase 3 of foundation.i18n enhancement.
- RTL layout demo records — covered by Arabic above.

## Dependencies

- Production `res.language.csv` already loaded (so the demo additions don't collide on `code`).

## Verification

```
ede migrate upgrade -t demo --with-demo=foundation.base
```

`select code, name from res_language;` shows the demo additions. Log in as `demo_user_ops_manager` → UI strings render in Hindi (where translated).

## Authoring order

1. Demo languages loaded before `demo_users.xml` references them.

---

*Back to [demo-usecase index](../../README.md).*
