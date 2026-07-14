---
name: secure-coding-practices
description: Use when writing or reviewing code that handles untrusted input, talks to external systems, or manages secrets — build-time defenses against injection, XSS, insecure deserialization, security misconfiguration, SSRF, and secret leakage. Covers parameterized queries, output encoding, boundary validation, secure defaults, and dependency hygiene. For after-the-fact verification see `testing-security` and its drill-down skills, not this one.
---

# Secure Coding Practices

The practices a developer applies *while writing code*, before anything ships: treat every external input as hostile until validated, and every output as unsafe until encoded for its destination. This is prevention, not testing — it's the checklist you run against a diff, not an attack you launch against a running service.

## When to use

- Writing or reviewing any code path that touches untrusted input: request bodies, query params, headers, cookies, file uploads, webhook payloads, messages off a queue.
- Code that builds a query, a shell command, a file path, or an outbound HTTP request from anything a caller influenced.
- New service scaffolding, a new integration, or a PR review pass before merge.
- Not for: verifying these defenses hold up under attack — that's `testing-security` (breadth sweep), `testing-pentest-authz` (privilege escalation), `testing-pentest-injection` (injection exploitation), `testing-data-fuzz` (input-space fuzzing).

## Core pattern

Two boundaries, both mandatory:

- **Input boundary** (entry point): validate against an explicit schema — types, bounds, enums, format — and reject what doesn't match. Don't "sanitize and continue" when reject is an option.
- **Output boundary** (every sink): encode/parameterize for that sink's language. SQL, shell, HTML, a log line, and a URL each have different escaping rules; there's no single "safe string."

Validate once, at the boundary, then trust the shape internally — don't scatter ad hoc re-checks through the call stack.

## Injection (SQL, NoSQL, command)

Never build a query or command by concatenating input into a string. Use parameterized queries / prepared statements everywhere, including inside an ORM's raw-query escape hatches:

```
// Bad — the value becomes part of the SQL text
query("SELECT * FROM users WHERE email = '" + input.email + "'")

// Good — the driver binds the value, never interpolates it
query("SELECT * FROM users WHERE email = ?", [input.email])
```

An ORM's query builder is safe by default, but its "raw fragment" or "raw where" methods are exactly as unsafe as hand-written SQL if input reaches them unbound. The same rule applies to NoSQL query objects (an unvalidated object can inject operators) and to shell commands: pass an argument array to an exec call, never build a shell string; if a command name itself is input-derived, allowlist the accepted values.

## Output encoding (XSS) and untrusted data

- Use your templating/rendering layer's default auto-escaping for anything rendered into HTML, and only bypass it (raw/unescaped output) for content you generated or already sanitized — never for a user-supplied field.
- Encoding is context-specific: a value safe in an HTML body can still break out of an HTML attribute, a `<script>` block, or a URL query string. Encode for the specific sink, not once "up front."
- If you must accept and render user-authored HTML (comments, bios), run it through a dedicated allowlist sanitizer library — don't hand-roll tag stripping.
- Don't deserialize untrusted data with a format that can reconstruct arbitrary objects or execute code on decode (native/language-level object serialization). Prefer a plain data format (JSON) with schema validation applied *after* decoding, not instead of it.

## Boundary validation and data handling

- **Mass assignment**: bind only the fields your validated schema names explicitly onto a persisted record. Never map a raw request body straight onto a model — that lets a caller set fields you never intended to expose (a role, a price, an internal flag) just by including them in the payload.
- **File uploads**: validate by sniffing actual content (magic bytes), not the client-supplied extension or `Content-Type` header. Store outside the served web root, generate a random filename/id server-side, and never use the client-supplied name to build a filesystem path.
- **SSRF**: if code fetches a URL built from input (webhook target, link preview, image proxy), resolve the hostname and check the resulting IP against a private/internal-range denylist, block redirects that lead to internal targets, and constrain scheme/port where feasible. A URL that "looks like" `https://example.com` can still resolve to `169.254.169.254` or an internal service.

## Secure defaults and configuration

- Deny by default: verbose error pages and stack traces off in production, debug/admin dashboards behind auth or disabled, unused routes removed rather than left reachable.
- Set security headers deliberately: HSTS, `X-Content-Type-Options: nosniff`, frame-ancestors / `X-Frame-Options`, a real CSP, `Referrer-Policy`.
- CORS is an explicit origin allowlist — never a wildcard combined with credentialed requests.
- Access control (who may act) is a separate concern from what's covered here (what a request may contain) — see `authorization-rbac`. Tenant data isolation is `multi-tenancy-rls`. Volume/abuse limits are `rate-limiting-abuse-prevention`. Credential and session mechanics are `authentication-sessions`; machine-token issuance is `api-tokens`.

## Secrets and sensitive data

- Never commit secrets to source control. Load them from environment variables or a secrets manager; commit only a placeholder/example file. For a companion service that needs a shared secret without an operator provisioning it, see `zero-config-secret-bootstrap`.
- Never log secrets, tokens, passwords, or PII — scrub known-sensitive fields and the auth header before anything gets written to a log or error tracker.
- Encrypt sensitive fields at rest using a vetted library/primitive, not a homemade cipher.

## Dependency hygiene

- Commit a lockfile; pin resolved versions so a build is reproducible.
- Run an automated vulnerability audit against dependencies in CI, and keep an automated update bot (Dependabot/Renovate-style) running so patches don't pile up.

## Pitfalls

- "Just this once" string-built SQL for a report or admin query — the exception is the one that gets exploited.
- Disabling a templating engine's auto-escape for convenience and forgetting to sanitize the one field that needed it.
- Trusting a client-supplied filename, extension, or `Content-Type` for anything touching the filesystem.
- Binding a whole request body onto a model instead of the validated, explicit field set.
- Deserializing untrusted data with a format that supports arbitrary type reconstruction.
- Debug mode, stack traces, or an admin/metrics endpoint left reachable in production.
- An outbound-fetch feature (webhooks, previews, proxies) with no check on where the resolved address actually points.

## Checklist

- [ ] All queries parameterized — no string-built SQL/NoSQL/shell commands, including raw-query escape hatches
- [ ] Output encoded per sink; auto-escaping left on, bypassed only for pre-sanitized content
- [ ] Input validated against an explicit schema at the boundary; unrecognized fields rejected, not passed through
- [ ] Untrusted data never deserialized with a code-executing format
- [ ] Outbound requests built from user input checked against an SSRF denylist before the fetch
- [ ] Debug mode off, security headers set, CORS is an explicit allowlist
- [ ] Secrets live in env vars / a secrets manager only — never logged, never committed
- [ ] Uploads validated by content, stored outside the web root, filenames randomized
- [ ] Dependency lockfile committed; automated vulnerability audit runs in CI
