---
name: testing-functional
description: Use when verifying that a product works correctly from a real user's perspective by driving the live UI — covers core user journeys (signup through primary action, resource management, billing, support, account/security, privileged roles), a standard error-state matrix (401/404/402/429/offline/validation/500), watching network+console alongside the DOM, and a structured bug-report format. For DOM/console/network-only inspection see [[testing-browser-ui]]; narrow-viewport see [[testing-mobile]]; accessibility see [[testing-accessibility]]; raw API coverage see [[testing-api]].
---

# Testing Functional / Browser QA

Functional QA driven through the real UI, as a user experiences it — no source, no server access, just the browser and the network traffic it generates. The goal is confirming that core journeys complete end-to-end and that error, loading, and edge states are handled gracefully, not just that a screen renders.

## When to use

- Verifying a release from the user's point of view, beyond what a smoke test's tiered API pings can see.
- Any bug report where "does the UI actually reflect what the API returned" is in question.
- Before/after a UI-affecting change, to catch regressions a pure API test would never see (spinners that never resolve, success responses the UI fails to render).

## Core pattern

**Watch four things on every screen**, not just what renders: the Network tab (right calls, right status codes, no stray 4xx/5xx), the Console (no JS errors or unhandled rejections), the rendered content (does it match what the API actually returned, or is there a stale/mismatched value), and loading/skeleton states (do they resolve, or hang).

**Walk core journeys end-to-end**, one journey = one realistic user goal, not one screen:
- **Onboarding → first value.** Sign up, complete any onboarding, and perform the product's primary action (add the first resource, send the first message — whatever "day one success" means for this product). Confirm no error toast, and that state transitions (queued → provisioning → active, or equivalent) are visible and correct.
- **Resource management.** For an existing resource, exercise every lifecycle action available in its detail view (start/stop, edit, duplicate, delete, any async operation) and confirm each maps to the expected API call and resulting status.
- **Billing/payment**, if applicable: invoice list and download, payment method management, balance/ledger updates, spend limits.
- **Support/communication channel**: create a request, confirm it threads correctly, reply, check status transitions.
- **Account & security**: profile edits, credential changes, MFA setup, session list and revocation.
- **Privileged roles** (admin/support-agent), if credentials are available: search/lookup tools, any "view as" or impersonation feature (confirm the banner appears and clears correctly, and that impersonated data is scoped correctly), and an audit trail if one exists.

**Assert the error-state matrix on top of the happy path**, not as an afterthought — for each journey, deliberately trigger:

| Scenario | How to trigger | Expected UI |
|---|---|---|
| Unauthenticated | Clear the stored credential, reload | Redirect to login, no blank screen |
| Not found | Navigate to a non-existent resource ID | A 404 page, not a blank/crashed screen |
| Payment/quota required | Trigger the paywall/limit path | A specific, actionable message — not a generic error |
| Rate limited | Rapid repeated submits | Retry guidance, not a silent failure |
| Offline/network error | Simulate offline, attempt an action | Graceful error, not an unhandled exception |
| Validation | Submit invalid/empty required fields | Inline messages, not a raw 400 |
| Server error | Force or wait for a 500 | Generic error surface, no stack trace leaked to the user |

## Design decisions

- **The API response is the source of truth for "is this a bug."** If the API returned the right data but the screen didn't update, that's a UI bug (state/render), distinct from an API bug — say which in the report.
- **Report format ties the visual bug to the network evidence**, so it's reproducible without re-driving the browser:

```
Page/Route:   /support/tickets/:id
Action:       Post reply
Expected:     Reply appears in thread immediately
Actual:       Spinner runs indefinitely; thread never updates
API call:     POST /api/tickets/:id/replies → 201 (response is correct)
Diagnosis:    UI success handler not firing after API success
Severity:     High (core journey broken)
```

- **Destructive actions are in scope** on your own/test-provided resources — delete, suspend, rebuild — since untested destructive paths are exactly where confirmation dialogs and cleanup logic rot. Capture state before destroying it so findings stay reproducible.

## Pitfalls

- **Trusting the screen without checking the Network tab** — a component can render stale or cached data while genuinely believing the API call succeeded; always cross-check the actual response.
- **Only testing the happy path** — the error-state matrix above catches a disproportionate share of real user-facing bugs (unhandled exceptions on network loss, generic 500 pages that leak stack traces).
- **Skipping loading-state observation** — a spinner that never resolves is invisible if you only check the end state.
- **Reporting a UI bug without the API evidence** — "it doesn't work" is not reproducible; always capture the actual request/response alongside the screenshot.
- **Ignoring role-scoped views** — data leaking across roles (a support agent seeing admin-only fields, one tenant's view showing another tenant's data) only shows up if you test with the actual role's credentials, not by simulating it.

## Checklist

- [ ] Network, Console, rendered content, and loading states all watched per screen, not just the final render
- [ ] Onboarding-to-first-value journey walked end-to-end
- [ ] Resource lifecycle actions each mapped to their expected API call and result
- [ ] Billing, support, and account/security journeys walked
- [ ] Privileged-role journeys walked if credentials available, including impersonation/view-as scoping
- [ ] Full error-state matrix exercised (401/404/paywall/429/offline/validation/500)
- [ ] Every finding includes route, action, expected vs. actual, and the actual API call/response
- [ ] Destructive actions tested on disposable resources with state captured first
