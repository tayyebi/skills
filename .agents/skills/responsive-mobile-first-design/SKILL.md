---
name: responsive-mobile-first-design
description: Use when authoring or refactoring UI layout for multiple screen sizes — mobile-first CSS authoring order, breakpoint strategy, per-size-class behaviour (not just reflow), touch target sizing, and performance budgets for constrained devices. Covers design/build-time layout methodology; see testing-mobile for verifying the result and rtl-bidi-layout for right-to-left mirroring.
---

# Responsive / Mobile-First Design

Author styles for the smallest viewport first, then progressively add complexity for larger ones. Never start from a desktop layout and squeeze it down — that produces "mobile-looking-desktop," not mobile-first, and it inverts who pays the cost of every unnecessary style.

Mobile-first is also **not only about expanding for larger screens**: a component often needs genuinely *different behaviour* on small screens versus large ones, not the same layout at a different density. See "Different behaviour per size class" below — it is the difference between a design that adapts and one that merely survives.

## When to use

Any UI viewed on more than one screen size — which, for a web or hybrid app, is effectively always.

## Core pattern

1. **Base styles are the mobile layout**, unconditional, no media query: single column, full-width elements, stacked content.
2. **Add complexity upward** with `min-width` media queries as the viewport grows. Never use `max-width` for layout — a `max-width` breakpoint means the design started from desktop and is being clawed back for mobile, so mobile users (usually the majority, and the least tolerant of bloat) inherit every desktop-only style unless it's explicitly undone.

```
/* base: mobile, no media query */
.grid { display: grid; grid-template-columns: 1fr; }

@media (min-width: 768px)  { .grid { grid-template-columns: repeat(2, 1fr); } }
@media (min-width: 1024px) { .grid { grid-template-columns: repeat(4, 1fr); } }
```

3. Pick 2-4 breakpoints tied to where *your* content actually breaks, not to specific device models — devices change yearly, content shape doesn't. A common starting set is ~768px (tablet) / ~1024px (small desktop) / ~1280px (desktop); adjust to what you observe.

## Touch targets

- Minimum interactive hit area **44x44px** — buttons, links, table-row click targets — even when the visible element is smaller. Expand with padding, not just visual size.
- Minimum **8px** spacing between adjacent tappable elements, so an imprecise tap doesn't land on the neighbor.
- No hover-*dependent* interaction. Hover can enhance (tooltips, previews) but must never be the only way to reach a control — touch has no hover state.
- Give an immediate active/pressed visual state on tap; a design that only defines a hover state is invisible to touch input.

## Layout patterns

- **One column by default.** Multi-column layout is something a wider viewport opts *into*, not something mobile opts out of.
- **Content before chrome.** Navigation, sidebars, and decoration are secondary — primary content sits first in both DOM order and visual priority. Nobody should have to scroll past a menu to reach what they came for.
- **Tables don't survive small screens.** Either transform rows into stacked label/value cards below a breakpoint, or make the table scroll horizontally inside its own bounded container — never let a wide table force the entire page to scroll sideways.
- **Navigation collapses.** A persistent sidebar or top-nav at wide viewports becomes an off-canvas panel (hamburger-triggered) below the point where it no longer fits without crowding content.
- **Modals go full-screen (or bottom-sheet) on mobile**, and become a centered, bounded-width card at wider viewports — a small centered dialog on a 360px screen wastes half the viewport on backdrop.

## Different behaviour per size class

Reflowing the same controls is the floor, not the goal. The distinguishing mark of a mobile-first component is that the small-screen arm changes *what the interaction is*, not just its metrics. A layout that only shrinks (tighter gaps, smaller fonts, columns that fade or truncate) still forces desktop interaction patterns — hover reveals, dense grids, wide steppers — onto a device where they don't work.

Concrete shape changes that mark real per-size-class behaviour:

- **A dense header becomes a collapsing app bar.** Desktop: identity + a full control cluster + a wide stage/progress stepper. Phone: a sticky bar holding only navigation and identity, with everything else in a fold that collapses as content scrolls and returns the moment the user scrolls back up. Scrolling away isn't hiding — it's deferring: the dropped controls are one scroll from the top, and anything width removes entirely must reappear in an overflow menu. *Nothing is hidden that isn't reachable another way.*
- **A wide stepper/slider becomes a list row + sheet.** A multi-step progress control that is hundreds of pixels of connectors and labels can't sit in a nested horizontal scroller — on touch, a horizontal scroller inside a vertical page fights the page on every swipe, because they're the same gesture. The phone form is a full-width list row (label above, current value below, chevron trailing — a ~48px target) opening a bottom sheet of options. Same data, different control. A timeline slider is the same story: distinct drag/tap behaviour per size class, not one widget scaled.
- **Overlays change species.** Anchored popovers and dropdown menus at desktop become bottom sheets on touch; centered dialogs become full-screen.
- **Hover reveals become always-visible.** Edit pencils, remove-x buttons, and row actions that appear on hover at desktop must simply *be there* on the touch arm.

Two implementation rules keep the arms from diverging into two apps:

- **Layout is CSS; behaviour is a media-query hook — and never the reverse.** Which columns exist, what stacks, what hides: breakpoint-scoped CSS, which is correct before any script runs and can't mismatch the server-rendered DOM. What JS may consult a matched media query for is *behavioural choice CSS cannot express*: which overlay component to mount (sheet vs popover), which input sensors to attach. Never gate "should this be a column" on a JS check — that's how hydration flashes and divergent trees start. If a JS breakpoint hook exists, it must report a consistent first-render value (server snapshot) rather than defaulting to "mobile" and flipping after mount.
- **State stays outside the arms.** When small and large screens render different markup for the same feature, both arms must be pure presenters of the *same* lifted state and callbacks — one source of truth, two projections. Duplicated state per arm desynchronizes the moment one arm mutates. (Corollary: if both arms mount an editable input bound to shared "editing" state, don't autofocus both — focus the one that's actually visible.)

One more motion rule for collapsing chrome: the scroll listener should flip a single boolean state with **hysteresis** (collapse past one offset, re-expand only below a smaller one — a dead band), and every moving property animates via CSS transitions off that one class change. Driving geometry from scroll position re-renders every frame on exactly the device least able to afford it; a single threshold makes the header flicker, because collapsing changes the content height that determines the scroll offset that fired the threshold.

## Performance budget

Mobile networks and devices are the tightest constraint, so budget for them, not for a developer's desktop connection:

- Cap the initial payload (critical JS + CSS + above-the-fold images) to a size that loads acceptably on a throttled connection, and enforce it as a build-time budget, not an aspiration.
- Lazy-load below-the-fold images and non-critical scripts.
- Prefer system fonts, or a small subset of one webfont, over loading multiple families/weights — font loading is a common invisible mobile-performance tax.
- Reserve layout space for images and async content so it doesn't shift once loaded — a shift is a larger fraction of a small viewport, and more disorienting there than on desktop.

## Design decisions

- **`min-width`, never `max-width`, for layout.** This one rule most reliably distinguishes a genuinely mobile-first codebase from a "responsive-as-an-afterthought" one — grep the stylesheet for `max-width` inside layout rules as a smell test.
- **Breakpoints follow content, not devices.** Resize the actual component until it visually breaks and put the breakpoint there, rather than at a device's reported width.
- **Progressive disclosure over cramming.** Don't force desktop information density onto a phone screen — show essential fields and let the user drill in for detail.

## Pitfalls

- Writing `max-width` media queries "because it was faster" — that's desktop-first design wearing a mobile-first label.
- Treating mobile-first as a pure layout exercise — shrinking gaps and truncating columns while keeping desktop interaction patterns (hover reveals, nested horizontal scrollers, dense multi-column grids) on the touch arm.
- Making features *disappear* below a breakpoint (`hidden` with no counterpart) instead of relocating them — everything the small screen drops must remain reachable via an overflow menu, a sheet, or a scroll back to the top.
- Gating layout on a JS media-query check instead of CSS — a guaranteed first-paint flash and a hydration mismatch waiting to happen.
- Sizing touch targets to the visual icon (e.g., a bare 16px icon with no padding) instead of the 44px hit area.
- Hover-only affordances (tooltip-only labels, hover-reveal menus) that are simply unreachable on touch.
- Wide, unscrollable tables that blow out the whole page's horizontal scroll instead of scrolling within their own container.
- Shipping one large, desktop-sized image asset to every viewport instead of serving responsive image variants.
- Calling it done once it "looks right" in a resized desktop browser window — verify on real touch devices and real network conditions (see `testing-mobile` for the QA pass), and check the same layouts mirrored for right-to-left locales (see `rtl-bidi-layout`).

## Checklist

- [ ] Base (unqueried) styles are the mobile layout; every media query is `min-width`
- [ ] The small-screen arm changes behaviour where needed (sheets over popovers, list rows over wide steppers, collapsing chrome), not just metrics — and everything it drops stays reachable another way
- [ ] Layout decisions live in CSS; JS media checks only pick behaviour (which overlay to mount), with SSR-consistent first render
- [ ] Both size arms present the same lifted state — no per-arm state duplication
- [ ] Breakpoints chosen from where content actually breaks, not device names
- [ ] All interactive elements ≥44x44px hit area, ≥8px apart
- [ ] No hover-only interaction
- [ ] Tables/wide content degrade to cards or scroll within their own container
- [ ] Navigation collapses to off-canvas below its breakpoint
- [ ] Performance budget defined and enforced for the critical path
- [ ] Verified on real devices and throttled networks — see `testing-mobile`
