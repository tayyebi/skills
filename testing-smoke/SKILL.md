---
name: testing-smoke
description: Use when you need a fast go/no-go verdict that a deployment is safe to declare live — a tiered, time-boxed sanity check (up? auth works? core data intact? critical write path? secondary domain? optional privileged access?) run in under 5-10 minutes with a hard stop-and-escalate rule on any tier-0/1 failure. Curl-only variant: [[testing-smoke-curl]]. Deep endpoint coverage: [[testing-api]]; full user journeys: [[testing-functional]].
---

# Testing Smoke Test & Deployment Sanity

A rapid post-deployment verification, not a regression suite. The goal is fast confidence that the critical path works — enough to say "safe to call this live" or "roll back now," not enough to catch everything.

## When to use

- Immediately after any deployment, before declaring it live or moving traffic to it.
- As a CI gate between "deploy succeeded" and "deploy is healthy."
- Time-boxed: budget 5–10 minutes total. If a tier fails, stop and escalate — don't keep testing a system you already know is broken.

## Core pattern

Structure the check as **tiers of increasing risk**, each gating the next, so a hard failure early saves the time of running everything downstream:

- **Tier 0 — Is it up?** Version/health endpoints respond, and the app's entry point serves 200. This confirms the process is running and its dependencies (DB, cache, queue) report connected — nothing more. Any failure here means the deployment is broken; stop immediately.
- **Tier 1 — Auth works.** Register or log in, use the returned credential on an authenticated read, and confirm any CSRF/anti-forgery middleware is actually enforced (a mutating request without the token should be rejected, not silently accepted). Auth is the single most common regression surface — it gates everything else a real user does.
- **Tier 2 — Core reference data is intact.** Whatever catalog/config data the app depends on (plans, regions, templates, feature flags — anything seeded or migrated) should be non-empty. Zero rows here usually means a wiped table or a failed migration, not a code bug.
- **Tier 3 — A critical write path round-trips.** Exercise the one write flow that matters most to the business (add-to-cart-and-checkout, create-and-fetch-a-record) including its expected *negative* path (e.g., checkout with no funds should fail with a specific error, not silently succeed). A write that silently no-ops or a guard that silently passes is the highest-severity smoke failure there is.
- **Tier 4 — A secondary domain still works.** Pick one more subsystem that's independent of the main path (support ticketing, notifications, search) as a broader signal the deployment isn't narrowly broken.
- **Tier 5 — Privileged access, optional.** If a privileged/admin credential is available out-of-band, confirm it can reach privileged routes and that a non-privileged credential still cannot. Never block the go/no-go decision on this tier — most smoke runs won't have an admin credential handy, and that's fine.

Each tier should print a one-line PASS/FAIL per check with the actual vs. expected value, so a human (or CI log) can diagnose without re-running anything.

## Design decisions

- **Go/no-go is binary per tier, not per check.** One hard failure in tier 0 or tier 1 is enough to stop; don't average failures into a "mostly passing" score.
- **The negative path is part of the check, not optional.** A checkout that should reject an empty wallet but instead succeeds is a worse bug than the happy path failing — smoke tests should assert the guard, not just the happy case.
- **Keep it fast by design.** Smoke intentionally excludes anything slow (full provisioning, payment gateway round-trips, load/perf) or hard to automate (email/webhook delivery). Those belong in targeted discipline-specific checks, not the 5-minute gate.
- **Clean up what you create.** A smoke run typically creates a throwaway account/record — delete or archive it as the last step so smoke runs don't pollute production data over time.

## Pitfalls

- **No tiering / flat check list** — a cosmetic tier-4 failure gets equal weight to "auth is broken," obscuring the real signal.
- **Continuing past a tier-0 failure** — wastes the time budget confirming things you already know are broken downstream.
- **Smoke test scope creep** — pulling in slow or flaky checks (full lifecycle, real payment capture) turns a 5-minute gate into a 30-minute one nobody trusts enough to block on.
- **No negative-path assertions** — only checking that things succeed misses the entire class of "guard silently disabled" bugs.
- **Leaving test data behind** — throwaway accounts and records accumulate and eventually pollute reporting/analytics.

## Checklist

- [ ] Tier 0: version/health/entrypoint all respond healthy
- [ ] Tier 1: register or login, authenticated read, CSRF/anti-forgery enforced
- [ ] Tier 2: reference/catalog data non-empty
- [ ] Tier 3: critical write path round-trips, including its negative path
- [ ] Tier 4: one secondary subsystem checked
- [ ] Tier 5 (optional): privileged access confirmed without gating go/no-go
- [ ] Any failure halts further checks and escalates immediately
- [ ] Throwaway test data cleaned up after a pass
