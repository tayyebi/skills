---
name: configuration-feature-flags
description: Use when settings must vary by install and by tenant — a two-tier configuration model (install-wide singleton + per-tenant settings blobs) plus feature toggles. Covers secret handling and safe public exposure of settings.
---

# Configuration & Feature Flags

Two levels of configuration, cleanly separated: **install-wide** (one row for the whole deployment) and **per-tenant** (one row per customer).

## When to use

Any multi-tenant app, and any app where operators need to change behavior without a redeploy. Feature flags fall out of this naturally.

## Core pattern

- **Install-wide settings** — a singleton table `system_settings` (one row) for deployment-level policy: default provider keys, global toggles, security posture (e.g. "restrict admin to base domain").
- **Per-tenant settings** — `site_settings(tenant_id, ...)`, typically a set of JSONB blobs grouped by concern: `email`, `sms`, `ai`, `localization`, `privacy`, `captcha`, custom field defs, etc. JSONB keeps the schema flexible as concerns are added.
- **Loader with defaults** — a `getSettings(tenant)` function merges tenant overrides onto system defaults onto hard-coded defaults, so a missing key never crashes a caller.
- **Feature flags** are just boolean settings (`jobAdsEnabled`, `aiScreeningDisabled`) read through the same loader. Per-tenant flags let you roll out gradually.

## Secret handling

Settings blobs contain secrets (SMTP passwords, API keys). Two rules:

1. **Strip secrets on public/less-privileged reads.** The GET that hydrates a settings *form* returns masked or omitted secret fields (`••••`), never the plaintext. Only write paths accept new secrets.
2. Have a single `stripSecrets()` / `maskSecret()` pass applied to any settings object leaving the trusted boundary.

## Pitfalls

- Returning secret values in a settings API that the frontend calls — audit every settings read path.
- No defaults — a newly added setting key is `undefined` for every existing tenant; the loader must supply a default.
- Putting rapidly-changing operational flags only in env vars (needs redeploy) — DB-backed settings can change live.
- JSONB with no validation — validate the shape on write so a bad settings blob can't break the loader.
- Caching settings too aggressively so a change doesn't take effect — use a short TTL or cache-bust on write.

## Checklist

- [ ] Install-wide singleton + per-tenant settings tables
- [ ] Loader merges tenant → system → hard-coded defaults
- [ ] Feature flags read through the same loader
- [ ] Secrets stripped/masked on read, accepted only on write
- [ ] Shape validation on write; short cache TTL or bust-on-write
