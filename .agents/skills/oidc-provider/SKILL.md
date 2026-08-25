---
name: oidc-provider
description: Use when your app must BE the identity provider — "sign in with <this workspace>" for third-party apps — covering client registration, the authorize/token/userinfo surface, opaque rotating tokens, consent a user can withdraw, tenant claims standard OIDC has no word for, and the public integration guide.
---

# Being an OpenID Connect Provider

The mirror of `sso-oidc-federation`: that skill consumes an external assertion, this one *issues* it. Your install becomes the OP, and other applications become relying parties that sign their users in with it.

The two directions look symmetric and are not. As a client you verify what someone else asserts and can be wrong only about yourself. As a provider you are the thing everyone else trusts, on a surface where each mistake has a named exploit — open redirector, code interception, token replay, `alg: none`. Read the RFCs rather than a blog post: the ordering of the checks *is* the feature.

## When to use

- Other applications — first-party tools, a customer's internal app — need to authenticate *your* users, and you would rather issue identity than mint another bespoke token scheme.
- You want a "connected applications" screen where a person can see and withdraw what they approved.
- Machine-to-machine access is already solved by API tokens; what is missing is *acting as a specific user*.

Skip it if every integration is a backend job with no end user. That is an API token, and adding an OP for it is strictly more surface for the same result.

## Scope of the build

Eight endpoints and one screen:

```
GET  /.well-known/openid-configuration   discovery — the whole integration surface
GET  /.well-known/jwks.json              public keys
GET  /oauth/authorize                    where a person says yes
GET  /oauth/consent  (+ POST api)        the screen they say it on
POST /oauth/token                        code → tokens, refresh → tokens
GET  /oauth/userinfo                     claims for a bearer token
POST /oauth/revoke                       RFC 7009
POST /oauth/introspect                   RFC 7662, for confidential clients
```

Support `authorization_code` + PKCE and `refresh_token`. Nothing else. No implicit, no hybrid, no `client_credentials` (that is your API-token feature wearing a hat), no dynamic registration. **Advertise exactly what works** — a discovery document that overstates support is worse than none, because the client library believes it and fails at the least convenient moment.

## Data model

Five tables, one prefix:

```
oidc_clients(id, client_id, secret_hash, name, description, logo_url, homepage_url,
             is_confidential, redirect_uris[], scopes[], restrict_to_org_id,
             require_consent, disabled_at, created_by, created_at, last_used_at)
oidc_codes(code_hash PK, client_id, user_id, org_id, scopes[], redirect_uri,
           code_challenge, code_challenge_method, nonce, auth_time,
           expires_at, consumed_at)
oidc_tokens(id, token_hash, kind('access'|'refresh'), client_id, user_id, org_id,
            scopes[], parent_id, auth_time, expires_at, revoked_at, last_used_at)
oidc_grants(id, client_id, user_id, scopes[], granted_at, revoked_at)
oidc_signing_keys(kid PK, alg, private_pem, public_jwk, created_at, activated_at, retired_at)
```

**Pick the prefix against what already exists.** If you have an `oauth_tokens` table holding tokens *you* hold *for* Google, do not add tokens you issue to the same name — one prefix per direction, so no reader has to work out which way the arrow points.

**Codes and tokens are stored hashed**, never plaintext: a database read must not be replayable into a session. Same discipline as API tokens.

## The parts that carry the security

**Redirect URIs match exactly.** Not by prefix, not by origin, no wildcard path. The code is *delivered* to that URL. Refuse to register anything that is not an absolute http(s) URL with no fragment (a fragment swallows the query the code rides in); refuse plaintext http except on loopback.

**Order the authorize endpoint the way the spec demands, not the way it reads.** Resolve the client and validate `redirect_uri` *first*; only then may anything be reported back on it. Everything failing before that point renders on your own error page — the only URL available to redirect to is one the caller supplied and you have not vouched for. Backwards, this is an open redirector for phishing.

**PKCE S256, mandatory, public and confidential clients alike.** Reject a missing `code_challenge`, reject `plain` (omitting the method *means* plain per RFC 7636), validate the challenge shape, and enforce the RFC's 43–128 verifier bounds — a one-byte verifier hashes to something perfectly valid and protects nothing.

**Single use is enforced in SQL, not in application logic.** Consumption is one `UPDATE ... WHERE consumed_at IS NULL ... RETURNING`, and the returned row is the only way the caller learns the code was valid. Two racing token requests cannot both win, because Postgres serialises the update. The same shape spends a refresh token.

**Return null for every code failure** — unknown, expired, already spent. A client has no business learning which.

**Reject `alg` before reading anything else** in any verifier you write, `none` included.

## Design decisions

- **Access tokens opaque and hashed, not JWTs.** One indexed lookup per userinfo call buys *instant revocation* — and instant revocation is the one behaviour a "connected apps" screen must have. A self-contained JWT keeps working after the user withdrew consent, which makes the screen a lie. Keep JWTs for the `id_token`, which is read once at sign-in and can be short-lived (10 min).
- **Refresh tokens rotate, and a replay revokes the family.** Presenting a spent refresh token is either a replay or a thief racing the legitimate holder, and there is no way to tell which — so revoke the whole chain and make both sign in again. Noisy on purpose; the alternative is permanent access for whoever stole it. Distinguish "spent" from "never existed" before nuking: only the first is evidence.
- **The signing key lives in the database, never in `.env`.** If your deploy regenerates env files, a key kept there silently rotates on an unrelated push and invalidates every `id_token` in the wild, with no diff to explain it. In the DB it survives a rebuild and rotation becomes deliberate. Mint the first key **lazily**, not as a seed step: SQL cannot generate RSA, and a fresh install and an upgrade should converge without ordering. Two boots racing both insert a key and both get published — the loser is a harmless spare, because verification is by `kid`.
- **Rotation is a non-event.** Keep the retired key in the JWKS for a grace window comfortably longer than the `id_token` lifetime, so a token signed a second before the switch still verifies. Use the RFC 7638 thumbprint as the `kid`: derived from the key, stable, collision-free.
- **Existing session-invalidation controls must reach in.** If a password change sets `sessions_valid_after`, refuse tokens whose `auth_time` predates it. Otherwise the OP is a documented way around the control that exists to end a compromised session.
- **No dependency for the JWT.** RS256 signing, verification and JWK export are ~60 lines on the standard crypto library. One less thing the deploy must install for logins to work.
- **Consent accumulates, it does not replace.** An app that had `openid profile email` and now wants `org` re-prompts for the addition; returning to a subset does not prompt. Compare the request against the *union*.
- **Revoking a grant kills the app's live tokens in the same request.** A connections screen that only forgets the consent, leaving a working refresh token behind, tells the user something untrue.

## Multi-tenancy

Standard OIDC has no notion of a tenant, and a relying party integrating with a multi-tenant app almost always needs one — otherwise it has an email address and no idea what to show.

- **One issuer for the whole install, not one per tenant.** N discovery documents and N key sets describing the same authentication is a cost with no buyer. The tenant travels in the claims instead.
- **A non-standard `org` scope** carrying `org_id`, `org_slug`, `org_name`, `role`. Unprefixed claim names are fine for a single-issuer OP built to be integrated against (they cannot collide with RFC 7519 §4.1 registered names); namespace under a URI only if you are a federation hub.
- **Register clients at the install altitude, not inside a tenant's settings.** That placement *is* the enforcement: a client with no tenant restriction can authenticate a user from **any** org on the install, which is not a call one tenant's admin gets to make for the others. Offer an explicit `restrict_to_org_id`, and enforce it at authorize.
- **Derive claims from the role, not from a nullable column.** An operator/super-admin promoted out of a workspace can still carry a stale `org_id`, and an `org_*` claim on them reads to a relying party as membership they do not have. Give them a roster of the workspaces they may act on instead — *membership OR roster, never both*, so `'orgs' in claims` identifies the operator as cleanly as `'org_slug' in claims` identifies a member. List only *active* workspaces, and narrow the roster to a restricted client's own tenant: signing into a single-tenant integration must not hand it the name of every tenant on the install.
- **Put the role claim on `profile`, not only on the tenant scope.** The default client scopes do not include your non-standard one, so an ordinary "sign in with us" integration otherwise authenticates a user and never learns what they may do.

## The surfaces around the protocol

- **A public, unauthenticated integration guide**, named as `service_documentation` in discovery. The developer wiring an app up is often not a user of your product; gating the docs behind a login they do not have is how integrations get guessed at instead of read. Nothing on it is secret — every URL it prints is already in the discovery document, which must be public for OIDC to work at all. Keep it in one language, like the OpenAPI spec beside it: a half-translated curl example helps nobody.
- **State the two facts that bite** right there: claims that are *absent* rather than null for some roles, and that a refresh token rotates — so retrying a failed refresh with the old value looks exactly like a leak and revokes the chain.
- **Two screens at two altitudes**: operator registration (install scope) and a per-user "connected applications" list. The second is what lets the consent screen honestly promise the user can withdraw later.
- **Empty state beats a hidden section.** "No applications are connected" tells a user the revoke surface exists and is currently empty. Hide the section only when the install has no client they *could* ever connect — a distinction only the server can make, so return it from the API alongside the list.
- **Events for the operator.** `consent_granted` / `consent_denied` / `consent_revoked` / `signed_in`. Keep the last one separate from the first: consent stands once given, so approval fires once per app while every subsequent sign-in is silent — subscribing only to the grant shows a burst of activity the day an app is adopted and nothing afterwards, however heavily it is used. Fire them where the audit lines sit (a deliberate answer, a code actually issued), never from the data layer, which cannot tell a decision from a replay.

## Deployment traps

- **Do not build same-origin redirects from the request's own origin.** Behind a reverse proxy the app sees its bind address, so `https://0.0.0.0:3000` ends up in a `Location` header. Resolve through the same issuer resolver the discovery document uses — one place decides the canonical host. It bites only the URLs you build; the redirect back to a registered client uses the client's own absolute URI and is never wrong.
- **The issuer string must be stable.** It is compared verbatim by every client library, so a configured base URL wins over the request host — an install reachable on both a canonical domain and a customer's custom domain must not mint tokens claiming two issuers.
- **Front-end routing needs two different rules.** Endpoints that read the session cookie (`authorize`, `consent`) belong on the canonical host and may be redirected there. `token`, `userinfo`, `revoke` and `introspect` must **not** be — a 307 to another host is a redirect most HTTP clients will not follow with a POST body intact. And exempt the whole OAuth surface, `.well-known` and the docs from any domain-parking or vanity-domain rewrite, or a token request on a custom domain gets rewritten into that customer's public page.
- **Never log a credential-shaped value that failed to resolve.** The classic support ticket is a client *secret* pasted into the `client_id` field — the two differ by a prefix and sit next to each other in every setup form. Detect it and say so in the error; redact the value, because warnings persist and fan out to webhooks, which turns a config mistake into a leaked credential.

## Testing

The endpoints are thin; everything that can go quietly wrong is in the library. Test that against a real database, and test the **rejections**:

- a code consumes once, and a second attempt returns nothing
- an expired code never consumes
- PKCE with a wrong verifier, and with a too-short one
- exact redirect matching: trailing slash, bare origin, different path, attacker host — all refused
- refresh rotation issues a new token; replaying the spent one revokes the descendants
- a password change invalidates already-issued tokens
- consent accumulates rather than replaces
- rotation: the retired key still verifies an older token, the JWKS never carries a private half
- `alg=none` and a tampered signature are refused

Exercise both paths a deploy can take — a fresh install, and an upgrade with the tables dropped and the migration journal row removed.

## Pitfalls

- **Redirecting an error before the redirect URI is trusted.** The single worst bug on this surface.
- **JWT access tokens for convenience.** You will not notice until the first person asks why disconnecting an app did nothing.
- **The signing key in an env file**, on a pipeline that regenerates env files.
- **`plain` PKCE, or PKCE only for public clients.** It costs a client library one line and closes code interception for everyone.
- **Advertising `client_credentials` because the metadata example had it.**
- **A `require_consent=false` first-party client with no connections screen** — nothing anywhere tells the user the app has access.
- **Forgetting that codes and tokens accumulate.** Prune spent and expired rows.

## Checklist

- [ ] `authorization_code` + mandatory PKCE S256 and `refresh_token`; nothing else advertised
- [ ] Client resolved and `redirect_uri` matched exactly *before* any error can redirect
- [ ] Codes and tokens stored hashed; single use enforced by the UPDATE's WHERE clause
- [ ] Access tokens opaque; refresh tokens rotate with family revocation on replay
- [ ] Signing key in the database, minted lazily, retired keys published through a grace window
- [ ] Session-invalidation controls (`sessions_valid_after`) reach issued tokens
- [ ] Tenant claims derived from role, not a nullable column; roster narrowed for restricted clients
- [ ] Client registration at install altitude with an explicit tenant restriction
- [ ] Consent accumulates; revoking a grant kills that client's live tokens
- [ ] Public integration guide, linked as `service_documentation`
- [ ] Issuer stable and proxy-aware; token endpoints exempt from host redirects and domain rewrites
- [ ] Rejection paths tested against a real database, on fresh install and upgrade
