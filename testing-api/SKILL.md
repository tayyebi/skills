---
name: testing-api
description: Use when testing a REST/HTTP API as a black box (curl, requests, or similar) with no source access — covers session-token and CSRF mechanics, rate-limit tiers, happy-path endpoint-group coverage, negative-path/status-code correctness, pagination shape, and error-body consistency.
---

# API-Layer Testing

Test the API at the HTTP level: send requests, read status codes and bodies, and confirm behavior matches contract for both the happy path and every negative/edge case. No source, no server access — everything is inferred from the wire.

**Goal:** every request gets the *right* status code, the *right* body shape, and the *right* headers — including the ones you expect to fail.

## When to use

- Regression-testing a REST API directly, independent of any UI.
- Negative-path and edge-case coverage that a UI can't easily trigger.
- Validating that responses match an implicit or explicit contract (see [[testing-contract]] for the formal spec-diff angle).

Distinct from [[testing-contract]] (documentation/spec coverage), [[testing-authz-matrix]] (systematic role×endpoint sweep), [[testing-data-fuzz]] (input-boundary depth), and [[testing-smoke]] (fast deploy gate).

## Core pattern

**Learn the auth/CSRF mechanics first, before writing a single test.** Every API has its own handshake: how a session token is minted and shaped (opaque random string vs JWT — decode it if it's a JWT, treat it as opaque otherwise), how it's presented (bearer header, cookie), and whether mutations require a second proof (CSRF double-submit token, signed nonce). Get this wrong and every subsequent test either 403s spuriously or silently skips a check.

**Build one reusable harness, not one-off curl calls.** A short helper library — plant the session, register/authenticate a throwaway account, wrap authenticated GET and CSRF-safe mutation calls — pays for itself after the third endpoint. Keep a `$BASE` variable so the whole suite retargets with one edit.

**Walk endpoint groups systematically for happy-path coverage.** Group endpoints by resource (auth, core CRUD resource, billing/account, sub-resources) and, for each, hit list/get/create/update/delete in turn, confirming the documented status code and response shape.

**Then invert every happy path into a negative case.** For each mutating endpoint: missing CSRF proof, missing/malformed auth, non-existent-but-well-formed ID, unknown JSON field, oversized payload, missing required field (both omitted and explicit `null` — they must behave identically), wrong `Content-Type`, and a type-confused field (string where a number is expected, etc.). None of these should ever produce a 5xx.

**Check pagination and list shape on every list endpoint**: consistent envelope (`data`/`total`/`page`/`perPage` or equivalent — flag any endpoint returning a bare array while siblings return an envelope), out-of-range page → empty data not an error, invalid page/perPage values → 400 not 500, oversized `perPage` capped or rejected.

## Status code expectations

| Situation | Expected |
|---|---|
| Successful read | 200 |
| Created resource | 201 |
| Async operation started | 202 |
| Successful delete | 204 |
| Bad request / validation | 400 |
| Missing/invalid auth | 401 |
| Payment/quota required | 402 |
| Wrong role or missing CSRF | 403 |
| Not found or wrong tenant | 404 |
| Conflict (duplicate) | 409 |
| Rate limit exceeded | 429 + `Retry-After` |
| Server error | 500 — should be rare, and the body must never leak a stack trace |

## Pitfalls

- Testing only the happy path — negative paths (bad auth, bad CSRF, malformed body) are where real bugs live.
- Treating a 500 as "the input was invalid enough" — a 500 is always a bug; the correct response to bad input is a clean 4xx.
- Comparing `null`-value and omitted-field behavior only for one, not both — they should be handled identically for required-field validation.
- Ignoring response *shape* consistency — a bare array on one list endpoint and an envelope on another breaks every client that assumes uniformity.
- Hardcoding a session token/cookie once and reusing it after a mutation rotates it — re-read the session/CSRF state after every mutating call.
- Rate-limit testing that turns into an actual volumetric attack — stay inside the number of requests needed to observe the boundary, and only on infrastructure you're authorized to test.

## Checklist

- [ ] Auth + CSRF/anti-forgery mechanics documented before testing starts
- [ ] Reusable harness (helpers for GET / CSRF-safe mutation) with a `$BASE` variable
- [ ] Happy-path pass over every endpoint group, status code + shape checked
- [ ] Negative-path pass: no-auth, bad-auth, missing-CSRF, bad-ID, unknown-field, oversized-payload, missing-vs-null, wrong-Content-Type, type-confusion — never a 500
- [ ] Pagination/list shape consistent and correct at the boundaries
- [ ] Rate-limit tier and `Retry-After` verified without mounting a real attack
- [ ] Error bodies checked for a single consistent key/shape across endpoints
