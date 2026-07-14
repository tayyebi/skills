---
name: testing-integration
description: Use when testing the seams between a product and the third-party services it depends on — payment gateway connectivity (cards, redirects, 3DS, crypto), incoming/outgoing webhook signature validation, transactional email delivery and authentication headers (SPF/DKIM/DMARC), SSO/OIDC login, SCIM provisioning, and graceful degradation when an external dependency is down or slow. Tests from the outside: trigger a flow and observe what comes out the other side. Distinct from testing-billing (wallet/ledger arithmetic correctness) and testing-security (webhook forgery/SSRF as an attack, not a connectivity check).
---

# Third-Party Integration Testing

Test the boundary, not the business logic. Every product depends on services it doesn't control — a payment processor, an email provider, an identity provider, a customer's own webhook endpoint. This skill verifies those seams work, fail safely, and can't be forged, without re-testing the internal logic those integrations feed into (that's testing-billing for money math, testing-security for adversarial signature/SSRF attacks).

## When to use

- Testing payment flows (card, redirect-based, or crypto), transactional email, SSO/SCIM, or any customer-facing webhook feature.
- After adding or reconfiguring any external provider, or rotating its credentials/signing secret.
- As part of an incident postmortem when a third-party outage caused unexpected product behavior — verify the degradation path actually matches what was assumed.

## Map the integration surface first

Before testing, enumerate what's actually in scope and its direction:

| Integration class | Direction | Typical protocol |
|---|---|---|
| Payment gateway(s) | outbound (charge) + inbound (webhook) | REST + signed webhooks |
| Transactional email | outbound only | SMTP/API |
| Customer-defined webhooks | outbound to a URL the customer controls | HTTP POST |
| SSO | inbound (an external IdP authenticates your user) | OIDC/SAML |
| SCIM provisioning | inbound (an external IdP manages your users) | SCIM 2.0 over REST |

## Core pattern

**1. Payment gateway happy paths — sandbox only.** Use each provider's published test card numbers / sandbox credentials (a successful card, a declined card, an insufficient-funds card, and a 3DS-challenge card, if the provider supports one). For each: add as a payment method, fund a balance, and complete a checkout — confirming the provider's own object id is genuinely present in the response (a real charge/session id, not a locally-fabricated stand-in), the resulting balance or order state updates, and a declined/failed attempt produces a clear error without silently succeeding. If a gateway can run in a demo/sandbox mode with no real credentials configured, explicitly verify whether it's actually reaching the provider or just simulating success — check for a real provider-issued id in the response.

**2. Incoming webhook signature validation.** Every gateway signs its webhooks; the signature must be verified server-side before the event is trusted. Send a request with no signature header, then one with a malformed signature — both must be rejected (400/403), never processed. If you have a valid sandbox signing secret, also send one correctly signed and confirm it *is* accepted — a validator that rejects everything isn't actually validating. Also test replay: resend a previously-valid, already-processed signed payload and confirm it's rejected or safely no-ops (idempotency), not double-applied.

**3. Outgoing webhook (customer-subscribed events).** Register a webhook pointed at a receiver you control, trigger the subscribed event, and confirm: delivery arrives within a reasonable window; the payload matches the documented event schema; a failing receiver (temporarily returning 5xx) triggers a retry with backoff, visible in a delivery log; deleting the subscription stops further deliveries. Also confirm URL validation on registration — accepting `http://localhost/...` or other internal-network targets is an SSRF risk (forgery/exploitation depth belongs to testing-security; this skill just confirms the validation exists).

**4. Transactional email — delivery and content.** Trigger each transactional email (registration/welcome, password reset, invoice, notification) to an inbox you control and check: it arrives within a couple of minutes; the subject is specific, not generic; links work; a password-reset or magic link is single-use, expiring, and delivered over HTTPS; a plain-text part exists alongside HTML; no more PII than necessary appears in the URL versus the body.

**5. Email authentication headers.** Open the raw source of a received message and read the `Authentication-Results` header (or platform equivalent): SPF should be `pass` (not `softfail`/`neutral`/`none`), DKIM should be `pass` with the signing domain aligned to the From domain, and DMARC should `pass` with a published policy of `p=quarantine` or `p=reject` (check via `dig TXT _dmarc.<domain>`) — a `p=none` policy means spoofed mail in your domain's name won't be rejected by receivers, which is worth flagging even though it's a DNS config, not application code.

**6. SSO / OIDC.** Probe the SSO entry point unauthenticated to see what it accepts and where it redirects. If configured end-to-end, walk the full flow (initiate → IdP login → redirect back → session established), and specifically test: an email that has both an SSO and a local-password account (which wins?), an expired/invalid IdP assertion (clean error, not a crash), and whether a role/claim asserted by the IdP is actually honored on the product side.

**7. SCIM provisioning.** Confirm the discovery endpoint (`ServiceProviderConfig`) is reachable and accurately describes supported features. With credentials, test list/create/update/deprovision — deprovisioning must actually disable the account (not just remove it from a list), and provisioned users must be able to log in normally afterward. Establish empirically (don't assume) what credential type SCIM actually accepts — a dedicated provisioning token versus a reused session credential is a meaningfully different security posture, worth documenting once confirmed.

**8. Graceful degradation.** For each dependency, reason through (and where safely possible, actually trigger) what happens when it's unreachable: does a failed gateway call leave a clear user-facing error without touching the wallet; does a failed webhook delivery retry instead of silently dropping; does a failed email send queue/retry rather than blocking the triggering action (unless that action is explicitly gated on email, e.g. verification); does a failed identity-provider call fail the whole provisioning step cleanly instead of leaving a partial record. Large-scale fault injection across many dependencies at once belongs to testing-chaos — this is a per-integration check.

## Reporting format

```
Integration:   Payment gateway webhook handler
Issue:         No signature validation on incoming payment-succeeded events
Test:          POST to the webhook endpoint with a valid-shaped payload and no signature header
Expected:      400/403 -- rejected for missing/invalid signature
Actual:        200 -- wallet balance for the referenced user increased
Impact:        Any unauthenticated caller can credit funds to an arbitrary account
Severity:      Critical
Fix:           Validate the provider's signature (HMAC over payload + timestamp) before processing
```

## Pitfalls

- Testing a "demo mode" gateway integration and reporting it as verified without confirming a real provider-issued id was ever returned.
- Treating an accepted webhook with no signature as merely a config gap instead of a critical, money-affecting vulnerability.
- Never testing the retry/backoff path for outbound webhooks — only ever testing the success case.
- Assuming SPF/DKIM passing means DMARC passes too — alignment (matching From domain) is a separate check from raw pass/fail.
- Skipping the "email + SSO both exist for the same address" edge case, which routinely produces an account-takeover or duplicate-account bug.
- Confusing this skill's scope with testing-billing — a passing payment-gateway test here says nothing about whether the resulting ledger math is correct.

## Checklist

- [ ] Sandbox payment flows exercised: success, decline, insufficient funds, and (if supported) 3DS challenge
- [ ] "Demo mode" gateways confirmed to either genuinely call the provider or be clearly labeled as simulated
- [ ] Incoming webhooks: unsigned and malformed-signature requests rejected; correctly-signed accepted; replay rejected/idempotent
- [ ] Outgoing webhooks: delivery timing, payload schema, retry-on-failure, and unsubscribe-stops-delivery all verified
- [ ] Webhook URL registration rejects internal/loopback targets
- [ ] Transactional emails: delivery time, content quality, working links, single-use/expiring reset links
- [ ] SPF/DKIM/DMARC all `pass` and aligned on a real delivered message; DMARC policy checked via DNS
- [ ] SSO flow walked end to end; dual-account and invalid-assertion edge cases checked
- [ ] SCIM create/update/deprovision verified; deprovisioned users actually can't log in
- [ ] At least one graceful-degradation scenario per critical dependency reasoned through or triggered
