---
name: api-tokens
description: Use when exposing an API to machines/integrations — issue bearer tokens that are hashed at rest, prefixed for identification, scoped, and logged per request. Covers the "show once" UX and safe revocation.
---

# API Tokens (machine access)

Bearer tokens for programmatic access, designed so a database leak doesn't leak usable credentials.

## When to use

Any time non-browser clients (integrations, scripts, CI) call your API. Distinct from user sessions: tokens are long-lived, scoped, and revocable individually.

## Core pattern

- **Format:** `<app>_<env>_<random>` e.g. `ats_live_9f3c…`. The prefix identifies the product and environment at a glance (and lets secret-scanners recognize it).
- **Store only a hash.** Keep `SHA-256(token)` in the DB, never the token itself. On each request, hash the presented token and look it up. The plaintext is shown to the user **exactly once** at creation.
- **Store a short display suffix** (last 4 chars) so users can recognize which token is which in the UI without you keeping the secret.
- **Scope tokens** to the same permission catalog as users (see `authorization-rbac`), so a token can be least-privilege.
- **Per-request audit log** — record token id, endpoint, timestamp, status. This powers usage analytics and incident response ("what did this token touch?").

## Data model

```
api_tokens(id, tenant_id, name, token_hash, prefix, suffix, scopes, created_by,
           last_used_at, expires_at, revoked_at)
api_request_log(id, token_id, method, path, status, ip, created_at)
```

## Pitfalls

- **Never log the raw token** (headers included) — scrub `Authorization` in request logs.
- Hash lookups must be indexed on `token_hash`.
- Support **expiry and revocation**; check both on every request. Revocation should be instant (no cache, or cache-bust on revoke).
- Don't reuse the user-session verification path — tokens skip cookies/CSRF and need their own middleware branch.
- Rate-limit by token, not just by IP.

## Checklist

- [ ] Prefixed token format (`app_env_random`)
- [ ] Only the SHA-256 hash stored; plaintext shown once
- [ ] Displayable suffix for recognition
- [ ] Scopes reuse the permission catalog
- [ ] Expiry + revocation checked per request
- [ ] Per-request audit log, Authorization header scrubbed
