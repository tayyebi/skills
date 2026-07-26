---
name: z-index-layer-governance
description: Use when a UI has overlapping surfaces — modals, popovers, toasts, sticky bars, drag overlays — and stacking order needs to be a designed contract rather than a race of ad-hoc numbers. Covers naming a layer scale, tokenizing it, stacking-context traps, and auditing an existing codebase.
---

# Z-Index Layer Governance

Every non-trivial UI accumulates overlapping surfaces: a sticky toolbar, a floating tab bar, a dropdown, a modal, a confirm dialog raised *from* that modal, toasts, a drag ghost, a tooltip. Left alone, each one gets a `z-index` picked at the call site by whoever wrote it, chosen by trying numbers until the bug went away. The result is not a stacking order; it's a folk memory — `9999` here, `100` there, two different components on `30` waiting to collide.

The fix is small and boring: **one named scale, defined once, referenced everywhere.** The value of a layer is then a decision recorded in one file, and "what covers what" becomes reviewable.

## When to use

- A UI with more than two or three overlapping surface types (any app with modals *and* popovers *and* notifications).
- The moment you catch yourself writing a bare number to fix an overlap — that's the signal, not a later cleanup task.
- Auditing an existing codebase where overlap bugs keep coming back in different places.

Skip it for a genuinely flat UI, or for ordering *inside* a single component's own stacking context (see "Local ordering stays local").

## The core pattern

Define a scale as design tokens — CSS custom properties, a theme object, whatever your stack's single source of truth is — with one entry per *kind of layer*, not per component:

```css
:root {
  --z-raised:        10;   /* in-flow elevation: a card over its own backdrop */
  --z-sticky:        20;   /* sticky headers/toolbars inside a scroller       */
  --z-nav:           30;   /* persistent chrome: floating tab bar, dock       */
  --z-anchored:      40;   /* popovers/menus anchored to in-page elements     */
  --z-page:          50;   /* full-page surfaces layered over the app         */
  --z-command:       60;   /* command palette, quick-find, shortcuts overlay  */
  --z-scrim:         80;   /* dimming scrim for a modal mode                  */
  --z-scrim-lift:    90;   /* what must stay ABOVE that scrim while it's up   */
  --z-toast:        100;   /* transient feedback                              */
  --z-capture:      120;   /* pointer-capture overlays during drag/resize     */
  --z-lightbox:     130;   /* media/preview lightboxes over a page            */
  --z-modal:        200;   /* full-screen modals                              */
  --z-dialog:       300;   /* confirm dialogs — top of the app                */
  --z-tooltip:    10000;   /* pointer-transparent, above everything           */
}
```

Then at every call site, reference the token — never a literal:

```css
.dialog  { z-index: var(--z-dialog); }
```
```html
<!-- utility-class frameworks take the var too -->
<div class="fixed inset-0 z-[var(--z-dialog)]">
```

### Rules that make it hold

1. **Anything that can overlap a *different* component uses a token.** No bare numbers, and no new magic number at a call site — a new number is a new unnamed layer nobody else knows about.
2. **Local ordering stays local.** `z-10` for an element over its own component's backdrop, inside a stacking context that component created, cannot collide with anything outside it. Leave those alone; tokenizing them adds ceremony for no safety.
3. **Adding a layer means adding a documented row**, with a comment saying what it must cover and what must cover it. If the reason can't be written in a line, the layer probably isn't a new kind.
4. **Two things may share a token only if they can never be visible at once.** Sharing otherwise re-creates the original bug: equal values break the tie on DOM order.
5. **Leave gaps between values** (10/20/30 … 100/120/130 … 200/300). A future layer should be insertable without renumbering the world.
6. **Order by *what must be reachable*, not by importance.** A confirm dialog raised from inside a modal must sit above that modal, or the user is stuck. Work through those "raised from" chains explicitly — they're the ones that produce trapped-UI bugs.

## Stacking contexts: the trap that makes tokens look broken

A `z-index` only competes with siblings **in the same stacking context**. A child of a `z-index: 50` context can set `z-index: 9999` and still lose to a `z-index: 51` element outside it — its 9999 is only meaningful *within* its parent. Before raising a number, check whether the two elements are even in the same context.

Properties that create a stacking context (a partial list, all common in real UIs): a positioned element with any `z-index` other than `auto`; `transform`, `filter`, `backdrop-filter`, `perspective`, `mask`, `clip-path`; `opacity` < 1; `will-change` naming any of those; `contain: paint`; `isolation: isolate`; a flex/grid *child* with a `z-index`.

Consequences worth internalizing:

- **An animation can create one.** Adding `transform` or `opacity` for a transition silently traps every descendant's stacking — a popover that worked yesterday now hides behind a sibling because someone added a fade.
- **A popover anchored inside a transformed ancestor cannot escape it.** No token value fixes that. Portal the popover to the document root (or a dedicated layer host) and position it with measured coordinates.
- **`isolation: isolate` is a feature, not just a hazard.** Deliberately creating a context around a component keeps its internal `z-10`/`z-20` from ever competing with app-level layers — exactly what makes rule 2 safe.
- **Modern escape hatches:** the top layer (`<dialog>` + `showModal()`, and the Popover API) renders above the whole page regardless of stacking context or `z-index`. Where the platform supports it, using it deletes a whole class of these bugs — but it also opts out of your scale, so it's a per-surface decision, not a blanket one.

## Auditing an existing codebase

1. **Grep every value.** `z-index:` in stylesheets, plus your framework's utility form (`z-50`, `z-[999]`, `zIndex:` props). Tally them.
2. **Group by kind, not by number.** Two components on `100` are one layer if they're the same *kind* of surface (both toasts) and two layers wearing one number if they aren't.
3. **Look for the tells.**
   - A duplicate value on two overlapping surfaces — the tie breaks on DOM order, which means the bug appears and disappears as markup moves.
   - An absurd value (`9999`, `2147483647`) — someone was fighting a stacking context and won by escalation. Check whether it now covers something it must *not*, like a confirm dialog raised from inside it.
   - Any surface with no `z-index` that visually sits above others — it's relying on DOM order alone and will break on the next re-order.
4. **Map old → token, migrate mechanically, then read the diff for precedence changes.** Changing precedence is sometimes the point (a dialog escaping a picker that used to cover it) and sometimes an unrequested behaviour change (lifting toasts above modals). Decide each one deliberately and write the choice into the token comment.
5. **Keep it from regressing.** A lint rule banning bare `z-index` outside the token file is the cheap version; a review habit of "which token, and why that one" is the version that actually holds.

## Documenting the decisions

The token block is the documentation — keep the rationale next to the values, not in a wiki nobody opens. Two comments earn their space: for the near-collisions (`anchored` is above `nav` *because* anchored popovers overlap the nav bar) and for deliberate low placements (a toast below modals so it can't swallow a click meant for the dialog that raised it). Reviewers then argue about the ordering, which is the useful argument.

## Pitfalls

- **Numbers at call sites.** The default failure mode: two components pick the same value independently and the tie breaks on DOM order, so the bug looks intermittent and "fixes itself" when unrelated markup moves.
- **Escalation as a debugging strategy.** Bumping to `9999` beats today's sibling and creates tomorrow's trapped dialog. If a token value isn't winning, the cause is usually a stacking context, not a value that's too small.
- **Tokenizing the wrong granularity.** One token per component (`--z-user-menu`, `--z-share-panel`) is the same anarchy with better spelling. Tokens name *kinds* of layer.
- **Consecutive values.** `1,2,3…` means the next insertion renumbers everything, so people invent `2.5` as `250` somewhere else instead.
- **Forgetting the raised-from chains.** Dialog over modal, tooltip over dialog, drag overlay over the thing being dragged — each is a reachability requirement, and each is a real trapped-UI bug when inverted.
- **Assuming a portal fixes stacking alone.** Portaling escapes the *ancestor context*, but the portal host still needs a token — otherwise it competes at wherever it happens to land in the body.
- **Scattering `!important` to win.** `!important` affects which declaration applies, not which stacking context an element lives in; it cannot lift a child out of its parent's context.

## Checklist

- [ ] One token scale, defined in a single place, with a comment per layer saying what it covers and what covers it
- [ ] Every cross-component surface references a token; no bare numbers outside the definition
- [ ] Purely local ordering left as plain values, ideally inside a deliberate `isolation: isolate` boundary
- [ ] Gaps left between values so a new layer needs no renumbering
- [ ] No two simultaneously-visible surfaces share a token
- [ ] "Raised from" chains verified: dialog above modal, tooltip above dialog, capture overlay above its drag source
- [ ] Stacking contexts checked before any value is raised (transform / opacity / filter / contain on an ancestor)
- [ ] Precedence changes from a migration reviewed deliberately, not absorbed silently
- [ ] A lint rule or review habit keeps new bare numbers out
