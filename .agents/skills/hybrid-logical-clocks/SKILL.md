---
name: hybrid-logical-clocks
description: Use when you need a single, comparable version stamp that orders events across machines without a coordinator — for last-writer-wins conflict resolution, causal ordering of a replication/event log, or "which write is newer" across replicas whose wall clocks disagree. A Hybrid Logical Clock (HLC) combines wall-clock time with a logical counter so timestamps stay monotonic and causally correct even when NTP drifts.
---

# Hybrid Logical Clocks (HLC)

Give every event a version stamp that is (a) close to real wall-clock time so it's human-readable and roughly sortable by "when", and (b) strictly monotonic and causally consistent so two machines never disagree about ordering. A plain `Date.now()` fails on both counts: two events in the same millisecond tie, and a backwards NTP step makes a newer event look older.

## When to use

- **Last-writer-wins (LWW) convergence** — you need a deterministic "newest write wins" rule across replicas (see `masterless-active-active-replication`).
- **Ordering a replication or event log** so peers apply entries in a consistent causal order.
- Any place you'd reach for `updated_at` to decide precedence but can't trust that clocks agree or that timestamps are unique.

Do **not** use it for strong consistency (use a real consensus protocol) — HLC gives you a total order for LWW, not linearizability.

## Core pattern

An HLC value is three parts: `physical_ms`, `logical_counter`, `origin_id`.

- `tick()` — called for every local event. If wall-clock ms moved forward, take it and reset the counter to 0; otherwise keep the same ms and increment the counter. Guarantees each local stamp is strictly greater than the last.
- `update(remote)` — called when receiving a remote stamp. Set physical = `max(local_pt, remote_pt, now)`, and set the logical counter to keep the result strictly greater than both what you've seen and what the peer sent. This is what enforces causality: any future local write is newer than anything you've observed.

**Encode it fixed-width and lexicographically sortable** so string comparison _is_ HLC comparison — this lets the database order and filter on it with a plain `ORDER BY version_hlc` / `WHERE version_hlc > $cursor`, no custom comparator:

```
${13-digit ms}-${6-digit logical}-${origin_id}     e.g. 0001700000000-000042-us-dallas
```

Zero-pad both numbers to fixed width. Parse by finding the first two dashes only — `origin_id` may itself contain dashes (`us-east-1`).

## Design decisions

- **`origin_id` breaks ties** between replicas that stamped in the same ms with the same counter, so causally-concurrent writes are still detected as distinct (and LWW still resolves deterministically — bigger id wins).
- **Clamp forward drift.** When merging a remote clock, cap its physical time at `now + MAX_DRIFT` (e.g. 60s). A peer with a broken clock set to next year would otherwise poison your clock forever. Clamping bounds the damage.
- **One monotonic source per instance.** If both application code and database triggers stamp HLCs (see `change-data-capture-triggers`), they must share the *same* counter — e.g. a `hlc_state` singleton row read `FOR UPDATE`, or a Postgres `next_hlc()` function. Two independent counters for one instance will collide.
- **Persist the state**, or at least advance past the newest value you've ever written on boot, so a restart can't re-issue a stamp you already used.

## Pitfalls

- Comparing HLCs numerically after decoding but forgetting the logical/origin parts — compare the *whole* encoded string, or all three fields in order.
- Variable-width encoding (unpadded numbers) silently breaks lexical sort: `"9-..." > "10-..."` as strings. Always zero-pad.
- Splitting `origin_id` with a naive `split('-')` — it corrupts ids that contain dashes. Split on the first two dashes only.
- Letting a peer's unclamped physical time flow into your own `tick()` — one bad clock drags the whole fleet's timestamps into the future.
- Two writers on the same node (app + DB trigger) using separate counters — they'll mint duplicate stamps for the same instant.

## Checklist

- [ ] Fixed-width, zero-padded, lex-sortable encoding: `pt-l-id`
- [ ] `tick()` monotonic locally; `update()` merges remote and stays strictly greater
- [ ] Forward-drift clamp (`min(remote_pt, now + MAX_DRIFT)`)
- [ ] Single shared counter per instance (app + DB agree)
- [ ] State persisted / advanced-past on boot
- [ ] Decoder splits on the first two dashes only
