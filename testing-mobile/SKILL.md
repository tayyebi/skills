---
name: testing-mobile
description: Use when testing how a web app behaves on phones, tablets, and narrow viewports — device/viewport breakpoint coverage, touch-target sizing, Playwright device emulation, manual DevTools responsive testing, mobile browser quirks (iOS Safari, Android Chrome), offline/poor-connectivity behavior, and PWA manifest correctness. Distinct from testing-accessibility (owns the 44x44 touch-target pass/fail criterion) and testing-browser-ui (general desktop UI/console/network auditing).
---

# Mobile & Responsive Testing

Confirm a responsive web app (no separate mobile build — same client-side code, different viewport) is actually *usable* on a phone or tablet, not just "renders without errors." A page that loads cleanly at 1280px and silently overflows or hides its primary action at 375px is a shipped bug.

## When to use

- Any request to test mobile behavior, responsive layout, touch interaction, or device-specific quirks.
- Before or after any layout change to a shared component — a fix at desktop width can easily break at 375px.
- When a bug report mentions "on my phone" — reproduce at the reporter's approximate viewport first.

## Device/viewport matrix

Cover a spread of real breakpoints rather than one arbitrary "mobile" size:

| Class | Representative width | Priority |
|---|---|---|
| Small phone | 360–375px | High |
| Standard/large phone | 390–428px | High |
| Small tablet / phone landscape | 768px | Medium |
| Tablet | 820–1024px | Medium |
| Small laptop | 1024px+ | Low (usually covered by desktop testing) |

Test at least one iOS (Safari) and one Android (Chrome) profile — the engines diverge on scroll, viewport units, and input behavior (see quirks below).

## Core pattern

**1. Resize and look for horizontal overflow first.** At each breakpoint, check `document.documentElement.scrollWidth > window.innerWidth` (or equivalent) — any positive result means something is forcing the whole page wider than the viewport, usually one unconstrained table or fixed-width element. This is the single highest-value automated check.

**2. Confirm navigation is reachable.** Sidebar/nav should collapse to a hamburger, drawer, or bottom bar at narrow widths — something has to give way. Verify it opens, closes, and its links work with touch.

**3. Walk the core flows with taps, not clicks.** Pick 3–5 flows that represent real usage (sign-in, view a list, submit a form, complete a purchase/checkout) and run them end to end using tap/touch input at a phone viewport. Confirm the virtual keyboard doesn't obscure the focused input, the keyboard dismisses after submit, and no layout overflow appears mid-flow.

**4. Check touch targets while you're there, defer the verdict to testing-accessibility.** Interactive elements should be roughly ≥44×44 CSS pixels (WCAG 2.5.5); flag anything obviously undersized as a convenience finding, but treat testing-accessibility as the canonical owner of the pass/fail threshold and evidence format.

**5. Exercise mobile browser quirks explicitly** (see table below) rather than assuming desktop-tested code behaves the same.

**6. Test degraded connectivity.** Throttle to a slow profile (e.g. via CDP `Network.emulateNetworkConditions`) and confirm the app shows a loading state and times out gracefully; then go fully offline and confirm a meaningful error appears instead of a blank/white screen.

## Device emulation sweep

Any browser-automation tool with device-emulation support (Playwright, Cypress, Selenium + a cloud device grid) can drive this same check — the pattern, not the tool, is what matters:

```
for device in [iphone-class-profile, android-flagship-profile, tablet-profile]:
    set viewport + user-agent to device profile
    open core page
    scrollWidth, clientWidth = measure(document)
    assert scrollWidth <= clientWidth + 1   # no horizontal overflow
```

## Mobile browser quirks

| Engine | Quirk | What to verify |
|---|---|---|
| iOS Safari | `100vh` includes browser chrome, causing overflow | Use `dvh`/`-webkit-fill-available`; check no clipped content |
| iOS Safari | Auto-zooms on input focus if `font-size` < 16px | Form inputs use ≥16px font |
| iOS Safari | Bounce/rubber-band scroll can scroll the page behind an open modal | Modals lock body scroll while open |
| iOS Safari | Fixed-position elements can misplace when the keyboard opens | Headers/footers don't overlap the focused input |
| Android Chrome | Swipe-back gesture can conflict with in-app swipe interactions | No accidental navigation from an in-app gesture |
| Android Chrome | Autofill injects extra UI into inputs | Layout doesn't break when autofill suggestions appear |

## Responsive tables

Data-dense pages (list views, invoices, logs, admin tables) are the most common source of horizontal overflow. At each breakpoint: does the table scroll *within its own container*, reflow into a card/list layout, or hide low-priority columns — or does it drag the whole page sideways? Only the first three are acceptable.

## Reporting format

```
Device:      iPhone SE (375x667, Safari 17)
Page:        /invoices
Issue:       Invoice table causes full-page horizontal scroll at 375px
Steps:       Open /invoices at 375px viewport; scroll right
Expected:    Table scrolls within its own container; page body doesn't move
Actual:      Whole page scrolls ~240px right, hiding surrounding UI
Fix:         overflow-x: auto on the table wrapper; overflow-x: hidden on body
Severity:    Medium — impairs visibility of this data on every phone-width viewport
```

## Pitfalls

- Testing only one viewport width and calling it "mobile-tested" — 375px and 768px behave very differently.
- Using mouse clicks instead of tap/touch emulation, which can mask keyboard-obscures-input and gesture-conflict bugs.
- Skipping the iOS-vs-Android split — many mobile bugs are engine-specific, not "mobile" in general.
- Forgetting to test with the virtual keyboard actually open — a huge source of SPA layout bugs.
- Declaring victory on "no console errors" without checking for horizontal overflow, which produces no error at all.

## Checklist

- [ ] Tested at ≥2 phone widths and ≥1 tablet width, plus one iOS and one Android profile
- [ ] No horizontal page overflow at any tested breakpoint
- [ ] Navigation collapses and remains reachable at narrow widths
- [ ] Core flows completed with tap/touch input, keyboard-aware (no obscured inputs, dismisses on submit)
- [ ] Obvious undersized touch targets flagged (canonical criterion: testing-accessibility)
- [ ] iOS Safari and Android Chrome quirks specifically exercised, not assumed
- [ ] Slow/offline connectivity produces a loading state or clear error, never a blank screen
- [ ] Data-heavy tables scroll within their container or reflow — never drag the page sideways
