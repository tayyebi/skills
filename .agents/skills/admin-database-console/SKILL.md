---
name: admin-database-console
description: Use when an application needs a built-in database GUI for operators — browse tables, run SQL, and watch size/bloat/index/connection health from inside the app instead of shelling into the box. Covers the single action-dispatched endpoint, identifier safety, cheap row counts, graceful profiling degradation, cache and retention observability, and the pitfalls that make such a console lie.
---

# Admin Database Console

Every non-trivial application eventually needs someone to answer "what does the data actually look like right now?" — and the answer is usually a shell on the production host, `psql`, and a query nobody else will ever see. That path has three costs: it needs credentials that shouldn't exist, it leaves no trail of who read or changed what, and it teaches the team that the truth about the system lives outside the system.

The alternative is not "install Adminer next to it." A bolted-on DB admin tool knows the tables but knows nothing about *your* application — it cannot show which of your caches are stale, which retention triggers are installed, or which columns are derived rather than stored. The pattern that earns its keep: **an operator console inside the app, behind the app's own super-admin authorization, that treats the database as an observable subsystem rather than a spreadsheet.**

Data browsing is the least interesting thing it does. The reason to build it is that it can put schema, live Postgres statistics, in-process cache state, and retention health on one screen — and that combination exists nowhere else.

## When to use

- A self-hosted or single-tenant-per-install product where the operator *is* the developer, and "check production" happens weekly.
- A system with meaningful in-process state (caches, pools, feature flags) that no external tool can see.
- Any environment where direct shell access to the database is something you're actively trying to eliminate — this is the thing that replaces it.

Skip it when a managed console already covers you (RDS Performance Insights, Cloud SQL Insights) *and* nobody needs the app-specific half. Skip it entirely for a product where end users could ever reach the surface: this is a super-admin tool, and there is no "read-only tenant" version of it worth building.

## The core pattern: one endpoint, action-dispatched

Resist a route per panel. The console is one resource — the database — inspected many ways, and a single endpoint keeps the authorization check in exactly one place instead of eight.

```
GET  /admin/database?action=tables|columns|browse|profile|caches|retention|computed
POST /admin/database   { action: vacuum|prune|invalidate-cache }  |  { query: "SELECT …" }
```

Every handler starts the same way, and this is non-negotiable:

```
auth = requireSuperAdmin(request)      // NOT "admin". Not tenant-scoped admin.
if (!authorized(auth)) return auth
```

A tenant admin must not reach this. The console reads across every tenant's rows by construction — row-level isolation is bypassed the moment you `SELECT * FROM <table>`. Treat "which role can open the database console" as an install-wide answer, never a per-tenant one.

**Unknown actions must fail loudly.** The dispatcher's fallthrough returns 400, which is correct on the server and silent on the client: a panel that fetches `action=history` against a server that never implemented it renders an empty list forever, indistinguishable from "no history yet". Enumerate the action names in one shared constant that both sides import, or accept that a typo becomes a permanently blank tab nobody reports.

## Identifier safety, and the one deliberate exception

Two different trust models coexist here, and conflating them is the classic way this feature becomes the vulnerability.

**Structural parameters are never interpolated.** Table names, column names, and sort directions arrive from the client and are validated against a strict pattern before they touch a query, then passed through the driver's identifier-quoting helper — not string concatenation:

```
if (!/^[a-zA-Z_][a-zA-Z0-9_]*$/.test(table)) return 400
orderCol = /^[a-zA-Z_][a-zA-Z0-9_]*$/.test(orderBy) ? orderBy : 'id'
orderDir = (dir === 'asc') ? ASC_FRAGMENT : DESC_FRAGMENT     // two literals, not a string

SELECT * FROM identifier(table) ORDER BY identifier(orderCol) <dirFragment> LIMIT ? OFFSET ?
```

The regex is the gate; the identifier helper is the seatbelt. Keep both. A name that survives validation is still quoted, so a table called `order` or `user` doesn't break the console.

**The SQL console is raw by design.** The query textarea executes what the operator typed, verbatim, including `UPDATE` and `DROP`. Do not attempt to make this safe by parsing — a keyword blocklist is trivially defeated (`/**/`, `DO $$`, a CTE that writes) and its real effect is to convince you the endpoint is safer than it is. The trust boundary for free-form SQL is **authorization plus an audit record**, not syntax analysis:

```
audit('database.query', truncate(query, 200), { userId, queryLength })   // BEFORE execution
result = execute(raw(query))
```

Write the audit entry *before* running the statement. A query that deadlocks, times out, or takes the process down with it must still leave a record of who ran what — a log line written after the fact never happens for exactly the queries you most want to know about.

Bound the damage of accidents instead: cap query length (~10k chars), cap returned rows (~500) and **say so in the response** with an explicit `truncated: true` that the UI renders. A silently truncated result set is a wrong answer that looks like a right one.

## Cheap row counts: the counter table

The table list wants a row count per table, and this is where the console gets slow enough to feel broken. `SELECT count(*)` in Postgres (and any MVCC engine) is a full heap or index scan — there is no covering shortcut — so listing 60 tables means 60 sequential scans on every page load, growing with your largest table.

`reltuples` from the catalog is the usual escape hatch, but it's an estimate that drifts between autovacuums and looks wrong next to a table the operator just imported into. When you want exact counts cheaply, maintain them:

```
table_row_counts(table_name PK, row_count bigint, updated_at)

AFTER INSERT/DELETE/TRUNCATE ON <table>
  FOR EACH STATEMENT REFERENCING NEW TABLE AS new_rows OLD TABLE AS old_rows
  EXECUTE row_counts_sync()      -- adjusts by the statement's delta
```

Three properties make this work, and all three are load-bearing:

- **Statement-level, not row-level.** A bulk import of 5 000 rows in one statement costs *one* counter update. The row-level version costs 5 000 updates to a single row — a lock convoy that turns your counter into the slowest thing in the system.
- **Incremental, not invalidate-and-recount.** Adjust by the delta rather than clearing a cached value, so no read ever pays for a recount and the number is never stale.
- **A repair function, not blind trust.** Provide `resync_table_row_count(name)`. Triggers can be bypassed — a restore under `session_replication_role = replica` does exactly that — and the fix must be one call, not a migration.

Reads then fall back gracefully: cached value if present, live `count(*)` if that table isn't maintained, and a missing counter table (migration not applied yet) logs a warning and returns empty rather than failing the page.

## The profiling panel: the part that isn't a spreadsheet

Five queries against the statistics catalogs turn "the database" into something with a health readout. In Postgres terms:

| Section | Source | What it answers |
|---|---|---|
| Overview | `pg_database_size`, `pg_stat_database` | Total size, cache hit ratio, live/total connections |
| Table sizes | `pg_total_relation_size` vs `pg_relation_size` | Which table is big, and how much of it is *index* |
| Table health | `pg_stat_user_tables` | Dead-tuple ratio, seq vs index scans, last (auto)vacuum/analyze |
| Indexes | `pg_stat_user_indexes` + `pg_index` | Never-scanned indexes — pure write cost, sorted to the top |
| Connections | `pg_stat_activity` | Who is connected, what they're running, for how long, blocked on what |

**Colour the numbers against thresholds, don't just print them.** Cache hit below 99%, dead ratio above 10%, an index with `idx_scan = 0`, a query past 1 s — an operator scanning the page should see the problem without knowing the healthy value for each metric. Exempt primary keys from the unused-index warning; they're never "unused", they're constraints.

**Degrade gracefully rather than showing a wall.** Cumulative slow-query stats need an extension (`pg_stat_statements`) that may not be loaded. The tempting implementation renders "extension not enabled — ask your DBA" and stops. Instead, cascade:

1. Query the view. If it works, you have cumulative stats — label the panel accordingly.
2. If it fails, try `CREATE EXTENSION IF NOT EXISTS` — this succeeds without a restart when the shared library is already preloaded, which is the common case.
3. If that fails too (library genuinely absent, needs an ops-level restart), fall back to the **longest-running live queries** from `pg_stat_activity`, and mark the panel `live` so the operator knows they're seeing a snapshot, not history.

Tier 3 is worth more than it looks. It answers "what is slow *right now*", which is the question being asked when someone opens this tab in anger. Exclude your own backend PID and the introspection query itself, or the panel reports on its own reflection.

The same cascade is the general lesson: **an observability surface that can be unavailable will be unavailable exactly when it matters.** Always have a degraded mode with an honest label.

## Application state the database can't see

This is what a generic DB tool cannot do, and the reason to build rather than install.

**Register every in-process cache with metadata, not just a size.** A number of entries tells an operator nothing about whether clearing it is safe:

```
{ name, module, description, type: map|pool|flag,
  ttl: "1 hour" | "CRUD-only" | "LFU (30 days)",
  invalidation: "Any candidate write",     // what SHOULD clear it
  clearable: true,
  entries, maxEntries, details: [{ key, hitCount, cachedAt, accessedAt }] }
```

`invalidation` is the field that earns the panel. When a cache holds data an operator believes is stale, the actionable question is "what was supposed to have cleared this and didn't" — and that only has an answer if each cache declares its contract. `clearable: false` for pools and flags keeps the UI from offering a button that can't work.

Per-entry `hitCount` and `accessedAt` turn the panel into a sizing tool: a cache whose entries are never re-read is pure memory cost, and you cannot learn that from an aggregate.

**Surface retention as installed-or-not, not as policy.** If rows are pruned by triggers (log tables, delivery records, query history as ring buffers), the console should report, per table: current count, the cap, **whether the trigger actually exists**, and the oldest surviving row.

```
SELECT EXISTS (SELECT 1 FROM pg_trigger
               WHERE tgname = 'trg_logs_auto_prune' AND NOT tgisinternal)
```

Checking for the trigger — rather than trusting that the migration ran — is what catches the failure mode where a table has been growing unbounded for months on one install. Pair it with a manual "prune now" so the fix is a click.

**List derived columns explicitly.** Generated columns from `information_schema.columns WHERE is_generated = 'ALWAYS'`, plus a hand-maintained list of the app-maintained ones (a JSONB column a background job recomputes). Both appear identical when browsing rows, and an operator who `UPDATE`s one is either rejected by the engine or silently overwritten by the next job run. Naming which is which prevents both.

## UI shape

- **Master list of tables on the left, panels as tabs on the right.** Filter box over the table list — a 60-table sidebar is unusable without it.
- **Put the active tab and selected table in the URL.** Operators paste console links into incident threads; a link that reopens on the default tab is worthless.
- **Load each panel lazily and only once**, on first activation, with an explicit refresh button and a "last updated HH:MM:SS" stamp. Profiling queries are not free — never poll them on a timer by default.
- **Cells truncate; clicking one opens the full value.** Wide JSONB and long text otherwise destroy the table layout, and the value an operator opened the console for is usually the truncated one.
- **Query history is a shared artifact.** Store it in a table (query, user, duration, row count, error, timestamp) rather than in `localStorage`, cap it as a ring buffer, and render it click-to-reload. "What did the last person run when this happened" is the console's second-most-used feature; per-browser history cannot answer it.
- **`Ctrl/Cmd+Enter` runs the query.** Anyone who reaches for this screen has that reflex.

## Pitfalls

- **Wiring the console to tenant-scoped authorization.** It reads across tenants by construction; scoping the *route* while not scoping the *queries* is a data-isolation breach with an audit trail that says "authorized".
- **Trying to make free-form SQL safe with a keyword filter.** It doesn't work, and it buys false confidence. Authorize hard, audit first, cap the blast radius.
- **Counting every table on every page load.** The single most common reason this screen gets abandoned as "too slow".
- **Truncating results without saying so.** An operator who reasons from a silently capped 500-row result reaches a wrong conclusion and blames the database.
- **Auditing after execution.** The queries worth auditing are disproportionately the ones that don't return.
- **A profiling tab that says "not available".** Cascade to a degraded, clearly-labelled view instead.
- **Trusting that retention migrations ran.** Check for the trigger; report its absence as a finding.
- **Letting the panel list drift from the server's action list.** A client-only action name yields a permanently empty tab and no error anywhere.

## Checklist

- [ ] One endpoint, action-dispatched, with a single super-admin check at the top of every handler
- [ ] Table/column/sort inputs regex-validated **and** identifier-quoted; no string interpolation of structure
- [ ] Free-form SQL audited before execution, length-capped, row-capped, with `truncated` surfaced in the UI
- [ ] Row counts served from a statement-level trigger-maintained counter, with a resync function and a live-count fallback
- [ ] Profiling covers size, bloat/vacuum age, unused indexes, connections — with thresholds coloured, not just printed
- [ ] Slow queries cascade: extension → try-enable → live-activity fallback, labelled with which mode is active
- [ ] Every in-process cache registered with its TTL, its invalidation contract, and whether it is clearable
- [ ] Retention reported as trigger-exists + count + cap + oldest row, with a manual prune action
- [ ] Generated and app-maintained columns listed distinctly from stored ones
- [ ] Panels lazy-loaded with explicit refresh; active tab and selected table reflected in the URL
- [ ] Query history persisted server-side, capped as a ring buffer, shared across operators
