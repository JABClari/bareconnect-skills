---
name: ui-ux-interaction-audit
description: >-
  House UI & UX interaction audit for every Sirius Arc / Bareconnect frontend. Run it before a UI
  commit, at every milestone, and whenever the vendored motion skills are updated. Combines the
  transitions.dev snippet catalog (jakubantalik/transitions-dev) with Emil Kowalski's design
  engineering and animation review bar (emilkowalski/skill) into one repeatable audit + fix + verify
  loop. Triggers on "audit the UI", "UX audit", "interaction audit", "does this feel right",
  "polish the UI", "review the animations", "run the ui audit".
version: 1.0.0
sources:
  - https://transitions.dev/            # npx skills add jakubantalik/transitions-dev
  - https://emilkowal.ski/skill         # npx skills add emilkowalski/skill
---

# UI & UX Interaction Audit (house skill)

One loop, run repeatedly: **recon → audit → fix → verify → record**. It is opinionated on
purpose. The bar is Shopify-Shop-grade compactness with Emil Kowalski's motion rules and the
transitions.dev catalog as the only allowed source of transition code.

## The vendored skills this wraps (read them, don't paraphrase them)

| Skill | Use it for | Path |
|---|---|---|
| `transitions-dev` | The only place to copy transition CSS from. 12 tuned snippets + one `:root` token block. | [transitions-dev/SKILL.md](./references/transitions-dev/SKILL.md) |
| `emil-design-eng` | Philosophy + component-level polish decisions (buttons, inputs, lists, toasts, focus). | [emil-design-eng/SKILL.md](./references/emil-design-eng/SKILL.md) |
| `animate` (+ RECIPES) | Building a new animation in the right order (should it animate → purpose → tool → props → curve → interrupt → exit). | [animate/SKILL.md](./references/animate/SKILL.md) |
| `review-animations` (+ STANDARDS) | Reviewing a diff that touches motion. Hard blocks live here. | [review-animations/SKILL.md](./references/review-animations/SKILL.md) |
| `improve-animations` (+ AUDIT, PLAN-TEMPLATE) | Whole-codebase motion audit with the 8 categories and exact target values. | [improve-animations/AUDIT.md](./references/improve-animations/AUDIT.md) |
| `find-animation-opportunities` | Places that *should* animate but don't. | [find-animation-opportunities/SKILL.md](./references/find-animation-opportunities/SKILL.md) |
| `animation-vocabulary` | Shared words for describing motion precisely in findings. | [animation-vocabulary/SKILL.md](./references/animation-vocabulary/SKILL.md) |
| `pick-ui-library` / `ask-sonner` | Don't hand-roll dialogs, menus, toasts. Sonner for toasts. | [pick-ui-library/SKILL.md](./references/pick-ui-library/SKILL.md) |

## When to run

1. **Before any commit that touches UI** — quick pass (sections A–D below, HIGH findings only).
2. **At every milestone / before a demo or deploy** — full pass, all sections, findings table.
3. **Whenever the vendored skills are updated** (see *Keep it updated*) — re-run the full pass so
   new rules are applied to old screens.

## Protocol

### 0. Recon (5 minutes, always first)
- Stack + motion surface: grep for `transition`, `animation`, `@keyframes`, `motion.`, `ease-in`,
  `transition: all`, `scale(0)`, `prefers-reduced-motion`, `transform-origin`.
- Existing tokens: is the transitions.dev `:root` block installed **once** in the global stylesheet?
- Frequency map: which screens/actions are hit 100+ times a day vs occasionally vs once.
- Audience: a store owner / member / clinic staff — non-technical by default.

### A. Product feel — "Shopify Shop compact"
- **Typography:** Inter (or the project font), 13–14px body, 12px meta, tight leading, tabular
  numbers for counts and money. No display fonts in app UI.
- **Density:** 8px grid; cards 12–16px padding; row height 40–48px; one accent colour; borders
  `1px` neutral-200, radius 8–12px; surfaces white on neutral-50.
- **Hierarchy:** one primary action per screen, black/brand filled; everything else secondary or
  ghost. Destructive is red only on confirm.
- **Icons:** SVG only — **never emojis** in product UI. Consistent 16/20px stroke set.

### B. Copy for humans
- No API words in UI: no "ID", "JSON", "endpoint", "payload", "422", "token".
- Errors say what happened + what to do next: *"Black Dress could not be uploaded. Retry or skip."*
- Buttons are verbs that name the outcome: *Choose folder*, *Upload all*, *Retry failed*.
- Empty, loading, success and error states all have copy written — none are blank.

### C. States & feedback (every interactive element)
- Hover (gated by `@media (hover: hover) and (pointer: fine)`), focus-visible ring, active/press
  (`scale(0.97)`, 100–160ms), disabled (not just opacity — also no pointer events + reason).
- Every async action has: optimistic or pending state → success confirmation → error with retry.
- Long lists: skeleton or count while loading; progress with numbers for multi-item work.
- Toasts via Sonner; success moments use the transitions.dev **success check**.

### D. Motion rules (hard blocks — from `review-animations/STANDARDS.md`)
| Never | Instead |
|---|---|
| `transition: all` | enumerate properties |
| `scale(0)` entrance | `scale(0.95–0.97)` + `opacity: 0` |
| `ease-in` on UI | `ease-out` / `cubic-bezier(0.23, 1, 0.32, 1)` |
| UI duration > 300ms without reason | 150–250ms |
| keyframes on toasts/toggles | CSS transitions |
| animating width/height/top/left | transform / opacity |
| animation on keyboard / 100+/day actions | none |
| missing `prefers-reduced-motion` | gentler variant, not zero |
| centered origin on trigger-anchored popover | origin at trigger (modals exempt) |
| everything entering at once | 30–80ms stagger |

Transition selection uses the transitions.dev decision rules: badge / dropdown / modal / panel /
page side-by-side / card resize / text swap / icon swap / number pop-in / success check / avatar
hover / error shake. Paste snippets verbatim; keep the reduced-motion block; keep the JS reflow.

### E. Accessibility
- Keyboard path through every flow; visible focus; `aria-live` for progress and results;
  labels on inputs; contrast ≥ 4.5:1 for text; touch targets ≥ 40px; no hover-only affordances.

### F. Findings table (full pass)
| # | Severity | Section | Location (file:line) | Finding | Fix |
|---|---|---|---|---|---|
Severity: **HIGH** feel-breaking / blocks the task · **MEDIUM** noticeably off · **LOW** polish.
Fix HIGH before the commit. Record the table in `docs/UI_AUDIT.md` with the date and commit.

### G. Verify
- Play motion at 2–5× (DevTools Animations panel) and step frame-by-frame.
- Test with *Reduce motion* on, keyboard only, and a 360px viewport.
- Re-run the grep sweep from §0 — anti-pattern hits must be zero or justified in a comment.

## Automated sweep (add to the project's `package.json` as `ui:audit`)

```bash
grep -rnE "transition:\s*all|scale\(0\)|[^-]ease-in[^-]" --include=*.css --include=*.tsx --include=*.ts --include=*.svelte src app components lib 2>/dev/null
grep -rnP "[\x{1F300}-\x{1FAFF}\x{2600}-\x{27BF}]" --include=*.tsx --include=*.svelte --include=*.ts src app components 2>/dev/null
grep -L "prefers-reduced-motion" app/globals.css src/app.css 2>/dev/null
```
Any output is a finding (emoji hits are always HIGH).

## Keep it updated (the "update always" rule)

The vendored copies under `references/` (or `skills/`) are snapshots. Refresh them and re-audit:

```bash
npx skills add jakubantalik/transitions-dev --skill '*' -y --copy
npx skills add emilkowalski/skill --skill '*' -y --copy
npx skills update -y
```
Then copy the refreshed folders back over the vendored ones, commit with
`chore(skills): refresh transitions-dev + emilkowalski/skill`, and run a full pass (§F).
