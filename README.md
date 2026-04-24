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

### 3. facebook/react — Fix false hydration mismatch warning with portals

**Issue:** [#12615](https://github.com/facebook/react/issues/12615) — Unexpected warning when hydrating with portal and SSR

**PR:** [#36321](https://github.com/facebook/react/pull/36321)

**Status:** Open / Under Review

**The Problem:**
When a component returned `null` on the server but rendered `ReactDOM.createPortal()` on the client, React emitted a false hydration mismatch error:
```
Hydration failed because the server rendered HTML didn't match the client.
  <span>
    <HoverMenu>
+     <div>   ← false positive: this is portal content, not inside span
```
The portal's `<div>` renders into a *separate container* and should never be compared against the parent's server HTML.

**Root Cause:**
`updatePortalComponent` calls `pushHostContainer` to switch to the portal's DOM container, but the hydration state variables (`isHydrating`, `nextHydratableInstance`, `hydrationParentFiber`) were not reset. Portal children then tried to claim hydration instances from the **parent container's** server nodes → false mismatch.

**The Fix:**
Added two functions to `ReactFiberHydrationContext`:
- `prepareToHydrateHostPortal` — saves hydration state to a stack and resets it when entering a portal
- `popHydrationStateAfterPortal` — restores saved state when the portal completes

Called from `updatePortalComponent` (begin work) and the `HostPortal` complete work case respectively. Stack-based so nested portals work correctly.

**Files Changed:**
- `packages/react-reconciler/src/ReactFiberHydrationContext.js` — added save/restore stack + two new exported functions
- `packages/react-reconciler/src/ReactFiberBeginWork.js` — call `prepareToHydrateHostPortal` in `updatePortalComponent`
- `packages/react-reconciler/src/ReactFiberCompleteWork.js` — call `popHydrationStateAfterPortal` in `HostPortal` case
- `packages/react-dom/src/__tests__/ReactDOMHydrationDiff-test.js` — two regression tests

**What I Learned:**
- React's hydration system tracks server DOM nodes via module-level cursor variables (`hydrationParentFiber`, `nextHydratableInstance`, `isHydrating`)
- `pushHostContainer` changes the DOM container context but is completely separate from the hydration cursor — a subtle but critical distinction
- Portals are excluded from the parent's DOM tree (`appendAllChildren` skips them) but the hydration cursor was NOT being excluded alongside them
- Save/restore stack pattern is the standard React approach for re-entrant contexts (Suspense uses it too)
- Always write a failing test first to confirm the bug is real before reading the source

---

### 4. facebook/react — Warn when NaN is passed as an aria-* attribute value

**PR:** [#36340](https://github.com/facebook/react/pull/36340)

**Status:** Open / Under Review

**The Problem:**
React already warns when a non-aria attribute receives a `NaN` value (in `ReactDOMUnknownPropertyHook`). But aria-* attributes skip that check entirely — they return early before the NaN guard is reached. The result is silent: `<div aria-valuenow={NaN} />` renders `aria-valuenow="NaN"` in the DOM with no warning, which is invalid for every ARIA attribute type and silently breaks assistive technologies.

A common trigger is a computed numeric value derived from state or data that can unexpectedly become `NaN`:

```jsx
const progress = (loaded / total) * 100;  // NaN if total is 0
<div role="progressbar" aria-valuenow={progress} />  // silently sets aria-valuenow="NaN"
```

**Root Cause:**
`ReactDOMUnknownPropertyHook.validateProperty` returns early for any `aria-*` prop at line 97-99 (delegating to the ARIA hook) — but `ReactDOMInvalidARIAHook.validateProperty` only checked the attribute *name*, never the *value*.

**The Fix:**
Added a NaN check inside `ReactDOMInvalidARIAHook.validateProperty`. Also threaded the prop value through from `validateProperties` so the inner function can inspect it. The check runs after the attribute name is confirmed valid and correctly cased, mirroring the exact message already used by the unknown property hook:

```
Warning: Received NaN for the `aria-valuenow` attribute.
If this is expected, cast the value to a string.
```

**Files Changed:**
- `packages/react-dom-bindings/src/shared/ReactDOMInvalidARIAHook.js` — extend `validateProperty` to accept and inspect `value`; add NaN guard for valid aria-* attributes
- `packages/react-dom/src/__tests__/ReactDOMInvalidARIAHook-test.js` — three new tests: NaN on numeric aria attr, NaN on string aria attr, no false-positive for valid integers

**What I Learned:**
- React's property validation is split across two hooks: `ReactDOMUnknownPropertyHook` (general DOM) and `ReactDOMInvalidARIAHook` (aria-specific) — each only covers its own domain
- An early-return for valid attribute names can accidentally exempt those attributes from value-level checks in the other hook
- The `warnedProperties` map is module-level and de-duplicates warnings across the component tree so the console isn't flooded during re-renders
- Writing tests alongside the fix is the norm in React: always test the exact warning string including the stack trace suffix

---

## Repo Structure

```
contributions/     # detailed notes per contribution
```
