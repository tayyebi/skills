---
name: masterless-active-active-replication
description: Use when you want N ≥ 3 nodes where any node serves any read or write with no primary/leader, each keeping its own local database, converging via an append-only mutation log gossiped between peers and merged last-writer-wins. Covers the replication log, pull-based gossip with per-peer cursors, LWW apply with monotonic HLC guards, final-deletion tombstones, dead-lettering, and log GC. The opposite of primary-replica: no failover, no split-brain, every node is writable.
---

# Masterless Active-Active Replication

Run the same application on every node, each with its own **local** database. Any node accepts writes. Nodes converge by shipping each other an append-only log of mutations and merging them deterministically. There is no primary, no failover step, and no single point of write. A node can be partitioned for hours and still catch up cleanly when it rejoins.

## When to use

- Multi-region deployments where every region must serve reads *and* writes with low latency and survive the loss of any peer (or the network between them).
- Workloads that tolerate **eventual** consistency and can express conflicts as last-writer-wins (config, catalogs, CRUD-style domain data). Not for strict invariants like "never double-spend" — that needs consensus.
- When operational simplicity of "every node is identical and writable" beats the throughput of a single primary.

Pairs with `hybrid-logical-clocks` (the version stamp), `change-data-capture-triggers` (auto-populating the log), and `cluster-membership-gossip` (knowing who the peers are).

## Core pattern

**1. Stamp every replicable row** with `version_hlc` (an HLC) and `origin_replica` (who wrote it). Give delete-able rows a `deleted_at` for soft-delete tombstones instead of hard `DELETE`.

**2. Append every write to a mutation log** *in the same transaction* as the row change:

```
replication_log(seq, hlc, origin_replica, entity_type, entity_id, op, payload jsonb, created_at)
UNIQUE (entity_type, entity_id, hlc)   -- makes appends idempotent
```

`op ∈ {insert, upsert, tombstone}`. Transactional append is critical: if the write rolls back, the log entry rolls back too, so peers never see a mutation that didn't commit locally.

**3. Gossip-pull to converge.** A background loop (e.g. every 5 min, plus an on-demand "sync now" trigger) picks a peer and pulls log entries **strictly greater than a stored per-peer cursor**, in HLC order:

```
GET /cluster/log?since=${cursor}&limit=500   →  { entries, high_water }
```

Keep a cursor per peer (`hlc_cursors(peer_instance_id, last_applied_hlc)`) so each pull resumes where it left off. If a batch fills the limit, continue pulling in a bounded loop until drained.

**4. Apply LWW with a monotonic guard.** Upsert with an HLC guard so a stale write can never overwrite a newer one:

```sql
INSERT INTO services (...) VALUES (...)
ON CONFLICT (id) DO UPDATE SET ...
  WHERE services.version_hlc < excluded.version_hlc     -- older HLC = silent no-op
    AND services.deleted_at IS NULL                     -- never resurrect a deleted row
```

**Deletion must be final and unconditional.** Apply tombstones by id *without* the `version_hlc <` guard — only the resurrection guard above keeps things consistent. (A locally-issued tombstone's clamped HLC can be numerically *behind* the row it deletes; a guarded delete would silently drop it and the row would live forever on the peer.)

## Design decisions

- **Prevent echo.** When applying remote entries, set a transaction-local flag (`SET LOCAL app.skip_replication_log = true`) so your CDC triggers don't re-log a remote mutation as if it originated locally — otherwise entries ping-pong forever.
- **One transaction per batch, SAVEPOINT per entry.** Applying 500 entries in 500 separate transactions exhausts the connection pool and starves the request path. Use one outer transaction with a `SAVEPOINT` per entry: a single poisoned entry rolls back only itself; the batch continues. If the *outer* transaction fails (pool timeout, DB restart), do **not** advance the cursor — retry the whole batch next cycle.
- **Cursor advances past the whole batch**, even over skipped entries — a skipped entry is dead-lettered, not lost (see repair below). This keeps a single bad row from stalling all convergence behind it.
- **Dead-letter, don't crash.** Collect entries that fail to apply into a skipped list with their error. Log at WARN, not ERROR — a skipped `upsert` isn't data loss: a periodic `repair()` can detect rows missing despite an upsert log entry and restore them.
- **Coerce types at one choke point.** JSON transport loses types (dates become strings). Derive the date/timestamp columns from the table schema and coerce centrally, so a newly-added timestamp column can't silently dead-letter every entry for that table.
- **GC the log.** Sweep entries older than a retention window (e.g. 30 days) **and** below the minimum cursor every live peer has acknowledged, so you never delete an entry a lagging peer still needs.
- **Catch-up barrier on boot** (optional): pull from all peers once before serving, so a freshly booted node isn't wildly stale.
- **The log alone cannot heal every replica.** It is swept, it never covered pre-trigger rows, and applied remote entries aren't re-logged — so a node joining after the log horizon converges to a *silently partial* dataset. Pair this with a row-state snapshot channel: see `replication-snapshot-bootstrap`.

## Pitfalls

- **Guarded tombstones** silently lose deletions when the local row's HLC is higher. Deletes are unconditional; resurrection is what the upsert guard prevents.
- **Non-idempotent apply** — gossip is at-least-once. The `UNIQUE(entity_type, entity_id, hlc)` constraint + `ON CONFLICT DO NOTHING` on the log, plus LWW upserts, make re-delivery a no-op.
- **Forgetting `SET LOCAL skip-log`** on the apply path → infinite replication echo.
- **Per-entry transactions** → pool exhaustion → "timeout when connecting" on unrelated endpoints. Batch it.
- **Advancing the cursor on a failed outer transaction** → the batch is skipped forever. Only advance on success.
- **Not deriving date columns from schema** → adding a column breaks replication for that whole entity, and it passes review because the hand-maintained coercion list looked fine.
- Treating this as strongly consistent — it is not. Concurrent writes to the same field resolve LWW; the "losing" edit is discarded.

## Checklist

- [ ] Every replicable row: `version_hlc`, `origin_replica`, soft-delete `deleted_at`
- [ ] Transactional append to `replication_log` with `UNIQUE(entity_type, entity_id, hlc)`
- [ ] Pull-gossip with a per-peer `last_applied_hlc` cursor, applied in HLC order
- [ ] LWW upsert guarded by `version_hlc <`; **unconditional** tombstones; resurrection guard
- [ ] `SET LOCAL skip_replication_log` on the apply path
- [ ] One transaction per batch, SAVEPOINT per entry; cursor advances only on outer success
- [ ] Dead-letter skipped entries (WARN) + a repair pass for missing rows
- [ ] Log GC by retention window ∩ min acknowledged cursor
