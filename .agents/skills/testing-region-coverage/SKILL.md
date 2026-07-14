---
name: testing-region-coverage
description: Use when confirming every region/location a platform advertises is actually usable — provisions one instance per available region, waits for each to reach a ready state, and checks basic reachability (ICMP, expected port/service open, protocol banner, outbound connectivity), producing a per-region availability matrix. This is an availability check, not a performance one — complementary to testing-region-benchmark, which adds performance numbers on top of the regions this confirms are alive.
---

# Region/Location Availability Coverage

Confirm that every region or location in the catalog actually works: an instance can be provisioned there, it comes up, and it's reachable. This is deliberately narrower than a performance comparison — pass/fail per region, not a score. Run this before `testing-region-benchmark`; a region that fails coverage doesn't need to be benchmarked.

## When to use

- Validating a newly added region before it's advertised to customers.
- A periodic health sweep across a growing region footprint.
- After an infrastructure or provider change that might silently break reachability in one location without affecting others.

## Core pattern

**Validate the catalog itself first.** Every region entry should have a stable, well-formed identifier and a display name, with no duplicates — a malformed or duplicated entry is a bug worth catching before you spend time provisioning against it.

**Provision one instance per region (or a chosen subset) in parallel**, then wait for each to reach its ready/active state under an individually bounded timeout. Bound each region's wait separately — a single stuck region should degrade to one row of failures in the final matrix, not hang the entire sweep.

**Run the same reachability checks against every instance**, ideally reusing whatever your project already defines as "reachable" for a single instance (network/connectivity and remote-access skills typically own the canonical definitions) rather than redefining them here:
- ICMP reachability with an acceptable packet-loss ceiling, not necessarily zero loss.
- The expected service port open within a bounded time after "ready."
- The expected protocol banner present and well-formed.
- Outbound internet connectivity confirmed from inside the instance (a simple "what's my IP" round trip works).

**Check cross-region isolation.** Every instance should get a distinct public address; a repeated address across two different regions signals a catalog or routing misconfiguration, not a coincidence.

**Optionally confirm geographic origin, not just uniqueness.** Resolve each instance's public address through a geolocation lookup and check it actually maps to the region it was provisioned in. This catches a region that's mislabeled or silently routed through a different physical location — a bug that IP-uniqueness alone can't see.

**Produce a coverage matrix** (region × check) as the deliverable, plus a single availability percentage across the whole footprint. Clean up (delete) every provisioned instance once the sweep completes.

## Design decisions

- Coverage is binary per check — a region either provisions and answers, or it doesn't. Leave numeric performance comparison to `testing-region-benchmark`; mixing the two muddies what should be a simple availability signal.
- Bound every wait individually so one broken region can't stall the whole sweep — collect a failure for that region and move on.
- Run this ahead of any performance comparison: there's no reason to benchmark a region that doesn't reliably come up first.

## Pitfalls

- No per-region timeout — one stuck region hangs the entire sweep indefinitely.
- Treating "an address was assigned" as sufficient without confirming actual reachability (ICMP, port, banner, outbound).
- Skipping the geoIP/origin check and missing a region that's mislabeled or misrouted.
- Not checking for duplicate addresses across regions, which would otherwise mask a routing bug.
- Running this as a mixed pass/fail-and-score report — keep it a clean availability matrix and leave scoring to the benchmark skill.

## Checklist

- [ ] Region catalog validated: well-formed, unique identifiers and names
- [ ] One instance provisioned per region (or subset) in parallel, each with an individually bounded ready-state timeout
- [ ] ICMP, port-open, protocol-banner, and outbound-connectivity checks run per instance
- [ ] Cross-region address uniqueness confirmed
- [ ] Optional geoIP origin check confirms each address maps to its advertised region
- [ ] Coverage matrix (region × check) and an overall availability percentage produced
- [ ] All provisioned instances deleted after the sweep
