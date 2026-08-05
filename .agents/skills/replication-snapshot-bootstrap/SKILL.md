---
name: replication-snapshot-bootstrap
description: Use when a replica cannot converge from the incremental replication log alone — it joined after the log's retention horizon, was partitioned longer than retention, or the log never covered rows that predate the triggers. Serve current ROW STATE as synthetic log entries through the same apply pipeline, paged by keyset, ordered by an FK-ordered entity registry, with a completion marker only a fully drained pull may set. The anti-entropy half of masterless-active-active-replication.
---

# Replication Snapshot Bootstrap

An incremental mutation log is not a description of your dataset — it is a description of *recent changes* to your dataset. Any node that wasn't listening when a change happened, and arrives after that change has been swept, can never learn about it from the log. Snapshot bootstrap is the second, coarse channel that closes that gap: a peer serves its **current row state**, and the joining replica applies it through the exact same pipeline it uses for log entries.

## When to use

You already have `masterless-active-active-replication` (or any log-shipping replication) and one of these is true:

- **New node.** A replica is added to a running cluster months after it started. The log only goes back 30 days.
- **Long partition.** A node was down or unreachable for longer than the log retention window, so the entries it needs are gone.
- **Pre-trigger rows.** CDC triggers were added in a migration; rows written before that migration never produced a log entry and never will.
- **Applied-remote entries aren't re-logged.** The apply path sets a skip-log flag (it must, or entries echo forever), so node C's log does *not* contain what C learned from B. The union of every peer's log still does not cover the dataset.

If any of these apply — and in practice all four do — gossip alone converges a fresh replica to a **silently partial** dataset. It looks healthy. It serves wrong answers.

Complements: `masterless-active-active-replication` (the incremental path this backstops), `hybrid-logical-clocks` (the version stamp), `change-data-capture-triggers` (why the log has a start date at all), `cluster-membership-gossip` (who the peers are).

## Core pattern: row state shaped as log entries

The single most important decision: **do not write a second apply path.** Serve current rows as synthetic log entries so the receiver reuses everything the incremental path already got right — LWW `version_hlc` guards, the skip-log flag, savepoint-per-entry isolation, date coercion, dead-lettering.

Server side — one keyset-paged endpoint, one entity type per request:

```
GET /cluster/snapshot?entity_type=service&after=<last_key>&limit=500
  → { entity_type, entries: LogEntry[], next_after: string | null }

# each row becomes a synthetic entry:
{ seq: 0, hlc: <derived, see below>, origin_replica: row.origin_replica ?? 'snapshot',
  entity_type, entity_id: row[key], op: 'upsert', payload: row, created_at: now }
```

- **Keyset paging, not OFFSET** — `WHERE key::text > $after ORDER BY key::text ASC LIMIT $n`. Stable under concurrent writes, and the cursor is just the last key you saw.
- **`op` is always `upsert`.** There are no deletes to transfer as deletes; soft-deleted rows are *included* with their `deleted_at` in the payload, so deletion state transfers as ordinary row state.
- **Cap the page loop** (e.g. 10 000 pages per entity) as a safety valve against a peer that never returns `next_after: null`.

Client side — drain every entity, in registry order, through `applyBatch`:

```
for spec in SNAPSHOT_SPECS:            # FK-ordered: parents before children
    after = ''
    loop:
        page = GET /cluster/snapshot?entity_type=spec.entity&after=after&limit=500
        if transport failure: complete = false; break     # see marker discipline
        if HTTP 400 (peer too old to know this entity): break   # NOT a failure
        applyBatch(page.entries)                          # the normal pipeline
        if not page.next_after: break
        after = page.next_after
```

Applying a snapshot is idempotent by construction: re-pulling converges to the same state, and gossip replaying the same mutations afterwards is a no-op under the LWW guard.

## The entity registry is the single source of truth

Snapshot order, apply dispatch, and the table↔entity mapping are the same knowledge. Keep exactly one FK-ordered array of specs and derive all three from it — the alternative is three hand-maintained lists that drift, and the drift is invisible until a fresh replica bootstraps.

```
EntitySpec {
  entity, table, kind: 'lww' | 'append-only' | 'custom',
  junctionKeys?: [...],          # append-only junctions key on their FK pair, not a row id
  snapshot: null | { hlc: HlcSource, key?: 'id', extraWhere?: '<constant SQL>' }
}
```

- **Array order IS snapshot order.** Parents before FK children, so pages apply as they stream in instead of dead-lettering en masse and waiting for a retry to sort out ordering.
- **`snapshot: null` excludes an entity.** Cluster membership is the canonical case: peers are owned by the join/leave handshake, and snapshotting them would resurrect decommissioned members from a peer with a stale view.
- **`extraWhere` bounds unbounded tables** — an audit log gets `AND timestamp > NOW() - INTERVAL '90 days'`. Constant SQL from the registry only; never anything derived from a request parameter.
- **Cross-check the registry in a unit test** against the trigger tables in your CDC migrations. A table with a replication trigger and no registry entry logs mutations nobody applies; a registry entry with no trigger snapshots rows that never gossip.

## Deriving an HLC for a snapshotted row

Snapshot rows must carry an HLC because the receiver's LWW guard compares against one. Three sources, declared per entity:

| Source | Use for | Behavior |
|---|---|---|
| `version_hlc` | rows that carry their own stamp | exact — real LWW against concurrent edits |
| `{ fromColumn: 'created_at' }` | append-only rows with a timestamp but no stamp | ms-precision synthetic HLC |
| `low` | pure junction rows (no meaningful version) | a constant floor HLC — **any** real mutation outranks it |

The `low` constant is the important one. A junction row has no version of its own, so stamping it low guarantees a snapshot can never overwrite a genuine later mutation that a concurrent gossip round is delivering.

## Completion-marker discipline

Boot-time bootstrap must be a one-shot, not a per-boot full transfer — but the marker that suppresses it is exactly where partial pulls become permanent data loss.

```
on boot:
    if hasCompletedInitialSnapshot(): return
    peers = live peers
    if peers is empty: log and return            # retry next boot; do NOT mark
    snapshot-pull every peer
    if any peer delivered a COMPLETE pull: markSnapshotCompleted()
    else: log a warning and leave the marker unset
```

- **Only a fully drained pull counts.** A pull that hit a transport error partway must not satisfy the check, or the replica serves a truncated dataset forever and never tries again.
- **A `400 unknown entity` from an older peer is complete-for-that-entity**, not a failure — that peer genuinely has nothing to serve for an entity it hasn't been upgraded to know about.
- **Store the marker in a local, never-gossiped table.** It describes *this* replica's history. Replicate it and every peer inherits "already bootstrapped" from the first node that finished.
- **Pull snapshots sequentially, one peer at a time.** A snapshot is a bulk apply; several concurrent ones fight over the connection pool and starve the request path — the same failure mode as per-entry transactions.
- **Follow the snapshot with a gossip round** to pick up anything written while the snapshot was streaming.
- Expose it as an operator action too (`POST /admin/resync` → `202` + progress in the overview), because "this node looks wrong" is a real incident and the snapshot path is the fix.

## How this changes log GC

Once snapshot bootstrap exists, the log sweeper can stop being infinitely patient. The healthy gate is still "older than retention **and** below the minimum cursor every active peer has acknowledged" — but add a **hard cap** (e.g. 90 days) past which entries are deleted regardless of acknowledgement. A peer that far behind is no longer a log problem; it is healed by a snapshot resync, and holding the log hostage for it only risks running the disk out.

Report *why* deletion was withheld, per peer — `unreachable`, `never-pulled`, `auth`. Without the reason, an operator misreads a rejected shared secret as a merely lagging peer and lets the log grow for weeks.

## Pitfalls

- **A second apply path for snapshots.** It will diverge from the log path — usually by missing the skip-log flag (echo storms), the resurrection guard (deleted rows come back), or date coercion (every entry dead-letters). Shape rows as log entries and reuse the one pipeline.
- **Marking bootstrap complete after a partial pull.** The single worst bug in this pattern: the replica is permanently, silently short of data and the boot check will never run again.
- **Snapshotting membership.** A peer with a stale view resurrects decommissioned nodes, which then get gossiped onward. Exclude it; membership belongs to the join/leave handshake.
- **Omitting soft-deleted rows.** Deletion state is state. Skip tombstoned rows and a bootstrapping replica cheerfully restores every record its peers deleted.
- **Snapshot pages in FK-arbitrary order** — children arrive before parents, the whole batch dead-letters, and convergence depends on a retry loop that may cap out first.
- **`OFFSET` paging** — concurrent inserts shift the window and rows are skipped, so the "complete" snapshot silently isn't.
- **Stamping junction rows with a real timestamp-derived HLC** instead of the low constant — a snapshot then wins LWW against a concurrent genuine mutation.
- **Interpolating a request-supplied entity/table/column name into the page query.** Table names, keyset columns, and `extraWhere` come from the registry (constants). The only user-controlled value is the `after` cursor, which is a bound parameter.
- **Believing the union of peers' logs covers the dataset.** It does not, and this is the assumption that makes teams skip this pattern entirely.

## Checklist

- [ ] Snapshot serves current row state shaped as synthetic `upsert` log entries, applied through the *existing* apply pipeline
- [ ] Keyset paging (`key > after ORDER BY key LIMIT n`) with a per-entity page cap
- [ ] Soft-deleted rows included, `deleted_at` carried in the payload
- [ ] One FK-ordered entity registry drives snapshot order, apply dispatch, and table↔entity mapping — cross-checked against CDC trigger tables in a test
- [ ] Per-entity HLC source declared (`version_hlc` / `fromColumn` / low constant); junctions use the low constant
- [ ] Membership (and anything else owned by a handshake) excluded via `snapshot: null`
- [ ] Unbounded tables bounded with constant `extraWhere`; no request-derived SQL identifiers
- [ ] Boot-time one-shot bootstrap gated on a **local, never-gossiped** completion marker
- [ ] Marker set **only** on a fully drained pull; a peer's `400 unknown entity` counts as complete, a transport error does not
- [ ] Snapshot pulls run sequentially per peer, followed by a gossip round
- [ ] Operator-triggered resync exposed asynchronously (`202` + progress), not as a blocking request
- [ ] Log GC has a hard cap past the acknowledgement gate, with per-peer withholding reasons reported
