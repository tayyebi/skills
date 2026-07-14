---
name: multi-tenancy-rls
description: Use when building a B2B/SaaS app where each customer's data must be isolated in a shared database. Enforce tenant isolation at the database layer with PostgreSQL Row-Level Security instead of trusting every query to add a WHERE clause.
---

# Multi-Tenancy via Row-Level Security

Isolate tenant data in the **database**, not in application code, so a forgotten `WHERE tenant_id = ?` can't leak one customer's data to another.

## When to use

Any shared-database multi-tenant app (`/tenant/[slug]/…` routing, org accounts, workspaces). Especially when correctness of isolation matters more than squeezing out the last drop of query performance.

## Core pattern

1. Every tenant-scoped table has a `tenant_id` (or `org_id`) column.
2. Enable **RLS** on those tables and add a policy:
   ```sql
   ALTER TABLE candidates ENABLE ROW LEVEL SECURITY;
   CREATE POLICY tenant_isolation ON candidates
     USING (org_id = current_setting('app.current_org_id')::uuid);
   ```
3. Wrap every request's DB work in a transaction that sets the tenant as a **transaction-local** variable:
   ```sql
   SELECT set_config('app.current_org_id', $1, true);  -- true = local to txn
   ```
   Now every query in that transaction is automatically constrained. Application code stops carrying isolation responsibility.

Provide a helper, e.g. `withOrg(orgId, fn)`, that opens the transaction, sets the config var, runs `fn`, and commits.

## Design decisions

- **Transaction-local (`set_config(..., true)`)** is critical: it prevents the setting from leaking to the next request on a pooled connection.
- The application DB role must **not** be a superuser or table owner — RLS is bypassed by `BYPASSRLS`/owner roles. Run app queries as a restricted role.
- Keep a separate, explicit **admin/migration path** that intentionally bypasses RLS, used only by trusted platform code.
- Route or middleware resolves the tenant (from slug/subdomain/session) once, then hands it to `withOrg`.

## Pitfalls

- **Connection pooling leakage** — the #1 bug. Always use transaction-local settings, never `SET` session-wide on a pooled connection.
- Forgetting RLS on a *new* table. Add a test/CI check that every tenant table has RLS enabled.
- Cross-tenant admin features must deliberately elevate; don't disable RLS globally to make them work.
- Background jobs run outside a request — they must set the tenant context explicitly too.

## Checklist

- [ ] `tenant_id` on every scoped table
- [ ] RLS enabled + policy per table (CI check for coverage)
- [ ] App runs as a non-owner, non-superuser role
- [ ] `withOrg()` sets a **transaction-local** setting
- [ ] Jobs/admin paths set tenant context explicitly
