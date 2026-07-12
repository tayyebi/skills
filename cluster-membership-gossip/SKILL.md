---
name: cluster-membership-gossip
description: Use when nodes must discover each other, track who is online, and let members join and leave without editing config on every box — a peer table bootstrapped from a seed list, periodic health checks with online/suspect/offline states, a shared-secret authenticated join/leave handshake, and stale-peer reconciliation. The membership layer that a masterless cluster sits on top of.
---

# Cluster Membership & Gossip

Let a fleet of identical nodes form a mesh on their own: a new node bootstraps from a **seed list**, registers itself with peers, and hydrates its local peer table from their view. A health loop keeps liveness current, and a shared secret authenticates all peer-to-peer calls. No node is special; membership survives any single node going away.

## When to use

- Any masterless / active-active cluster (see `masterless-active-active-replication`) that needs to know "who are my peers and which are up" to gossip with.
- Fleets where hosts are added/removed by CI or autoscaling and you don't want to hand-edit a static peer list on every box.

## Core pattern

**Peer table** (local to each node): `peers(id, instance_id, url, region, status, last_seen_at, first_seen_hlc, decommissioned_at, error_message)`. Unique on `instance_id`, with `url` as a fallback key.

**Seed bootstrap on startup.** From `CLUSTER_SEEDS` (a comma-separated URL list, minus self), POST an identity to each seed until one ACKs:

```
POST /cluster/join { instance_id, url, region, first_seen_hlc }
  → { peer_list[], hlc_high_water, self_instance_id }
```

Then **hydrate** the local peer table from `peer_list`. Retry with a backoff ladder (e.g. 1s, 2s, 5s, 10s×3 ≈ 38s total, each round trying every seed) so a rolling deploy where a seed isn't up yet still converges. If all seeds fail, continue standalone — don't crash.

**Join handshake is idempotent and an upsert.** Look up the caller by `instance_id`, then fall back to `url` — this handles both "same node, new IP" and "re-provisioned node, same IP, new instance_id" without hitting the unique constraint. Gossip the join (append a `peer` upsert to the replication log) so *every* node learns the new member, not just the seed.

**Health loop** (~60s): GET each peer's `/cluster/health`, update `status` + `last_seen_at`. Map success→`online`, failure→`offline` (optionally `online → suspect → offline` to avoid flapping on one blip). A peer is "live" for gossip if it's online/suspect, not decommissioned, and `last_seen_at` is within a heartbeat window.

**Leave** is best-effort: POST `/cluster/leave` to known peers on shutdown, mark `decommissioned_at`, gossip a tombstone. Peers that miss it will time out the node via the health loop anyway.

## Design decisions

- **Shared-secret peer auth.** Every peer-facing endpoint checks a constant `X-Cluster-Secret` header against `CLUSTER_SECRET`; reject if unset or mismatched. Simple, symmetric, good enough for a trusted fleet behind TLS. Require it whenever more than one node is configured — fail fast at deploy time if it's missing.
- **Health checks run sequentially, not fanned-out.** Each check ends in a DB write; firing N concurrent checks can grab N pool connections at once and, coinciding with a replication burst, exhaust the pool. Background liveness is cheap — one at a time is fine.
- **Reconcile stale peers after a seed join.** Decommission local peer rows whose `instance_id` isn't in the seed's authoritative list — leftovers from replaced servers or changed instance ids. Only act on rows that *have* an instance_id; leave unknown ones to the health loop.
- **`instance_id` is the stable identity, not the URL.** IPs and hostnames change; the logical instance id shouldn't. Address peers by URL but *identify* them by instance_id.
- **Post-deploy active join.** After a rolling deploy, have CI hit each node's `/cluster/self-join` so nodes that booted before their seeds were reachable re-register. Then gate the pipeline on a convergence check (see `rolling-fleet-deploy`).

## Pitfalls

- **Concurrent health-check fan-out** exhausting the connection pool → unrelated requests fail with connection timeouts. Serialize the loop.
- **Identifying peers by URL only** → a re-IP'd or re-provisioned node either duplicates or collides on the unique constraint (500 on join). Two-step lookup: instance_id, then url.
- **Crashing when seeds are unreachable** → a node that boots first in a cold start never comes up. Back off, then run standalone and let gossip catch it up later.
- **No stale reconciliation** → dead nodes linger in every peer's table, get gossiped to, and skew "online peers" counts.
- **Unauthenticated peer endpoints** → anyone who can reach the node can inject peers or read the change log. Gate all of them on the shared secret.
- **Flapping status** on a single failed probe → use a suspect state or a `last_seen_at` window before declaring a peer dead.

## Checklist

- [ ] Local `peers` table keyed on `instance_id`, `url` fallback
- [ ] Seed bootstrap with a backoff ladder; standalone if all seeds fail
- [ ] Idempotent `join` upsert (instance_id → url lookup) that gossips the new member
- [ ] `X-Cluster-Secret` on every peer endpoint; required for multi-node, checked at deploy
- [ ] Sequential health loop → online/suspect/offline + `last_seen_at`
- [ ] "Live peer" = online/suspect ∧ not decommissioned ∧ within heartbeat window
- [ ] Stale-peer reconciliation against the seed's authoritative list
- [ ] Best-effort leave + tombstone on shutdown
