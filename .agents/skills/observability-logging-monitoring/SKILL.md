---
name: observability-logging-monitoring
description: Use when instrumenting a service for production visibility — log levels, metrics (counters, gauges, histograms), correlation/trace IDs that follow a request across services, dashboards, and alerting thresholds. Covers the operational observability stack; defers to structured-logging-audit for structured log-field shape and actor-tagged audit trails.
---

# Observability: Logging, Metrics & Monitoring

Three pillars, each answering a different question: **logs** (what happened, in detail, for one instance), **metrics** (aggregated numbers over time, for trends and alerts), and **traces** (how one request moved across services). None substitutes for the others — a spike in the error-rate metric tells you *that* something broke; a correlated log line tells you *what*.

## When to use

Any service running unattended in production. Instrument before the first incident, not after — you cannot retroactively add logs to a spike that already happened.

## Log levels

| Level | When |
|-------|------|
| `debug` | Verbose diagnosis; disabled in production by default |
| `info` | Normal operation milestones — request handled, job completed |
| `warn` | Unexpected but handled — retried failure, approaching a limit |
| `error` | An operation failed and a human may need to look — payment declined, upstream timeout |
| `critical`/`fatal` | A whole process or dependency is down |

Default the production threshold to `info` or above. Treat every `error`-level line as something a human should be able to explain — logging routine, expected failures (a validation error, a 404) at `error` trains people to ignore the level entirely.

For the structured-field shape (what goes in `context`), secret scrubbing, and the separate actor-tagged audit trail, see `structured-logging-audit` — this skill assumes that foundation exists and covers the operational layer built on top of it.

## Metrics

Three shapes cover nearly everything you need:

- **Counter** — monotonically increasing (requests served, jobs failed). The rate of change is the useful signal, not the raw cumulative number.
- **Gauge** — a value that moves up and down (queue depth, open connections, memory used).
- **Histogram** — the distribution of a measured value (request latency, payload size), so you can read percentiles (p50/p95/p99) instead of a misleading single average.

Emit metrics with a name plus bounded tags/labels — `http_request_duration{route, method, status}` — not one metric per combination. A tag with unbounded values (raw user ID, raw URL) causes cardinality explosion, which breaks most metrics backends' cost and query speed.

## Correlation and tracing

- Generate or propagate a request/correlation ID at the edge (inbound HTTP request, queue message consumed) and attach it to every log line and outbound call made while handling it. This is what lets you pull one request's entire story out of an ocean of concurrent logs.
- In a multi-service system, propagate that ID — and a trace/span ID if using distributed tracing — through headers or message metadata on every downstream call, so the same logical request can be stitched together as it crosses service boundaries.
- Treat the correlation ID as mandatory context, on the same footing as level and timestamp — every log line in a request's path should carry it, not just the entry point.

## Dashboards and health checks

- Minimal per-service dashboard: request rate, error rate, and latency percentiles (the RED trio — **R**ate, **E**rrors, **D**uration), plus whatever resource the service is actually bottlenecked on (queue depth, DB connections, memory).
- Expose **liveness** and **readiness** as distinct checks. Liveness answers "is the process up"; readiness answers "are its dependencies (DB, cache, queue) reachable, so it's safe to route traffic here." Conflating them causes a slow dependency to pull a healthy process out of rotation, or a broken process to keep receiving traffic it can't serve.

## Alerting

- Alert on **symptoms users feel** — error rate, latency, queue backlog — not on internal implementation details that don't affect anyone yet.
- Use two tiers: a level that pages a human because it needs action now, and a level that just posts to a channel for review during business hours. Routing everything to the same channel trains people to ignore it.
- Tune thresholds per environment — staging noise should never share a pager rotation with production.
- Every alert should be actionable. If firing twice produces no different response either time, delete it or fix the threshold.

## Pitfalls

- Logging expected, handled conditions at `error` — reserve that level for things a human should actually investigate.
- Unbounded metric-label cardinality (raw IDs, free text as a tag) — degrades or breaks the metrics backend.
- No correlation ID — a multi-service incident turns into grep-and-guess across unrelated log streams.
- Alerting on everything — alert fatigue means the one alert that mattered gets lost in the noise.
- Reporting latency as an average instead of percentiles — the average hides the p99 tail your slowest, angriest users actually experience.
- Conflating liveness and readiness checks in one endpoint.

## Checklist

- [ ] Log levels used consistently; production threshold at `info` or above
- [ ] Metrics cover counters/gauges/histograms with bounded-cardinality tags
- [ ] Correlation ID generated at the edge and propagated through every downstream call and log line
- [ ] RED dashboard (rate/errors/duration) exists per service
- [ ] Liveness and readiness checks are separate endpoints
- [ ] Alert thresholds are actionable and tuned per environment
- [ ] Structured-field shape and audit trail deferred to `structured-logging-audit`, not re-implemented here
