---
name: testing-benchmark
description: Use when verifying that a freshly provisioned compute instance (VM, container, bare-metal node) actually delivers the CPU, memory, disk, and network resources it advertises — covers system-fact checks against advertised specs, CPU/memory/disk-throughput/IOPS/network micro-benchmarks, disk-fill and sustained-I/O stability, a combined stress test, and a tiered scoring table. Tool availability is never assumed; degrade to pre-installed fallbacks rather than failing the run.
---

# In-Instance Resource Benchmarking

Log into a provisioned compute instance and measure whether it actually delivers the CPU cycles, memory bandwidth, disk throughput/IOPS, and network throughput its spec sheet promises. Everything runs from inside the instance, driven remotely (SSH or equivalent) — you're not trusting the provider's dashboard, you're measuring it yourself.

## When to use

- Confirming a newly provisioned instance or plan actually matches its advertised spec.
- Investigating a specific "this instance feels slow" report.
- As the reusable unit of work that a cross-tier comparison (`testing-benchmark-compare`) or cross-region comparison (`testing-region-benchmark`) runs once per instance and aggregates.

## Core pattern

**System facts before synthetic benchmarks.** Read vCPU count, total RAM, and disk size directly from the OS (`/proc`, `nproc`, filesystem size) and compare against the advertised spec. Allow reasonable overhead rather than demanding an exact match — OS and kernel reservations typically eat some RAM and disk (a common tolerance: RAM ≥ ~90% of advertised, disk ≥ ~80%), but the vCPU count should match exactly.

**Benchmark each resource with a workload that isolates it:**
- **CPU** — a compute-bound workload (prime counting, hashing, or a dedicated CPU benchmark tool) at both full thread count and single-thread, since single-thread throughput matters for serial workloads that multi-core scores hide.
- **Memory bandwidth** — a large sequential memory-copy throughput measurement.
- **Disk** — sequential read and write **and** random small-block IOPS as separate measurements; they diverge sharply, especially on network-backed or thin-provisioned storage. Use direct I/O (bypass the page cache) or the numbers measure RAM, not disk. Capture tail latency (e.g. p99 of random-read latency) alongside IOPS — a device can hit a target IOPS number while still having unacceptable tail latency.
- **Network** — outbound throughput to a stable, well-known target, plus latency (ping) to a couple of anchor hosts.

**Watch for degradation under sustained load, not just a burst.** A short benchmark can be flattered by burst credits (common on thin-provisioned storage or bursty network tiers) that throttle hard once exhausted. Run a longer sustained write and look at the lowest per-interval throughput samples, not just the average — a floor well below the average is the signature of a burst-then-throttle backend.

**Finish with a brief combined stress test.** Load CPU, memory, and disk simultaneously for a short window and confirm the instance stays responsive throughout (no OOM kill, no connection timeout). This is a stability smoke check, not a performance measurement.

**Score against a tiered threshold table**, not just a pass/fail line — a plain "minimum / expected / excellent" banding turns a raw number into an interpretable rating and makes results comparable across runs:

| Resource | Minimum | Expected | Excellent |
|---|---|---|---|
| CPU throughput | baseline for tier | 3× baseline | 6×+ baseline |
| Memory bandwidth | baseline for tier | 2.5× baseline | 5×+ baseline |
| Disk sequential | baseline for tier | 4× baseline | 10×+ baseline |
| Disk random IOPS | baseline for tier | 20× baseline | 100×+ baseline |
| Network throughput | baseline for tier | 5× baseline | 10×+ baseline |

Set the actual baseline numbers from the advertised spec of whatever tier you're testing; keep them adjustable (env vars or config), since the same benchmark logic should run unchanged against an entry-level and a high-end tier.

## Design decisions

- **Never assume a benchmarking tool is installable.** Probe for a usable package manager before attempting any install, and treat a failed/unavailable install as expected, not fatal — fall back to whatever ships with the base OS image (a hashing utility as a CPU proxy, a block-copy utility for disk/memory throughput, a plain HTTP client for network). Fallback tools are proxies, not exact substitutes for a purpose-built benchmark — note that in the results, and prefer the real tool when it's available.
- **Measure sequential and random I/O separately.** They are genuinely different workloads and a storage backend can be strong on one and weak on the other.
- **Compare measured-vs-advertised as the primary signal**, with the tiered table as a secondary sanity check — a customer's complaint is about the contract, not an abstract score.
- Bound every remote command with a timeout; a hung benchmark on one instance shouldn't hang the whole run.

## Pitfalls

- Measuring disk/network without direct I/O or with a target so small it's served from cache — inflated numbers that don't reflect real throughput.
- Skipping a warm-up request/run and letting cold-start variance skew the first measurement.
- Judging CPU only by a multi-threaded score, missing a single-thread regression that hurts serial workloads.
- Treating "the benchmark tool wouldn't install" as a benchmark failure instead of falling back to a proxy measurement.
- No sustained-load test — burst-then-throttle storage or network looks fine on a 10-second run and fails badly on a 60-second one.
- Hardcoding absolute thresholds instead of scaling them to the tier under test.

## Checklist

- [ ] vCPU / RAM / disk read from the OS and compared against advertised spec (with reasonable overhead tolerance)
- [ ] CPU measured at full thread count and single-thread
- [ ] Memory bandwidth measured with a sequential copy workload
- [ ] Disk sequential read/write and random IOPS + tail latency measured separately, with direct I/O
- [ ] Network throughput and latency-to-anchor measured
- [ ] Sustained/disk-fill run checks for burst-then-throttle behavior, not just a short average
- [ ] Combined CPU+memory+disk stress confirms the instance stays responsive
- [ ] Results scored against a tiered (minimum/expected/excellent) table, adjustable per tier
- [ ] Missing benchmark tooling degrades to a pre-installed fallback instead of failing the run
