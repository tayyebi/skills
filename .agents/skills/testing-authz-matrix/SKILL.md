---
name: testing-authz-matrix
description: Use when systematically verifying role-based access control across an API — sweeping every endpoint against every role in a loop, checking unauthenticated/wrong-role/cross-tenant access all fail correctly, probing self-escalation and impersonation, and reporting a pass/fail role×endpoint matrix. This is coverage verification, NOT penetration testing — see testing-pentest-authz for adversarial depth.
---

# RBAC & Authorization Matrix Testing

Sweep role × endpoint combinations systematically and report a matrix, not a narrative. A user hitting a route they shouldn't should always get the same wrong-role response — if even one combination returns success, that's a bug, not an ambiguity. The deliverable is coverage: every private route checked against every role, every result recorded.

> Owns the RBAC policy matrix (role × endpoint expected-code regression). For adversarial IDOR/privilege-escalation depth see [[testing-pentest-authz]]; for the broader OWASP methodology see [[testing-security]].

## When to use

- After adding a new role, permission, or protected endpoint.
- As a regression suite that re-runs on every deploy to catch an authorization check that got dropped in a refactor.
- Before shipping any change that touches middleware, route guards, or the permission model.

## Core pattern

**Establish the ground rules before sweeping.** Typically: no credential → 401; wrong role/permission → 403; resource that exists but belongs to another tenant → 404 (not 403 — a 403 on a specific resource ID confirms the resource exists, which is itself a disclosure). Confirm which convention this system actually uses before asserting failures against it.

**Acquire one token per role** (self-service registration for the lowest privilege tier; request or provision the rest) and build a small helper that fires the same request shape across all of them, collecting a status code per (role, endpoint) cell.

**Sweep 1 — unauthenticated.** Every private endpoint, GET and mutation, with no credential at all. All must return the "no auth" code; anything else is an information leak or a broken guard.

**Sweep 2 — lowest-privilege role against every higher-privilege route.** Loop every admin/privileged endpoint with the lowest role's token; every one must return the "wrong role" code. Do this for both reads and mutations — a route often guards GET correctly but leaves PUT/POST/DELETE unchecked.

**Self-escalation attempts.** Try to set a privileged field (role, permission list, `isAdmin`) via every mutation endpoint that touches the actor's own record — profile update, registration body, mass-assignment via extra/prototype-polluting fields. All must be silently stripped or rejected server-side; verify by re-reading the record after the attempt, not just trusting the response body.

**Impersonation/delegation features, if present.** Confirm the elevated role can use the feature and the returned identity actually changes; confirm a non-privileged actor sending the same header/param is ignored, not honored; confirm an impersonation target that doesn't exist degrades to 400/404, not 500.

**Cross-tenant resource access.** For every resource type, create it (or locate one) under identity A, then request/mutate it under identity B. Expect not-found, not forbidden. Cover read, update, and delete — a delete endpoint that leaks a 403-vs-404 distinction, or worse, succeeds, is a critical finding.

**Session/token lifecycle.** A revoked token must fail immediately on the next request; other live sessions for the same account must be unaffected by one session's revocation (unless the system intentionally revokes all sessions together — confirm which behavior is intended).

**Role/field value sanity.** Every identity's role field should be one of the documented enum values — this catches silent data-model drift as a side effect of the sweep.

## Pitfalls

- Sweeping only GETs — mutation endpoints (POST/PUT/DELETE) are checked far less often and are where privilege bugs concentrate.
- Accepting 403 as equivalent to 404 for cross-tenant resource access when the system's own convention is 404 — a 403 discloses existence.
- Trusting the mutation's response body to prove a field was rejected — always re-fetch the record independently to confirm state didn't change.
- Testing impersonation only for the elevated role — the more important check is that a non-privileged actor's attempt is ignored.
- Treating "one critical/high finding" as a signal to stop the whole sweep — finish the matrix; a partial matrix hides other gaps.

## Checklist

- [ ] Ground-truth convention established: no-auth code, wrong-role code, cross-tenant code
- [ ] One token per role, one helper firing the same request shape across all
- [ ] Unauthenticated sweep across every private GET and mutation
- [ ] Lowest-role-vs-privileged-route sweep, reads and mutations both
- [ ] Self-escalation attempts on every self-mutating endpoint, verified by re-read
- [ ] Impersonation/delegation checked both for the privileged role and for a non-privileged actor attempting it
- [ ] Cross-tenant read/update/delete swept per resource type
- [ ] Token revocation and session-independence verified
- [ ] Role/permission field values checked against the known enum
- [ ] Results reported as a role × endpoint matrix, not just a pass count
