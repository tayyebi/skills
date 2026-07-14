---
name: rolling-fleet-deploy
description: Use when deploying the same app across a matrix of hosts and you must keep the cluster serving through the rollout — a staggered rolling deploy (each host waits an offset before recycling so quorum never drops at once), a hard post-deploy convergence gate that polls every node until they see each other, and pre-flight CI-variable validation. Also covers the discipline of onboarding a new host as a small, well-defined config change.
---

# Rolling Fleet Deploy

Deploy to many hosts without a fleet-wide flicker. Each host is recycled on a **stagger** so the previous one is back up before the next goes down; after the rollout a **convergence gate** blocks the pipeline until every node reports a healthy cluster; and a pre-flight check fails fast on missing config before any host is touched.

## When to use

- Rolling a new build across N hosts (especially an active-active cluster — see `masterless-active-active-replication`) that must stay available throughout.
- Any multi-host deploy where "all hosts recycle at the same instant" would drop quorum or blank the service.

Sits on top of `host-auto-provisioning` and `build-once-deploy-everywhere`; sequences their per-host work across the matrix.

## Core pattern

**Host matrix, single source of truth.** Define hosts once (`SERVER_ID: [de, us, au, uk]`) and drive every per-host stage (`provision`, `ship`, `migrate`, `deploy`) from it. Onboarding a host is a small, fixed change — e.g. append the id to the matrix **and** to the `load-vars` case block **and** the validation list. Document that "adding an id in one place is not enough"; keep the lists in sync or stages fail with "unknown server id".

**Validate before touching hosts.** A `validate` stage runs first: every host has an address and an SSH credential; the cluster secret is set when >1 host is configured; the registry is enabled. A missing variable becomes an immediate, clear error instead of a cryptic SSH timeout deep in a later stage.

**Stagger the recycle.** Each host waits `DEPLOY_OFFSET_SECONDS` before its `compose up` (e.g. 0 / 180 / 360 / 540 — a few minutes apart). While a host waits, the previously-deployed host has already come back and is serving. Without the stagger the whole matrix hits its downtime window simultaneously.

**Flat stage barriers, not per-cell DAG.** Every cell of `ship` waits for the *entire* `provision` matrix, `migrate` for all of `ship`, etc. A single host failing a stage halts the next stage for the whole fleet — deliberately. A partial-fleet, mixed-version deploy is worse than not deploying.

**Post-deploy convergence gate.** After deploy, actively trigger each node to re-join peers (best-effort), then **poll every node's status endpoint until each sees ≥ N-1 peers** or a timeout fires. This is the authoritative pass/fail — the best-effort join before it never gates.

## Design decisions

- **Migrations are per-host and re-entrant.** Each node has its own database, so run migrations independently per host; make the runner advisory-locked/idempotent so parallel runs across hosts are safe.
- **Interruptible jobs + auto-cancel on new commit.** If a newer commit lands mid-rollout, cancel the in-flight pipeline so stale code doesn't keep rolling. The stagger and the convergence gate absorb the resulting partial-update window, and the fresher pipeline redeploys the right code.
- **The convergence gate polls in parallel and tolerates lag.** Fire all node status requests concurrently each round, allow a grace period for gossip to propagate, and only pass when *all* nodes are healthy *and* every peer count has caught up. Fail the pipeline on timeout — a silent under-converged cluster is the outage you're trying to prevent.
- **Best-effort join, authoritative verify.** Split "nudge nodes to re-register" (exits 0 even if some don't answer) from "prove the cluster converged" (hard gate). A transient hiccup during the nudge shouldn't abort a deploy that actually succeeded.
- **Single-node degrades gracefully.** With one host configured, skip the cluster secret requirement, the stagger, and the convergence gate — the same pipeline serves a standalone install.

## Pitfalls

- **No stagger** → every host recycles at once → full-fleet downtime during every deploy.
- **Per-cell DAG across stages** → a host that skipped a stage proceeds while others didn't → mixed-version cluster. Use flat barriers.
- **Trusting the best-effort join as the gate** → the pipeline goes green while nodes never actually found each other. Poll status until converged.
- **Adding a host id in one place only** → "unknown server id" mid-pipeline. Change all the coupled lists together; a `check-deprecated`/lint step helps.
- **No pre-flight validation** → a missing SSH key surfaces as a 3-minute SSH timeout in the ship stage instead of an instant "variable X is unset."
- **Convergence gate with no timeout** → a broken node hangs the pipeline forever; bound it and fail loudly.

## Checklist

- [ ] Host matrix as the single source; per-host stages driven from it; onboarding = coupled multi-list change (documented)
- [ ] `validate` stage fails fast on missing address/credential/cluster-secret/registry
- [ ] Per-host recycle stagger (`DEPLOY_OFFSET_SECONDS`)
- [ ] Flat stage barriers; one host's failure halts the next stage fleet-wide
- [ ] Per-host, re-entrant migrations
- [ ] Best-effort re-join, then a hard convergence gate polling until every node sees ≥ N-1 peers, with a timeout
- [ ] Interruptible + auto-cancel on new commit
- [ ] Single-node path skips multi-node requirements
