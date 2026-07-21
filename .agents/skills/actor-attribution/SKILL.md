---
name: actor-attribution
description: Use when a database record needs to remember *who* created or last changed it — created_by/updated_by (actor_id) columns on mutable tables, nullable-for-system-actor handling, and how this differs from a separate audit-log trail. Defers to structured-logging-audit for full who-did-what event history and authorization-rbac for binding the actor at the auth gateway.
---

# Actor Attribution

Stamp every mutable row with *who touched it*, not just *when* — `created_by`/`updated_by` columns that answer "who created this record" and "who last changed it" directly from the row itself, no join to a log table required.

## When to use

Any table where "who did this" is a real question someone will ask — support debugging a customer's data, an admin reviewing a suspicious record, a compliance requirement, or just `updated_by` showing up in the UI ("last edited by Jane"). Almost every mutable business-entity table in a multi-user system qualifies; pure lookup/reference tables usually don't.

This is **not** the same problem as full audit logging (see `structured-logging-audit`). Actor attribution answers "who is currently responsible for this row's current state" — one answer per row, overwritten on every update. An audit trail answers "what happened, in what order, by whom" — an append-only history of every change. Many systems need both: attribution columns for fast "who owns this" lookups, plus an audit log for "show me everything that happened." Don't build the audit log first if all the product actually needs is attribution columns, and don't skip attribution columns just because an audit log exists — joining to a log table to render "edited by" in every list view is a needless N+1.

## Core pattern

```
<table>(
  ...,
  created_by  <actor_fk>  NOT NULL,
  updated_by  <actor_fk>,          -- nullable: never updated since creation
  created_at  timestamptz NOT NULL,
  updated_at  timestamptz,
  ...
)
```

- `created_by` is set once, at insert, and never changes — treat it as immutable, same as `created_at`.
- `updated_by` is overwritten on every mutating write. It reflects the *last* actor only; it is not a history.
- Pair each `_by` column with its `_at` sibling (`created_at`/`updated_at` — see `database-schema-design`) so "who" and "when" travel together.
- If soft-delete is in use, add `deleted_by` alongside `deleted_at` for the same reason — don't make deletion the one mutation nobody is attributed for.

## Setting the actor

- Bind attribution at the same single choke point that binds the audit actor (the auth gateway/middleware — see `authorization-rbac`), not at each call site. A per-call-site `updated_by: currentUser.id` parameter is exactly the kind of thing one call site will forget.
- Prefer a database-level default sourced from a request-scoped session variable (e.g. Postgres `current_setting('app.actor_id', true)` wired into a `BEFORE UPDATE` trigger or `DEFAULT`) when writes can also happen from raw SQL, a migration, or an admin console — an application-layer-only convention gets silently bypassed the first time someone runs a manual `UPDATE`.
- Whichever layer sets it, set it in exactly one place. Two code paths independently deciding how to stamp the actor is how you end up with a mix of user IDs, email strings, and `"system"` literals in the same column.

## System and automated actors

Real systems have writes with no human behind them: scheduled jobs, migrations, webhooks, imports. Decide up front how these are represented — don't leave the column nullable-by-accident and let "unknown" and "system" collapse into the same NULL.

- **Preferred:** give system actors real rows in the same actor table (a `system` user, one row per integration/service account) so `created_by`/`updated_by` stays a single non-nullable, always-joinable foreign key with no special-cased NULL handling anywhere that reads it.
- If that's not feasible, allow NULL explicitly for "no actor" and document what NULL means (never let it mean two different things in the same column).
- Never default to a magic sentinel ID (`0`, `-1`) instead of a real row or an explicit NULL — it silently breaks the foreign key constraint's usefulness and confuses the first person who joins on it expecting a real actor.

## Implementation notes

- Type the `_by` columns as a proper foreign key to the actor/user table, not a loose string — you want the FK constraint and the ability to join for a display name.
- Index `updated_by` (and `created_by` if you'll query by it) when "show me everything Jane touched" is a real query — same indexing discipline as any other foreign key (see `database-schema-design`).
- In an admin/audit UI, resolve `updated_by` to a display name via the actor join; don't store a denormalized name string on the row — actors get renamed, and a stale cached name is a support ticket waiting to happen.
- For a multi-actor action (a bulk import touching many rows, an automated rule triggered by one user's action), attribute each affected row to the actor that's actually responsible for it, not indiscriminately to whoever's session happened to be active — a rule engine acting on behalf of a user should usually attribute to the rule/system actor, with the triggering user recorded separately if that distinction matters to the product.

## Pitfalls

- Making `created_by` nullable "to be safe" — every row has a creator; a nullable creator just means the column was added after the fact without a backfill plan.
- Treating `updated_by` as a history — it's a single overwritten value. If "list everyone who ever edited this row" is a real requirement, that's an audit-log problem, not an attribution-column problem.
- Stamping the actor at scattered call sites instead of one gateway — guaranteed to have gaps.
- Denormalizing the actor's name/email onto the row instead of joining — goes stale the moment the actor is renamed.
- Silently defaulting missing attribution to a sentinel ID rather than a real system-actor row or an explicit, documented NULL.
- Forgetting `deleted_by` when soft-delete exists — deletion is a mutation like any other and deserves attribution too.

## Out of scope here

- Append-only, per-event history of every change — see `structured-logging-audit`.
- Trigger-based change-data-capture into a separate log table — see `change-data-capture-triggers`.
- Deciding roles/permissions for who is *allowed* to act — see `authorization-rbac`.
- General column/index/migration conventions — see `database-schema-design`.

## Checklist

- [ ] Every mutable table has `created_by` (NOT NULL) and `updated_by` (nullable), paired with `created_at`/`updated_at`
- [ ] `deleted_by` added alongside `deleted_at` wherever soft-delete exists
- [ ] Actor is bound at one choke point (auth gateway or DB session variable), not per call site
- [ ] System/automated writes have a defined actor representation — real system-actor row, or explicit documented NULL, never a sentinel ID
- [ ] `_by` columns are real foreign keys to the actor table, indexed if queried by actor
- [ ] Display name resolved via join, never denormalized onto the row
