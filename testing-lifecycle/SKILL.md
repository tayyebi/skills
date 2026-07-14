---
name: testing-lifecycle
description: Use when testing a provisioned, stateful resource (VM, container, database instance, k8s pod) through its full state machine — provision, power/lifecycle transitions, child-resource operations (snapshots/backups), and deletion. Covers poll-and-diff verification against a timeout, format-checking identifiers and timestamps, stuck-state and orphan detection, invalid-transition (409) checks, and correlating billing/metering with actual reachable state. The flagship pattern other resource-specific testing skills (SSH, console, network, catalog coverage) plug into.
---

# Lifecycle & State-Machine Testing

Verify a provisioned resource by treating it as an explicit state machine: every transition is triggered, timed, and polled to a terminal state; every response is format-checked, not just glanced at; and state is snapshotted before and after each operation so the diff — not your memory of what should have changed — is the evidence. The goal is to catch the failure modes a single happy-path smoke test misses: stuck states, transitions that return "accepted" but never complete, invalid transitions that silently succeed instead of being rejected, orphaned child resources, and a resource that keeps being billed after it stops working.

## When to use

- Testing any create → operate → destroy lifecycle for a stateful cloud/infra resource: VM/VPS, container, managed database, k8s pod, queue, or similar.
- Whenever "the API returned 202" is being trusted as proof that an asynchronous operation actually finished.
- Pairs with `testing-platform-coverage` (does every catalog variant provision correctly), `testing-ssh` and `testing-vnc` (does the provisioned resource's access paths actually work), `testing-network` (is it reachable), and `testing-billing` (does metering track real state).

## Core pattern

**Define the state machine explicitly.** Write down every state (queued, provisioning, running, stopped, rebooting, snapshotting, error, deleted...) and every legal transition between them before testing. Anything not on the list — a transition that "shouldn't" be possible — is exactly what you probe for next.

```
queued → provisioning → running ⇄ stopped
                                → rebooting → running
                                → snapshotting → running
                                → error (terminal)
```

**Poll to terminal, don't trust the mutation response.** A `POST /power-off` returning 200/202 only means the request was accepted, not that the resource is stopped. Poll the resource's own status field on an interval, log each observation with elapsed time, and stop on either the target state or a terminal failure state (e.g. `error`) — treating an error state as terminal lets a test fail in seconds instead of spinning out the full timeout. Bound every poll with a timeout and fail loudly on expiry; an unbounded poll just hangs the suite the first time a transition breaks.

**Snapshot and diff, don't eyeball.** Before an operation, capture a normalized projection of the resource (status, IP, name, updated-at — not the whole noisy payload) and capture it again after the operation resolves. Diff the two. This makes "did the status actually change" a mechanical check instead of a human reading two JSON blobs and missing the field that didn't move.

**Format-check every value, not just its presence.** IDs should match the identifier scheme the API actually uses (UUID vs. slug — don't assume); timestamps should parse as the documented format; assigned addresses should match the expected shape. A field that's merely non-null can still be garbage.

**Probe invalid transitions deliberately.** Fire the same action twice in a row (power-off while already stopped, delete while already deleted) and confirm the second call is rejected (409/400), not silently accepted. A state machine that accepts illegal transitions will eventually land a resource in an inconsistent state under real concurrent usage.

**Test concurrent operations.** Fire two conflicting mutations at the same resource near-simultaneously (e.g. snapshot and reboot). Exactly one should be accepted; the other should be rejected with a conflict status. Afterward, poll to confirm the resource still converges to a terminal state — a race that leaves it stuck is worse than the race itself.

**Exercise child-resource lifecycles the same way.** For a resource that owns child objects (snapshots, backups, volumes): create one, poll it to its own ready state, confirm it appears in its list endpoint, and confirm it's deletable in any state — including a stuck non-terminal one. A management API must always accept a delete request even when the underlying operation is wedged.

**Check for orphans on deletion.** Before deleting the parent, snapshot its child-resource count and IDs. After deletion, every child resource must also become inaccessible (404) — a child left dangling after its parent is gone is capacity or billing quietly leaking. Also re-check the parent's list endpoint for one fewer entry, and re-check the count-consistency invariant on any paginated list (`total` should never be less than the items actually returned).

**Correlate billing/metering with actual state.** The single highest-value cross-cutting check: a resource that is unreachable, stopped, or deleted must stop accruing usage charges. Snapshot the metering ledger for the resource, wait, snapshot again — no new entries should appear once the resource is in a non-billable state. Do this at both "unreachable while supposedly running" and "confirmed deleted" checkpoints.

## Pitfalls

- **Trusting the mutation's HTTP status as completion.** 202 means "accepted," not "done" — always poll to a terminal state.
- **Unbounded polling.** No timeout turns one broken transition into a hung test suite; bound every poll and fail with the last-observed state.
- **Not testing invalid transitions.** Only ever exercising the happy path never proves the guard rails exist.
- **Deleting a parent without checking for orphaned children.** The most common source of unbilled-but-still-running (or billed-but-gone) resources.
- **No post-delete billing check.** Deletion succeeding at the API layer doesn't prove metering actually stopped.
- **Sequential-only testing.** Concurrency bugs (double-accepted conflicting operations, stuck-after-race states) are invisible unless you deliberately race two operations.
- **Format-checking only on the happy path's final state**, and skipping it on intermediate/error responses where malformed data is more likely to leak through.

## Checklist

- [ ] State machine and legal transitions documented up front
- [ ] Every async mutation polled to a terminal state with a bounded timeout, treating error/failed as terminal
- [ ] Before/after snapshots diffed for each transition (normalized projection, not raw payload)
- [ ] Identifiers, timestamps, and addresses format-checked, not just presence-checked
- [ ] Invalid transitions (repeat/no-op action) rejected with 409/400
- [ ] Concurrent conflicting operations: exactly one accepted, resource still converges afterward
- [ ] Child-resource lifecycle (create → ready → list membership → delete-in-any-state → 404) exercised
- [ ] Orphan check: child resources and list-count consistency verified after parent deletion
- [ ] Billing/metering checked to stop when the resource is unreachable, stopped, or deleted
