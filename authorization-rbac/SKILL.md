---
name: authorization-rbac
description: Use when an app needs more than one kind of user — role-based access control with a fine-grained permission catalog, a single enforcement gateway, and per-request audit. Covers roles→permissions mapping and cross-tenant admin overrides.
---

# Authorization (RBAC + permission catalog)

Separate **roles** (what someone *is*) from **permissions** (what they can *do*), and enforce both through one chokepoint.

## When to use

As soon as you have more than "logged in vs. not." The moment you write `if (user.role === 'admin')` in a second place, adopt this pattern.

## Core pattern

1. **Permission catalog** — an explicit, enumerated list of fine-grained capabilities, grouped by domain:
   `candidates.read`, `candidates.write`, `users.invite`, `settings.edit`, `system.logs.view`, `database.manage`, …
2. **Role → permission map** — each role (e.g. `admin`, `recruiter`, `manager`, `viewer`, `super_admin`) maps to a set of permissions. Roles are sugar over permissions; permissions are the real currency.
3. **One enforcement gateway** — a single `requireAuth({ permission })` function that every protected handler calls. It:
   - validates the session,
   - blocks disabled users,
   - confirms tenant membership,
   - checks the required permission,
   - returns an `audit()` helper bound to the actor.

```
{ user, audit } = requireAuth(request, permission: "candidates.write")
... do the work ...
audit("candidate.updated", { id })
```

## Design decisions

- **Deny by default.** Handlers must *opt in* by declaring a required permission; a handler with no declared permission should be treated as a bug, not "public."
- **Check permissions, not roles, in business code.** `can(user, 'settings.edit')` survives role renames and new roles; `role === 'admin'` does not.
- **Super-admin / cross-tenant override** — a platform operator role can act across tenants. Make this an explicit branch in the gateway (and always audit it loudly), never an implicit "admin sees everything."
- Keep the catalog **centralized** in one module so the full surface is greppable and reviewable.

## Pitfalls

- Scattering role checks across the codebase — centralize in the gateway.
- Forgetting that authorization is **per-tenant**: a user may be admin in tenant A and viewer in tenant B. Resolve the effective role in the current tenant context.
- UI-only enforcement. Hiding a button is UX; the server permission check is the actual boundary.
- Permission sprawl: prefer a modest set of meaningful permissions over one-per-endpoint.

## Checklist

- [ ] Enumerated permission catalog in one module
- [ ] Role → permission map
- [ ] Single `requireAuth`/`can` enforcement gateway
- [ ] Deny-by-default; handlers declare required permission
- [ ] Per-tenant effective role resolution
- [ ] Cross-tenant override is explicit + audited
