---
name: authentication-sessions
description: Use when adding user login to an app — session management with signed cookies, multiple sign-in providers (password, OAuth, chat-app, SMS OTP), and two-factor auth. Covers the "self-hiding provider" pattern and secure session verification.
---

# Authentication & Sessions

Build authentication without a heavyweight framework: a signed session cookie plus pluggable sign-in providers.

## When to use

Any app with user accounts. Reach for this instead of a full auth SaaS when you want to own the session format, support several login methods, and avoid vendor lock-in.

## Core pattern

**Session = a signed, self-describing cookie.** Store the minimum identity in the cookie payload (`userId`, `tenantId`, issued-at), then sign it with HMAC-SHA256 using a server secret. No server-side session store is needed for the common case; the signature is the trust anchor.

```
cookie value = base64url(payload) + "." + base64url(hmac_sha256(secret, base64url(payload)))
```

- Verify on every request: recompute the HMAC, constant-time compare, reject on mismatch or expiry (e.g. 7-day TTL).
- Set cookie flags: `HttpOnly`, `Secure`, `SameSite=Lax`, scoped `Path=/`.
- Keep a short (~30s) in-process cache of the user row keyed by `userId` so per-request auth doesn't hammer the DB — but always re-check `disabled` status.

## Sign-in providers (pluggable)

Model each method as a provider that ultimately resolves to (or creates) a user and issues a session:

- **Password** — bcrypt/argon2 hash at rest, never log the plaintext.
- **OAuth (Google, etc.)** — standard code-exchange flow; store the external `sub`/id on the user row to link accounts.
- **Chat-app login (Telegram widget, etc.)** — verify the signed payload from the provider before trusting it.
- **SMS one-time code** — short-lived numeric code, rate-limited, single-use.

### Self-hiding providers

Each provider reads its own config (client id/secret, API keys) from settings. If unconfigured, the provider **hides itself** from the login UI rather than erroring. This lets you ship every method but only surface the ones an operator has actually set up. Expose an endpoint that returns the list of *enabled* providers and render the login page from that.

## Two-factor (TOTP)

- Generate a per-user secret; show it as an `otpauth://` QR code.
- Verify 6-digit codes with a small time window (±1 step) using a TOTP library.
- Issue single-use **backup recovery codes** (hash them at rest) so a lost authenticator isn't a lockout.
- Gate 2FA setup/disable behind a fresh password re-entry.

## Pitfalls

- **Don't put mutable authority in the cookie.** Roles/permissions should be looked up fresh (or cached briefly), not trusted from the cookie — otherwise a demoted user keeps their old powers until expiry.
- **Constant-time compare** the signature; a naive `==` leaks timing.
- **Rotate the signing secret** deliberately: support two valid secrets during rotation so live sessions survive.
- Always re-check `user.disabled` on each request — a signed cookie outlives an account being turned off.
- Rate-limit every credential-accepting endpoint (see `rate-limiting-abuse-prevention`).

## Checklist

- [ ] HMAC-signed cookie, HttpOnly + Secure + SameSite
- [ ] Constant-time signature verification, expiry check
- [ ] Password hashing (bcrypt/argon2), never logged
- [ ] Providers self-hide when unconfigured
- [ ] TOTP 2FA + hashed backup codes
- [ ] Disabled-user check on every request
- [ ] Secret rotation strategy
