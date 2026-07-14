---
name: testing-browser-ui
description: Use when testing UI flows in a real, live browser rather than against the API directly — driving an actual tab (via a browser-automation MCP tool or Playwright) to walk user journeys, read console errors, inspect network requests, audit the HTML head for stale metadata, and spot-check responsive layout. Covers session setup, per-page console/network auditing, and a findings-report format. For deeper mobile-only concerns see testing-mobile; for WCAG see testing-accessibility; for scripted end-to-end journeys see testing-functional; for raw API testing see testing-api.
---

# Live Browser UI Testing

Black-box test a web app by driving a real browser tab — not by asserting against the API, but by looking at what a user actually sees: rendered pages, console noise, network calls, and page metadata. This catches a class of bug that API tests can't: broken client-side wiring, leaked debug info in the DOM, requests firing without auth, stale branding in `<head>`, and visual layout breakage.

## When to use

- Any request to "test the UI," "check for console errors," "verify a page renders," or "walk this journey in a browser."
- As a first pass before or alongside deeper specialist passes (accessibility, mobile, functional journeys) — this skill establishes the baseline: does the page load clean, and does the network traffic look sane.
- When you have (or can get) a real browser tab under control — via a browser-automation MCP tool (CDP- or extension-backed) or a Playwright script. Either works; the checks below are tool-agnostic.

## Core pattern

**1. Get a controlled tab and enable network capture.** Open or attach to a tab, navigate to the target URL, and turn on network monitoring before doing anything else — you want to catch every request from the first paint, not just ones after you remember to look.

**2. Walk the journey, screenshotting each stop.** Navigate to each page in the flow under test (login, primary dashboard/home, list views, a create/edit form, checkout or settings pages) and capture a screenshot before and after any state-changing action. Screenshots are your evidence trail, not decoration — attach them to findings.

**3. Read console and network on every page, not just the first.** After each navigation:
   - Pull the console log — flag any uncaught error, CSP violation, deprecation warning, or failed network-triggered exception.
   - Pull the recent network requests — flag any unauthenticated call to an endpoint that should require auth, any 4xx/5xx on a read path the page depends on, any unexpected third-party domain, and any endpoint that reveals more than the current user should see (e.g. feature flags, internal config).

**4. Audit the HTML `<head>` once per session.** Read the `<head>` DOM and check `<title>`, `<meta name="viewport">`, Open Graph / Twitter tags, and `<link rel="canonical">` for correctness and internal consistency (same product name and domain everywhere). Also scan for leaked HTML comments — internal ticket references, TODOs, or debug notes should never ship to production markup.

**5. Exercise interactive elements without committing side effects.** Open modals, wizards, and multi-step forms far enough to confirm they render and their dependent data loads (e.g. a dropdown's options came back with data), then back out (Escape, browser-back) instead of submitting, unless the test explicitly calls for a real submission.

**6. Spot-check responsive layout while you're already in the tab.** A quick resize-and-screenshot at a narrow viewport (see testing-mobile for the full breakpoint matrix) and a rough touch-target/contrast glance (see testing-accessibility for the canonical criteria) are cheap wins here, even though they're not this skill's specialty — flag anything obviously broken and hand deeper investigation to the specialist skill.

## What to flag

| Console signal | Typical severity | Why it matters |
|---|---|---|
| Uncaught `TypeError` / "Cannot read properties…" | Medium–High | Client-side logic broke, possibly silently |
| Failed `fetch`/XHR for data the page needs (401/500) | High | Feature is broken even if the page "looks" fine |
| CSP violation | Low–Medium | Either a real policy gap or overly strict policy blocking legitimate resources |
| Deprecated API / meta-tag warning | Info | Cheap to fix, low urgency |

| Network signal | What to check |
|---|---|
| Unauthenticated call to a should-be-protected endpoint | Does it 401/403 correctly, or does it leak data? |
| A "reporting"/telemetry endpoint failing (e.g. 403) on every load | Low severity but noisy — flag as tech debt |
| Endpoints exposing feature flags or config | What's visible to an ordinary user that shouldn't be? |
| Unexpected third-party domains in the request list | Should be minimal and match the CSP allowlist |

## Reporting format

```
Page:       Dashboard (/dashboard)
Finding:    POST /api/errors returns 403 on every page load
Console:    Failed to load resource: 403 (/api/errors)
Request:    POST /api/errors — no Authorization header sent
Expected:   Either accept unauthenticated error reports, or don't call the endpoint before auth
Actual:     403 on every load — generates console noise and may silently drop real error reports
Severity:   Low
Fix:        Rate-limit and allow unauthenticated POSTs, or gate the call behind an active session
```

## Pitfalls

- Only screenshotting — a page can look fine and still be throwing console errors or firing broken network calls underneath.
- Testing only the happy path's first screen and never checking console/network on subsequent pages in the journey.
- Submitting destructive test actions (real wizard completions, real deletes) when a read-only exploration would have answered the question.
- Treating a clean screenshot as proof of "no bugs" — always cross-check against the console and network tabs for that same page load.
- Never verifying page-metadata consistency until a rebrand or domain migration ships half-finished across `<head>` tags.
- Using a real or plausible-looking credential in test scripts or reports — use an obviously-fake placeholder (e.g. `qa+test@example.com`) and pull real secrets from environment variables, never hardcode them in a checked-in script.

## Checklist

- [ ] Tab under control, network monitoring enabled before first navigation
- [ ] Each page in the journey: screenshot + console log + recent network requests reviewed
- [ ] No uncaught JS errors; no failed fetches on required data endpoints
- [ ] Auth-gated endpoints reject unauthenticated calls; no unexpected data exposure
- [ ] `<head>` metadata internally consistent (title, canonical, OG/Twitter tags, no leaked comments)
- [ ] Modals/wizards explored read-only where a real submission isn't the point of the test
- [ ] Findings include page, exact request/response, expected vs. actual, and severity
