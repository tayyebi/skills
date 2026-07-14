---
name: testing-performance
description: Use when testing an application's runtime behavior under load and timing pressure from the outside — covers latency baselining, throughput/load testing, rate-limit enforcement (and bypass via spoofable headers), race conditions in concurrent request handling (double-spend, TOCTOU, async-job races), and timing-attack detection (user enumeration, blind injection via response-time deltas). Requires coordinating a load-test window with stakeholders before any volumetric run.
---

# Application Performance & Timing Testing

Black-box testing of an application's performance, concurrency, and timing behavior — no source access needed, just the ability to send requests and measure what comes back. This splits into two very different risk profiles: **load testing** (throughput, rate limits) which can degrade a live system and needs coordination, and **timing analysis** (enumeration, blind injection, races) which is low-traffic and safe to run any time.

## When to use

- Establishing a latency baseline before or after a change, to catch regressions.
- Verifying rate limits actually trigger at their documented threshold, and can't be trivially bypassed.
- Suspecting a race condition in any flow that debits a balance, issues a single-use resource, or revokes a credential.
- Suspecting a timing side-channel: does the response take measurably longer for a valid-but-wrong input than an invalid one (enumeration), or for an injected payload that reaches a real interpreter (blind injection)?

## Core pattern

**Baseline before judging anything "slow."** Send a batch of serial (not concurrent) requests per endpoint, discard the first as warm-up, and record mean and p99. A high mean means average requests are slow; a fast p50 with a slow p99 points at GC pauses or connection-pool exhaustion under normal use rather than a systemic issue. Re-run the same baseline after any change you're evaluating — a single unbaselined measurement tells you nothing.

**Verify rate limits at the threshold, not just "eventually."** Fire requests until the limit should trigger and confirm the exact attempt number matches the documented threshold, that the throttling response carries a sane `Retry-After` (present, positive, not absurd), and that a burst well past the threshold never leaks through. Then attempt a bypass: rate limits are frequently keyed off a client-supplied header (`X-Forwarded-For`, a session id) — rotate that value across requests and see whether the limit resets each time. A limit enforced at an edge/proxy layer may not hold if you can reach the origin directly.

**Race conditions need concurrency, not sequence.** A serial loop of N requests never finds a TOCTOU bug — you need genuinely concurrent fire against your own resources. The canonical shape: put a resource into a state where exactly one action should succeed (a balance with credit for exactly one purchase, a single-use token, a job that should only ever be scheduled once), fire a small burst (2–10) of the competing action concurrently, then check whether more than one succeeded. Apply the same harness to session/credential revocation (does a token stop working exactly once, or can it be used again in the revoke-vs-use race window?) and to async job creation (fire N concurrent "create" calls, then check how many were actually created — a mismatch is a concurrency bug in the scheduler).

**Timing attacks reuse one harness: baseline, then payload, then compare.** Measure a clean/baseline request time over enough repetitions to get a stable mean and stdev, then measure the suspect input the same way, and flag a difference beyond a fixed threshold (tens of milliseconds is usually enough signal, well above typical jitter). This same shape catches two distinct bugs: **user enumeration** (a login or "forgot password" endpoint that takes measurably longer — or shorter — for a registered identifier than an unregistered one, even though both return an identical error body) and **blind command/SSRF injection** (a payload like a shell sleep or an SSRF target that never returns output but adds exactly its delay to the response time). Keep the payload's delay well above your measurement noise floor and set the pass/fail threshold below it.

## Discipline

- **Coordinate a load-test window with stakeholders before any volumetric run.** Being authorized to test security or functionality is not the same as being authorized to run sustained load — confirm a window explicitly before running any load-generation tool at a meaningful rate.
- Timing-attack probes are serial and low-traffic by nature — safe to run without special coordination.
- Race-condition bursts stay small (single digits to low tens of concurrent requests) and target only your own test resources, never shared or another tenant's.
- Never direct load at operations that could affect other tenants (bulk admin actions, shared queues) even during a coordinated window.
- Volumetric denial-of-service is out of scope full stop — "proving it's possible" is not a justification.

## Pitfalls

- Treating a single unbaselined measurement as evidence of a regression — always compare against a recorded baseline.
- Testing "concurrency" with a sequential loop — it will never reproduce a genuine race.
- Rate-limit testing that stops at "did I eventually get throttled" without checking the exact threshold or attempting a header-rotation bypass.
- Setting a timing-attack threshold below normal network/GC jitter, producing false positives on every run.
- Assuming a security-testing engagement implicitly authorizes volumetric load — it doesn't; ask.
- Skipping the "identical error body" check on enumeration-prone endpoints — a timing leak is just as real when the *content* looks identical.

## Checklist

- [ ] Serial baseline recorded (mean + p99) per endpoint before judging anything slow
- [ ] Rate limit triggers at the documented threshold with a sane `Retry-After`
- [ ] Rate-limit bypass attempted via spoofable-header rotation
- [ ] Race-condition harness fires genuine concurrent bursts (2–10+) against single-success resources
- [ ] Concurrent-revocation and async-job-count races checked, not just balance double-spend
- [ ] Timing-attack harness: baseline + payload, fixed delta threshold above measurement noise
- [ ] Load-test window explicitly coordinated with stakeholders before any volumetric run
- [ ] No load or race testing directed at shared/other-tenant resources
