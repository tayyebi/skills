---
name: testing-security
description: Use when pen-testing your own web application or API before attackers do — a breadth-first OWASP sweep covering authentication attacks, session management, authorization bypass/IDOR, CSRF, injection (command/SQL/XSS/SSRF), business-logic abuse, webhook signature bypass, info disclosure, and transport security. Gray-box: bring whatever intel you have on likely weak spots, then verify empirically. For deeper authorization drill-downs see testing-pentest-authz; for deeper injection drill-downs see testing-pentest-injection.
---

# Web App Security Testing (OWASP Sweep)

A systematic, black-box (or gray-box) pass over your own application's attack surface, organized around the OWASP Top 10 categories. The goal is to find real, reproducible vulnerabilities and document them so they get fixed — not to demonstrate cleverness.

## When to use

- Before shipping a new auth flow, payment path, or any feature that touches money, PII, or another tenant's data.
- Periodically against a live system you are authorized to test, as a broad first pass before drilling into specific attack classes.
- **Only against systems you own or have explicit authorization to test.** This is authorized security testing on your own platform, not opportunistic probing of someone else's. Coordinate with stakeholders before any high-blast-radius action (volumetric load, bulk account actions, anything that could affect other live users) — time these deliberately. If you find you can reach another tenant's data, stop at proof of access; don't actually exfiltrate or destroy it. Handle any PII you touch minimally and mask it in reports.

## Core pattern

**Set up two identities.** Register (or obtain) two separate accounts — an "attacker" and a "victim" — each with their own session. Nearly every authorization and IDOR check needs a second account whose resources you attempt to reach from the first.

**Work the categories in order, each with a clear pass/fail:**

1. **Authentication** — does login/registration leak whether an email exists (identical error body and timing for wrong-password vs. unknown-email)? Does a rate limit exist on login, and can it be bypassed by rotating a spoofable header (e.g. `X-Forwarded-For`)? Is the password-reset token single-use, expiring, high-entropy, and delivered out of the URL (not logged via referrer)? Does MFA actually gate the protected endpoint, or can the partial (pre-MFA) session reach it?
2. **Session management** — does logout invalidate the token server-side (not just client-side)? Does "revoke all other sessions" actually revoke them? Does a password change invalidate old tokens?
3. **Authorization / IDOR** — cross-account access to another user's resources (read, modify, delete) should be denied, conventionally with 404 (not 403, which leaks existence). Admin-only routes must reject a non-admin token. See `testing-pentest-authz` for the full escalation matrix — this pass only needs representative smoke checks.
4. **CSRF** — state-changing requests without a valid anti-CSRF token (missing, empty, or wrong value) must be rejected. Confirm cookie flags (`SameSite`, `Secure`, `HttpOnly` as appropriate). Webhook/callback endpoints are usually CSRF-exempt by design — but they must then require their own signature verification (see item 7 below).
5. **Injection** — command injection in any field that reaches a shell or subprocess call; SQL injection in search/filter/id fields; SSRF in any field where the server fetches a URL you control (webhook targets, callback domains, avatar/image-by-URL). See `testing-pentest-injection` for the full payload corpus — this pass only needs a handful of representative payloads per field class.
6. **Business logic** — negative/zero quantities, client-supplied price/amount fields, self-referral loops, race conditions on balance-affecting concurrent requests (e.g. simultaneous "add funds" and "spend"), and any "field is silently ignored" behavior that a user would reasonably expect to take effect.
7. **Signed callback / webhook bypass** — endpoints that accept externally-triggered events (payment providers, third-party callbacks) and are CSRF-exempt must validate a cryptographic signature. Forge an event with no or a garbage signature and confirm it's rejected — an unsigned "payment succeeded" event that credits an account is free money.
8. **Stored XSS** — submit script payloads into any field a privileged viewer (admin, support agent) will later render (names, ticket bodies, notes), then check whether it executes in that privileged view. Also test file-upload fields for SVG/HTML content smuggled past a content-type check.
9. **Info disclosure** — internal/debug/metrics routes reachable from outside, permissive CORS on doc/spec endpoints, exposed `.git`/backup files, and stack traces or internal errors leaking in 5xx bodies.
10. **Transport & headers** — HSTS, CSP, `X-Frame-Options`, and correct cookie flags on every auth-relevant cookie.

## Reporting

For every finding, capture the exact HTTP exchange and separate **confirmed** (reproduced) from **suspected** (a lead you didn't fully verify). A finding needs: title, severity, relevant OWASP category, preconditions, exact repro steps, expected vs. actual, impact, and a concrete fix.

## Pitfalls

- Reporting a lead as confirmed without reproducing it — wastes engineering time chasing phantoms.
- Testing destructively on a shared resource instead of stopping at proof-of-access.
- Skipping the "identical response" check on enumeration-prone endpoints — timing differences count too.
- Treating a 403 as equivalent to a 404 on cross-tenant access — 403 confirms the resource exists, which is itself a (usually low-severity) leak.
- Chasing every category to full depth here instead of handing authz/injection depth off to their dedicated passes.

## Checklist

- [ ] Two independent accounts (attacker + victim) set up
- [ ] Authentication: enumeration, rate-limit bypass, reset-token strength/single-use, MFA gating
- [ ] Session: logout revokes server-side, revoke-all works, password change invalidates old tokens
- [ ] Authorization smoke: cross-account IDOR (404, not 200), admin routes reject non-admin
- [ ] CSRF enforced on mutations; cookie flags correct; webhook exemption backed by signature checks
- [ ] Injection smoke: command/SQL/SSRF payloads in a few representative fields
- [ ] Business logic: negative/zero values, price tampering, race conditions, self-referral
- [ ] Signed-callback bypass: forged event with no/garbage signature rejected
- [ ] Stored XSS: payloads render safely in privileged viewer contexts and upload paths
- [ ] Info disclosure: debug routes, CORS, backup files, stack traces in errors
- [ ] Transport: HSTS/CSP/X-Frame-Options and cookie flags present
