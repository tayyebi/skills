---
name: database-schema-design
description: Use when designing relational tables, migrations, indexes, or query patterns — primary key strategy, naming, composite indexes, N+1 prevention, chunked processing, and transactions. Defers to multi-tenancy-rls for tenant isolation and change-data-capture-triggers for trigger-based CDC/audit logging.
---

# Database Schema Design

Get the shape of the data right before optimizing queries against it — most performance problems are schema problems wearing a query's clothes.

## When to use

Designing a new table, adding a migration to an existing one, or diagnosing why a query that "should be fast" isn't. Applies to any relational store (Postgres, MySQL, SQL Server, etc.) regardless of the language or ORM sitting on top of it.

## Primary keys

- Default to a globally-unique, sortable ID (UUIDv7 or an equivalent time-ordered ULID) over a plain auto-increment integer. It's generatable client-side or in any service without a round trip to the database, doesn't leak the row count to anyone who can see an ID, and merges painlessly across shards or during a data migration.
- A sequential integer is still fine for purely internal, single-writer tables where none of those properties matter and you want the smallest possible index — don't cargo-cult UUIDs onto a lookup table with ten rows.
- Never expose an auto-increment integer as a public-facing identifier (in a URL, in an API response) — it invites enumeration and reveals business volume (competitor counts your daily signups by diffing IDs).

## Naming conventions

Pick one convention and apply it everywhere — the value is in consistency, not in which convention wins:

- Table names: plural (`orders`, `order_items`). 
- Join/pivot tables: both table names, singular, alphabetical (`order_product`, `role_user`) — a consistent order means nobody has to guess.
- Foreign key columns: `{singular_referenced_table}_id` (`user_id`, `order_id`).
- Index names: `{table}_{columns}_idx` so an `EXPLAIN` plan or a slow-query log is self-describing without a schema lookup.

## Columns

- Avoid nullable columns as a default; a `NULL` forces every reader to handle a third state ("present," "absent," "did we forget to set it") for what's often really two states. If a value is genuinely optional and semantically separate (an address that not every user has), consider a separate table over a wide table full of nullable columns.
- Store money as an integer in the smallest unit (cents) or a fixed-point decimal type — never as a floating-point type, which cannot represent most decimal fractions exactly and will eventually produce a one-cent discrepancy.
- Prefer unbounded text over an arbitrary short varchar limit unless a real constraint (a legal field length, a downstream system's limit) demands one — a `varchar(255)` cap that exists only because it was the ORM's default will eventually truncate someone's legitimate input.
- Every table gets `created_at`/`updated_at`. Add a nullable `deleted_at` only if the table actually needs soft-delete semantics — don't add it reflexively to tables where a hard delete is fine and soft-delete would just add a `WHERE deleted_at IS NULL` everyone has to remember.

## Migrations

- Every migration is reversible: write the down/rollback step at the same time as the up step, not as an afterthought once something breaks.
- Never edit a migration that has already run in any shared environment (staging, production). Write a new migration — editing history that others have already applied causes their local state and the migration file to silently diverge. Operational default: once a migration file is committed/pushed, **assume it has already run** (most runners apply on every deploy/startup), so never renumber or rewrite it — always add the next-numbered file. This holds even for "fixing a typo" or "it only just merged": the safe move is a new forward migration, not an in-place edit.
- Preview the SQL a migration will generate before running it against production; don't trust the migration tool's abstraction blindly for anything touching a large table.
- For zero-downtime deploys, a migration must be safe against the *currently running* code, not just the new code — see `ci-cd-pipeline-discipline` for the surrounding deploy sequencing. Concretely:
  - Adding a column: give it a default or make it nullable so in-flight writes from old code don't fail.
  - Renaming a column: expand/contract — add the new column, dual-write both, backfill, cut reads over, then drop the old column in a later deploy. Never rename in place while old code is still running.
  - Dropping a column: deploy the code that stops reading/writing it first; drop the column in a subsequent migration once you're sure nothing references it.

## Indexing

- Index every foreign key column — most databases don't do this automatically, and an unindexed foreign key turns every join or cascade delete into a full scan.
- Index columns that appear in `WHERE`, `JOIN`, and `ORDER BY` clauses of hot queries — but every index also slows down writes and costs storage, so index for the queries you actually run, not speculatively.
- Composite index column order matters: put the column used for equality filtering before the column used for range filtering/sorting, so the index can seek then scan. `(status, created_at)` serves "pending orders sorted by date"; `(created_at, status)` serves that query far worse.
- Run `EXPLAIN`/`EXPLAIN ANALYZE` on any query touching a large table before shipping it — don't guess whether an index is being used.

## Avoiding N+1

Fetching a list, then issuing one additional query per row to fetch a related record, turns an O(1) round trip into O(n). This is the single most common relational-database performance bug, independent of ORM.

```
// N+1: one query for orders, one more per order for its user
orders = Order.all()
for order in orders: print(order.user.name)   // triggers a query per iteration

// fixed: one query for orders, one query (or one join) for all their users
orders = Order.all().eagerLoad("user")
```

If your ORM supports a "strict mode" or "no lazy loading" flag for non-production environments, turn it on — it turns a silent N+1 into a loud test failure instead of a production latency surprise.

## Processing large datasets

- Never load an entire table into memory (`SELECT *` with no bound) when the row count is unknown or unbounded — page/chunk instead, fetching a few hundred to a few thousand rows at a time.
- When chunking *while updating* the same rows you're paging through, page by a stable, previously-seen ID boundary (`WHERE id > last_seen_id ORDER BY id LIMIT n`), not by numeric offset — an offset-based chunk drifts and skips/repeats rows as the underlying result set shifts.
- Select only the columns a query actually needs; `SELECT *` on a wide table pulls bytes across the wire that get discarded, and defeats index-only scans.

## Transactions

Wrap multi-statement writes that must succeed or fail together in a transaction — a partial write (order created, inventory not decremented) is a data-integrity bug waiting to surface under load or a crash mid-operation.

```
transaction:
    order = createOrder(input)
    createOrderItems(order, input.items)
    decrementInventory(input.items)
// commits all three, or rolls back all three on any error
```

Keep transactions short — a long-running transaction holds locks and blocks other writers; do the expensive/slow work (calling an external API) outside the transaction boundary.

## Query approach by workload

| Approach | Best for |
|---|---|
| ORM / high-level query builder | Single-record CRUD, straightforward eager loading |
| Lower-level query builder (bulk update/delete) | Bulk operations that don't need to hydrate full objects into memory |
| Raw SQL | Window functions, CTEs, reporting queries, anything the ORM's abstraction can't express cleanly |

Bulk-updating through an ORM that insists on hydrating every row into an object first wastes memory for no benefit — drop to a bulk statement for that case.

## Out of scope here

- Isolating tenant data in a shared database — see `multi-tenancy-rls`.
- Trigger-based audit/change-data-capture logging — see `change-data-capture-triggers`.

## Pitfalls

- UUIDv4 (fully random) as a clustered/primary index on a high-write table — it fragments the index on every insert; prefer a time-ordered variant (UUIDv7/ULID) when insert order matters for locality.
- Two separate single-column indexes where one composite index would serve the actual query pattern.
- Editing a migration already applied in a shared environment instead of writing a new one.
- Renaming a column in place instead of expand/contract, breaking whichever app version is still running old code.
- Floating-point currency columns.

## Checklist

- [ ] Primary key strategy chosen deliberately (sortable UUID vs. integer), not by ORM default
- [ ] Foreign keys, join columns, and index names follow one consistent convention
- [ ] Every foreign key is indexed; composite indexes ordered equality-then-range
- [ ] Migrations are reversible and safe for the currently-running code (expand/contract for renames)
- [ ] N+1 queries eliminated via eager loading; strict/no-lazy-load mode on in non-prod
- [ ] Large tables processed in ID-keyed chunks, never unbounded `SELECT *`
- [ ] Multi-statement writes wrapped in short-lived transactions
- [ ] Money stored as integer cents or fixed-point decimal, never float
