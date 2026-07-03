# Client Action — Authoring Guideline

**Status:** Active · **Layer:** `foundation.presentation` (frontend) · **Applies to:** every `action_type="client"` action

This is the engineering contract for building a client action. Its companion, [`foundation-client-action-design-guideline.md`](foundation-client-action-design-guideline.md), covers **what it should look like**; this one covers **how to wire it up and what it must handle** before it ships. Follow both — a client action is done only when it satisfies the checklist in §9.

---

## 1. What a client action is

A client action mounts a bespoke React component into the workspace instead of the generic ListView / FormView. It funnels through one host, `ClientActionPage`, which owns registry lookup, param validation, the RBAC gate, the `onOpen` gate, `Suspense`, the error boundary, and the lifecycle-veto machinery. You write the component; the host gives it a home.

**Two mount surfaces, one host:**
- **Page** — full-pane, reached by navigation. Navigation *is* the intent, so there is no close-veto.
- **Dialog** — mounted in `DialogActionHost` (Radix dialog). `Esc` / outside-click / the close ✕ run the `onClose` veto chain first.

Key files:
- `src/frontend/src/managers/ClientActionPage.tsx` — the host + `useClientActionLifecycle` / `useClientActionTitle`.
- `src/frontend/src/managers/ClientActionLayoutRenderer.tsx` — optional DSL-driven chrome (header / body / footer slots).
- `src/frontend/src/managers/ClientActionRegistry.tsx` — the descriptor.
- `src/frontend/src/managers/DialogActionHost.tsx` / `DialogActionStack.tsx` — the dialog surface + stack.

---

## 2. The descriptor

Register the action with a descriptor (or the bare `(key, component)` form). Fields that drive behaviour:

| Field | Purpose |
|---|---|
| `key` | Stable action key (`module.action_key`) |
| `component` | The React component; may be `lazy()` for a code-split chunk |
| `paramSchema` | Shape of the params (zod, or `uuid` / `uuid?` / `any`) — validated before mount |
| `requires` | RBAC capability codes gated before mount |
| `onOpen` | Async gate run before the component renders (prefetch / precondition) |
| `layout` | `{ modelKey, viewId? }` to paint DSL chrome via the layout renderer |

Keep params minimal and validated — the host rejects a bad param shape with a dedicated state (§6), so never hand-parse `window.location` inside the component.

---

## 3. The chrome contract

Render **only into the content pane**. Never repaint the top bar, module nav, or (when present) the left rail. Those are the app shell; a client action that touches them is wrong. See the design guideline §1.

---

## 4. Pick a pattern

Choose one and follow its structure (design guideline §4):

| Pattern | Use for | Persistence | Actions |
|---|---|---|---|
| **Settings** | `ir.config`-style configuration | **Auto-save** on change | none (no Save button) |
| **Form** | Create / edit a record or config object | Explicit **Save** | page header, top-right |
| **Dashboard** | Read-mostly overview (usage, KPIs, health) | n/a (read) | page header, top-right |

Do not blend persistence models: a settings screen auto-saves and has no Save button; a form screen saves explicitly via the header primary. Mixing them confuses the user about when their change took effect.

---

## 5. Page header & actions

Every client action opens with a page header (design guideline §3):
- Title + one-line subtitle on the left.
- Actions top-right, **primary right-most**, `--accent-blue`, imperative verb, one primary per pane.
- Settings pattern: **no** action buttons (auto-save) — title + subtitle only.

For a dialog-mounted **form**, the footer follows the same order (primary right-most). The dialog's own close ✕ / `Esc` route through the `onClose` veto before dismissing.

---

## 6. States you MUST handle

The happy path is not enough. The host distinguishes these; your component (and its empty/loading views) must present each distinctly — never one shared grey blob:

| State | Where it comes from | Requirement |
|---|---|---|
| **Loading** | `onOpen` / layout fetch / `Suspense` | Spinner + muted label, not bare "Loading…" |
| **Empty** | Component has no data to show | Heading + explanation + one primary next-step CTA |
| **Forbidden** | `requires` RBAC gate | Message naming the missing capability |
| **Error** | Error boundary | Card with title + detail + Try again / Reload |
| **Bad param** | `paramSchema` validation | Clear "opened incorrectly" message + the offending key |

Loading and error placeholders are owned by the host; empty and forbidden copy are yours to make specific to the action.

---

## 7. Validation (form & settings)

- Every field has a **visible label** (never placeholder-as-label); required fields marked `*`.
- On invalid input, turn helper text into an inline error (`--accent-red`) with an icon and a red field border. Say what is wrong and how to fix it.
- **Never disable the primary/submit** to signal an incomplete form — validate on submit and point to the fix.

---

## 8. Styling & accessibility rules

- **Tokens only.** No raw hex, no raw px, no inline style values. Use the client-action theme tokens (design guideline §2); if a token is missing, add it — don't inline. This mirrors the root `CLAUDE.md` frontend rules.
- **Tight radii, full width.** Corners ≤ 6–8px; content fills the pane with no `max-width` cap.
- **Focus & keyboard.** 2px `--accent-blue` focus ring at 2px offset on every control; tab order = visual order; every control keyboard-reachable; `Esc` dismisses dialog-mounted actions.
- **Reduced motion & contrast.** Respect `prefers-reduced-motion`; meet contrast on text and semantic colours.

---

## 9. Definition of done — checklist

A client action is shippable only when all of these hold:

- [ ] Registered with a descriptor; `paramSchema` and `requires` declared.
- [ ] Renders only into the pane — shell untouched.
- [ ] Opens with a page header (title + subtitle); actions top-right with a single `--accent-blue` primary, right-most — **or** no actions for the auto-save settings pattern.
- [ ] Follows exactly one pattern (settings / form / dashboard) and its persistence model.
- [ ] Presents all five states distinctly: loading · empty · forbidden · error · bad-param.
- [ ] Fields have visible labels; validation is inline; the primary is never disabled to signal incompleteness.
- [ ] Every value is a token; corners are tight; content fills the full pane width.
- [ ] Keyboard-navigable with a visible focus ring; `Esc` dismisses (dialog); reduced-motion respected.
- [ ] Verified in both light context and (where relevant) the dialog surface.

---

## 10. Reference implementations

- **Settings pattern** — the *General Settings* client action (three-column bands, auto-save).
- **Dashboard pattern** — the *Subscription & Usage* client action (summary-first, meters, semantic colour, area chart).

Both were built to these guidelines on 2026-07-03 and are the canonical examples.
