# Open Source Journey

Tracking my contributions to open source projects — what I worked on, why, what I learned.

---

## Contributions

### 1. mui/material-ui — Accordion Auto-generate IDs

**Issue:** [#48305](https://github.com/mui/material-ui/issues/48305) — opened by oliviertassinari (MUI core maintainer)

**PR:** [#48326](https://github.com/mui/material-ui/pull/48326)

**Status:** Open / Changes Requested — `mj12albert` asked for tests and docs, responded with explanation

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

**Status:** Open / Commented by `mj12albert`, awaiting re-review

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

### 5. razorpay/blade — Add zIndex as a global design token

**PR:** [#3357](https://github.com/razorpay/blade/pull/3357)

**Status:** Open / Under Review

**The Problem:**
Blade is Razorpay's design system used across all their products. Design systems rely on "tokens" — named values that components read instead of hardcoding numbers. Blade had tokens for spacing, motion, border radius, and colors. But z-index — which controls which UI layer sits on top of which — was never tokenized.

Instead, all z-index values lived in a utility file `utils/componentZIndices.ts` as plain hardcoded numbers:
```ts
// TODO: Move these properly to tokens at some point
export const componentZIndices = {
  modal: 1000,
  drawer: 1001,
  popover: 1100,
  tooltip: 1100,
  // ...
};
```

Four component files (Popover, Tooltip, TourPopover) each had their own `// TODO: Tokenize zIndex values` comment flagging this gap. The top-level TODO in the utility file itself said the same thing.

The real consequence: consumers building white-label products on Blade couldn't override z-index values through the theme system. Every other design decision was overridable — except this one.

**The Fix:**
Created `src/tokens/global/zIndex.ts` following the exact pattern of `spacing.ts` — a typed, documented constant with semantic layer names:
```ts
export const zIndex = {
  hide: -1,      // element hidden behind all other content
  base: 0,       // normal document flow
  sticky: 100,   // top nav, bottom nav, bottom sheet handles
  overlay: 1000, // modal backdrops
  drawer: 1001,  // side panels
  dropdown: 1002,// select/dropdown overlays
  popover: 1100, // tooltips, popovers, tour masks
};
```

Added `ZIndex` to `ThemeTokens` so the type system enforces it. Added to `bladeTheme` so it flows through the default theme. Updated `componentZIndices.ts` to reference the token instead of hardcoded numbers — backward-compatible, no existing component imports needed to change.

**Files Changed:**
- `packages/blade/src/tokens/global/zIndex.ts` — new file, token definition
- `packages/blade/src/tokens/global/index.ts` — export `ZIndex` type and `zIndex` constant
- `packages/blade/src/tokens/theme/theme.ts` — add `zIndex: ZIndex` to `ThemeTokens` type
- `packages/blade/src/tokens/theme/bladeTheme.ts` — add `zIndex` to default theme object
- `packages/blade/src/utils/componentZIndices.ts` — replace hardcoded numbers with token references
- `Popover.web.tsx`, `Popover.native.tsx`, `TourPopover.web.tsx`, `Tooltip.native.tsx` — remove resolved TODO comments
- `.changeset/swift-tokens-rise.md` — changeset entry (required by repo)

**What I Learned:**
- **Design token architecture** — tokens are the same "single source of truth" pattern as environment variables in backend config. Change one value, every consumer updates.
- **z-index stacking in CSS** — elements sit on a Z-axis. Higher number = closer to the user. You need coordinated values across the whole system or layers conflict. The gaps between values (100, 1000, 1001, 1002, 1100) are intentional — room to insert future layers without renumbering everything.
- **`ThemeTokens` as a TypeScript contract** — adding a field to the type forces every custom theme to declare that field. The type system becomes the documentation.
- **Backward-compatibility bridge pattern** — keeping `componentZIndices.ts` as a thin adapter meant 20+ existing component files needed zero changes. Only the internals changed.
- **Changeset workflow** — Blade uses `@changesets/cli` to track what changed for release notes. Every PR that touches a published package needs a `.changeset/*.md` file declaring the semver bump and description.
- **How to read a large codebase quickly** — grep for TODO comments, then trace the import chain from that file upward to understand where values should live.

---

### 6. razorpay/blade — Use existing border.radius.large and motion.easing.linear tokens

**PR:** [#3360](https://github.com/razorpay/blade/pull/3360)

**Status:** Open / Under Review

**The Problem:**
Two components had `// TODO` comments pointing at missing tokens — but the tokens already existed and were just never wired up.

**BottomSheet** had this for its rounded top corners:
```ts
// TODO: we do not have 16px radius token
borderTopLeftRadius: makeSpace(theme.spacing[5]),
```
`spacing[5]` is 20px, not 16px. The developer used it as a workaround. Meanwhile `theme.border.radius.large` was sitting in `border.ts` with a value of exactly 16. Two separate codepaths for the same value — and one was wrong.

**ProgressBar** had this for its indeterminate animation easing:
```ts
const easing = 'linear'; // TODO: Add this in motion tokens
```
`theme.motion.easing.linear` (`makeBezier(0,0,0,0)`) had existed in the motion token file all along. The ProgressBar just never plumbed it through — it passed the raw string instead of reading from the design system.

**The Fix:**
- BottomSheet: Replace `makeSpace(theme.spacing[5])` with `makeSpace(theme.border.radius.large)` in both `BottomSheet.native.tsx` and `getBottomSheetGrabHandleStyles.ts`. Remove the TODO comments.
- ProgressBar: Add `motionEasing` to the styled-component's `Pick<ProgressBarFilledProps, ...>` type so it can receive the prop. Replace `const easing = 'linear'` with `const easing = castWebType(getIn(theme.motion, motionEasing))`. Pass `motionEasing={motionEasing}` in the JSX.

No visual changes — both values resolve to the same output. But now the components are controlled by the design system instead of hardcoded strings.

**Files Changed:**
- `packages/blade/src/components/BottomSheet/BottomSheet.native.tsx` — correct radius token, remove TODO
- `packages/blade/src/components/BottomSheet/getBottomSheetGrabHandleStyles.ts` — same
- `packages/blade/src/components/ProgressBar/ProgressBarFilled.web.tsx` — thread motionEasing through styled-component, read from token
- `.changeset/warm-tokens-land.md` — changeset entry

**What I Learned:**
- **Searching for TODOs before writing new code** — the fastest way to find real bugs in a design system is `grep -r "TODO" --include="*.ts"`. Most TODO comments are breadcrumbs left by the original author pointing at exactly what needs to be fixed.
- **`styled-components` TypeScript generics** — `styled(Box)<Pick<Props, 'someProp'>>` — you must explicitly list every prop in the `Pick` that the style function needs to receive. The prop doesn't exist in the template literal scope otherwise.
- **`getIn(theme.motion, motionEasing)`** — Blade uses dot-path strings like `'motion.duration.xquick'` as prop values, then resolves them at runtime with `getIn`. The pattern decouples component logic from the token structure — you pass the path, not the value.
- **`castWebType()`** — a type-narrowing helper that tells TypeScript the return type is the web variant (string) rather than the platform-agnostic union. Required because `getIn` returns a general type.
- **Wrong workaround compounds over time** — `spacing[5]` happened to equal 20px but was semantically wrong. If the spacing scale ever changed, the radius would silently change too. Using the correct token by name insulates the component from unrelated scale changes.

---

### 7. razorpay/blade — Fix FormLabel double announcement for screen readers

**PR:** [#3366](https://github.com/razorpay/blade/pull/3366)

**Status:** Open / Under Review

**The Problem:**
`FormLabel` is the label component used across every input in Blade — TextInput, Select, DatePicker, Checkbox, and so on. When `necessityIndicator="required"` or `"optional"` is set, the component renders two things:

1. A `VisuallyHidden` node containing the text "required" or "optional" — invisible to sighted users but read by screen readers.
2. A visible indicator (`*` for required, `(optional)` for optional) — sighted users see this, but screen readers also read it.

Since both exist in the DOM, screen readers announced both. NVDA/VoiceOver said **"Email required asterisk"** instead of **"Email required"**. For optional fields: **"Email optional open paren optional close paren"** instead of just **"Email optional"**. The extra words are confusing noise for assistive technology users.

The TODO comment in the source code had been sitting there since the component was first built:
```tsx
{/* TODO: Hide from screen readers to prevent double announcement */}
{necessityLabel}
```

**The Fix:**
Wrap the visible `necessityLabel` in `makeAccessible({ hidden: true })`. Blade already ships a cross-platform `makeAccessible` utility used throughout the Form folder — this follows the exact same pattern as `SelectorGroupField` and `SelectorInput`.

On web, `makeAccessible({ hidden: true })` produces `aria-hidden="true"`. On React Native it produces `accessibilityElementsHidden={true}` + `importantForAccessibility="no-hide-descendants"`. Both platforms: the visible indicator is skipped by assistive technology while remaining visible to sighted users.

```tsx
// Before
{/* TODO: Hide from screen readers to prevent double announcement */}
{necessityLabel}

// After
{necessityLabel ? (
  <BaseBox {...makeAccessible({ hidden: true })}>{necessityLabel}</BaseBox>
) : null}
```

No visual change. One import added. One TODO resolved.

**Files Changed:**
- `packages/blade/src/components/Form/FormLabel.tsx` — import `makeAccessible`, wrap `necessityLabel` in aria-hidden container
- `.changeset/calm-labels-speak.md` — changeset entry

**What I Learned:**
- **Double announcement pattern** — in any UI that shows both a visible indicator AND has hidden text for screen readers, the visible element must be marked `aria-hidden`. Without it, AT users get redundant information which breaks the listening flow.
- **`aria-hidden` does not remove from DOM** — it tells the accessibility tree to skip the element during reading. The element still renders, still takes up space, still looks the same. Only screen readers are affected.
- **`makeAccessible({ hidden: true })` is cross-platform** — on web → `aria-hidden`, on iOS → `accessibilityElementsHidden`, on Android → `importantForAccessibility`. One call handles three targets.
- **How to find well-scoped accessibility bugs** — grep for `// TODO` in component source, especially in UI primitives used by many other components. A fix to `FormLabel` fixes the accessibility of every form field in the design system in one PR.
- **`VisuallyHidden` is the inverse of `aria-hidden`** — `VisuallyHidden` renders text that is visually hidden but announced by AT. `aria-hidden` renders content that is visually visible but skipped by AT. You use them together: one for the semantic text, one to silence the decorative duplicate.

---

### 8. razorpay/blade — Add headingLevel prop to Accordion for correct aria-level

**PR:** [#3367](https://github.com/razorpay/blade/pull/3367)

**Status:** Open / Under Review

**The Problem:**
Every `AccordionItem` trigger in Blade renders inside a heading wrapper. The heading level was hardcoded to `3` with a comment acknowledging the problem:

```tsx
// a11y guidelines suggest having an apt heading surround a button but heading level is hardcoded here
{...makeAccessible({ role: 'heading', level: 3 })}
```

This is a real accessibility bug. Screen readers like NVDA and VoiceOver allow users to navigate a page by heading levels — jumping from h1 → h2 → h3 to understand page structure. When an Accordion lives inside an `<h1>` section, its items announce as h3, creating a skipped level (h1 → h3). When it lives inside an `<h4>` section, the items announce as h3 which is *higher* in hierarchy than the parent, completely breaking the document outline.

The WAI-ARIA Accordion pattern explicitly states: "The heading element has a meaningful heading level relative to the context of the document."

**The Fix:**
Added `headingLevel?: 1 | 2 | 3 | 4 | 5 | 6` to `AccordionProps` (defaults to `3` so nothing breaks). The value flows from the root `Accordion` component through `AccordionContext` down to `AccordionButton.web.tsx` which uses it instead of the hardcoded `3`.

```tsx
// Before — always h3, no matter where it lives
<Accordion>...</Accordion>

// After — consumer can match the document hierarchy
<Accordion headingLevel={2}>...</Accordion>
<Accordion headingLevel={4}>...</Accordion>
```

**Files Changed:**
- `Accordion/types.ts` — add `headingLevel?: 1 | 2 | 3 | 4 | 5 | 6` to `AccordionProps` with JSDoc
- `Accordion/AccordionContext.tsx` — add `headingLevel` to `AccordionContextState` type
- `Accordion/Accordion.tsx` — destructure `headingLevel = 3`, include in context value and `useMemo` deps
- `Accordion/AccordionButton.web.tsx` — read `headingLevel` from context, use instead of hardcoded `3`
- `.changeset/wise-heading-flows.md` — changeset (marked as `minor` because it's a new prop)

**What I Learned:**
- **Document outline / heading hierarchy** — screen reader users navigate pages by jumping between headings, like a table of contents. Skipped or inverted heading levels (h1 → h3, or h3 inside h4) break this navigation. The heading level must match the document's section depth, not a hardcoded number.
- **`aria-level` attribute** — when an element has `role="heading"`, you must also specify `aria-level` (1–6) to tell AT what depth it is. Without it, AT defaults to level 2. The WAI-ARIA spec doesn't infer the level from document structure — you have to be explicit.
- **React Context as a prop relay** — instead of threading a prop through every intermediate component manually, the `headingLevel` is set once at the `Accordion` root and read directly by `AccordionButton`. This is exactly what React Context is for: avoiding prop-drilling through components that don't need the value themselves.
- **`useMemo` dependency arrays** — after adding `headingLevel` to the context value object inside `useMemo`, it also needs to be in the dependency array. If it weren't, the memoized value would always be the initial `headingLevel` even after it changed.
- **`minor` vs `patch` in changesets** — adding a new optional prop is a `minor` semver bump (new capability without breaking change), not `patch` (bug fix). Pure bug fixes that don't add/remove API surface are `patch`.

---

### 9. razorpay/blade — Add prefers-reduced-motion support across the entire design system

**PR:** [#3368](https://github.com/razorpay/blade/pull/3368)

**Status:** Open / Under Review

**The Problem:**
Blade had zero support for `prefers-reduced-motion`. This is an OS-level accessibility setting — macOS, iOS, Windows, and Android all let users enable "Reduce Motion" to minimize animations. When set, the browser emits the CSS media query `prefers-reduced-motion: reduce`, and applications are expected to honor it.

Users who rely on this setting include people with vestibular disorders (where animation triggers dizziness and nausea), epilepsy (flashing animations can trigger seizures), and motion sensitivity. For these users, using a Blade-based product is physically harmful.

Searching the entire codebase returned **exactly one match** for `prefers-reduced-motion` — a JSDoc comment in `BaseMotion.tsx` that claimed the component "handles reduced motion" but had zero implementation. Every animation — entrance/exit transitions, scale effects, stagger sequences, interactive hover states — ran at full speed regardless.

This is a WCAG 2.1 Success Criterion 2.3.3 violation.

**The Fix:**
Two-part solution that required understanding the full animation architecture:

**Part 1 — `BladeProvider.web.tsx` (the elegant part)**

framer-motion v11 ships `<MotionConfig reducedMotion="user">`. When you wrap components with it, framer-motion reads the OS `prefers-reduced-motion` setting and sets all animation durations to 0 when it's enabled. `BladeProvider` is already the root provider for all Blade system-wide configuration (theme, color scheme, floating-ui, drawer stacks). Adding `MotionConfig` there means every component in the entire design system that uses framer-motion — `BaseMotion`, `Scale`, `Stagger`, `AnimateInteractions` — automatically respects reduced motion with **zero changes to any individual component**. One import, one wrapper.

```tsx
<MotionConfig reducedMotion="user">
  <DrawerStackProvider>
    <BottomSheetStackProvider>{children}</BottomSheetStackProvider>
  </DrawerStackProvider>
</MotionConfig>
```

**Part 2 — `useReducedMotion` hook (for consumers and CSS animations)**

CSS transitions in styled-components (Toast, Modal backdrop) are not controlled by `MotionConfig`. For those, and for Blade consumers building custom components, I created a `useReducedMotion()` utility hook that reads `window.matchMedia('(prefers-reduced-motion: reduce)')` directly and subscribes to OS-level changes (so it reacts in real-time without a page reload).

```tsx
const prefersReducedMotion = useReducedMotion();

<Box transitionDuration={prefersReducedMotion ? 'duration.2xquick' : 'duration.moderate'} />
```

Exported from `~utils` as part of the public Blade API.

**Files Changed:**
- `BladeProvider.web.tsx` — import `MotionConfig`, wrap children in `<MotionConfig reducedMotion="user">`
- `utils/useReducedMotion.ts` — new hook, JSDoc, OS change subscription via `addEventListener`
- `utils/index.ts` — export the new hook

**What I Learned:**
- **`prefers-reduced-motion` media query** — an OS-level signal, not just a CSS feature. Users set it in System Settings. The browser exposes it as both a CSS media query and a JavaScript `window.matchMedia` API. Applications are expected to honor it automatically.
- **framer-motion `MotionConfig`** — a provider component that configures default behavior for all framer-motion animations inside it. `reducedMotion="user"` makes it read the OS preference dynamically. `reducedMotion="always"` disables everything (useful for testing). `reducedMotion="never"` forces animations regardless of user preference.
- **Provider-level vs component-level fixes** — the architectural insight here is that fixing it in the provider means every component gets it for free, vs patching individual components. This is the same reason `ThemeProvider` exists: set once at the top, consume anywhere below.
- **WCAG 2.1 SC 2.3.3** — "Animation from Interactions: Motion animation triggered by interaction can be disabled, unless the animation is essential to the functionality or the information being conveyed." This is a Level AAA criterion. SC 2.3.1 (Three Flashes) is Level A.
- **`MediaQueryList.addEventListener('change', ...)`** — the correct way to subscribe to media query changes in modern browsers. The older `addListener` API is deprecated. The event fires immediately when the user changes their OS setting, so the UI reacts without a page reload.
- **`window.matchMedia` in SSR** — SSR environments (Next.js, Remix) don't have `window`. The hook initializes with `false` on the server and subscribes on the client via `useEffect`, which is the correct SSR-safe pattern.
- **Why this is different from other accessibility fixes** — most a11y fixes are about labeling (aria-label, aria-hidden, aria-describedby). Reduced motion is about behavior: the UI must change how it acts based on the user's physical needs. It requires understanding both the OS accessibility model and the animation framework's internals.

---

## Repo Structure

```
contributions/     # detailed notes per contribution
```

```
contributions/     # detailed notes per contribution
```
