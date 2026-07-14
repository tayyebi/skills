---
name: performance-optimization
description: Use when a code path is slow or about to be built in a way that will become slow — the build-time discipline of profiling before optimizing, spotting algorithmic complexity blowups (especially N+1 query patterns), layering caches with an invalidation story, and deferring non-critical work. Covers the design-time half of performance; see `testing-performance` and `testing-benchmark` for black-box, after-the-fact verification.
---

# Performance Optimization

Performance work has two halves: designing a code path so it doesn't have a cheap-to-avoid bottleneck, and verifying afterward that it holds up under load. This skill is the first half — decisions made while writing or reviewing code, before anything ships. `testing-performance` (latency/throughput under load, from outside) and `testing-benchmark` (raw hardware capability of an instance) cover the second half.

## When to use

- Before shipping any loop that calls out to a database, API, or filesystem once per item.
- When a dashboard, report, or list endpoint is "slow" and the instinct is to guess at a fix.
- When designing a read path that will be hit often (dashboards, list views, hot API routes) — get the caching/batching shape right at design time, not after a fire.

## Core pattern: measure, then optimize

Never optimize from intuition. Profile the actual code path (a profiler, an APM trace, a query log, or a simple timer around a suspect block) and find where the time really goes before touching anything. Three questions gate any optimization:

1. **Where is the time going?** CPU-bound computation, network/IO wait, or database round-trips are fixed differently — don't cache your way out of an O(n²) algorithm, and don't rewrite an algorithm to fix a missing index.
2. **Is this path hot?** A slow endpoint hit twice a day is not worth the same effort as one hit a thousand times a minute.
3. **Would the fix actually move the number?** Shaving 5ms off a step that's 2% of total latency is wasted effort; find the dominant cost first.

## Algorithmic complexity and the N+1 pattern

The most common real-world complexity bug is not a bad sort algorithm — it's a loop that issues one round-trip per iteration instead of one round-trip total:

```
# N+1: one query per row
for order in orders:
    order.customer = fetch_customer(order.customer_id)

# batched: one query total
customer_ids = orders.map(order => order.customer_id)
customers    = fetch_customers_by_ids(customer_ids)     # single IN(...) query
customer_by_id = index_by(customers, c => c.id)
for order in orders:
    order.customer = customer_by_id[order.customer_id]
```

The same shape shows up beyond databases: an API call per item in a loop, a file stat per item, a cache lookup per item. Batch the round-trip, not just the database case. Also watch for algorithms that are accidentally quadratic on data that grows — a linear scan (`list.includes(x)`) inside a loop over the same list turns O(n) into O(n²); a hash/set lookup keeps it O(n).

## Caching layers

Three distinct layers, each with its own invalidation story — don't conflate them:

- **Computation cache** — memoize an expensive calculation (aggregate stats, a report) behind a cache-aside pattern:
  ```
  value = cache.get(key)
  if value is absent:
      value = expensive_computation()
      cache.set(key, value, ttl=300)
  return value
  ```
  Group related keys under a tag/namespace so a single write can invalidate everything derived from it, instead of guessing at individual keys.
- **Query/reference-data cache** — near-static lookups (settings, categories, country lists) are safe to cache for a long time, but only if every write path explicitly invalidates the cached key. A long TTL with no invalidation path is a correctness bug waiting to surface as "I changed it and nothing happened."
- **HTTP/CDN cache** — `Cache-Control`/`ETag` on responses that are safe to share across requests or replay conditionally; push whole responses to a CDN when they don't vary per user. This is the cheapest cache because it skips your application entirely.

## Lazy loading and deferral

- Fetch and render only what's on screen or about to be needed — paginate or stream large result sets instead of materializing them fully in memory (see `search-filtering-pagination` for the query-side pattern). Lazy-load below-the-fold assets and relations that most requests never touch.
- Move genuinely non-critical work off the request path into a background job (sending an email, generating a report, calling a slow third-party webhook) — see `background-jobs-scheduling` for what belongs in a queue versus what must stay synchronous. Anything the caller needs to see the result of, or that must fail before the response is sent, stays synchronous.

## Pitfalls

- Optimizing before profiling — guesses are usually wrong about where the time goes.
- Eager-loading everything "to be safe" — turns a fast query into a wide join that moves the cost from N round-trips to one enormous one; select only what the response needs.
- Caching without an invalidation path — a permanently-cached value that silently goes stale is worse than no cache.
- Queuing everything "just in case" — queues add latency, a second failure mode, and an operational surface; reserve them for work that's genuinely deferrable.
- Loading an unbounded collection fully into memory instead of paging or streaming it.
- Treating a cache as a substitute for fixing a genuinely slow query or algorithm — it hides the cost on the first request and every cache miss.

## Checklist

- [ ] Bottleneck identified by profiling/measurement, not guessed
- [ ] No loop issuing one DB/API/IO call per item — batched instead
- [ ] Hot algorithms reviewed for accidental quadratic behavior on growing input
- [ ] Each cache layer (computation/query/HTTP) has an explicit invalidation path
- [ ] Large/unbounded result sets paginated or streamed, not fully materialized
- [ ] Only genuinely non-critical work deferred to a background job
