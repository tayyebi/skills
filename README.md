# Software Development Skills

A library of reusable, framework-agnostic **skills** for the cross-cutting infrastructure that almost every non-trivial multi-user / SaaS application ends up needing. Each skill captures a battle-tested pattern — the data model, the key decisions, and the pitfalls — so it can be lifted into a new project instead of re-derived.

These were distilled from production systems — a multi-tenant SaaS application (Next.js + PostgreSQL + Drizzle) and an active-active, masterless, multi-region status platform — but the patterns are deliberately written to be portable across stacks.

## How to use

Each folder contains a `SKILL.md` with YAML frontmatter (`name`, `description`) and a body describing when to use it, the core pattern, concrete implementation notes, and gotchas. They follow the Claude Code / Agent Skills format, so they can be dropped into `.claude/skills/` (or any agent skill directory) as-is — or just read as engineering reference docs.

## Index

### Application & SaaS building blocks

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

### Distributed systems & data synchronization

Patterns for running the same app on N ≥ 3 writable nodes that converge without a coordinator — distilled from an active-active, masterless status platform.

| Skill | What it covers |
|-------|----------------|
| [hybrid-logical-clocks](hybrid-logical-clocks/SKILL.md) | Comparable, causal version stamps across drifting clocks (HLC) |
| [masterless-active-active-replication](masterless-active-active-replication/SKILL.md) | N writable nodes converging via a gossiped LWW mutation log |
| [change-data-capture-triggers](change-data-capture-triggers/SKILL.md) | DB triggers that log every write (replication / outbox / audit) |
| [cluster-membership-gossip](cluster-membership-gossip/SKILL.md) | Seed bootstrap, peer health, join/leave, shared-secret peer auth |
| [leader-election-via-shared-config](leader-election-via-shared-config/SKILL.md) | One-of-N singleton work via a replicated config value, no consensus |
| [idempotency-keys](idempotency-keys/SKILL.md) | Dedupe write retries across replicas with a cached-response table |

### Deployment & fleet operations

Turning bare hosts into a running fleet — distilled from an SSH-driven, multi-host GitLab CI pipeline.

| Skill | What it covers |
|-------|----------------|
| [host-auto-provisioning](host-auto-provisioning/SKILL.md) | Idempotently ready a bare SSH host: runtime, dirs, mount self-heal, preflight |
| [build-once-deploy-everywhere](build-once-deploy-everywhere/SKILL.md) | Build one SHA-tagged image, pull-and-run it identically on every host |
| [rolling-fleet-deploy](rolling-fleet-deploy/SKILL.md) | Staggered rollout across a host matrix + post-deploy convergence gate |
| [zero-config-secret-bootstrap](zero-config-secret-bootstrap/SKILL.md) | A service self-mints its token on first boot and shares it via a volume |

## License

Provided as-is for reuse. Add your own license file before distributing.
