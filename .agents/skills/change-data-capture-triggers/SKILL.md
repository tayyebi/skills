---
name: change-data-capture-triggers
description: Use when every write to a set of tables must also produce a durable log entry (for replication, an outbox, or an audit trail) and you cannot trust application code to remember to write it. Database triggers append the log entry inside the same transaction as the row change — automatically, for raw SQL, ORM writes, restored backups, and manual edits alike. Self-healing: there is no code path that mutates the row without logging it.
---

# Change Data Capture via Database Triggers

Make the database itself record every change, so the log can never drift from the data. A `BEFORE` trigger fills in version metadata; an `AFTER` trigger appends the row's new state to a log table — both in the same transaction as the write. Application code, migrations, ad-hoc `psql`, and a restored dump all produce log entries without knowing the log exists.

## When to use

- **Replication** — feeding a gossip/mutation log (see `masterless-active-active-replication`) where a missed entry means two nodes silently diverge.
- **Transactional outbox** — reliably emit events/webhooks only for committed writes, with no dual-write race.
- **Audit trails** that must be complete even for out-of-band edits.

Use triggers specifically when *completeness* matters more than keeping logic in the app. If only your own code ever writes (and you're sure), an application-layer append is simpler — but it's one forgotten call away from a silent gap.

## Core pattern

Two triggers per synced table, plus a DB-side version-stamp generator.

**1. Version generator** (so writes that bypass the app still get a valid stamp). A `next_hlc()` SQL function backed by a `FOR UPDATE`-locked singleton row, using the *same* encoding as the application clock so values are directly comparable (see `hybrid-logical-clocks`).

**2. `BEFORE INSERT/UPDATE` — ensure metadata:**

```sql
IF INSERT and version_hlc is null/boot-default  → version_hlc := next_hlc();
IF UPDATE and version_hlc unchanged by writer    → version_hlc := next_hlc();  -- raw UPDATEs still advance
IF origin_replica is null                        → origin_replica := current instance id;
```

This guarantees LWW always advances even when someone runs a bare `UPDATE ... SET name=...`.

**3. `AFTER INSERT/UPDATE/DELETE` — append the log entry:**

```sql
IF current_setting('app.skip_replication_log', true) = 'true' THEN RETURN; END IF;  -- see below
op := INSERT→'insert' | DELETE→'tombstone' | (deleted_at just set)→'tombstone' | else 'upsert';
payload := to_jsonb(NEW);   -- full row state
INSERT INTO change_log (hlc, origin, entity_type, entity_id, op, payload)
VALUES (...) ON CONFLICT (entity_type, entity_id, hlc) DO NOTHING;   -- idempotent
```

Pass the entity name as a trigger argument (`EXECUTE FUNCTION log_change('service')`) so one generic function serves every table.

## Design decisions

- **The skip flag is essential.** When your own code applies a *remote* change, it must set `SET LOCAL app.skip_replication_log = true` for that transaction, or the trigger re-logs the remote write as locally-originated and it echoes around the cluster forever. `SET LOCAL` is transaction-scoped and survives `ROLLBACK TO SAVEPOINT`.
- **Idempotent append.** A `UNIQUE (entity_type, entity_id, hlc)` + `ON CONFLICT DO NOTHING` means it's harmless if application code *also* appends (belt-and-braces): both target the same triple, one survives.
- **Full-row `to_jsonb(NEW)` payloads** make the consumer simple (upsert the whole row) and self-describing. Note the type caveat: JSON turns timestamps into strings — the consumer must coerce back.
- **Pass instance id via a session GUC** (`current_setting('app.instance_id')`), set once per connection, so the DB stamps `origin_replica` correctly without the app threading it through every write.
- **Attach triggers in a migration loop** over the table list, so adding a synced table is a one-line change and the trigger bodies stay identical.

## Pitfalls

- **No skip flag on the apply path** → infinite echo between nodes. This is the #1 CDC-trigger bug.
- **Session vs transaction scope.** Use `SET LOCAL` (transaction) for the skip flag, not `SET` (session), or a pooled connection leaks the flag into the next unrelated request and silently stops logging real writes.
- **`DELETE` has no `NEW`** — read `OLD.id`, emit a `tombstone`, and mint a fresh HLC (there's no row left to carry one).
- **Soft-delete detection**: an `UPDATE` that sets `deleted_at` from null is a tombstone, not an upsert — branch on it.
- **Trigger recursion**: if the log table itself is synced, or the apply writes trigger more logging, you get loops. Keep the log table out of the synced set and rely on the skip flag.
- **Bulk operations** fire the trigger per row — a million-row backfill writes a million log entries. Gate big migrations with the skip flag if they shouldn't replicate.

## Checklist

- [ ] DB-side version generator sharing the app's HLC encoding
- [ ] `BEFORE` trigger fills `version_hlc` + `origin_replica`, bumps HLC on every UPDATE
- [ ] `AFTER` trigger appends full-row `to_jsonb` payload with correct `op`
- [ ] `UNIQUE(entity_type, entity_id, hlc)` + `ON CONFLICT DO NOTHING`
- [ ] `SET LOCAL app.skip_replication_log` honored by the trigger, set on the apply path
- [ ] Instance id supplied via a session GUC
- [ ] Triggers attached in a migration loop over the synced-table list
