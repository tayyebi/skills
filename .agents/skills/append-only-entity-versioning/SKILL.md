---
name: append-only-entity-versioning
description: Use when a table needs full change history and cheap undo without a separate audit-log bolt-on — every insert *and* update writes a brand-new full-row snapshot (never UPDATE in place), snapshots share a stable group id, and a shared sequence table hands out the row identity. "Current state" is the latest snapshot per group. Defers to change-data-capture-triggers for trigger-based CDC on tables that still mutate in place, and structured-logging-audit for a side audit trail that doesn't replace the primary table.
---

# Append-Only Entity Versioning

Never `UPDATE` a row in place. Every change — insert or edit — writes a brand-new, complete row. A stable "group id" ties every version of the same logical record together; the current value is just "the newest row in the group." History isn't a bolt-on log of *diffs* — it's the table itself, because nothing is ever overwritten or destroyed.

## When to use

A table where "what did this record look like last week" or "who changed this and when" needs to be answerable **without** a separate audit subsystem, and where undo should be as cheap as "point back at an older row." Good fits: user profiles, resumes/documents, pricing/config records, anything with a compliance or dispute-resolution need for prior versions. Skip it for high-frequency, low-value-per-write data (metrics, session pings) — the storage growth isn't worth it there; use plain mutation instead.

This is a different tool from CDC triggers or an audit log: those *observe* a table that still mutates in place and record what happened *elsewhere*. Here, the versioning **is** the table's storage strategy — there is no in-place mutation to observe.

## Core pattern

```
-- the shared backbone: hands out identity + groups versions + tombstones deletes
revisions(
  revision_id   serial/identity PRIMARY KEY,   -- global, monotonic, cross-entity sequence
  group_id      <stable id>  NOT NULL,          -- same value across every version of one logical record
  event_at      timestamptz NOT NULL DEFAULT now(),
  is_deleted    boolean NOT NULL DEFAULT false  -- tombstone marker
)

-- one snapshot table per entity type; its PK *is* a revision_id, not its own sequence
users(
  id            int PRIMARY KEY REFERENCES revisions(revision_id),
  -- ...every current field, as of this snapshot...
)
```

- **Write = insert a `revisions` row, then insert a full snapshot row** whose primary key is that new `revision_id`. There is no `UPDATE users SET ...` anywhere in the write path — "updating" a record is indistinguishable from creating it, at the storage level.
- `group_id` is what makes these snapshots one logical record instead of N unrelated rows — assign it once (at first creation) and carry it forward unchanged on every subsequent snapshot.
- Deletion is a `revisions` row with `is_deleted = true` and no matching entity snapshot (or one final snapshot, if you want the last state preserved) — a tombstone, not a `DELETE FROM`.

## Reading current state

Current state is a query, not a stored fact: newest non-deleted revision per group.

```sql
CREATE VIEW users_current AS
SELECT u.*, g.group_id, g.revision_count
FROM (
  SELECT group_id, MAX(revision_id) AS revision_id, COUNT(*) AS revision_count
  FROM revisions
  WHERE is_deleted = false
  GROUP BY group_id
) g
JOIN users u ON u.id = g.revision_id;
```

- The `revision_count` falls out of the same aggregation for free — a per-record edit counter with no extra column to maintain.
- If this view is queried on a hot path and the table has grown large, materialize it (materialized view refreshed on write, or a maintained `current_revision_id` pointer column on a lightweight group table) rather than re-aggregating on every read.
- History for one record: `SELECT * FROM users WHERE id IN (SELECT revision_id FROM revisions WHERE group_id = ?) ORDER BY id`. No separate history table to join.

## A single write gateway for every entity

Since every entity's write path is mechanically identical (insert a `revisions` row → insert a snapshot keyed by the new id), implement it once and have every entity-specific writer extend it:

```
base.Insert(group_id, snapshot):
    revision_id = revisions.insert(group_id, is_deleted=false)
    return revision_id            // caller stamps snapshot.id = revision_id, inserts it

base.Update(group_id, snapshot):
    return base.Insert(group_id, snapshot)   // update IS insert — same code path

base.Delete(group_id):
    revisions.insert(group_id, is_deleted=true)   // tombstone, no snapshot row
```

Entity-specific code overrides only "how do I build and save my own snapshot row" — it never reimplements the revision/group bookkeeping. This collapses insert and update into one code path, which also means there's no drift between "the fields insert sets" and "the fields update sets."

## Concurrency and uniqueness

- A uniqueness rule ("username taken") can't be a database constraint on the snapshot table anymore — old snapshots of *other* usernames, and old snapshots of freed-up group ids, pile up right alongside current ones. Enforce uniqueness by querying the **current-state view**, inside a transaction with appropriate locking, not the raw snapshot table.
- Two writers racing to create the next revision for the same `group_id` is a last-writer-wins race by default (whichever insert commits last "wins" as current). If that's not acceptable, serialize writes per `group_id` (a per-group advisory lock, or a `WHERE current_revision_id = expected` optimistic-concurrency check before appending) — same discipline as any other optimistic-concurrency write path.

## Pitfalls

- Forgetting that storage is unbounded by design — every edit is a full new row, forever. Have an explicit retention/archival story (move revisions older than N versions or T time to cold storage) before this ships on a high-churn table; don't discover it during a disk-space incident.
- Putting a uniqueness or foreign-key constraint directly on the snapshot table expecting it to mean "unique among current records" — it will reject a value that's only reused because an *old, superseded* snapshot holds it, or fail to catch a genuine duplicate among current rows.
- Querying the raw snapshot table instead of the current-state view/materialization in application code — every such query is a latent bug that surfaces the moment a record gets its second revision.
- Skipping the transaction around "insert revision, insert snapshot" — a crash between the two leaves a `revisions` row with no matching snapshot, which the current-state view's join will silently drop (invisible data loss) unless you've planned for that failure mode.
- Using this pattern for a field that changes on every request (view counters, last-seen timestamps) — that's not "versioning," that's high-frequency mutation wearing a versioning table's clothes, and it will drown the real history in noise.

## Out of scope here

- Trigger-based change capture on a table that keeps mutating in place — see `change-data-capture-triggers`.
- A side audit-log table recording who-did-what across the whole app, independent of any one entity's storage strategy — see `structured-logging-audit`.
- Multi-node append-only replication logs merged across peers — see `masterless-active-active-replication`.
- General primary-key/indexing/migration conventions — see `database-schema-design`.

## Checklist

- [ ] Shared `revisions`-style table hands out identity, `group_id`, timestamp, and delete tombstone
- [ ] Entity snapshot tables key off `revision_id`; no `UPDATE` statement anywhere in the write path
- [ ] One base write-gateway shared by every entity type; entity code only builds its own snapshot row
- [ ] Current-state view (or materialization) computed as newest non-deleted revision per group
- [ ] Uniqueness rules enforced against current-state view inside a transaction, not the raw snapshot table
- [ ] Per-group write concurrency handled (lock or optimistic-concurrency check), not left to last-writer-wins by accident
- [ ] Retention/archival plan exists before this lands on a high-churn table
- [ ] Revision-insert + snapshot-insert wrapped in one transaction
