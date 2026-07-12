---
name: leader-election-via-shared-config
description: Use when exactly one node in a cluster must run a singleton task (a sync job, an external poller, a scheduled report) but you don't want a full consensus protocol. Elect the leader by naming it in a replicated config value; every node re-reads it each tick and only the named node acts. Simple, operator-controllable, and safe for idempotent/self-correcting work — not for tasks where a brief two-leader overlap is catastrophic.
---

# Leader Election via Shared Config

Designate the single node allowed to run a piece of singleton work by writing its instance id into a **replicated setting**. Every node runs the same scheduler, but on each tick it reads the setting and no-ops unless the setting names *it*. Flipping the leader is a config change that gossips to the fleet — no Raft, no ZooKeeper, no lease renewals.

## When to use

- One-of-N singleton jobs in a masterless cluster: pulling from an upstream source, sending a digest, running a cleanup — work that would double-execute or race if every node did it.
- When you already have a replicated config/settings blob (see `masterless-active-active-replication`) and an operator who can pick which node should own the work.
- When the work is **idempotent or self-correcting**, so a rare overlap during a leader change is harmless.

Do **not** use it when a momentary two-leader window is unacceptable (money movement, exclusive locks). That needs fencing tokens or a real consensus/lease system.

## Core pattern

Store the elected instance in a replicated config value; default to "nobody":

```
settings.cluster.singleton_task_instance_id = null   -- safe default: no one runs it
```

Gate the scheduled job on it, re-reading every tick so a change takes effect within one interval:

```
every 60s:
  if isRunning: return                          // no overlapping runs on the leader itself
  leader = settings.cluster.singleton_task_instance_id
  if leader is null or leader != myInstanceId: return
  isRunning = true
  try { runTheSingletonWork() } finally { isRunning = false }
```

Expose a small admin API to read the current leader + the list of candidate instance ids (peers + self), and to set it. Validate the input against the known instance ids.

## Design decisions

- **Default to null = nobody runs it.** On a fresh or newly-scaled cluster, auto-picking a leader risks two nodes both deciding they're it before config converges — they'd race, double-create, and thrash LWW. "Off until an operator elects one" is the safe resting state.
- **Re-read every tick, don't cache the decision.** Leadership is soft: when the setting flips, the old leader stops within one interval (plus one gossip cycle to receive the change) and the new one starts. No handoff protocol needed.
- **Guard against overlapping runs** on the leader with a local `isRunning` flag — a slow run must not stack on the next tick.
- **Allowlist config keys** written through the API, so the gossip-replicated config blob can't accumulate arbitrary keys from a malformed request.
- **Manual election is a feature, not a limitation.** An operator choosing the region closest to the upstream source, or steering work off a degraded node, is often exactly what you want. Pair with `cluster-membership-gossip` to populate the candidate picker.

## Pitfalls

- **Auto-pickup across N nodes** (e.g. "lowest instance id wins", computed independently per node) races during membership changes and config lag — two nodes act at once. Make election explicit and single-valued.
- **This is not fencing.** During a leader change both the old and new leader can briefly run. Only acceptable because the work is idempotent — verify that assumption before using this pattern.
- **Caching the leader value** for the process lifetime means flipping it requires a redeploy. Re-read each tick.
- **No `isRunning` guard** → a run longer than the interval overlaps itself on the leader.
- **Config not actually replicated** → each node has a different idea of who the leader is, and you get zero or many leaders. The setting must be part of the converging state.

## Checklist

- [ ] Leader stored as an instance id in a **replicated** config value; default null
- [ ] Scheduler re-reads it every tick; no-op unless it names this node
- [ ] Local `isRunning` guard against overlapping runs
- [ ] Admin API to view candidates + current leader and to set it, input-validated
- [ ] Used only for idempotent/self-correcting singleton work (documented as such)
