---
name: outbound-webhooks
description: Use when your app must notify external systems of events — outbound webhooks with HMAC-signed payloads, a delivery log, and retries with backoff. Covers signing, replay protection, safe delivery to untrusted URLs, event-catalog design, and the self-amplifying feedback loop that a failing receiver creates.
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

## The event catalog is a design surface

- **Name events after decisions, not table writes.** Fire from the place that made the call — the route that recorded a deliberate answer, the code path that actually issued something — never from the data layer, which cannot tell a decision from a replay or a backfill.
- **A standing permission and the activity under it are two events.** "User approved app X" fires once, ever; the sign-ins that follow are silent. A subscriber to the grant alone sees a burst of activity the day the app is adopted and nothing afterwards, however heavily it is used. Emit both.
- **Cover the gaps, not just the edges.** A catalog with login/logout and nothing between it reports a silent gap followed by activity nobody can attribute to anyone arriving. An explicit "resumed after N hours" event is often the one subscribers actually want.
- **Emit management events for the webhooks themselves.** A subscription whose URL is quietly repointed is the one change that hides every other change. `webhook.updated` should carry the *field names* the request set — that is what separates a rename from a redirection — and the delete handler should read the row before removing it, so the event can say what stopped working instead of naming a dead id.
- **Never put a secret in a payload.** These payloads travel *over* webhooks, so their audience is every subscriber to the event — precisely the wrong audience for a signing key. Report `secretChanged: true` and nothing more.

## Feedback loops: the failure mode nobody predicts

If error/warn logs fan out as `system.error` / `system.warning` events, **anything a delivery logs is self-feeding**: warn → fire → deliver → warn → fire. A failing receiver then costs two outbound calls and two log lines per real event, and one bad relay can exhaust the whole fan-out budget for every other alert in the app.

- **The guard must be an async-context scope, not a boolean.** `let firing = true` set around a fire-and-forget call is cleared in the same synchronous tick, long before the delivery — or anything it logs — has run. A boolean cannot span an `await`. Open an `AsyncLocalStorage` (or equivalent) scope around each delivery attempt; every continuation beneath it, however deep, can see it is inside one.
- **Store that scope on a global symbol.** A dev hot-reload re-evaluates the module, and a second storage instance reports "not in a delivery" for a delivery opened by the first — silently reopening the loop.
- **Match log contexts by prefix, not by a list.** An exact-match allowlist of three context names goes stale the first time someone adds a `logWarn` to the delivery module. Suppress everything under `webhooks.*`.
- **Suppression is fan-out only.** The log row is still written and still shows in the admin log view. Silencing the *record* to break the loop trades one outage for a blind spot.
- **Cap the fan-out anyway.** A rolling-window limit on how many error/warn lines may become events is the backstop for the loop you did not foresee.
- **A single-event `event !== 'webhook.delivery_failed'` guard is not enough.** It closes the direct self-loop and none of the indirect ones.

## A dead receiver must not look green

- **`last_status_code` lies.** When the receiver never answers there is no status code, so the column stays null forever and the UI shows an integration that has failed 100% of the time for days as healthy. Derive displayed health from *consecutive failures*, not the last status.
- **Circuit-break per subscription.** After N consecutive failures stop attempting for a cooldown, log **one** line instead of N, and suppress the fan-out. The first success closes it. In-memory is the right default — no schema, no state to go stale — and the admin UI reads it to say "delivery suspended" rather than inventing its own health rule.
- **Update the breaker before the failure fan-out**, or the delivery that trips it fires the event the breaker exists to suppress.

## Delivery realities

- **`e.message` is "fetch failed" and nothing else.** Resolve the `.cause` chain before storing it: `ENOTFOUND` (dead URL) and a connect timeout (blocked egress) are different tickets and the raw message distinguishes neither.
- **Retry only on a transport failure or 5xx.** A 4xx is the receiver's considered verdict; repeating it just repeats the rejection.
- **Relays answer a POST with a redirect.** Apps Script and Zapier-style receivers reply `302` pointing at the result of the call they already ran, and `fetch` never chases a `Location` for a POST — so a working integration is recorded as a permanent failure. Follow **exactly one** hop by hand: re-run the SSRF checks on the target, make it a GET with no payload and no signature header (nothing is re-sent to a host nobody configured), and treat a second 3xx as the receiver's final answer. Keep `redirect: 'manual'` so the recorded status is the one actually produced.
- **Mask URLs in logs** — a token embedded in the path otherwise ends up in every log line and every log export.
- **Invalidate every cache key an event can use.** Install-wide events fire with no tenant id and cache under a `global` key; invalidating only `org:<id>` means a new subscription misses them and a deleted one keeps receiving.
- **Cache a *failed* lookup briefly too**, or a struggling database is asked the same dead question once per event.

## Testing

- Assert the loop is closed **across an `await`**, not just at the top of the call — that is exactly what the boolean version passed.
- Exercise a real delivery against a **loopback** receiver. The SSRF guard blocks loopback, so this needs a test-only seam that no configuration can enable; every other blocked range must still throw, which is what lets you assert that a `Location` pointing at `169.254.169.254` is refused.
- Cover `302 → 200` (delivered) and `302 → 500` (still failed).
- Watch for tests that self-skip without a database. The live half of a delivery suite can die the day an unrelated guard ships and nobody notices for months.

## Pitfalls

- No signature → receivers can't verify authenticity, and anyone can forge events.
- **A failing receiver feeding its own error log** — the single most expensive webhook bug, and it looks like a database problem until you read the log contexts.
- **Health derived from the last status code** rather than a failure streak.
- **A secret in a management payload**, broadcast to every subscriber of that event.
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
- [ ] Log-driven events cannot re-enter delivery: async-context scope, prefix-matched contexts, rolling fan-out cap
- [ ] Per-subscription circuit breaker; UI health from consecutive failures, not last status
- [ ] Transport errors resolved through the `.cause` chain; retry only on transport/5xx
- [ ] One hand-made redirect hop, re-validated, unsigned, GET
- [ ] Events fired from decision points; management events carry changed field names and no secrets
