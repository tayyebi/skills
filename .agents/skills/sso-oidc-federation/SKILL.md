---
name: sso-oidc-federation
description: Use when an app must accept sign-in from external identity providers (Keycloak, Entra, Google, Cloudflare Access) alongside local accounts — OIDC discovery and PKCE, one-user-many-identities linking, JIT provisioning, group→role mapping, and IdP-outage break-glass.
---

# SSO / OIDC Federation

Let people sign in with the organisation's identity provider **without** giving up local accounts. One user, many identities; providers configured as data, not code.

Extends `authentication-sessions` (which covers the session itself and the self-hiding-provider idea) with the federation half: verifying an external assertion, deciding which local account it belongs to, and deriving authorization from it.

The mirror direction — your app *issuing* identity to third-party applications — is `oidc-provider`.

## When to use

- More than one organisation, or one organisation with an existing IdP, needs into your app.
- You want SSO **and** a way in when the IdP is down.
- Access should follow IdP group membership rather than being maintained twice.

Skip it if a single shared IdP is the only login method you will ever have and lockout is not a concern — then a plain OAuth client is enough.

## Core pattern

**Providers are rows, not classes.** One generic OIDC implementation; "Keycloak", "Okta", "Cloudflare Access" are *presets* that prefill an issuer shape and a claim path. Adding a provider is a config change.

```
auth_providers(id, key, kind, preset, display_name, enabled,
               issuer, client_id, client_secret_enc, scopes,
               discovery_cache jsonb, discovery_fetched_at,
               claim_source, auto_provision, allowed_domains[],
               enforce_local_2fa, logout_url)
```

**One user, many identities.** Credentials live on the identity, never on the user:

```
users(id, email, display_name, disabled)          -- no credential columns
identities(id, user_id, provider_key, subject, email, email_verified, claims jsonb,
           password_hash, totp_secret_enc, totp_enabled, last_login_at)
           UNIQUE(provider_key, subject)
```

`provider_key='password'` is just another identity. That single decision is what makes "link Keycloak to my existing account", "sign in either way", and "break-glass" all fall out for free.

**The flow**, written out rather than delegated, because the checks *are* the feature:

1. Discovery: fetch `<issuer>/.well-known/openid-configuration`, verify the advertised `issuer` matches what you configured, cache it **in the database** (shared across replicas, survives restart), serve stale on fetch failure rather than locking everyone out.
2. Authorization request: `code` flow with **PKCE** (`S256`), plus `state` and `nonce`. Store all three in a short-lived signed cookie — nothing to persist, nothing to clean up.
3. Callback: constant-time compare `state`, exchange the code with the `code_verifier`.
4. Verify the `id_token` — **all five**, every time:
   - signature, against JWKS fetched fresh (providers rotate keys)
   - `iss` equals the configured issuer
   - `aud` contains your `client_id` (it is legitimately a list)
   - `exp` in the future
   - `nonce` equals the one you issued
   Restrict the accepted algorithms to an explicit allowlist — this is what stops `alg: none` and HMAC-confusion.

## Design decisions

- **Never auto-link on an unverified email.** Linking an IdP subject to an existing account by matching addresses is the convenient path and the takeover path: anyone who can register that address at a loosely-configured IdP inherits the account. Auto-link only when the IdP asserts `email_verified` *and* the domain is on that provider's allowlist. Otherwise require the user to sign in by their existing method and link deliberately from their profile.
- **JIT provisioning off by default.** "Anyone at the IdP gets an account" is a decision, not a default. Ship `auto_provision=false`, and provision with **no grants** — existing is not the same as authorised.
- **Recompute group→role mappings on every login.** A role removed at the IdP must disappear at the next sign-in, not linger until someone audits. Store IdP-derived grants with `source='idp'` and reconcile only those — an administrator's manual grant must not be collateral damage of a sync.
- **Claim paths are configuration.** Keycloak puts realm roles at `realm_access.roles` and client roles at `resource_access.<client>.roles`; most others use a flat `groups`. A dotted-path reader plus a config field beats a per-provider adapter.
- **Delegate MFA only when the IdP says so.** `amr` containing `mfa`/`otp`/`hwk` is an assertion you can act on. Absence of the claim is *not* evidence MFA happened — let a per-provider `enforce_local_2fa` flag decide whether to still demand your own second factor.
- **Sessions stay yours.** Exchange the assertion for your own session record; do not treat the IdP token as the session. You need revocation, `disabled`-user checks, and a consistent session table regardless of how someone signed in.
- **Back-channel logout.** Store the IdP's `sid` on the session row and expose a logout endpoint that verifies the logout token and revokes matching sessions. Without it, signing out of the IdP leaves a live session in your app — the exact gap SSO is meant to close.

## Break-glass

**At least one privileged local account must always survive.** If the IdP is the only way in and it goes down — or its config is wrong, or the client secret expired — nobody can fix anything, including the IdP integration itself. Enforce it in code:

> refuse to remove the last password identity (with a second factor) from the last active owner

Say why in the error message. Someone will hit this and think it is a bug.

The failure mode is worse the closer your app sits to infrastructure: a tool that manages DNS, and can only be reached through an IdP that resolves via that DNS, has a circular dependency waiting for a bad afternoon.

## Provider notes

- **Keycloak** — confidential client, standard flow. Roles at `realm_access.roles` (realm) or `resource_access.<client>.roles` (client). Supports back-channel logout; configure the URL in the client.
- **Cloudflare Access** — there is **no** public "sign in with Cloudflare" for third-party apps. Two real options: a Zero Trust *SaaS application* with OIDC (discovery at `https://<team>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client-id>/.well-known/openid-configuration`), or *proxy mode* where Access fronts the app and injects `Cf-Access-Jwt-Assertion`, verified against `https://<team>.cloudflareaccess.com/cdn-cgi/access/certs` with an audience-tag check.
- **Entra ID / Google** — generic OIDC. Entra needs the tenant-scoped issuer; both need `groups` requested explicitly and, at scale, group-overage handling.

## Pitfalls

- **Skipping `nonce`** because state "already covers CSRF" — it does not cover token replay from a different sign-in.
- **Caching JWKS forever.** Rotation then breaks every sign-in until a restart. Fetch per verification, or cache with a short TTL and refetch on unknown `kid`.
- **Trusting `email` as an identifier.** The stable key is `(provider_key, sub)`. Emails get reassigned; `sub` does not.
- **Echoing the token endpoint's error body** into a log or a UI — you just sent it your client secret.
- **Storing the IdP access token** you have no use for. If you are not calling the provider's API, keep the identity claims and drop the token.
- **Testing only the happy path.** The value is in the rejections: wrong issuer, wrong audience, expired, replayed nonce, unknown signing key, tampered payload. Mint tokens with a local key in tests and assert each is refused.
- **One provider hard-coded as "the" SSO.** The second one always arrives.

## Checklist

- [ ] Providers stored as data; presets only prefill
- [ ] One user, many identities; `UNIQUE(provider_key, subject)`
- [ ] Discovery cached in shared storage, issuer verified, stale-on-failure
- [ ] PKCE + state + nonce; flow state in a short-lived signed cookie
- [ ] `id_token` checked for signature, `iss`, `aud`, `exp`, `nonce`, restricted algorithms
- [ ] Auto-link requires verified email **and** allowlisted domain
- [ ] JIT provisioning opt-in, provisioned users start with no grants
- [ ] Group→role mappings recomputed per login; manual grants untouched
- [ ] MFA delegated only on an explicit `amr` assertion
- [ ] Back-channel logout revokes local sessions by `sid`
- [ ] Last local privileged account cannot be removed (break-glass)
- [ ] Rejection paths covered by tests, not just the happy path
