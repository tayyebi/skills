---
name: share-links-tracking
description: Use when users need to share a record or file with someone who has no account — expiring, hashed, revocable share links, plus optional open/view tracking. Covers capability-URL security and the tracking-pixel pattern.
---

# Share Links & Open Tracking

Give outsiders scoped, temporary access to one resource via a URL — no account required — and optionally know when they viewed it.

## When to use

"Send this candidate/document/report to a hiring manager (or client) without inviting them to the app." Also public preview links with an expiry.

## Share links (capability URLs)

The URL itself *is* the credential ("capability URL"), so treat it like one:

- **Unguessable token**: a long random token in the URL. Store only its **hash** server-side (`SHA-256`), like an API token — a leaked DB row can't reconstruct live links.
- **Scoped**: each link grants access to exactly one resource and one action (view). Never a broad session.
- **Expiring**: set `expires_at`; reject after. Short by default; let the sharer choose.
- **Revocable**: a `revoked_at` column; check it on every access. Sharers need an "un-share" button.
- **Optional gate**: password, email-verification, or view-count cap for sensitive shares.

```
share_links(id, token_hash, resource_type, resource_id, tenant_id,
            created_by, expires_at, revoked_at, max_views, view_count)
```

Because it's a capability URL: it can be forwarded, appears in browser history, and may hit referer headers — keep expiries short, avoid putting secrets in the path where it lands in logs, and set `Referrer-Policy: no-referrer` on shared pages.

## Open / view tracking

- **Web view**: increment `view_count` and log an access row (timestamp, IP, UA) when the shared page loads.
- **Email open tracking**: embed a 1×1 transparent pixel whose URL carries a per-message token; the request to it records an "open." Store `email_tracking(message_id, …)` and `email_tracking_opens(token, opened_at, ip)`.
  - Caveat: image-blocking and privacy proxies make opens **undercounts** — treat as a signal, not truth. Disclose tracking per privacy norms/regulation.

## Pitfalls

- Storing the raw token instead of its hash — a DB leak becomes a link leak.
- No expiry/revocation — a forwarded link lives forever.
- Guessable/sequential ids in the URL — enumeration exposes every resource.
- Leaking the token via `Referer` to third-party assets on the shared page.
- Treating email opens as accurate — they're systematically low.

## Checklist

- [ ] Long random token in URL; only its hash stored
- [ ] Per-resource scope, `expires_at`, `revoked_at` checked on access
- [ ] Optional password / view-cap for sensitive shares
- [ ] `Referrer-Policy: no-referrer`; no secrets in logged paths
- [ ] View logging; email open pixel (treated as an undercount + disclosed)
