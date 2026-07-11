# Software Development Skills

A library of reusable, framework-agnostic **skills** for the cross-cutting infrastructure that almost every non-trivial multi-user / SaaS application ends up needing. Each skill captures a battle-tested pattern — the data model, the key decisions, and the pitfalls — so it can be lifted into a new project instead of re-derived.

These were distilled from a production multi-tenant application (Next.js + PostgreSQL + Drizzle), but the patterns are deliberately written to be portable across stacks.

## How to use

Each folder contains a `SKILL.md` with YAML frontmatter (`name`, `description`) and a body describing when to use it, the core pattern, concrete implementation notes, and gotchas. They follow the Claude Code / Agent Skills format, so they can be dropped into `.claude/skills/` (or any agent skill directory) as-is — or just read as engineering reference docs.

## Index

| Skill | What it covers |
|-------|----------------|
| [authentication-sessions](authentication-sessions/SKILL.md) | Signed-cookie sessions, SSO providers, 2FA |
| [authorization-rbac](authorization-rbac/SKILL.md) | Roles + fine-grained permission catalog |
| [api-tokens](api-tokens/SKILL.md) | Hashed bearer tokens for machine access |
| [multi-tenancy-rls](multi-tenancy-rls/SKILL.md) | Tenant isolation via Postgres Row-Level Security |
| [multi-channel-notifications](multi-channel-notifications/SKILL.md) | One dispatch → in-app + email + SMS + chat, with preferences |
| [structured-logging-audit](structured-logging-audit/SKILL.md) | Structured logs + who-did-what audit trail |
| [rate-limiting-abuse-prevention](rate-limiting-abuse-prevention/SKILL.md) | Sliding-window IP bans + CAPTCHA on public endpoints |
| [configuration-feature-flags](configuration-feature-flags/SKILL.md) | Two-tier (install + per-tenant) settings & toggles |
| [versioned-http-api](versioned-http-api/SKILL.md) | Versioned REST surface + OpenAPI/Swagger |
| [background-jobs-scheduling](background-jobs-scheduling/SKILL.md) | Cron, async job queue, trigger→action automation |
| [content-addressed-file-storage](content-addressed-file-storage/SKILL.md) | Blob store with SHA-256 dedup + ref-counting |
| [outbound-webhooks](outbound-webhooks/SKILL.md) | HMAC-signed event delivery with retries & log |
| [data-import-export](data-import-export/SKILL.md) | CSV/tabular import & export |
| [search-filtering-pagination](search-filtering-pagination/SKILL.md) | Trigram search, filtering, stable pagination |
| [internationalization](internationalization/SKILL.md) | i18n/l10n, RTL, locale-aware digits |
| [backup-restore](backup-restore/SKILL.md) | Scheduled backups with offsite copies + restore |
| [share-links-tracking](share-links-tracking/SKILL.md) | Expiring signed share links + open tracking |

## License

Provided as-is for reuse. Add your own license file before distributing.
