---
name: ci-cd-pipeline-discipline
description: Use when setting up or auditing a CI/CD pipeline — ordered lint/test/build gates, branch strategy, required checks, and what makes a pipeline fast and trustworthy enough that people don't bypass it. Defers to build-once-deploy-everywhere and rolling-fleet-deploy for the actual multi-host rollout mechanics.
---

# CI/CD Pipeline Discipline

A pipeline is only as good as the trust engineers place in it. If it's slow, flaky, or skippable, people route around it and it stops preventing anything. Design for speed and determinism first; gates are worthless if nobody waits for them.

## When to use

Any repo with more than one contributor, or any change that reaches production without a human re-checking it by hand. Set this up before the pipeline grows organically into a pile of unordered, half-trusted scripts.

## Core pattern: staged gates

Stages run in order of increasing cost; a failure at any stage halts the pipeline before the next one starts, so the most expensive stage (deploy) only ever runs on a change that already passed everything cheaper.

```
1. Checkout + cache restore
2. Install dependencies (locked, not resolved — see below)
3. Lint / static analysis / type check      (seconds)
4. Unit tests                                (seconds–low minutes)
5. Integration tests                         (minutes)
6. Build artifact (see build-once-deploy-everywhere)
7. Deploy (see rolling-fleet-deploy)
```

- Order by signal-per-second: a lint failure is cheap to detect and cheap to fix, so put it first, not last. Don't let a 10-minute integration-test run be the first thing to catch a typo.
- Fail fast: stop the pipeline at the first red stage. Don't burn CI minutes building an artifact from code that failed unit tests.
- Run independent stages (lint, unit tests, type check) in parallel jobs, not sequentially — they share no state, so serializing them only adds wall-clock time.

## Branch strategy and required checks

- Protect the trunk branch: no direct pushes, merges only through a pull/merge request that has run the full pipeline.
- Make the pipeline a **required status check** at the platform level (not just a convention) — otherwise "merge without waiting for CI" is one click away and eventually someone takes it under deadline pressure.
- Keep feature branches short-lived. A branch that lives for weeks accumulates a merge/rebase cost that scales with its age; prefer trunk-based development with feature flags (`configuration-feature-flags`) over long-lived branches for incomplete work.
- Tag or record the exact commit SHA that produced each deployed artifact — the pipeline's output must be traceable back to the diff that produced it.

## What makes a pipeline trustworthy

- **Deterministic builds.** Install dependencies from a lockfile, not by re-resolving version ranges — the same commit must produce the same dependency tree today and a year from now. A pipeline that occasionally fails on an unrelated dependency bump erodes trust fast.
- **Fast feedback.** Target a red/green signal on a pull request in single-digit minutes for the cheap stages. Contributors context-switch away from slow pipelines and stop treating failures as urgent.
- **Hermetic runs.** Each run starts from a clean environment (fresh container/VM), not a machine with leftover state from the previous run — a "worked last time" pipeline that depends on prior-run side effects is not reproducible.
- **No flaky required checks.** A test that fails ~5% of the time for reasons unrelated to the change trains people to re-run-until-green or to bypass the gate entirely. Quarantine or fix flaky tests immediately; a flaky required check is worse than no check.
- **Secrets never touch the diff.** Injected by the CI platform's secret store at run time, scoped to the minimum stages that need them, never printed to logs.

## Environment strategy

- Distinct configuration per stage — CI, staging, production — each with its own settings, never a single config file conditionally branching on environment name inside application code.
- Staging mirrors production's topology (same datastore engine, same queue) closely enough that "worked in staging" is actually predictive; a CI-only in-memory substitute (e.g. a fake queue) is fine for unit tests but should not stand in for staging.
- Never commit real secrets; commit a template file with every required key and a placeholder value so a new environment's config surface is discoverable without reading source.

## Pitfalls

- Gates that only run on merge, not on the pull request — the whole point is to catch the problem before it lands.
- A "required" check that's actually advisory because a human can merge past a red X — enforce it at the branch-protection level.
- Rebuilding the artifact separately for staging and production ("build twice") instead of promoting one immutable build through environments — see `build-once-deploy-everywhere`.
- Long-running integration/E2E suites blocking every commit instead of running on a schedule or a merge queue, with only the fast stages required per-PR.
- Silent pipeline-config drift: pipeline definitions reviewed with less scrutiny than application code, letting someone quietly disable a check.

## Checklist

- [ ] Stages ordered cheapest-first; pipeline halts on first failure
- [ ] Independent stages run in parallel jobs
- [ ] Trunk branch protected; pipeline is a required status check, not a convention
- [ ] Dependencies installed from a lockfile — builds are deterministic
- [ ] Each run starts hermetic (clean container/VM), no cross-run state
- [ ] Flaky required checks are quarantined or fixed, not tolerated
- [ ] Secrets injected at run time by the CI platform, scoped and unlogged
- [ ] One artifact built once and promoted through environments (see `build-once-deploy-everywhere`)
