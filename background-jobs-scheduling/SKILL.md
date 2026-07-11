---
name: background-jobs-scheduling
description: Use when work must happen outside the request cycle — an async job queue for slow/retryable tasks, cron-driven scheduled jobs, and a trigger→action automation engine. Covers job auth, idempotency, and delayed actions.
---

# Background Jobs, Scheduling & Automation

Three related mechanisms: **async jobs** (do slow work off the request), **scheduled jobs** (do work on a clock), and an **automation engine** (do work in reaction to events).

## When to use

Any task that is slow (resume parsing, AI calls, image processing), must retry (email/webhook delivery), or must happen on a schedule (reports, cleanup, backups).

## Async job queue

- Table `background_jobs(id, type, payload jsonb, status, attempts, run_after, locked_at, result, error, created_at)`.
- A worker claims jobs (`SELECT ... FOR UPDATE SKIP LOCKED`), runs them, records result/error, and increments `attempts` with backoff on failure.
- **Make handlers idempotent** — a job may run twice (crash after work, before marking done). Key side effects so re-runs are safe.
- Enqueue from request handlers so the user's action returns immediately (e.g. notifications, exports).

## Scheduled jobs (cron)

- A cron scheduler hits app endpoints (or invokes workers) on a schedule: process-queue every minute, reports at a fixed time, cleanup/backup periodically.
- **Authenticate cron calls** — the endpoints are internal but network-reachable. Use a rotating shared token (from a shared volume/secret), not "trust the source IP."
- Emit a **heartbeat** per scheduled job and alert if one stops running — a silent dead cron is a classic outage.
- Keep run history (`*_runs` tables) for observability.

## Automation engine (trigger → action)

Let operators define rules without code:

```
automation_rules(id, tenant_id, trigger, conditions jsonb, action, params jsonb,
                 delay_minutes, rate_limit, enabled)
```

- On a domain event, match enabled rules; enqueue matching actions.
- Support **delayed actions** via a `scheduled_jobs(run_after, ...)` table (e.g. "send follow-up 2 days after stage change").
- **Rate-limit** automations so a bulk import doesn't fire 10,000 emails.

## Pitfalls

- Non-idempotent handlers + at-least-once delivery = duplicate side effects.
- Unauthenticated cron endpoints — a public URL that triggers a backup or report is an abuse vector.
- No visibility — without heartbeats and run history you won't notice a stalled worker.
- Poison jobs retrying forever — cap attempts, move to a dead-letter state.
- Doing slow work inline in the request because "the queue is more setup" — it isn't, and users feel every second.

## Checklist

- [ ] Job table with status/attempts/backoff, `FOR UPDATE SKIP LOCKED` claiming
- [ ] Idempotent handlers; attempt cap + dead-letter
- [ ] Cron endpoints authenticated (rotating token), with heartbeat + run history
- [ ] Automation rules: trigger→action, delayed actions, rate-limited
