---
name: testing-contract
description: Use when auditing how well a live API's actual surface matches its published spec (OpenAPI/Swagger/etc.) — covers diffing documented routes against known/observed routes, probing for undocumented endpoints, grep-checking response field-format consistency (casing, ID format, timestamp format), and reporting a coverage percentage. Distinct from testing-api's runtime status-code correctness.
---

# API Contract & Documentation Coverage Testing

Audit spec accuracy and completeness, not runtime correctness. Pull the live spec, diff it against every route you actually know about (from docs, client code, or prior exploration), probe for routes that exist but aren't documented, and grep every response for format drift. The output is a coverage number that should only go up over time.

> Distinct from [[testing-api]] (runtime status-code/error-body correctness) and [[testing-security]] (adversarial probing). This skill's product is a documentation gap list, not a pass/fail on behavior.

## When to use

- The API has a machine-readable spec (OpenAPI, GraphQL SDL, gRPC proto, etc.) that may be stale or incomplete.
- You want to quantify "how much of the real API surface is documented" and track it as a regression metric.
- Onboarding a new API and need to build the known-route inventory that other testing skills (e.g. [[testing-api]], [[testing-authz-matrix]]) will sweep.

## Core pattern

**Extract the documented surface.** Parse the spec into a normalized `METHOD /path` list (collapse path parameters to a placeholder like `{id}` so `/services/{id}` and `/services/{uuid}` compare equal).

**Build the known-route list independently of the spec.** Source it from UI network traces, client SDKs, changelogs, or prior exploration — never from the spec itself, or the diff is meaningless. Diff known routes against documented routes; every route present in the former and absent from the latter is a documentation gap. Report `covered / known = coverage%` and track it run over run — it should be monotonically non-decreasing.

**Probe for undocumented endpoints actively, not just passively diff.** Build a short wordlist from naming patterns you've already seen (a `metrics` sibling of `logs`, a `bulk` sibling of a singular-resource route, common ops/debug paths like `health`, `internal`, `debug`) and request each with both an unauthenticated and an authenticated identity. Anything that doesn't 404 is either genuinely undocumented (add it to the spec) or a route that should 404 and doesn't (a separate bug).

**Grep response bodies for format consistency**, not just correctness. Pick one convention per axis (all-camelCase keys vs snake_case, UUID vs slug identifiers, RFC3339 timestamps) and flag every field that violates the house convention — including legitimate slugs that happen to look identifier-like, so validate against *both* accepted shapes rather than assuming every `id`-suffixed field is a UUID.

**Check structural conventions across every list endpoint**: does every list respond with the same envelope shape (bare array vs `{data, total, page, ...}`)? A single outlier breaks generic client code.

**Verify HTTP method and Content-Type enforcement** as a contract concern: unsupported methods on a resource should return 405, not silently succeed or 500; malformed/missing Content-Type should degrade to 400/415, never a crash.

## Design decisions

- **Coverage percentage as a tracked metric**, not a one-off number — re-run after every doc update and expect it to trend toward 100%.
- **Known-route list lives outside the spec.** If you seed it from the spec you're testing, you can only ever discover your own blind spots' complement — sourcing it independently (traffic capture, client SDK, changelog) is what surfaces real gaps.
- **Treat "found but undocumented" and "missing from spec" as two different bug classes** — the former needs an audit (is it intentionally hidden, or a security gap?), the latter just needs a spec update.

## Pitfalls

- Deriving the known-route list from the spec itself — guarantees 100% "coverage" against a tautology.
- Not normalizing path parameters before diffing — `/x/{id}` vs `/x/{uuid}` register as different routes and inflate the gap count.
- Flagging a legitimate slug identifier as a "non-UUID" format violation — confirm the field's actual identifier convention before asserting a violation.
- Treating an undocumented-but-working endpoint as harmless — it bypassed spec-driven review, which often means it also bypassed rate-limiting or auth review.

## Checklist

- [ ] Live spec parsed into a normalized, path-param-collapsed route list
- [ ] Independently sourced known-route inventory (not derived from the spec)
- [ ] Coverage percentage computed and tracked run over run
- [ ] Active probe for undocumented endpoints (auth + unauth), each hit triaged
- [ ] Field-format grep: casing, ID format (UUID vs slug), timestamp format
- [ ] List-endpoint envelope shape consistent across the whole API
- [ ] Method enforcement (405) and Content-Type handling (400/415, never 500) checked
- [ ] Findings reported as a gap list, not just a pass/fail count
