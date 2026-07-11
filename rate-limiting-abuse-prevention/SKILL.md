---
name: rate-limiting-abuse-prevention
description: Use to protect public endpoints (login, signup, forms, APIs) from brute-force and abuse — sliding-window per-IP rate limiting with automatic bans (fail2ban-style) and fail-closed CAPTCHA. Covers ban lifecycle and CAPTCHA verification.
---

# Rate Limiting & Abuse Prevention

Two complementary layers: **rate limiting / auto-ban** (slow down or block bad actors) and **CAPTCHA** (prove humanity on public entry points).

## When to use

Anything reachable without a session: login, password reset, signup, public forms, SMS/OTP send, public API. Also any endpoint that costs you money per call (SMS, AI).

## Rate limiting (fail2ban pattern)

Track events per key (usually IP, sometimes account or token) in a sliding window; when the rate crosses a threshold, ban the key for a cooldown.

```
jails: named rules → { window, max_events, ban_duration }
events(jail, key, created_at)          -- sliding-window counter
bans(jail, key, until, reason)         -- active bans
```

- On each request, record the event, count events in the window, and if over the limit **insert/extend a ban** and return `429` until `until`.
- **Different jails for different endpoints** — login failures deserve a stricter jail than general API calls.
- Count **failures**, not all attempts, for auth endpoints (a user typing a wrong password 3 times is fine; 50 is not).
- **Master switch** to disable the whole system quickly if a legitimate spike gets caught.
- Cache active bans in-process (short TTL) so the hot path doesn't hit the DB every request.
- Optionally feed repeated offenders to a network-level block (nginx / OS firewall) for cheap rejection.

## CAPTCHA (fail-closed)

- Put CAPTCHA on public, unauthenticated, abusable forms.
- **Provider registry** (reCAPTCHA / hCaptcha / Turnstile) selected by config, behind a common `verify(token)` interface.
- **Fail closed**: if verification errors or is misconfigured, **reject** the request. A CAPTCHA that fails open protects nothing.
- Verify server-side; the client widget alone proves nothing.

## Pitfalls

- **Keying only on IP** punishes users behind shared NAT/proxies — combine with account/username where possible, and set thresholds generously.
- Trusting `X-Forwarded-For` blindly — only trust it from your own proxy hop.
- CAPTCHA that fails open on provider outage — decide deliberately (usually fail closed for signup, maybe degrade for login).
- No unban path — always have ban expiry + a manual unban control.
- Rate-limiting *cost* endpoints (SMS/AI) by IP only; also cap per-account spend.

## Checklist

- [ ] Sliding-window counters + auto-ban with expiry, per-jail thresholds
- [ ] Count failures for auth endpoints; 429 with retry info
- [ ] Master kill-switch + in-process ban cache + manual unban
- [ ] CAPTCHA on public forms, server-verified, fail-closed
- [ ] Trusted-proxy handling for client IP
