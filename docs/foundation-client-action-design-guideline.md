# Client Action — Design Guideline

**Status:** Active · **Layer:** `foundation.presentation` (frontend) · **Applies to:** every client action mounted through `ClientActionPage`

A *client action* (`action_type="client"`) mounts a bespoke React component into the workspace instead of the generic ListView / FormView — Settings, the Subscription & Usage dashboard, the Approval Inbox, the Storage Drive, and so on. Because each one is hand-built, they drift visually unless they share a design contract. This document is that contract: the tokens, layout patterns, components, and states every client action uses so they read as one product, not a folder of one-off screens.

Its companion, [`foundation-client-action-authoring-guideline.md`](foundation-client-action-authoring-guideline.md), covers the *engineering* rules (descriptor, mount surfaces, states to handle, checklist). Read this one for **what it should look like**; read that one for **how to wire it up**.

> **Reference mockups** (validated 2026-07-03): the *General Settings* client action (settings pattern) and the *Subscription & Usage* client action (dashboard pattern). Both were built to this guideline and are the canonical visual examples.

---

## 1. The chrome contract — never repaint the shell

The app shell is fixed. A client action renders **only into the content pane below the module nav**. It must never redraw, restyle, or overlap:

- **Top bar** — organization switcher · app switcher · utility icons · avatar. Sticky, `z-index: 30`.
- **Module nav** — the horizontal tab strip (`General settings`, `Imports & exports`, …). Sticky under the top bar, `z-index: 20`.
- **Left settings rail** (when present) — grouped navigation into the app's sub-pages.

The action owns the pane, and nothing above it. If you find yourself styling the top bar or the module nav, stop — that is a shell concern, not a client-action concern.

---

## 2. Design tokens — never hardcode

Every colour, space, radius, and type size comes from a token in the client-action SCSS layer (`src/frontend/src/theme/`). Raw hex (`#3b82f6`), raw px (`13px`), and inline style values are prohibited — see the frontend rules in the root `CLAUDE.md`. If a value you need has no token, add the token; don't inline the value.

### Surfaces
| Token | Role |
|---|---|
| `--surface-canvas` | The pane background (behind cards) |
| `--surface-panel` | Cards, tiles, inputs, the shell bars |
| `--surface-sunken` | Wells: meter tracks, code chips, active rail item |
| `--surface-hover` | Hover state on interactive rows / icons |

### Text
| Token | Role |
|---|---|
| `--text-primary` | Titles, values, labels |
| `--text-secondary` | Subtitles, descriptions, secondary button text |
| `--text-muted` | Helper text, metadata, placeholders, counts |
| `--text-on-accent` | Text on a filled accent button |

### Borders, radius, elevation
| Token | Role |
|---|---|
| `--border-subtle` | Hairline dividers, card borders |
| `--border-default` | Input borders |
| `--border-strong` | Input hover, toggle track |
| `--radius-sm` (3px) | Chips, meter tracks |
| `--radius-md` (5px) | Inputs, buttons, rail items |
| `--radius-lg` (6px) | Cards, tiles, panels |
| `--radius-xl` (8px) *(propose)* | Hero banners — the largest radius allowed; add if absent |
| `--shadow-1` | Resting card elevation |
| `--shadow-2` *(propose)* | Overlays, drawers, dialogs — add if absent |

> **Tight corners.** Client actions use **crisp, small radii** — the scale tops out at 6–8px. Do not use large rounding (12px+ / `rounded-lg`-style); it reads consumer-app, not enterprise. Fully-round elements (pills, status chips, meter fills, toggles) stay at `999px`.

### Spacing & sizing
The `--space-1 … --space-5` scale on an 8px rhythm carries all gaps and padding. Page-level bands (group separation, dashboard row gaps) compose from these; if a larger step is genuinely needed, add `--space-6`+ to the scale rather than inlining `32px`. Control heights use `--h-control-sm`.

### Type
| Token | Role |
|---|---|
| `--fs-2xs` | Uppercase eyebrows, counts, chips |
| `--fs-xs` | Helper text, descriptions |
| `--fs-sm` | Labels, body, buttons, table cells |
| `--fs-base` | Emphasis body, subtitles |
| `--fs-md` | Group / panel titles |
| `--fs-lg` / `--fs-xl` *(propose)* | Page title · dashboard display numerals — add if absent |

Use `--font-mono` for model keys, config keys, and IDs. Digits that line up in columns get `font-variant-numeric: tabular-nums`.

### Accent & semantic colour
| Token | Role |
|---|---|
| `--accent-blue` | **Primary action + selection state** (buttons, chart line, active nav) |
| `--accent-green` / `-soft` | Semantic **success** — healthy meters, positive deltas, "Active" pills |
| `--accent-amber` / `-soft` | Semantic **warning** — meters approaching a limit |
| `--accent-red` / `-soft` | Semantic **danger** — over-limit, validation errors |

> **Decision (2026-07-03):** client-action **primary actions use `--accent-blue`**, unified with the chrome's selection state. This supersedes the legacy `--accent-orange` "primary" token *for client actions* — orange is not used as the primary action colour here. Keep the accent to one job; health/status colour is semantic and must stay off the accent so state reads cleanly.

---

## 3. Page header — every client action has one

The pane opens with a header. This closes a real gap: page-mode client actions historically had no header chrome.

- **Left:** title (`--fs-lg`, bold, `--text-primary`) + one-line subtitle (`--fs-base`, `--text-secondary`). Sentence case.
- **Right:** the pane's actions, primary **right-most**. Primary is a filled `--accent-blue` button; secondary is a default/outline button to its left. A muted status hint (e.g. "Last saved 2 hours ago") sits left of the buttons.
- Imperative verbs only: "Save changes", "Manage plan" — never "Submit" / "OK".
- One primary per pane.

The **settings pattern is the exception**: settings auto-save, so those pages carry a title + subtitle and **no** Save/Cancel (see §4.1).

---

## 4. The three layout patterns

Pick the pattern that fits the action's job. All three share the chrome, tokens, and page header above.

### 4.1 Settings pattern — three-column bands, auto-save
For configuration screens (`ir.config`-style). The pane is a stack of **group bands**, each a three-column grid:

```
┌ group meta (≈268px) ─┬─ setting list (1fr) ──────────────┬─ control (right) ┐
│ ▎ Group title        │  Setting label  🌐                │  [ select ▾ ]    │
│ short description     │  muted description of the setting  │                  │
│ N settings            │  ──────────────────────────────── │  [ toggle ]      │
└──────────────────────┴────────────────────────────────────┴──────────────────┘
```

- **Meta column:** a 3px `--accent-blue` accent bar + bold title, a short description, and an italic "*N settings*" count.
- **Setting rows:** bold label + a scope icon (globe = organization-wide) + a muted one-to-two-line description. Controls pin to the **right edge** — no dead right-hand gutter.
- Dividers: `--border-subtle` between settings within a group; a full-width hairline between groups.
- **Auto-save:** each control persists on change. No global Save button.
- **Full-width** — never a narrow centred column; the right-pinned control is what balances the row.

> **Fill the pane.** The content owns the **full width** of the pane (pane padding is the only inset) — **no `max-width` cap**. A capped column on a wide screen leaves a dead right-hand gutter, which reads as broken. This applies to every pattern: settings rows pin controls right, dashboards let tiles/panels stretch, form cards span the width.

### 4.2 Form pattern — labeled rows, header actions
For create/edit of a record or a bounded config object. Two-column rows: label + helper on the left, control on the right, grouped in cards. Actions live in the **page header, top-right** (§3), primary right-most. Every field has a **visible label** (never placeholder-as-label); required marked with `*`.

### 4.3 Dashboard pattern — summary first, information-dense
For read-mostly overview screens (usage, KPIs, health). Information design takes over from form layout:

- **Summary before detail:** a hero band (e.g. plan banner) → headline stat tiles → trend + breakdowns. The reader gets the state in one glance.
- **Encode state in form, not just number:** meters change colour by health (green/amber/red); deltas ride semantic chips.
- **Charts get real care** (§5.6).
- **Spend colour in one place:** exactly one multi-hue moment (e.g. a storage stacked bar); everything else stays calm.

---

## 5. Component specs

### 5.1 Buttons
| Variant | Use | Style |
|---|---|---|
| Primary | The one main action per pane | filled `--accent-blue`, `--text-on-accent` |
| Default | Secondary actions | `--surface-panel` + `--border-default`, `--text-secondary` |
| Subtle | Low-emphasis (Cancel) | transparent, hover `--surface-hover` |

Ordering: in a **page header / modal footer**, primary is **right-most**. Never disable the primary to signal an incomplete form — validate on submit and explain the fix.

### 5.2 Fields (input / select)
Height from `--h-control-sm`, `--border-default`, `--radius-md`, `--fs-sm`. Focus: 2px `--accent-blue` ring, 1px offset, border turns accent. Numeric inputs are right-aligned with `tabular-nums`. Selects carry the chevron affordance.

### 5.3 Validation
Helper text → inline error in `--accent-red` with an icon; the field border turns red (`is-error`). Error copy says what's wrong and how to fix it ("Must be 8 or more."). The control is **never disabled**.

### 5.4 Toggle
40×22 track; off = `--border-strong`, on = `--accent-green`. Focus ring on the track. Use for boolean settings.

### 5.5 Meters, tiles, chips, pills
- **Meter:** 8px `--surface-sunken` track, fill coloured by **health** (`--accent-green` healthy, `--accent-amber` approaching, `--accent-red` over).
- **Stat tile:** uppercase eyebrow + optional icon → big value (`--fs-xl`, `tabular-nums`) → meter or sparkline → footer with a muted note + a delta chip.
- **Chip:** small pill for deltas — `--accent-green/-soft` positive, `--accent-amber/-soft` warning, `--accent-red/-soft` negative.
- **Status pill:** e.g. "Active" — `--accent-green/-soft` with a leading dot.

### 5.6 Charts (dataviz)
- Faint gridlines in `--border-subtle`; brand line in `--accent-blue`, 2.5px.
- Area fill = a top-down `--accent-blue` gradient fading to transparent.
- **Emphasize the endpoint** (a ringed dot at the latest value).
- Axis labels in `--text-muted`, `--fs-2xs`, `tabular-nums`.
- Categorical breakdowns use a small qualitative palette (4–5 distinct hues) and appear **once** per screen; sequential/share bars use a single accent hue.

### 5.7 Section message
Mandatory icon + bold title + neutral body (prose stays `--text-primary`, **not** tinted to the background). Types: information / success / warning / danger, mapped to the semantic soft backgrounds.

---

## 6. States a client action must present

| State | Treatment |
|---|---|
| Loading | Centred spinner + muted label — not bare "Loading…" text |
| Empty | Sentence-case heading + one-line explanation + one primary next-step CTA |
| Forbidden (RBAC) | Distinct message naming the missing capability — not the same blob as "unknown" |
| Error | Card with title, detail, and a "Try again" / "Reload" action |
| Bad param | Clear "this action was opened incorrectly" message with the offending key |

Differentiate them — never collapse all five into one centred grey line.

---

## 7. Accessibility baseline
- Every control shows a **2px `--accent-blue` focus ring** at 2px offset.
- **Tab order follows visual order**; selects, numeric inputs, toggles, and buttons are all keyboard-reachable.
- `Esc` dismisses overlays (dialog-mounted actions, the design-notes drawer in mockups).
- Respect `prefers-reduced-motion`; meet contrast on text and semantic colours.

---

## 8. Quick do / don't
- **Do** render only into the pane; keep the shell untouched.
- **Do** drive every value from a token; add a token before inlining a value.
- **Do** keep `--accent-blue` for the one primary action; keep health colour semantic and separate.
- **Do** pick a pattern (settings / form / dashboard) and follow its structure.
- **Don't** cap the content with a `max-width` — fill the full pane width; a centred column leaves a dead right gutter.
- **Don't** use large corner radii (12px+) — keep corners tight (≤ 6–8px); pills/meters stay fully round.
- **Don't** disable the primary button to signal an incomplete form.
- **Don't** scatter colour; spend it in one deliberate place per screen.
- **Don't** ship only the happy path — handle loading / empty / forbidden / error / bad-param.
