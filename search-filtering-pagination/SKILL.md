---
name: search-filtering-pagination
description: Use when a list of records needs to be searched, filtered, and paged — Postgres trigram/full-text search, composite indexes for common filters, and cursor pagination that stays stable under writes. Covers keyset vs offset and index design.
---

# Search, Filtering & Pagination

Make large record lists fast and correct to browse, search, and page — without an external search engine for the common case.

## When to use

Any app with a growing list users scroll, filter, and search (candidates, orders, tickets). Postgres handles this well up to substantial scale before you need Elastic/OpenSearch.

## Search (Postgres, no extra infra)

- **Fuzzy / substring** search: `pg_trgm` extension + a **GIN trigram index** on the searched column. Great for "contains" and typo-tolerant name/email search.
  ```sql
  CREATE INDEX ON candidates USING gin (name gin_trgm_ops);
  ```
- **Word/relevance** search: `tsvector` + `tsquery` full-text with a GIN index and ranking (`ts_rank`). Use for longer text (descriptions, notes).
- Normalize input first (trim, lowercase, locale digit folding) so search matches user intent.

## Filtering

- Build **composite/covering indexes** for the filters users actually combine (e.g. `(tenant_id, status, created_at)`), not one index per column. Order columns by selectivity and match the query's `WHERE`/`ORDER BY`.
- Always lead tenant-scoped indexes with `tenant_id`.
- A **covering index** (`INCLUDE (...)`) lets hot list queries be index-only scans.

## Pagination

- **Prefer keyset (cursor) pagination** over `OFFSET` for large/among-changing data:
  ```sql
  WHERE (created_at, id) < ($cursor_ts, $cursor_id)
  ORDER BY created_at DESC, id DESC LIMIT $n
  ```
  - O(1) regardless of page depth (offset gets slower the deeper you go).
  - **Stable under inserts/deletes** — offset pagination skips or repeats rows when the list shifts under the user.
  - Tie-break on a unique column (`id`) so the order is total and the cursor is unambiguous.
- Offset pagination is fine for small, admin-only lists where page numbers matter.

## Pitfalls

- `OFFSET` deep pagination — slow and skips/duplicates rows during writes.
- `LIKE '%term%'` with no trigram index → full table scan.
- Filter combinations with no supporting index → seq scans under load.
- Unstable sort (non-unique sort key) → cursors land mid-tie and drop rows.
- Not folding locale digits/case before search → "misses" that look like bugs.

## Checklist

- [ ] Trigram GIN index for substring/fuzzy; full-text for word search
- [ ] Composite indexes matching real filter+sort combos, tenant-first
- [ ] Keyset pagination with a unique tie-breaker for large lists
- [ ] Input normalization before matching
