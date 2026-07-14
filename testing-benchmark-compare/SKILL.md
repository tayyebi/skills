---
name: testing-benchmark-compare
description: Use when verifying that pricing/plan tiers actually deliver proportional performance — provisions one instance per tier, runs the same in-instance benchmark (testing-benchmark) on all of them in parallel, and produces a comparative table that surfaces non-linear scaling, tiers sharing the same underlying hardware, and poor price-to-performance ratios. Requires cleanup of every provisioned instance immediately after the run.
---

# Cross-Tier Benchmark Comparison

Provision one instance per pricing tier, run the same benchmark on each, and compare. The question isn't "is this instance fast" (that's `testing-benchmark`) — it's "does paying more actually get proportionally more," and "do any two tiers turn out to be the same hardware with a different price tag."

## When to use

- After a new tier launches, to confirm its advertised specs are real and distinct from neighboring tiers.
- When a customer reports a paid tier performing the same as a cheaper one.
- As a periodic capacity/oversell audit across the tier lineup.

## Core pattern

**Provision every tier under comparison in parallel**, not sequentially — pick a representative set (cheapest, a couple of mid tiers, most expensive) if provisioning every tier is too costly or slow; the cheapest/mid/priciest spread already reveals most non-linearity. Run them concurrently so time-of-day variance doesn't get conflated with a real tier difference, and so one instance's provisioning failure doesn't stall the whole sweep.

**Run the identical benchmark (see `testing-benchmark`) against every instance**, holding everything except the tier itself constant: same OS image, same benchmark parameters, overlapping run windows. Any variable you don't pin becomes a confound in the comparison.

**Check proportionality, not exact multiples.** If tier B advertises 2× the vCPU of tier A, its CPU score should be *close to* 2× — not necessarily identical, since virtualization and scheduling overhead cost some efficiency. Pick a realistic floor (for example, the higher tier should deliver at least 75% of what the resource ratio predicts) and flag anything below it as non-linear scaling.

**Flag any dimension where a pricier tier scores *no better or worse* than a cheaper one.** A resource that doesn't move at all across tiers (identical disk throughput on every plan, say) usually means that dimension isn't actually tier-isolated — it's a shared backend the pricing model doesn't reflect. Likewise flag a pricier tier that scores *lower* than a cheaper one on any dimension — that's a placement or oversell bug, not noise.

**Compute a price-efficiency figure** (benchmark score per unit price) per tier and look at the trend across tiers — a sharp drop at the high end means customers are paying more per unit of delivered performance, which is worth surfacing even without a hard failure.

**Clean up every provisioned instance immediately after the run.** This test multiplies cost by the number of tiers under comparison and running them all simultaneously.

## Design decisions

- Fair comparison requires pinning every variable except the one under test (tier) — same image, same benchmark config, same rough time window.
- Parallel execution should tolerate partial failure: if one tier's instance fails to provision or a benchmark step errors, collect what you can from the rest rather than aborting the whole comparison.
- A small, deliberately chosen subset of tiers (not every tier) is usually enough to catch non-linear scaling and is far cheaper to run repeatedly.

## Pitfalls

- Running tiers at different times of day — conflates time-of-day noise with genuine tier differences.
- Treating "not exactly 2×" as a failure — set a realistic proportionality floor instead of demanding a perfect multiple.
- Forgetting cleanup — N tiers running concurrently multiplies the cost of every run.
- Accidentally varying OS image, region, or benchmark parameters alongside the tier, so the comparison is no longer isolating price/tier as the only difference.
- Reporting "identical scores across tiers" as automatically damning without first confirming it's the resource itself (not a benchmark ceiling, like a capped download target) that's flat.

## Checklist

- [ ] Representative tier subset (or all tiers) selected and provisioned in parallel
- [ ] Same OS image, benchmark config, and run window across every tier
- [ ] Same benchmark (`testing-benchmark`) run per instance; partial failures collected, not fatal
- [ ] Proportionality checked against a realistic floor, not an exact multiple
- [ ] Any dimension flat across tiers, or worse on a pricier tier, flagged explicitly
- [ ] Price-efficiency trend computed across tiers
- [ ] All provisioned instances deleted immediately after the run
