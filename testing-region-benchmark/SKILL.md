---
name: testing-region-benchmark
description: Use when verifying that performance is consistent (or fairly explained by geography) across every region/data-center location a platform offers — provisions one instance per region on an identical plan and OS, runs the same in-instance benchmark (testing-benchmark) on all of them in parallel, adds per-region latency-to-anchor and boot-time measurements, and produces a ranked comparison table. Structurally the same sweep-and-compare mechanics as testing-benchmark-compare, but the axis under test is location, not price tier.
---

# Cross-Region Benchmark Comparison

Provision one instance per region, benchmark each identically, and rank the results. This is the same provision-in-parallel / run-the-same-benchmark / compare mechanics as `testing-benchmark-compare` — see that skill for the shared discipline (variable pinning, partial-failure tolerance, cleanup cost). The difference is what's being held constant and what's varying: here the plan and OS are pinned identical and **region** is the only variable, so any performance spread is a location effect, not a pricing effect.

## When to use

- Verifying per-region network/latency claims before advertising a new location.
- Investigating a "region X feels slow" report.
- A periodic consistency audit as the region footprint grows.

## Core pattern

**Pin everything except location.** Provision one instance per region (or a specified subset) using the exact same plan tier and OS image everywhere — location is the only variable under test, same as `testing-benchmark-compare` pins everything but tier.

**Run the same benchmark suite (`testing-benchmark`) against every region in parallel**, and add the one measurement that's specific to a region comparison: **latency to a small set of well-known anchor hosts**, measured from inside each instance. Throughput and disk/CPU numbers tell you whether the hardware is comparable; latency tells you whether the network path is doing what geography would predict.

**Track provisioning/boot time per region too.** Infrastructure maturity varies by location, and a region that's slow to bring an instance up before it's even reachable is its own finding, independent of in-instance performance.

**Rank, don't just pass/fail.** Customers pick a region by relative performance, not an absolute threshold — present results as a ranked table (fastest to slowest per metric) alongside any hard pass/fail checks, and call out the spread between best and worst explicitly.

**Sanity-check surprises before calling them regressions.** A large throughput or latency spread across regions isn't automatically a platform bug — it can be legitimate (different upstream transit providers, genuine physical distance to whatever target the benchmark measures against). Cross-check against a second network target and against the latency numbers before concluding a region's *platform* is under-performing versus its *geography* explaining the gap.

## Pitfalls

- Everything covered in `testing-benchmark-compare`'s pitfalls (time-of-day skew, forgetting cleanup, partial-failure handling) applies here too.
- Attributing a throughput gap to the platform when it's actually explained by the benchmark's own target server being closer to some regions than others.
- Comparing regions provisioned on different plan tiers or OS images by accident, which reintroduces the tier-comparison confound this skill is specifically trying to exclude.
- Skipping the boot-time measurement and missing a region whose provisioning pipeline itself is slow.

## Checklist

- [ ] Same plan tier and OS image pinned across every region under comparison
- [ ] Same benchmark (`testing-benchmark`) run per region, in parallel, with partial failures tolerated
- [ ] Latency-to-anchor measured from inside each instance
- [ ] Provisioning/boot time captured per region
- [ ] Results presented as a ranked table, not just pass/fail
- [ ] Outlier throughput/latency cross-checked against a second target before calling it a platform regression
- [ ] All provisioned instances deleted immediately after the run
