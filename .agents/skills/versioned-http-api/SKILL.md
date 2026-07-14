---
name: versioned-http-api
description: Use when exposing an HTTP API that third parties depend on — a versioned REST surface with a generated OpenAPI spec and interactive docs. Covers layering internal vs public APIs, versioning strategy, and keeping docs honest.
---

# Versioned HTTP API + OpenAPI

Give external consumers a stable, documented, versioned surface — separate from your internal endpoints.

## When to use

The moment anyone outside your own frontend calls your API. Internal UI endpoints can churn freely; a public API is a contract.

## Core pattern

Layer your HTTP surface into distinct namespaces:

1. **Internal/UI endpoints** (`/api/...`, `/tenant/[slug]/api/...`) — serve your own frontend, free to change.
2. **Public versioned API** (`/tenant/[slug]/api/v1/...`) — the external contract. Version in the path (`/v1`) so `/v2` can coexist during migrations.

- Public endpoints authenticate with **API tokens** (see `api-tokens`), not session cookies.
- Public endpoints enforce the same **permission catalog** (see `authorization-rbac`) via token scopes.

## OpenAPI + interactive docs

- **Generate the OpenAPI spec** from a single source of truth (route definitions / schemas), served dynamically at `/api/v1/openapi.json`. Generated-from-code beats hand-written YAML because it can't drift.
- Serve interactive docs (Swagger UI / Redoc) that read that spec, so consumers can explore and try calls.
- Validate request/response bodies against the same schemas you publish — the spec then *is* the behavior, not a description of it.

## Versioning strategy

- **Additive changes** (new optional field, new endpoint) don't need a version bump.
- **Breaking changes** (removing/renaming a field, changing types, changing auth) require `/v2`; keep `/v1` alive through a deprecation window.
- Announce deprecations in headers (`Deprecation`, `Sunset`) and docs.

## Pitfalls

- Docs drifting from reality — generate them; don't maintain a parallel document.
- Exposing internal endpoints as if they were the public contract, then being unable to change them.
- Leaking internal fields / secrets in public responses — define explicit response schemas, allow-list fields.
- No pagination/rate-limit story on public list endpoints (see `search-filtering-pagination`, `rate-limiting-abuse-prevention`).
- Inconsistent error shape — standardize `{ error: { code, message } }` across all endpoints.

## Checklist

- [ ] Public API in its own versioned namespace (`/v1`)
- [ ] Token auth + scoped permissions for public endpoints
- [ ] OpenAPI spec generated from code, served live
- [ ] Interactive docs (Swagger/Redoc) from that spec
- [ ] Explicit response schemas (no field leakage)
- [ ] Standard error shape; deprecation policy for breaking changes
