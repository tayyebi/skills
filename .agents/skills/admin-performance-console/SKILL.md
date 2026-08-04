---
name: admin-performance-console
description: Use when an application needs its own performance page — in-process timings for every request, statement and named function, surfaced to operators with an advisory layer that says which row is the problem and what to do about it. Covers the in-memory aggregate store, global instrumentation, label cardinality, percentiles from a ring buffer, per-request query attribution, snapshot trends, and turning a table of numbers into ranked findings with actions.
---

# Application Performance Console

"Is it slow?" is answerable from outside. "*What* is slow?" is not — not from a load balancer graph, not from a log grep, and not from a synthetic check. The honest answer needs a timing on every code path, and most teams get there by buying an APM, which knows how to time an HTTP request but knows nothing about *your* resume parser, *your* AI call, *your* nightly reconciliation.

The pattern that earns its keep is smaller than an APM and more useful than a dashboard: **an in-process store that times everything, and a page inside the app that interprets what it collected.** The store is a few hundred lines. The interpretation is what makes it get used.

That second half is the part everyone skips. A page with three hundred labels and eight numeric columns is complete and unreadable — it hands the reader a table and a research project. The console is finished when it can say *"this route issues forty queries per call, here is the trend, here is the button that isolates it."*

## When to use

- A self-hosted or small-fleet product where the operator is the developer, and an APM's per-host pricing buys observability nobody looks at.
- An application whose interesting work is *not* HTTP: background jobs, imports, model calls, document parsing. These are exactly what generic tooling cannot name.
- Anywhere "which of our own functions is slow" is currently answered by adding a `console.time` and redeploying.

Skip it if you already run distributed tracing end to end and the questions you ask are cross-service. This pattern is deliberately single-process: it has no notion of a trace spanning two machines, and pretending otherwise is how it becomes a bad APM instead of a good console.

## The core pattern: aggregate in memory, never write on the hot path

One rule dominates the design: **recording a timing must not touch the database.**

A per-call `INSERT` is the obvious implementation and it is a trap — it makes instrumentation cost a connection, so the busiest code paths (the ones you most want measured) are the ones that exhaust the pool. Worse, the failure is self-reinforcing: the pool saturates, everything slows down, and the slowdown generates more timing writes.

So the store is a map of label → aggregate, held in process memory:

```
recordTiming(label, durationMs, { error?, meta? })   // O(1), synchronous, never async
```

Two consequences to design around, both fine, both worth stating in the UI:

- **It resets on restart.** Counts and totals are "since this process started", not "today". Say so on the page, or someone will read a deploy as a traffic collapse.
- **It is per process.** Behind N app instances you are reading one of them. Either accept it and label the page accordingly, or aggregate the snapshots (below) by instance.

Hold the state on a global/module-level singleton keyed by a well-known symbol, not a plain module variable — dev hot-reload re-evaluates modules and would otherwise silently double-count or reset.

## Instrument globally, not at call sites

The tempting design is a `withTiming()` wrapper applied to each route handler. It produces a metric with holes in it: two hundred route files, one of them added next week by someone who didn't know the convention, and the gap is invisible because a missing label looks exactly like a code path nobody called.

Time the chokepoints instead — the two places every unit of work funnels through:

- **Inbound requests:** patch the HTTP server's request emission (what APM agents do), not the framework's middleware. Middleware typically runs *before* the handler and returns immediately, so it can only measure itself. The server-level hook sees everything regardless of which layer answered: handlers, server-rendered pages, redirects, static files.
- **Database statements:** wrap the *connection pool*, not the ORM handle. This is the one that catches people. `orm.transaction(cb)` hands the callback a raw handle that never touches your wrapper again — and transactions are where the writes are. Wrap the pool's checkout, and through it each client's query method, and you see pooled queries, transactional queries and raw SQL exactly once. (Wrapping both the pool's `query` *and* the client's will double-count the common path.)

Then add **opt-in spans** for the things worth naming individually:

```
result = measure('ai.autoScreen', () => screen(candidate))
stop = startSpan('resume.extract'); … ; stop()
```

Global instrumentation answers "which route", opt-in answers "which of our operations". A console with only the first has a blind spot exactly where the expensive work lives — and that absence is itself worth reporting to the reader (see the advisory layer).

Every hook must be wrapped so a failure *inside* instrumentation can never fail the thing it measures, and installed idempotently so a reload doesn't patch twice.

## Cardinality is the whole game

The store caps its label count — it must, or a memory leak is one interpolated id away. Which means every label has to be a **shape**, not a value:

```
/candidates/6b1f…/ratings   →   GET /candidates/:id/ratings
select * from c where id=42 →   db.select.candidates
```

The rule that keeps this straight: **the label answers "which code path", the meta answers "which row".** Anything that varies per row goes in meta and never in the label.

Put normalization in its own module, so both global sources route through one set of rules. Be conservative about what counts as an id: a false positive silently merges two real code paths into one row, which is worse than one extra label. Digits, UUIDs, long hex, percent-encoded segments — yes. Short words — no; handle those by position instead (the segment after `/users/` is a user).

**Report the cap.** When the store starts dropping new labels, every code path added from that moment on is invisible, and nothing in a table of existing rows will ever tell you. Expose `{ labelCount, maxLabels, capped }` and let the page raise it as a finding. This is also why the footer should say "showing 200 of 1 340 labels" rather than "200 labels" — a capped list is indistinguishable from a short one.

## The aggregate: what to keep per label

```
label, count, errorCount, totalMs, minMs, maxMs, lastMs, lastTs,
maxMeta,                 // meta from the slowest call ever seen
samples[N], sampleLen, sampleIdx      // ring buffer for percentiles
```

Two of these do more work than they look:

- **`maxMeta`** — the meta attached to the worst call, kept for as long as that call holds the record. This is what lets the table show "the slowest call to this route issued 40 queries" without storing per-call rows.
- **The ring buffer** — a fixed-size window (a few hundred samples) of recent durations. Percentiles are computed over it on read, by sorting a copy. Bounded memory per label, no time-series database, and no percentile estimator to explain to anyone.

Be explicit about the mixed time base this creates, because it will confuse someone: **`count` and `totalMs` are cumulative since process start, while percentiles describe only the last N calls.** A label that was catastrophic this morning and fine since reads as fine. That is usually what you want on an operations page — but only if the page says so.

## Attribution: queries per request

The single highest-value number on the page, and the reason to bother with a request context at all.

Open an async-local context per request; have the pool instrumentation increment `queryCount` and `queryMs` on whichever context is current; attach both to the request's timing meta.

```
GET /candidates/:id   p95 320ms   queries@max 47
```

Without that column, an N+1 is invisible: the route looks *mildly* slow, and every individual statement looks fast, because each one is. With it, the diagnosis is immediate and the fix is structural (a join, an `IN (…)`, a batch loader) rather than a micro-optimization that recovers nothing.

Capture the context **synchronously**, while still on the caller's stack. Reading it inside the query's completion handler returns nothing — the continuation has already left the context. For pooled clients that outlive a single call, stamp the borrowing context onto the client at checkout, so a recycled connection cannot bill its queries to the previous borrower.

## Trends: snapshot periodically, and know what can't be plotted

In-memory data dies at deploy time, which is precisely when someone asks "was it like this yesterday?" Flush one row per label to a table on a timer (a minute is plenty), prune past a retention window, and read it back for a per-label sparkline.

**Snapshot percentiles, not counters.** `count` and `totalMs` are cumulative since process start, so plotted across a restart they sawtooth to zero and read as an outage that never happened. Percentiles describe a recent window and stay comparable across restarts. If you want a rate, difference the counters between consecutive snapshots and drop the negative deltas — but percentile-only is the honest default.

The flush must be fire-and-forget: a broken snapshot table can degrade the trend, never the application.

## The switch, and sampling

Two settings, read synchronously from a process-local snapshot (the hooks are far too hot to await a settings load — refresh the snapshot on the same timer as the flush):

- **enabled** — a master off switch. Off must mean *nothing recorded*, and the page must say the table is frozen; a stale table with no explanation is worse than an empty one.
- **sampleRate** — record one request in N under load.

Sample requests, **not** statements. Most SQL is issued by jobs and cron ticks that have no request to sample, and a partially-sampled query profile is a misleading one. Keep the per-request query counters running even for unsampled requests, so the counts that *are* recorded stay honest.

Anything sampled must be labelled as sampled where its numbers are read. A reader comparing a 10%-sampled call count against a business metric will otherwise conclude the feature is unused.

## Reading the page: what to watch, in order

This belongs *in the product*, not only in a runbook. A collapsed "how to read this page" panel costs one afternoon and is the difference between a screen three people can use and a screen everyone can.

| Column | What it actually means |
|---|---|
| Calls | Cumulative since process start — not since you opened the page. A deploy resets it. |
| Err | Threw, or answered 5xx. Compare to Calls, never to other rows. |
| Avg | The least useful number here. One 30-second outlier in 10 000 fast calls barely moves it. |
| p50 | The typical call. |
| p95 | The default sort and the number to argue about: 1 call in 20 is at least this slow. What a user notices. |
| p99 | The tail. Far above p50 means the same code behaves in two different ways. |
| Max | The single worst call ever recorded — the one `maxMeta` describes. |
| Total | Calls × duration: the actual bill. Where the server spends its day. |
| Queries@max | Statements issued by that worst call. Ten or more is how an N+1 announces itself. |

The reading order that finds problems fastest:

1. **Connection pool first.** Anything queued for a connection makes every other number on the page wrong — the wait is *inside* them.
2. **Then queries-per-call.** A route at 300 ms issuing 40 statements is a worse problem than a route at 900 ms issuing one, and a cheaper fix.
3. **Then Total, not p95.** p95 finds the annoying call; Total finds the one worth an afternoon. They are rarely the same row: a cheap call made constantly outweighs an expensive call made rarely.
4. **Then the trend, before changing anything.** A number that has always been this size is not an incident.

And what the page is *not*, stated on the page: in-memory (resets on deploy), mixed time base (cumulative counts, windowed percentiles), server-side only (says nothing about the network or the browser), and per process.

## From table to advice: the advisory layer

Everything needed to say which row is a problem is already in the response. Encode the thresholds an experienced reader applies by eye, and surface them as ranked findings above the table.

**Make the rules a pure function.** Input: the already-fetched response. Output: an array of findings. No fetch, no database, no UI framework. Three things follow, and all three matter:

- The rules are testable without a running application — table-driven, fast, and pinned to a threshold rather than to a screenshot.
- They can re-run on every poll for free, so the findings track the live data.
- They stay out of the server, which means the console can be added to an existing page without touching the collection path.

A finding is:

```
{ id, severity: critical|warning|info,
  titleKey, bodyKey, params,        // i18n keys, never literal prose
  evidence: [label, …],             // the rows it fired on
  detail?: "select … from …",       // untranslatable supporting text
  actions: [ … ] }                  // at least one, always
```

A starting rule set, roughly in the order they should be read:

| Finding | Fires when | Why it leads |
|---|---|---|
| Collection off / sampled | The switch is off, or rate < 1 | Every finding below is read through this |
| Label cap reached | Store is dropping new labels | New code paths are invisible from now on |
| Connections queued | pool.waiting > 0 | The wait is inside every other timing |
| Error rate | ≥ 2% over ≥ 20 calls | A fast failure is still a failure — and error paths are *quick*, so a rising error rate looks like an improvement |
| Many queries per call | ≥ 10 statements in one call | Structural fix, biggest payoff |
| Slow routes | p95 ≥ 1 s over ≥ 5 calls | What a user notices |
| Slow statements | p95 ≥ 100 ms over ≥ 5 calls | At that scale it is reading far more rows than it returns |
| Latency spread | p99 ≥ 4× p50, ≥ 50 calls, above an absolute floor | The same code behaving two ways: cold cache, lock contention, input size |
| Time sink | One label ≥ 25% of all measured time **and** ≥ 2× the runner-up | Where optimization pays |
| Nothing hand-instrumented | Requests timed, no named spans | Names the blind spot rather than hiding it |
| Nothing to act on | No rule fired | Silence should be explicit, not ambiguous |

Design notes that are easy to get wrong:

- **Analyse the unfiltered set.** If the page filters server-side and you analyse the filtered rows, a finding vanishes when the reader narrows the view — filter to HTTP and the statement starving the pool stops being reported. Give the advisor its own read with fixed parameters (everything, ordered by total), independent of the table's controls.
- **Every threshold needs a minimum sample and, for ratios, an absolute floor.** A p95 over three calls describes three calls. A 4× spread between 0.1 ms and 0.4 ms is arithmetic, not a defect. One failure in five calls is not a 20% error rate.
- **Share is not dominance.** "This label is 33% of all time" is guaranteed when three labels exist. Require the leader to be a multiple of the runner-up before naming it.
- **Rank by severity and show the count collapsed.** The panel should be scannable when there are twelve findings, not just when there are two.
- **Findings carry evidence.** "Some routes are slow" is a horoscope. List the labels, with the number that triggered the rule, so the claim can be checked against the row it came from.
- **Thresholds are constants in code with the reasoning in a comment — not settings.** A configurable threshold gets raised until the page is quiet again, and a quiet page is the goal it was supposed to serve.
- **All copy goes through i18n keys**, including finding bodies. They are the most-read prose in the console. Write two sentences: what the number means, then what to do.

## Actions that belong on the page

A finding without a next step is a finding nobody acts on. Every rule should hand back at least one thing to click, and the page — not the rules — decides what each one does:

- **Focus** — set the group, sort and search so the table shows exactly the evidence. The single most useful action, and free to implement.
- **Trend** — open the label's snapshot history. Answers "is this new?", which decides whether to act at all.
- **Reset one label** — clear one label's counters, exercise the path, read a clean number. This is a *measurement* tool, and the page should say so, or someone will treat it as a fix. Most implementations already accept a label parameter on the reset endpoint and never expose it in the UI, leaving "reset everything" as the only option — which destroys the baseline you were about to compare against.
- **Copy `EXPLAIN ANALYZE <statement>`** — for a slow statement, put the runnable query on the clipboard and link to the SQL console. Two clicks from "this is slow" to a query plan.
- **Deep-link to the neighbouring console** — the database profile for indexes and vacuum age, the log viewer filtered to the failing label, the settings page for the collection switch. The finding knows which one is relevant; the reader shouldn't have to.

Reset, and anything else that mutates, needs the same authorization and audit treatment as the rest of the admin surface — it is install-wide state, and "who cleared the counters an hour before the incident review" is a real question.

## Pitfalls

- **Writing a row per timing.** Instrumentation that consumes a connection exhausts the pool under exactly the load you built it to observe.
- **Wrapping the ORM handle instead of the pool.** Transactions bypass it, and transactions are where the writes are — so the missing half is the interesting half.
- **A raw path or raw SQL as the label.** The cap is reached within an hour of real traffic, and the page becomes noise on the way there.
- **Plotting cumulative counters across restarts.** Sawtooth graphs that look like outages and aren't.
- **Percentiles without a minimum sample count.** The loudest row on the page is a code path called four times.
- **An advisor that fires on everything.** It gets collapsed on the second day and never opened again. Tune for silence when the system is healthy — and make that silence explicit with a "nothing to act on" finding, so the reader can tell it apart from a broken panel.
- **Advice with no action.** The reason the table was ignored in the first place; a paragraph of advice is just a longer table cell.
- **Analysing the filtered view.** Findings that disappear when the reader narrows the table teach people not to trust the panel.
- **Not saying the page is per-process, in-memory, and server-side.** Every one of those produces a confident wrong conclusion within a week.

## Checklist

- [ ] Recording is synchronous, in-memory, O(1), and never touches the database
- [ ] State on a global symbol, so hot-reload can't double-count or reset it
- [ ] Requests timed at the server hook; statements timed at the pool's checkout; both installed idempotently and unable to throw
- [ ] Opt-in `measure()` / `startSpan()` for named operations, plus a rule that reports when nobody has used them
- [ ] Every label normalized to a shape by one shared module; ids and values live in meta
- [ ] Label cap enforced, and `{ labelCount, maxLabels, capped }` exposed and surfaced in the UI
- [ ] Per-label aggregate keeps a ring buffer for percentiles and the meta of the worst call
- [ ] Per-request query count and query time attributed via an async-local context, captured synchronously
- [ ] Periodic snapshot of percentiles for trends; counters excluded and the reason documented
- [ ] Master switch plus request-only sampling, both readable synchronously, both stated in the UI when active
- [ ] A "how to read this page" panel: every column, the reading order, and what the page is not
- [ ] Advisory rules as a pure function, unit-tested, analysing the unfiltered snapshot
- [ ] Every threshold has a minimum sample count; every ratio has an absolute floor
- [ ] Every finding names its evidence and carries at least one action
- [ ] Actions cover: focus the table, open the trend, reset one label, copy a runnable query, deep-link to the neighbouring console
- [ ] Reset (all and per-label) is authorized and audited like any other admin mutation
