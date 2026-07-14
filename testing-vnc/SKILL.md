---
name: testing-vnc
description: Use when verifying a remote-console/VNC (or similar out-of-band screen-access) feature for a provisioned instance — covers the console API returning a valid session URL/token, detecting and handling both native-protocol and WebSocket-proxied schemes, probing the connection handshake, token TTL and single-use enforcement, cross-tenant authorization (another customer must not obtain a console session for someone else's instance), direct-port exposure on the instance's own address, and behavior when the instance is stopped.
---

# Remote Console / VNC Access Testing

Verify the out-of-band console pathway — the "screen access that works even when the network stack or SSH is broken" feature — end to end: that the API actually issues a usable session descriptor, that a client can complete the protocol handshake with it, and that the authorization boundary around it holds.

## When to use

- Any product offering a browser-based or client-based remote console/screen (VNC, RDP-over-web, serial console) for a provisioned instance.
- Whenever a "console" or "VNC" button exists in the product UI and you need to confirm it does more than open a URL that never connects.
- Pairs with `testing-lifecycle` (the console endpoint's basic HTTP availability is part of the resource's overall correctness) and `testing-network` (port-level reachability of the underlying instance).

## Core pattern

**Don't assume the protocol — detect it from the response.** The console/session endpoint may return a native remote-framebuffer URL (`vnc://host:port`) or a WebSocket-proxied one (`ws://`/`wss://`). Parse the scheme out of whatever URL/token payload the API returns and branch the handshake probe accordingly: a native scheme gets an RFB-header probe on its port, a WebSocket scheme gets an HTTP upgrade / WebSocket handshake attempt. Forcing a WebSocket probe against a native-protocol URL (or vice versa) produces a false failure that has nothing to do with whether the feature actually works.

**Validate the response shape defensively.** Check for a URL/token under the field names the API might plausibly use, confirm a URL is present and matches one of the expected schemes, and treat a missing secondary field (token, expiry) as informational rather than an automatic failure until you've confirmed against the live API which fields it actually returns — don't hardcode assumptions about an unconfirmed response format.

**Attempt the actual handshake.** For a native protocol, a bare TCP connection should immediately return the protocol's version header; for a WebSocket proxy, the connection should complete the `101 Switching Protocols` upgrade. A console feature that only ever gets tested at "the API returned 200" misses the entire point — the URL/token generation step working is necessary but not sufficient.

**Test token/URL freshness.** Request a console session twice for the same instance and compare. If the token or URL differs each time, the product implements a single-use/rotating pattern (more secure); if identical, it's a static, reusable credential — document which, since this is a real security-relevant design choice, not just an implementation detail. If a TTL/expiry field is present, note whether it's an absolute timestamp or a relative duration.

**Verify cross-tenant authorization.** Have a second, unrelated account request a console session for the first account's instance. This must be rejected (403/404) — a console session handed to the wrong tenant is a full compromise of that instance, so this check deserves a critical severity if it fails, not a routine one.

**Probe for direct protocol exposure.** Check whether the instance's own address exposes the raw protocol port directly (bypassing whatever proxy/token gate the console API implies). If the product's design is "access only through the token-gated proxy," a directly reachable raw port is itself a finding, independent of whether the proxy works.

**Check console behavior on a non-running instance.** A stopped/powered-off instance's console request should not 500 — it should either issue a session that shows a boot/POST screen (if the platform supports it) or return a clear 4xx explaining why not.

**Exercise the actual client surface when one exists.** If there's a browser-based console UI, drive it: does the canvas/viewer element render, does a connection attempt appear in the network layer, does the status indicator move past "connecting," and do any console errors appear? An API-only pass can miss failures that only manifest in the client's protocol implementation.

## Pitfalls

- **Forcing one handshake style regardless of the returned scheme** — always detect the scheme first.
- **Treating "API returned a URL" as proof the feature works** — attempt the handshake.
- **Skipping the cross-tenant check** — this is the highest-severity bug class in this space and easy to overlook because the happy path looks fine.
- **Assuming the raw protocol port is closed on the instance** without checking — proxy-only designs need to verify that, not assume it.
- **500ing (or not testing) console access on a stopped instance.**
- **Hardcoding an assumed response shape** instead of confirming which fields the live API actually returns before asserting on them.

## Checklist

- [ ] Console/session endpoint checked for basic availability (not 404/500/401)
- [ ] Response scheme detected (native vs. WebSocket-proxied) and handshake probe branched accordingly
- [ ] Handshake actually attempted and its result (not just HTTP 200) recorded as pass/fail
- [ ] Token/URL freshness tested (two requests compared) and documented as single-use vs. static
- [ ] Cross-tenant console access confirmed rejected (403/404) — treated as critical if it isn't
- [ ] Direct protocol-port exposure on the instance's own address checked against the product's proxy-only claim
- [ ] Console request against a stopped instance verified not to 500
- [ ] In-UI console flow (canvas render, connection status, console errors) exercised when a client surface exists
