---
name: outbound-webhooks
description: Use when your app must notify external systems of events — outbound webhooks with HMAC-signed payloads, a delivery log, and retries with backoff. Covers signing, replay protection, and safe delivery to untrusted URLs.
---

# Outbound Webhooks

Let customers subscribe to your events and receive signed HTTP callbacks — reliably and verifiably.

## When to use

Any time integrations need to react to things happening in your app (record created/updated, stage changed). The push counterpart to a polling API.

## Core pattern

```
webhooks(id, tenant_id, url, secret, events[], enabled, created_at)
webhook_deliveries(id, webhook_id, event, payload, status, attempts,
                   response_code, next_retry_at, created_at)
```

On a domain event: find enabled subscriptions matching the event, create a delivery row per subscription, enqueue it (see `background-jobs-scheduling`), and POST the payload.

## Signing (so receivers can trust you)

- Give each webhook a **secret** at creation.
- Sign every payload: `signature = HMAC_SHA256(secret, timestamp + "." + body)`, sent in a header (`X-Signature`, `X-Timestamp`).
- Receivers recompute and constant-time compare. Include the timestamp in the signed content and have receivers **reject stale timestamps** (replay protection).

## Reliable delivery

- **Retry with exponential backoff** on non-2xx / timeout; cap attempts, then mark failed.
- Record every attempt (status, response code, body snippet) in the delivery log for debugging and a customer-facing "recent deliveries" view.
- Support **manual redelivery** from the UI.
- Set aggressive **timeouts** — a slow receiver must not tie up a worker.

## Security (SSRF)

Outbound requests to user-supplied URLs are an **SSRF risk**:

- Block private/link-local/loopback ranges and metadata IPs (`169.254.169.254`, `10.*`, `127.*`, `::1`).
- Resolve DNS and validate the *resolved* IP (guard against rebinding), or route through an egress proxy that enforces this.
- Only allow `https`; don't follow redirects to disallowed hosts.

## Pitfalls

- No signature → receivers can't verify authenticity, and anyone can forge events.
- No replay protection → a captured payload can be resent.
- Synchronous delivery in the request path → a dead endpoint stalls your app.
- SSRF via internal URLs → treat every subscription URL as hostile.
- Unbounded retries → poison deliveries hammer a dead endpoint forever.

## Checklist

- [ ] Subscription + delivery-log tables
- [ ] HMAC-signed payloads with timestamp + replay rejection
- [ ] Async delivery, exponential backoff, attempt cap, manual redeliver
- [ ] SSRF guards on subscription URLs (block private ranges, https-only)
- [ ] Per-attempt logging + customer-visible delivery history
