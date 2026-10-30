# Software Development Skills

A library of reusable, framework-, language-, and project-agnostic **skills** — for the cross-cutting infrastructure almost every non-trivial application needs, for testing/QA methodology, and for general engineering practice. Each skill captures a battle-tested pattern — the core method, the key decisions, and the pitfalls — so it can be lifted into a new project instead of re-derived. Code examples are pseudocode or generic notation, not tied to one language or framework, unless the skill's subject *is* a specific surface syntax (e.g. CSS logical properties for RTL layout).

## Repository layout

Every skill lives once, under `.agents/skills/<name>/SKILL.md` — that's the canonical copy. Everything else is a thin adapter:

```
.agents/skills/<name>/SKILL.md   ← canonical source, edit here
.claude/skills/<name>            → symlink to .agents/skills/<name>
.codex/skills/<name>             → symlink to .agents/skills/<name>
.opencode/skills/<name>          → symlink to .agents/skills/<name>
```

`.agents/skills/` is itself a real, emerging cross-tool convention: Google Antigravity reads it natively as its project-scope skills directory, OpenAI's Codex CLI and OpenCode both auto-load it as one of their supported skill locations. So dropping this repo into a project (or symlinking it in) already works for those three agents with zero extra setup. Claude Code specifically looks for `.claude/skills/`, so that directory holds one symlink per skill pointing back into `.agents/skills/` — same content, no duplication, no drift. `.codex/skills/` and `.opencode/skills/` are populated the same way for explicitness, even though both tools also pick up `.agents/skills/` directly.

## Working with skills

### Reading a skill

Each `SKILL.md` has YAML frontmatter (`name`, `description`) and a body describing when to use it, the core pattern, concrete implementation notes, pitfalls, and a checklist. Read it as an engineering reference doc, or point an agent at this repo so it's picked up automatically.

Open the canonical copy: `.agents/skills/<name>/SKILL.md`. The symlinks in `.claude/`, `.codex/`, and `.opencode/` all point back to it, so reading any path shows the same file.

### Updating an existing skill

Edit the canonical copy at `.agents/skills/<name>/SKILL.md`. The symlinks in the adapter directories follow automatically — no duplication, no drift, nothing to sync.

Before updating, read the existing skill end-to-end. Match its voice, structure, and level of detail. If the change affects the skill's scope, remember to update the `description` in the YAML frontmatter (agents use it for skill routing) and the README index entry if the one-line summary is no longer accurate.

### Adding a new skill

1. **Write the skill.** Create `.agents/skills/<name>/SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: <kebab-case-name>
   description: Use when <condition> — <one-line summary>.
   ---
   ```
   Study an existing skill (e.g. `database-schema-design`) for the expected structure: when-to-use, core pattern, concrete guidance, pitfalls, and a checklist.

2. **Symlink into adapter directories.** From the repo root:
   ```
   ln -s ../../.agents/skills/<name> .claude/skills/<name>
   ln -s ../../.agents/skills/<name> .codex/skills/<name>
   ln -s ../../.agents/skills/<name> .opencode/skills/<name>
   ```

3. **Add a row to the README index.** Find the right category table under `## Index` and add a row linking to the canonical path (`.agents/skills/<name>/SKILL.md`). Match the existing row format exactly — pipe-separated columns, relative link, one-line summary.

## Index

### General engineering practices

Language-independent principles — apply the *idea*, translate the pseudocode into whatever stack you're in.

| Skill | What it covers |
|-------|----------------|
| [clean-code-fundamentals](.agents/skills/clean-code-fundamentals/SKILL.md) | Naming, function shape, comments, DRY, guard clauses, error handling |
| [solid-principles](.agents/skills/solid-principles/SKILL.md) | The five SOLID principles, with pseudocode and when to pragmatically break them |
| [object-oriented-design-principles](.agents/skills/object-oriented-design-principles/SKILL.md) | Encapsulation, composition over inheritance, coupling/cohesion |
| [design-patterns-catalog](.agents/skills/design-patterns-catalog/SKILL.md) | GoF-style patterns + when a pattern is over-engineering |
| [clean-architecture-layering](.agents/skills/clean-architecture-layering/SKILL.md) | Dependency rule, domain/application/adapter layers, ports |
| [plugin-architecture-microkernel](.agents/skills/plugin-architecture-microkernel/SKILL.md) | Kernel + plugins split: contract-as-code, hook gating, disabled-state semantics, UI registries, caller-first migration |
| [static-typing-discipline](.agents/skills/static-typing-discipline/SKILL.md) | Strict typing, static analysis in CI, null-safety, immutability |
| [database-schema-design](.agents/skills/database-schema-design/SKILL.md) | Keys, naming, migrations, indexing, N+1 avoidance |
| [actor-attribution](.agents/skills/actor-attribution/SKILL.md) | created_by/updated_by (actor_id) columns, system-actor handling |
| [append-only-entity-versioning](.agents/skills/append-only-entity-versioning/SKILL.md) | Full-row snapshots keyed off a shared revision sequence instead of UPDATE-in-place |
| [api-design](.agents/skills/api-design/SKILL.md) | Resource naming, HTTP verbs, status codes, response/error envelope |
| [performance-optimization](.agents/skills/performance-optimization/SKILL.md) | Profile-first discipline, caching layers, batching, lazy loading |
| [observability-logging-monitoring](.agents/skills/observability-logging-monitoring/SKILL.md) | Log levels, metrics, correlation IDs, alert tiers, dashboards |
| [secure-coding-practices](.agents/skills/secure-coding-practices/SKILL.md) | Build-time OWASP-adjacent prevention: injection, XSS, secrets, deserialization |
| [git-workflow-conventions](.agents/skills/git-workflow-conventions/SKILL.md) | Commit conventions, branch naming, PR discipline, rebase vs. merge |
| [ci-cd-pipeline-discipline](.agents/skills/ci-cd-pipeline-discipline/SKILL.md) | Staged gates, required checks, what makes a pipeline trustworthy |
| [responsive-mobile-first-design](.agents/skills/responsive-mobile-first-design/SKILL.md) | Mobile-first CSS authoring, breakpoints, per-size-class behaviour, touch targets |
| [rtl-bidi-layout](.agents/skills/rtl-bidi-layout/SKILL.md) | Logical CSS properties, icon mirroring, bidi text handling |
| [keyboard-shortcuts-layouts](.agents/skills/keyboard-shortcuts-layouts/SKILL.md) | Layout-proof shortcuts: e.key vs e.code matching, non-Latin layout fallback, IME/editable guards, per-platform labels |
| [z-index-layer-governance](.agents/skills/z-index-layer-governance/SKILL.md) | Named layer scale as tokens, stacking-context traps, auditing ad-hoc z-index |
| [user-journey-engineering](.agents/skills/user-journey-engineering/SKILL.md) | Goal-driven flow design: actor/goal mapping, friction analysis, journey anti-patterns |
| [dense-workspace-design](.agents/skills/dense-workspace-design/SKILL.md) | Compact, connected workspaces for thousands of records: UI density, task-based UX, bounded backend APIs, safe edits, and bulk operations |
| [living-architecture-diagram](.agents/skills/living-architecture-diagram/SKILL.md) | Generated, zoomable system map from a data file — data/layout/render split, viewer, anti-rot discipline |

### Application & SaaS building blocks

| Skill | What it covers |
|-------|----------------|
| [authentication-sessions](.agents/skills/authentication-sessions/SKILL.md) | Signed-cookie sessions, SSO providers, 2FA |
| [sso-oidc-federation](.agents/skills/sso-oidc-federation/SKILL.md) | OIDC sign-in (Keycloak, Entra, Cloudflare Access) alongside local accounts, group→role mapping, break-glass |
| [oidc-provider](.agents/skills/oidc-provider/SKILL.md) | Being the OP: client registration, authorize/token/userinfo, opaque rotating tokens, withdrawable consent, tenant claims |
| [authorization-rbac](.agents/skills/authorization-rbac/SKILL.md) | Roles + fine-grained permission catalog |
| [api-tokens](.agents/skills/api-tokens/SKILL.md) | Hashed bearer tokens for machine access |
| [mcp-server-design](.agents/skills/mcp-server-design/SKILL.md) | MCP tool access for AI assistants: owner-acting tokens, layered guards, owner-readable call log, and a guard-exempt "wish" feedback tool |
| [multi-tenancy-rls](.agents/skills/multi-tenancy-rls/SKILL.md) | Tenant isolation via Postgres Row-Level Security |
| [multi-channel-notifications](.agents/skills/multi-channel-notifications/SKILL.md) | One dispatch → in-app + email + SMS + chat, with preferences |
| [interactive-email](.agents/skills/interactive-email/SKILL.md) | Actionable email: ICS invites, Gmail JSON-LD cards, signed action links, RFC 8058 unsubscribe |
| [mail-client-sync](.agents/skills/mail-client-sync/SKILL.md) | Mirroring a remote mailbox (IMAP) locally: mirror + write-back op queue, leased budgeted sync runs, folder fairness, the UID-renumber trap, honest staleness |
| [spf-record-validation](.agents/skills/spf-record-validation/SKILL.md) | Resolving SPF the way a receiver does: the ten-lookup ceiling, void lookups, flattening, and why it needs monitoring rather than one-time validation |
| [routing-filters](.agents/skills/routing-filters/SKILL.md) | User-defined rules routing items to destinations: priority/ignore/stop/catch-all, plus overlap detection proved by witness, a simulator on the live path, and a matrix |
| [structured-logging-audit](.agents/skills/structured-logging-audit/SKILL.md) | Structured logs + who-did-what audit trail |
| [rate-limiting-abuse-prevention](.agents/skills/rate-limiting-abuse-prevention/SKILL.md) | Sliding-window IP bans + CAPTCHA on public endpoints |
| [configuration-feature-flags](.agents/skills/configuration-feature-flags/SKILL.md) | Two-tier (install + per-tenant) settings & toggles |
| [versioned-http-api](.agents/skills/versioned-http-api/SKILL.md) | Versioned REST surface + OpenAPI/Swagger |
| [background-jobs-scheduling](.agents/skills/background-jobs-scheduling/SKILL.md) | Cron, async job queue, trigger→action automation |
| [content-addressed-file-storage](.agents/skills/content-addressed-file-storage/SKILL.md) | Blob store with SHA-256 dedup + ref-counting |
| [outbound-webhooks](.agents/skills/outbound-webhooks/SKILL.md) | HMAC-signed event delivery with retries & log, event-catalog design, feedback-loop and dead-receiver guards |
| [data-import-export](.agents/skills/data-import-export/SKILL.md) | CSV/tabular import & export |
| [mock-data](.agents/skills/mock-data/SKILL.md) | Seeding realistic fake data that mirrors the production schema |
| [search-filtering-pagination](.agents/skills/search-filtering-pagination/SKILL.md) | Trigram search, filtering, stable pagination |
| [internationalization](.agents/skills/internationalization/SKILL.md) | i18n/l10n, RTL, locale-aware digits |
| [multi-calendar-datetime](.agents/skills/multi-calendar-datetime/SKILL.md) | Gregorian + Jalali/Hijri end to end: instant storage, Intl-only calendar math, calendar-aware fields/grids, zone traps |
| [backup-restore](.agents/skills/backup-restore/SKILL.md) | Scheduled backups with offsite copies + restore |
| [admin-database-console](.agents/skills/admin-database-console/SKILL.md) | In-app DB GUI for operators: browse, SQL console, size/bloat/index profiling, cache & retention observability |
| [admin-performance-console](.agents/skills/admin-performance-console/SKILL.md) | In-app timings page: in-memory percentile store, global HTTP/SQL instrumentation, N+1 attribution, and an advisory layer with actions |
| [share-links-tracking](.agents/skills/share-links-tracking/SKILL.md) | Expiring signed share links + open tracking |
| [social-preview-cards](.agents/skills/social-preview-cards/SKILL.md) | Generated per-page OG/Twitter cards, content-addressed so they never go stale |

### Distributed systems & data synchronization

Patterns for running the same app on N ≥ 3 writable nodes that converge without a coordinator — distilled from an active-active, masterless status platform.

| Skill | What it covers |
|-------|----------------|
| [hybrid-logical-clocks](.agents/skills/hybrid-logical-clocks/SKILL.md) | Comparable, causal version stamps across drifting clocks (HLC) |
| [masterless-active-active-replication](.agents/skills/masterless-active-active-replication/SKILL.md) | N writable nodes converging via a gossiped LWW mutation log |
| [replication-snapshot-bootstrap](.agents/skills/replication-snapshot-bootstrap/SKILL.md) | Row-state transfer for replicas the incremental log can't heal — paged snapshots through the same apply pipeline |
| [change-data-capture-triggers](.agents/skills/change-data-capture-triggers/SKILL.md) | DB triggers that log every write (replication / outbox / audit) |
| [cluster-membership-gossip](.agents/skills/cluster-membership-gossip/SKILL.md) | Seed bootstrap, peer health, join/leave, shared-secret peer auth |
| [leader-election-via-shared-config](.agents/skills/leader-election-via-shared-config/SKILL.md) | One-of-N singleton work via a replicated config value, no consensus |
| [idempotency-keys](.agents/skills/idempotency-keys/SKILL.md) | Dedupe write retries across replicas with a cached-response table |

### Deployment & fleet operations

Turning bare hosts into a running fleet — distilled from an SSH-driven, multi-host GitLab CI pipeline.

| Skill | What it covers |
|-------|----------------|
| [host-auto-provisioning](.agents/skills/host-auto-provisioning/SKILL.md) | Idempotently ready a bare SSH host: runtime, dirs, mount self-heal, preflight |
| [build-once-deploy-everywhere](.agents/skills/build-once-deploy-everywhere/SKILL.md) | Build one SHA-tagged image, pull-and-run it identically on every host |
| [rolling-fleet-deploy](.agents/skills/rolling-fleet-deploy/SKILL.md) | Staggered rollout across a host matrix + post-deploy convergence gate |
| [zero-config-secret-bootstrap](.agents/skills/zero-config-secret-bootstrap/SKILL.md) | A service self-mints its token on first boot and shares it via a volume |

### Testing & QA

Black-box, framework-agnostic testing methodology — how to verify a system from the outside, not how to write unit tests against its source.

| Skill | What it covers |
|-------|----------------|
| [testing-recon](.agents/skills/testing-recon/SKILL.md) | Attack-surface mapping and scoping at the start of a test engagement |
| [testing-smoke](.agents/skills/testing-smoke/SKILL.md) | Tiered go/no-go deployment sanity check |
| [testing-smoke-curl](.agents/skills/testing-smoke-curl/SKILL.md) | Zero-dependency curl+jq smoke-test playbook |
| [testing-functional](.agents/skills/testing-functional/SKILL.md) | Scripted regression across core user journeys |
| [testing-exploratory](.agents/skills/testing-exploratory/SKILL.md) | Session-based exploratory testing (SBET): charters, tours, HICCUPPS |
| [testing-api](.agents/skills/testing-api/SKILL.md) | Black-box HTTP/REST testing: auth, CSRF, status codes, pagination |
| [testing-contract](.agents/skills/testing-contract/SKILL.md) | Diffing a live API against its documented spec, undocumented-route discovery |
| [testing-data-fuzz](.agents/skills/testing-data-fuzz/SKILL.md) | Input-space fuzzing: boundaries, Unicode, type confusion |
| [testing-authz-matrix](.agents/skills/testing-authz-matrix/SKILL.md) | Systematic role×endpoint RBAC coverage sweep |
| [testing-automation](.agents/skills/testing-automation/SKILL.md) | Building runnable black-box test harnesses with no source access |
| [testing-security](.agents/skills/testing-security/SKILL.md) | Breadth-first OWASP penetration-testing sweep |
| [testing-pentest-authz](.agents/skills/testing-pentest-authz/SKILL.md) | Deep privilege-escalation and IDOR probing |
| [testing-pentest-injection](.agents/skills/testing-pentest-injection/SKILL.md) | Deep SQL/OS-command/SSTI/NoSQL injection probing |
| [testing-compliance](.agents/skills/testing-compliance/SKILL.md) | Privacy/GDPR, data retention, audit-trail testing |
| [testing-chaos](.agents/skills/testing-chaos/SKILL.md) | Failure-injection and resilience testing |
| [testing-lifecycle](.agents/skills/testing-lifecycle/SKILL.md) | State-machine testing for a provisioned/stateful resource |
| [testing-ssh](.agents/skills/testing-ssh/SKILL.md) | SSH access correctness: key injection, auth, host-key consistency |
| [testing-vnc](.agents/skills/testing-vnc/SKILL.md) | Remote console/VNC session correctness |
| [testing-network](.agents/skills/testing-network/SKILL.md) | Reachability, DNS, port state vs. billing/control-plane state |
| [testing-platform-coverage](.agents/skills/testing-platform-coverage/SKILL.md) | Catalog/variant-matrix coverage (images, tiers, device profiles) |
| [testing-performance](.agents/skills/testing-performance/SKILL.md) | Response times, throughput, race conditions, timing attacks |
| [testing-benchmark](.agents/skills/testing-benchmark/SKILL.md) | In-resource micro-benchmarking vs. advertised specs |
| [testing-benchmark-compare](.agents/skills/testing-benchmark-compare/SKILL.md) | Cross-tier benchmark comparison, catching oversold plans |
| [testing-region-benchmark](.agents/skills/testing-region-benchmark/SKILL.md) | Cross-location performance comparison |
| [testing-region-coverage](.agents/skills/testing-region-coverage/SKILL.md) | Per-region availability matrix |
| [testing-browser-ui](.agents/skills/testing-browser-ui/SKILL.md) | Live browser-driven interactive UI testing |
| [testing-mobile](.agents/skills/testing-mobile/SKILL.md) | Responsive/mobile device-emulation testing |
| [testing-accessibility](.agents/skills/testing-accessibility/SKILL.md) | WCAG-referenced a11y testing and remediation reporting |
| [testing-billing](.agents/skills/testing-billing/SKILL.md) | Financial-correctness testing: ledger, invoices, refunds |
| [testing-integration](.agents/skills/testing-integration/SKILL.md) | Third-party seams: payment gateways, email, SSO/SCIM, webhooks |

## License

Provided as-is for reuse. Add your own license file before distributing.
