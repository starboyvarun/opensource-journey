# Open Source Journey

Tracking my contributions to open source projects — what I worked on, why, what I learned.

---

## Contributions

### 1. mui/material-ui — Accordion Auto-generate IDs

**Issue:** [#48305](https://github.com/mui/material-ui/issues/48305) — opened by oliviertassinari (MUI core maintainer)

**PR:** [#48326](https://github.com/mui/material-ui/pull/48326)

**Status:** Open / Under Review

**The Problem:**
Developers using the `Accordion` component had to manually assign `id` and `aria-controls` props on `AccordionSummary` for screen readers to work correctly:

```jsx
// Before — you had to do this manually
<AccordionSummary id="panel1-header" aria-controls="panel1-content">
  Header
</AccordionSummary>
```

If forgotten → accessibility silently broken.
If duplicated across multiple instances → duplicate IDs in the DOM → screen readers confused.

**The Fix:**
Used `React.useId()` inside `Accordion` to auto-generate unique IDs, passed via React context to `AccordionSummary`. The component now handles accessibility automatically.

```jsx
// After — just works, no manual IDs needed
<AccordionSummary>
  Header
</AccordionSummary>
```

User-provided `id`/`aria-controls` still override auto-generated ones (fully backwards compatible).

**Files Changed:**
- `packages/mui-material/src/Accordion/Accordion.js` — added `React.useId()`, passed IDs via context
- `packages/mui-material/src/Accordion/AccordionContext.js` — updated context type
- `packages/mui-material/src/AccordionSummary/AccordionSummary.js` — read IDs from context, apply as defaults

**What I Learned:**
- How React Context works to pass data between parent and child components
- `React.useId()` — generates stable unique IDs per component instance, SSR-safe
- WAI-ARIA accordion pattern — `aria-controls`, `aria-labelledby`, `role="region"`
- How large monorepos (Lerna + Nx) are structured
- MUI contributing workflow: fork → branch → PR → comment on issue

---

### 2. mui/material-ui — Menu text shifts on first hover

**Issue:** [#40414](https://github.com/mui/material-ui/issues/40414) — text on MenuItem moves slightly on first hover

**PR:** [#48329](https://github.com/mui/material-ui/pull/48329)

**Status:** Open / Under Review

**The Problem:**
When you open a MUI Menu and hover over a MenuItem for the first time, the text jumps ~1px. Most visible on Windows with 150% display scaling.

**Root Cause (after correction):**
The `Grow` animation (used when Menu opens) sets `transform: 'none'` when it finishes:
```js
entered: { opacity: 1, transform: 'none' }  // removes GPU compositing layer
```
This causes the browser to demote the element from its GPU layer. The first hover triggers a background-color change → CPU repaint → text lands at slightly different subpixel positions.

**Initial wrong fix:** Added `-webkit-font-smoothing` to `ButtonBase` — reviewer `mj12albert` correctly pointed out this only affects macOS, not Windows. Corrected immediately.

**The Real Fix:**
Use `getScale(1)` instead of `'none'` in Grow's `entered` state. `scale(1,1)` is visually identical to `none` but keeps the GPU layer alive:
```js
entered: { opacity: 1, transform: getScale(1) }
```

**Files Changed:**
- `packages/mui-material/src/Grow/Grow.js` — 1 line change

**What I Learned:**
- GPU compositing layers — browsers move elements to GPU for animations, but remove them after
- Subpixel text rendering — text positions can shift by 1px depending on compositing state
- CSS stacking context — any `transform` (even 2D identity) creates a stacking context, which can break drag-and-drop positioning
- `transform: 'none'` vs `transform: scale(1,1)` — seemingly identical visually but very different in browser rendering model
- Deliberate decisions in open source are documented in linked PRs — always trace history before changing things
- How to respond to reviewer feedback, correct wrong analysis, and propose the next direction constructively
- `mergeSlotProps` merge order in MUI: `additionalProps` → `externalForwardedProps` → `slotProps`

---

## Repo Structure

```
contributions/     # detailed notes per contribution
```
