---
name: testing-automation
description: Use when asked to write a runnable black-box test harness — regression suites, E2E browser tests, API test scripts, or fuzzers — against a live system with no source access. Covers building a reusable session/auth harness, structuring positive+negative test cases, browser (Playwright-style) journey tests, and shell-based endpoint enumeration. The meta-skill of constructing the automation itself, not any single test's methodology.
---

# Automated Black-Box Test Writing

Write test **code**, not a one-off transcript of manual checks. Every test must be reproducible by anyone with network access and no internal knowledge — self-contained, handling auth/session setup automatically, producing clear pass/fail output.

> Provides the harness scaffolding (requests/Playwright/shell templates) that other testing skills' checks get encoded into as regression tests. For the specific things to check, see [[testing-api]], [[testing-authz-matrix]], [[testing-data-fuzz]], etc. For the fast deploy gate, see [[testing-smoke]].

## When to use

- Turning a manually-discovered bug or behavior into a regression test that runs on every future change.
- Building a suite from scratch for a system with no existing test automation and no source access.
- Any request for "write tests for X" against a live HTTP surface or browser UI.

## Core pattern

**Build one authenticated-session helper class and make every test use it.** It should: perform the auth handshake (register/login, capture the token, plant/track any CSRF or anti-forgery cookie), expose thin `get`/`post`/`put`/`delete` wrappers that attach the right headers automatically, and re-derive any rotating anti-forgery token after each mutation. Every test file imports this instead of reimplementing the handshake — the handshake is exactly the part that breaks tests mysteriously when done ad hoc per-file.

**Provision test identities inside the test, never via hardcoded credentials.** Self-registration with a timestamped/random throwaway email keeps the suite runnable by anyone, anytime, without a shared account to coordinate. Use fixture scoping (module/session-scoped) so a suite that needs two independent identities doesn't recreate them per-test.

**Structure suites as classes/groups by concern**, each with both a positive and a negative case: auth, CSRF/anti-forgery enforcement, authorization (see [[testing-authz-matrix]] for what to cover), rate limiting, public/unauthenticated endpoints, input validation. A test file that only has positive assertions is a smoke test wearing a regression suite's clothes.

**Name tests `test_<what>_<expected_outcome>`** so a failure's name alone tells you the broken contract (`test_idor_service_returns_404`, `test_mutation_without_csrf_returns_403`).

**Mark known, accepted failures explicitly** (an `xfail`/known-issue marker with a reason and a link to the tracking ticket) rather than deleting or skipping the test silently — this keeps the suite green while still flagging the gap, and the test flips to a hard failure the moment someone "fixes" the regression back in.

**Browser/E2E tests self-provision through the API, not through hardcoded UI-only credentials.** Use the same throwaway-account technique via a direct API call before driving the browser, so the E2E suite doesn't depend on a shared test account existing in a particular state.

**Shell-based endpoint enumeration is a separate, lighter-weight tool** from the pytest/Playwright suites — a wordlist of plausible undocumented-path fragments (ops/debug/internal-sounding words, siblings of known routes) probed with curl, escalating to a proper fuzzer (e.g. ffuf) if available. Treat any non-404 as worth a manual look, not an automatic finding.

## Design decisions

- **The harness's session object owns anti-forgery-token rotation** so individual tests never have to remember to re-read it — a forgotten re-read is the single most common false-failure in hand-written suites against CSRF-protected APIs.
- **Fixtures over setup/teardown boilerplate per test** — cheaper to read, and framework-native fixture scoping controls how much state is shared vs isolated.
- **Runnable with a one-line install+run command** (`pip install ... && pytest ...`, `npm i && npx playwright test`) — a suite nobody can run in under a minute won't get run.

## Pitfalls

- Reimplementing the auth/CSRF handshake per test file instead of sharing one harness — guarantees drift and mystery 403s.
- Hardcoding a shared test account's credentials — breaks the moment two people or two CI runs use it concurrently.
- Silently deleting a test for a known-broken behavior instead of marking it as an expected/tracked failure — the regression suite stops protecting against that exact bug re-appearing later.
- Writing only positive-path browser journeys — the highest-value E2E tests are the ones that prove a guard/redirect fires (unauthenticated access, wrong-role access, 404 handling).
- Treating shell enumeration hits as confirmed findings without a manual follow-up — a non-404 is a lead, not a verdict.

## Checklist

- [ ] One shared session/auth harness; no per-file reimplementation of the handshake
- [ ] Throwaway self-provisioned identities, not shared hardcoded credentials
- [ ] Suites structured by concern with both positive and negative cases
- [ ] Test names encode the expected outcome
- [ ] Known-broken behaviors marked as tracked xfail, not deleted or silently skipped
- [ ] Browser/E2E tests self-provision via API before driving the UI
- [ ] One-command install+run documented
- [ ] Endpoint-enumeration hits treated as leads requiring manual triage, not auto-findings
