---
name: idempotency-keys
description: Use when a client (or a CDN/load balancer) may retry a write and you must not perform it twice — a mutating endpoint that caches its response under a client-supplied Idempotency-Key and replays the stored result on a repeat. Essential when requests can be retried against a different replica in an active-active cluster, or when a flaky network makes clients resend POSTs.
---

# Idempotency Keys

Let a client safely retry a write. The client sends a unique `Idempotency-Key` header; the first request runs the handler and caches its response keyed by that value; any replay returns the *stored* response verbatim instead of re-executing. A double-submit, a network retry, or a CDN failover to another replica all become no-ops.

## When to use

- Mutating endpoints (`POST`/`PATCH`) where "the client sends it twice" must not create two of something or charge twice.
- **Active-active clusters** (see `masterless-active-active-replication`): a retry may land on a *different* replica than the original, so the dedup must not rely on in-memory state — persist it.
- Public/partner APIs where you can't control client retry behavior.

## Core pattern

A table keyed by the idempotency key:

```
idempotency_keys(key PK, method, path, status, response jsonb, created_at)
```

Middleware, mounted only on opt-in write routes:

1. No `Idempotency-Key` header → pass through untouched (most reads want nothing to do with this).
2. **Validate** the key: bounded length, restricted charset (RFC 7230 token). Reject malformed keys with 400.
3. **Look up** the key. Hit → replay stored `status` + `response`, skip the handler entirely.
4. Miss → run the handler, and on `finish` **cache only 2xx responses** (4xx/5xx should stay retryable). Capture the body by wrapping `res.json`.
5. Sweep entries older than a retention window (e.g. 24h) periodically.

## Design decisions

- **Persist, don't memoize.** In an active-active cluster the retry can hit any replica. An in-process cache dedups only same-replica retries — exactly the case a CDN failover breaks. A shared/replicated table is what makes cross-replica retry safe.
- **Cache success only.** A cached 500 would pin a transient failure forever; a cached 409/422 would block a client from fixing its input and retrying. Only 2xx is safe to replay.
- **`ON CONFLICT DO NOTHING` on save**, so two truly-concurrent requests with the same key don't both try to insert and error — the first wins, the second's identical work is harmless (or, for stricter guarantees, insert a "pending" row first and have the racer wait/replay).
- **Fail open on lookup errors.** If the idempotency store is briefly unavailable, prefer double-executing to taking the endpoint down — log and continue. (Choose fail-closed instead only when a duplicate is worse than an outage.)
- **Bounded retention.** Keys are for retry windows, not forever. A 24h TTL keeps the table small; sweep on a schedule.

## Pitfalls

- **Caching non-2xx** → a one-off error becomes permanent for that key, or a validation error can never be corrected.
- **In-memory only** → useless the moment a retry hits another replica; this is the whole reason the pattern exists in a cluster.
- **No key validation** → unbounded/garbage keys bloat the table or enable a cache-poisoning vector. Constrain length and charset.
- **Racing inserts** without `ON CONFLICT` handling → the second concurrent request 500s on the unique constraint.
- **Applying it to reads** → pointless work and a cache that can serve stale data; mount only on the writes that need it.
- **Key scoped too broadly/narrowly** — decide whether a key is global or per-user/per-endpoint; a global key lets one tenant's key collide with another's. Namespacing by caller is usually right.

## Checklist

- [ ] `idempotency_keys(key, method, path, status, response, created_at)` table
- [ ] Middleware mounted only on opt-in write routes
- [ ] Key validated (length + charset); malformed → 400
- [ ] Hit → replay stored status+body; miss → run then cache
- [ ] Cache **2xx only**; capture body via wrapped `res.json`
- [ ] `ON CONFLICT DO NOTHING` save; fail-open on lookup errors
- [ ] Periodic sweep past a retention window
