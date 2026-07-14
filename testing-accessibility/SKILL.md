---
name: testing-accessibility
description: Use when auditing a web app against WCAG (2.1/2.2 AA) — keyboard navigation, screen reader compatibility, color contrast, ARIA semantics, focus management in SPAs, motion/animation, form labeling, and the 44x44 touch-target rule, combining automated scans (axe-core, Lighthouse) with manual verification. Produces WCAG-referenced findings with remediation guidance. Canonical owner of the touch-target criterion referenced by testing-mobile.
---

# Accessibility (WCAG) Testing

Audit a web app so it's usable by people with motor, visual, cognitive, and auditory disabilities. Target WCAG 2.1 Level AA as the baseline (the common legal/SaaS standard); note where WCAG 2.2 AA tightens a criterion. Automated tools catch roughly a third of real issues — the rest require driving the app with a keyboard and a screen reader yourself.

## When to use

- Any request to audit, test, or report on accessibility.
- Before shipping a new form, modal, or navigation pattern — these are where focus-management bugs concentrate.
- Periodically against the whole app as a compliance sweep (see also testing-compliance for regulatory framing).

## Test setup

- **Automated:** axe-core (via Playwright or a browser extension) for rule-based scanning; Lighthouse for a scored audit.
- **Manual:** a real screen reader (NVDA on Windows, VoiceOver on macOS — both free/built-in) and a keyboard with the mouse physically disconnected or ignored.
- **Visual:** OS-level "reduce motion" and high-contrast/forced-colors settings to verify color/motion independence.

## Core pattern

**1. Automated scan first, on every major page.** Run an axe-core scan (`wcag2a`, `wcag2aa`, `wcag21aa` tags) against each route via Playwright, and/or a Lighthouse accessibility pass. Log every violation with its rule ID, affected element, WCAG success criterion, and impact level (critical/serious/moderate/minor). Treat this as the floor, not the ceiling — automated tools cannot judge whether an alt text is *meaningful*, only whether it's present.

**2. Keyboard-only pass.** Tab through the entire app. Verify: every interactive element is reachable; tab order matches visual/reading order; there are no keyboard traps; focus is *visibly* indicated at all times (WCAG 2.4.7); modals trap focus while open and return it to the trigger on close; dropdowns/menus respond to arrow keys, Enter, and Escape as expected. Walk 2–4 representative flows (sign-in, a multi-field form submission, an MFA-style step-up flow) keyboard-only end to end, not just individual widgets in isolation.

**3. Screen reader pass.** With NVDA or VoiceOver running, verify: a logical heading hierarchy (one `<h1>`, no level skips); labeled landmarks (`main`, `nav`, `header`); a skip-to-content link; every button/link has a meaningful accessible name (not "click here" or a bare icon); every input has an associated `<label>`; loading states, form errors, and toasts are announced via `aria-live`/`role="alert"`; modals expose `role="dialog"`, `aria-modal="true"`, and a labeled title; data tables use `<th scope>` and a caption or `aria-label`.

**4. Color & contrast.** Normal text needs ≥4.5:1 contrast, large text ≥3:1, UI components and focus indicators ≥3:1 (WCAG 1.4.3/1.4.11). Run the automated `color-contrast` rule, then manually confirm status indicators, validation errors, and disabled states don't rely on color alone (WCAG 1.4.1) — a status dot needs a text label or distinct icon alongside its color.

**5. Forms, one by one.** Every input has a programmatic label; required fields are indicated beyond color; error messages name the specific field and are announced (`aria-live`/`role="alert"`), not just shown as a red border; instructions appear before the input, not only after a failed submit; `autocomplete` attributes are set where applicable (`email`, `current-password`, `new-password`).

**6. Focus management in SPAs — the category automated tools miss entirely.** Route changes should move focus to the new page's heading or a skip link; opening a modal moves focus in, closing it returns focus to the trigger; async completions (a background job finishing, an item being added) should either move focus or announce via `aria-live`; form errors move focus to the error summary or first invalid field.

**7. Motion, images, page metadata.** Animations respect `prefers-reduced-motion`; nothing flashes >3×/second; images have `alt` (empty `alt=""` for decorative ones); every page has a unique `<title>` (WCAG 2.4.2) and the document sets `lang` (WCAG 3.1.1).

**8. Touch targets (mobile a11y, canonical here).** Interactive elements should be ≥44×44 CSS px (WCAG 2.5.5; WCAG 2.2 lowers the AA floor to 24×24). Pinch-to-zoom must not be disabled (`user-scalable=no` is itself a failure, WCAG 1.4.4). Content should reflow at 320px width without horizontal scroll (WCAG 1.4.10). testing-mobile may spot-check this during viewport testing but should defer the authoritative pass/fail here.

## Reporting format

```
WCAG:      1.4.1 Use of Color (Level A)
Component: Status badge on the services list
Issue:     Status conveyed by dot color only (green=running, red=stopped)
Impact:    Serious — unusable for users with color vision deficiency (~8% of men)
Evidence:  [screenshot with color-blindness simulation]
Fix:       Add a text label or distinct icon alongside the color dot
```

Severity mapping: `critical` blocks task completion (WCAG A), `serious` significantly impairs use (WCAG AA), `moderate` creates confusion/extra effort (WCAG AA), `minor` is best-practice/AAA — log but deprioritize.

## Pitfalls

- Stopping at the automated scan — it misses meaningful-alt-text quality, tab order sanity, and screen-reader announcement correctness entirely.
- Testing widgets in isolation instead of walking full flows keyboard-only — a single-widget check can pass while the flow around it traps focus.
- Treating a visible focus ring as optional styling instead of a WCAG 2.4.7 requirement.
- Reporting color-only status indicators as cosmetic instead of a Level A conformance failure.
- Skipping SPA-specific focus management, which has no browser default and must be implemented explicitly on every route change and async completion.
- Auditing only desktop viewport and skipping the 44×44/reflow criteria — mobile a11y is not a separate concern, it's part of this pass.

## Checklist

- [ ] Automated scan (axe-core `wcag2a`/`wcag2aa`/`wcag21aa`, and/or Lighthouse) run on every major page, violations logged with rule ID + WCAG criterion + impact
- [ ] Full keyboard-only walk of 2–4 representative flows; no traps; visible focus throughout
- [ ] Screen reader pass: heading hierarchy, landmarks, labeled controls, announced dynamic content
- [ ] Contrast checked (text ≥4.5:1/3:1, UI components ≥3:1); status never conveyed by color alone
- [ ] Every form: labels, field-specific announced errors, autocomplete attributes
- [ ] SPA focus management verified on route change, modal open/close, and async completion
- [ ] Reduced-motion respected; no flash >3/sec; every image has correct alt handling
- [ ] Unique page titles; `lang` attribute set
- [ ] Touch targets ≥44×44 (or 24×24 under WCAG 2.2); pinch-zoom not disabled; reflow at 320px
