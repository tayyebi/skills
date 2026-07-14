---
name: testing-platform-coverage
description: Use when verifying that every selectable entry in a variant/template catalog (OS images, container base images, plan/tier sizes, device profiles — whatever axis your product lets a user pick from) actually builds and runs correctly, not just that the catalog API lists it. Covers catalog-completeness baselining, per-entry field validation, build/provisioning-path coverage, sizing feasibility across multiple resource dimensions, provisioned-identity verification, and cross-variant invariants such as no shared identity/secret reuse between variants.
---

# Catalog / Matrix Coverage Testing

A catalog of selectable variants (OS templates, base images, instance sizes, device types) is a promise: pick any entry and you get a correctly built, correctly sized thing running that exact variant. This skill tests that promise across the whole catalog rather than the one or two variants a smoke test happens to exercise — catching the entry that lists fine but silently fails to provision, or is sized against the wrong resource requirement, or shares an identity with every other entry built from the same underlying image.

## When to use

- Any product where users pick from a list of variants at creation time — OS/image templates, container base images, hardware/plan tiers, device profiles — and each entry is expected to actually work, not just be listed.
- Whenever "we support N options" needs to become "we verified N options," especially after the catalog has grown organically and nobody has re-checked the older entries.
- Pairs with `testing-lifecycle` (each provisioned variant goes through the same state machine), `testing-ssh` (cross-variant host-key uniqueness is a canonical invariant here), and `testing-network`/`testing-benchmark` (per-variant reachability and performance).

## Core pattern

**Baseline the catalog and diff against it deliberately.** Keep an explicit "expected entries" list (IDs and/or families) and diff the live catalog against it on every run. This catches two different regressions that a bare "catalog returns >0 items" check misses entirely: an entry silently dropped, and a new, undocumented entry added without anyone updating the test (or the docs). Update the baseline as a conscious act, not by letting the diff go unchecked.

**Validate every entry's fields, not just the ones you provision.** For each catalog entry, assert identifier format matches the scheme the API actually uses, required descriptive fields are non-empty, and every resource-requirement field (minimum memory, minimum disk, whatever the sizing dimensions are) is present and a sane positive number. A malformed entry can sit in the catalog for a long time if nothing ever provisions it.

**Group by underlying build/provisioning mechanism and cover every mechanism at least once.** Variants sharing a mechanism (e.g. a common cloud-init path vs. an agent-based firstboot path vs. a platform-specific initialization path) tend to fail or succeed together — a full sweep of every single catalog entry is expensive and mostly redundant. Default to **representative sampling**: one entry per mechanism, exercising every distinct build path at minimal cost, with a full-sweep mode available as an explicit, opt-in (e.g. scheduled/nightly) run rather than the default.

**Check multi-dimensional sizing feasibility.** Every catalog entry must fit at least one real plan/tier — compute the cheapest fit by comparing *all* of its resource-requirement dimensions against a candidate tier (memory **and** disk, not just one). Never assume the smallest/cheapest tier is sufficient for every entry by default; explicitly assert the heaviest entries actually require a larger tier than a naive "pick the cheapest" selection would choose, since a sizing bug here means a whole class of variants is silently unprovisionable.

**Provision a representative instance per mechanism and verify identity, not just success.** A "reached running state" result is not proof the *correct* variant was applied — fingerprint the running instance (via whatever introspection the platform allows: an in-instance identity file, a reported image tag, a version string) and confirm it actually matches the family/version that was selected. Mechanisms that only ever get checked for "did it boot" can silently apply the wrong image indefinitely.

**Assert cross-variant invariants.** The canonical one: no two provisioned instances — regardless of which catalog variant they came from — should share an identity/secret that's supposed to be freshly generated per instance (an SSH host key is the typical example). Collect the relevant fingerprint from each provisioned instance and assert uniqueness across the whole set. A shared fingerprint across variants usually means instances are being cloned from a shared golden image instead of generating fresh identity material at first boot — a real security defect, not a cosmetic one.

**Check catalog consistency and determinism.** The catalog should return the same content whether the caller is authenticated or anonymous (unless access-gating is an intentional feature) and should be deterministic across repeated calls in the same run — a catalog that reorders or intermittently omits entries makes every other check in this pattern unreliable.

## Pitfalls

- **"Catalog returns 200" mistaken for "every entry provisions correctly."** Listing and building are different guarantees; test both.
- **Assuming the cheapest tier fits every catalog entry.** Sizing must compare every resource dimension, not just the cheapest common one (e.g. memory alone while ignoring disk).
- **Full-matrix sweep as the default.** Cost and flakiness scale with catalog size; sample by mechanism and reserve full sweeps for an explicit, less-frequent run.
- **Treating "reached running" as proof of correct identity.** Always verify the running instance actually matches the selected variant.
- **Skipping the cross-variant uniqueness check.** Golden-image cloning is invisible until you explicitly compare fingerprints across different variants.
- **No baseline diffing.** Without an explicit expected-entries list, both catalog regressions and silent additions go unnoticed indefinitely.

## Checklist

- [ ] Live catalog diffed against an explicit, deliberately maintained baseline (entries and families)
- [ ] Every catalog entry's fields format-checked: identifier scheme, required descriptive fields, positive resource-requirement values
- [ ] Every distinct build/provisioning mechanism represented in the catalog exercised at least once
- [ ] Sizing feasibility computed across all resource dimensions per entry; heaviest entries confirmed to need a larger tier, not the cheapest
- [ ] At least one representative instance provisioned per mechanism, with running identity verified against the selected variant (not just "reached running")
- [ ] Cross-variant identity/secret uniqueness asserted (e.g. no shared host key across differently-sized or differently-imaged instances)
- [ ] Catalog checked for authenticated-vs-anonymous consistency and determinism across repeated calls
- [ ] Full-sweep mode available but not the default; representative sampling used for routine runs
