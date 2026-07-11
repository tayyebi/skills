---
name: structured-logging-audit
description: Use when an app needs operational visibility and accountability — structured JSON logging mirrored to a queryable store, plus an actor-tagged audit trail of who-did-what. Covers log levels, searchable context, and separating logs from audit.
---

# Structured Logging & Audit Trail

Two related but distinct concerns: **logs** (for operators, to debug the system) and **audit** (for accountability, to answer "who changed this?").

## When to use

Every app past the prototype stage. Structured logging from day one is far cheaper than retrofitting it after your first production incident.

## Logging

- **Structured, not string-concatenated.** Every log line is an object: `{ level, message, context, ...fields, timestamp }`. Emit JSON to stdout (for the platform's log collector) and optionally mirror to a queryable store (a `logs` table) so support staff can search without shell access.
- **Levels**: `debug/info/warn/error`. Default production threshold at `info`.
- **Context propagation**: attach `tenant_id`, `user_id`, `request_id` to every line in a request so logs can be filtered to one request or one customer.
- **Searchable store**: `logs(id, level, message, context, fields jsonb, created_at)` with indexes on level + created_at, and trigram/text search on message. Add an in-app log viewer with level/context/text filters and a histogram.
- **Auto-prune**: logs grow without bound — schedule deletion beyond a retention window (e.g. 30 days), or partition by day and drop old partitions.
- Extract structured detail from errors (e.g. the DB error *code*, not just the message) so you can aggregate by failure type.

## Audit

Audit is **not** logging. Audit entries are business facts, retained longer, never auto-pruned casually, and always actor-tagged:

```
audit(actor, action, target, metadata)
→ audit_log(id, tenant_id, actor_id, action, target_type, target_id, meta jsonb, created_at)
```

- Bind an `audit()` helper to the authenticated actor at the auth gateway (see `authorization-rbac`) so call sites can't forget who did it.
- Record **security-relevant events explicitly**: login, permission change, data export, cross-tenant admin action, token creation, data deletion.
- Audit entries should be **append-only** — no updates or deletes from application code.

## Pitfalls

- **Logging secrets/PII** — scrub tokens, passwords, `Authorization` headers, and sensitive fields before writing. Have a `maskSecret()` helper and use it.
- Recursive logging: if writing a log can itself fail and log, you get a loop — add a recursion guard around the log-store writer.
- Conflating logs and audit — different retention, different consumers, different tamper-resistance needs.
- Unbounded growth — always have a retention/prune plan.

## Checklist

- [ ] Structured JSON logs with level + context fields
- [ ] Queryable log store with filtered viewer + retention/prune
- [ ] Secret/PII scrubbing before write; recursion guard
- [ ] Separate append-only audit trail, actor-bound
- [ ] Security-relevant actions explicitly audited
