---
name: testing-chaos
description: Use when testing how a system behaves under failure, interruption, and partial-state conditions — dependency outages, interrupted async operations, session expiry mid-flow, malformed/truncated requests, invalid state-machine transitions, orphaned resources after a crash, and concurrent-modification races. Verifies every failure is recoverable, clearly communicated, and free of stuck/zombie state. Black-box: failures are induced from the outside via timing, interruption, and deliberate bad input — distinct from security testing.
---

# Chaos & Resilience Testing

Deliberately induce failure, interruption, and partial-state conditions against a system you're responsible for, and verify it degrades safely: every failure ends in a terminal state, is clearly communicated to the caller, and never leaves an orphaned or stuck resource. This is a resilience axis, not a security one — the goal is to break things by timing and bad state, not by exploiting a vulnerability.

## When to use

- Before shipping any long-running/async operation (provisioning, batch jobs, multi-step workflows) that reports progress via polling or webhooks.
- Any flow with more than one side effect that must stay consistent (debit a balance AND create a record; both happen or neither does).
- Coordinate with stakeholders before running chaos tests against a shared or production environment — even non-destructive interruption testing can affect other users if it touches shared capacity or triggers real side effects (charges, provisioning, notifications). Prefer a staging environment when the blast radius is uncertain.

## Core pattern

**Chaos testing reveals these failure modes at the black-box level:**

| Failure mode | Observable symptom | How to trigger |
|---|---|---|
| Async op fails mid-way | Resource stuck in a non-terminal state forever | Interrupt, then watch status |
| Orphaned state | A staged/cart-like record persists after a crash | Interrupt mid-transaction |
| Duplicate operation | Two side effects from one logical action | Double-submit (race) |
| Stale client state | UI/cache shows a status that no longer matches the server | Mutate via API, don't refresh caller |
| Missing error propagation | Caller hangs or shows nothing when the server call failed | Force an API failure, watch the caller |
| Session/context expiry mid-flow | A multi-step operation fails silently partway through | Let the session expire, then continue the flow |
| Partial multi-step commit | Step 1 succeeded, step 2 (dependent) didn't | Interrupt between the two |

**1. Interrupt async operations mid-flight.** Start a long-running operation (anything that responds 202-and-poll), then either abandon it or immediately fire a second, conflicting operation against the same resource. A conflicting concurrent action should be rejected (409) or queued — not accepted twice to fight over the same resource. Poll status to completion or timeout: a status that never reaches a terminal value (success/error) is a stuck-state bug regardless of what caused it. Separately, submit an operation with intentionally invalid input and confirm it fails fast and synchronously (e.g. 400) rather than being accepted and left to die asynchronously.

**2. Multi-step transaction chaos.** For any flow with two or more side effects that should be atomic (e.g., debit a balance and create the corresponding resource), interrupt between the steps — kill the network client-side right after the first side effect would have landed — and verify the system ends up either fully committed or fully rolled back, never half-done. Race the same flow concurrently against itself (multiple simultaneous submissions competing for one limited resource, like an account balance) and assert only the invariant-respecting number succeed.

**3. Session/context expiry mid-flow.** Revoke or expire the session, then attempt to continue using it — every subsequent call should fail cleanly (401-equivalent), and a caller mid-workflow should get a clear, actionable failure rather than a silent hang or a blank state. Multi-step wizards that persist partial server-side state are worth extra attention: does expiry mid-wizard leave orphaned partial state behind?

**4. Malformed and truncated input.** Send deliberately malformed or truncated payloads (cut off mid-object, trailing commas, wrong types) to mutation endpoints and confirm a clean 4xx with a parse error — never a 500, an internal stack trace, or (worse) a request that's partially applied before the parse failure.

**5. State-machine violations.** Enumerate the resource's valid states and attempt every invalid transition (stop-while-stopped, start-while-already-started, delete-while-mid-operation). Each invalid transition should be rejected (409/400), not silently accepted and queued — an accepted invalid transition either does nothing (confusing) or corrupts state (dangerous). Confirm the status field itself updates correctly after a *valid* transition; a transition that returns success but never updates the visible status is its own bug class.

**6. Resource deletion and orphan cleanup.** Delete a resource that has dependent children (sub-resources, attached configuration) and confirm the children are either cascade-deleted or the deletion is blocked (409) pending explicit cleanup — never left dangling and reachable after the parent is gone.

**7. Concurrent modification.** Fire several concurrent writes to the same field of the same resource and confirm the system converges to exactly one of the submitted values with no 500/constraint-violation — last-write-wins is an acceptable outcome, a crash is not.

**8. Recovery verification.** After every induced failure above, confirm the system actually recovers: is there a way to retry or reset a stuck operation, does the resource eventually reach a terminal state, and can the caller successfully perform another action on it afterward? A failure that's "handled" but leaves the resource permanently unusable is not actually handled.

## Pitfalls

- Treating "the request returned 200" as success without polling to a terminal status — async operations lie at the surface and tell the truth only in their eventual state.
- Not checking both directions of a transaction (side effect A happened but B didn't, and vice versa) — testing only the happy interruption point.
- Running heavy concurrency/race tests against shared production capacity without warning stakeholders.
- Accepting an invalid state transition that returns success without checking whether the underlying status actually changed — a stale-status bug is easy to miss if you only check the HTTP code.
- Stopping at "it failed as expected" without verifying the system can still make forward progress afterward.

## Checklist

- [ ] Conflicting concurrent operations on one resource → rejected/queued, not both accepted
- [ ] Every async operation reaches a terminal state or is verifiably stuck (flagged as a bug)
- [ ] Multi-step/multi-side-effect flows interrupted mid-way → fully committed or fully rolled back
- [ ] Concurrent race against a shared limited resource → only the invariant-respecting count succeeds
- [ ] Expired/revoked session mid-flow → clean failure, no orphaned partial state
- [ ] Malformed/truncated payloads → clean 4xx, never 500 or partial apply
- [ ] Every invalid state transition rejected; every valid transition's status field actually updates
- [ ] Deleting a parent resource cascades or blocks — no reachable orphans
- [ ] Concurrent writes to one field converge to one consistent value, no crash
- [ ] Post-failure recovery verified: resource reaches terminal state, further actions succeed
