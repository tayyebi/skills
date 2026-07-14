---
name: testing-recon
description: Use when starting a new test engagement on an unfamiliar system and you need to map what's testable before diving into any other discipline — covers pulling the API contract, fingerprinting versions/health, mapping client-side routes, recovering source maps, fuzzing for undocumented endpoints, observing auth/session mechanics, and checking for edge/WAF bypass. Read-only; feeds every other testing skill.
---

# Testing Recon (Attack-Surface Mapping)

Black-box reconnaissance of a target system before any other testing begins: no source, no server access, only what the network surface exposes. The goal is a complete map — routes, auth mechanics, tech stack, data shapes, and any information leakage — that every subsequent test discipline can work from.

## When to use

- The first hat worn on any new engagement, before smoke, functional, or adversarial testing.
- Re-run after a major release: routes and contracts drift, and yesterday's map goes stale.
- Read-only by design — no mutations — so it's safe to run without coordination or a testing window.

Feeds [[testing-security]], [[testing-pentest-injection]], and [[testing-pentest-authz]] with an accurate target map. Spec conformance itself is owned by [[testing-contract]]; deep endpoint-by-endpoint behavior by [[testing-api]].

## Core pattern

Work outside-in, cheapest signal first:

1. **Grab the API contract.** If an OpenAPI/Swagger/GraphQL schema is published, fetch it before anything else — it's a free enumeration of every route, parameter, and schema the vendor is willing to admit to. Diff it later against what you actually find; the gap is the undocumented surface.
2. **Fingerprint the target.** Hit version/health/status endpoints and any disclosure files (`/.well-known/security.txt`). Note exact version strings (map to release notes for risk), and flag any internal hostnames, queue names, or DB labels a health payload leaks.
3. **Map client-side routes.** For an SPA, walk every nav item with DevTools' Network tab open and record which API calls back each screen. Then probe routes directly by URL — a route that renders without a login redirect is a client-side auth-guard miss (still confirm the backing API enforces auth server-side; the UI guard is cosmetic).
4. **Recover source maps.** Modern bundlers can ship `.map` files even when the browser doesn't auto-load them. A live source map hands you the full component tree, hidden routes, and feature flags — huge leverage for every later phase.
5. **Enumerate beyond the spec.** Fuzz a short, targeted wordlist (`internal`, `debug`, `admin`, `metrics`, `graphql`, `config`, `.git/HEAD`, `backup.zip`…) against the base path. A 401/403 still confirms the route exists — that's an undocumented endpoint worth targeting later.
6. **Observe auth/session mechanics.** Do one real login and read everything the server sets: token format (opaque session vs. JWT — check for a `header.payload.sig` shape), cookie flags (`HttpOnly`, `Secure`, `SameSite`), and where the client stores the credential (cookie vs. `localStorage`).
7. **Check for edge/WAF bypass.** If the app sits behind a CDN or WAF, look for a reachable origin (historical DNS, certificate-transparency logs, leaked IPs). Confirming direct origin access means downstream testing can bypass rate limits and edge protections that would otherwise mask real behavior — flag this loudly, it changes the threat model for every other hat.
8. **Inventory public vs. gated endpoints.** Loop a known list of "should be public" paths and confirm 200s; loop a list of "should require auth" paths and confirm 401 (not 200, not 500). This pair of loops is cheap and catches an entire class of "auth guard fell off" regressions.

```bash
BASE=https://your-target.example
curl -sS "$BASE/api/openapi.yaml" -o openapi.yaml && wc -l openapi.yaml
curl -sS "$BASE/api/version" | jq .
curl -sS "$BASE/api/health" | jq .
```

## Pitfalls

- **Skipping the published spec** and fuzzing blind — the spec is free, exhaustive, and usually accurate for the documented surface; use it as your diff baseline.
- **Treating a client-route redirect-to-login as proof of server-side auth** — it isn't. Always confirm the API itself 401s.
- **Missing "hidden" source maps** because DevTools doesn't auto-load them — try fetching `.map` files directly.
- **Stopping at 404-only fuzzing results** — a 401/403 on an unlisted path is still a finding; it proves the route exists.
- **Exfiltrating real data found during recon** — capture the minimum proof, mask it, don't hoard it.
- **Forgetting recon goes stale** — re-run the endpoint inventory after every material release.

## Checklist

- [ ] Published API/schema fetched and saved as a diff baseline
- [ ] Version/health fingerprinted; any info leakage in the health payload noted
- [ ] Client-side routes enumerated; unguarded routes flagged (and server-side auth re-verified)
- [ ] Source maps checked; if live, component tree/routes/flags extracted
- [ ] Targeted fuzz run beyond the documented spec; non-404 hits logged
- [ ] Auth/session mechanics recorded: token format, cookie flags, storage location
- [ ] Edge/WAF bypass checked (origin reachability)
- [ ] Public-endpoint and auth-required-endpoint inventories both verified
