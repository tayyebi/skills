---
name: testing-ssh
description: Use when verifying SSH access to a provisioned VM/instance actually works end-to-end — covers whether a key uploaded to the control-plane API is really injected onto the machine (vs. merely accepted into the API's own database), public-key vs. password auth, host-key consistency across reconnects and its expected change (or non-change) after reboot vs. rebuild, weak cipher/MAC/KEX absence, root-login policy, and key-revocation semantics. Requires a running instance with an assigned address.
---

# SSH Access Testing

Prove that SSH access works the way the product claims, not the way the control-plane API's response implies. The single most valuable check in this space is whether a key the API "accepted" actually lands in the instance's authorized-keys file — a huge class of bugs live exactly at that seam, where the control plane and the provisioned machine silently disagree about what happened.

## When to use

- Any product that offers "inject an SSH key at provisioning time" or "manage SSH keys" as a feature.
- Whenever you need to confirm remote shell access to a freshly provisioned or freshly rebuilt instance is both possible and correctly gated.
- Pairs with `testing-lifecycle` (provisioning state must be `running`/ready before SSH is expected to work) and `testing-network` (port-22 reachability is a prerequisite, covered there in L3/L4 depth); this skill owns auth correctness, cipher policy, and host-key behavior.

## Core pattern

**Prove injection, don't assume it.** Upload a key via the API, provision (or rebuild) an instance referencing it, then attempt to log in with the corresponding private key. A 201 from the key-upload endpoint only proves the API stored the key — it says nothing about whether the provisioning pipeline wrote it to the machine. Treat "key accepted by the API" and "key usable for login" as two separate assertions, and test both. If injection fails, fall back to whatever the product's documented alternate access path is (typically an API-issued initial password) so the rest of the suite can still exercise the machine.

**Test rejection, not just acceptance.** Generate a second, unregistered key pair and confirm login with it fails cleanly. A suite that only ever tries the correct key never proves the auth boundary exists — "any key works" is a critical, silent finding that a happy-path-only test would never surface.

**Scan and fingerprint the host key.** Collect all offered host-key types (prefer ed25519). Scan twice in quick succession and confirm the fingerprint is identical — a mismatch within seconds means either a MITM proxy or a broken/rotating key generator, both serious.

**Model host-key behavior against the lifecycle, not just at rest.** A host key must be:
- **identical** across a plain reconnect (no unexpected rotation mid-session),
- **identical** after a reboot (the machine's identity persists — regenerating host keys on every boot breaks TOFU/known_hosts and any automation pinned to the fingerprint), and
- **different** after a rebuild/re-image (a rebuild that keeps the old host key means the "new" image was never actually regenerated, or — worse — every instance built from the same golden image shares one host key, which is a cross-customer identity collision).

**Audit the cipher/KEX/MAC suite.** Enumerate offered algorithms (via a dedicated scanner or by parsing the client's negotiation log) and assert two things: a fixed list of legacy algorithms (arcfour, `3des-cbc`, `hmac-md5`, `hmac-sha1-96`, `diffie-hellman-group1-sha1`) is **absent**, and at least one modern cipher (`chacha20-poly1305`/`aes*-gcm`) and modern KEX (`curve25519-sha256`) is **offered**. This is cheap, deterministic, and catches `sshd_config` drift instantly.

**Determine and document root-login policy.** Some base images permit root login directly; others disable it and expect a sudo-capable default user instead. Both are legitimate — detect which applies rather than assuming, and if root is blocked, try the small set of default usernames the product's images are known to use. Regardless of policy, confirm a wrong password is rejected (not just a wrong key).

**Check key-revocation semantics against the product's actual claim.** Deleting a key from the control-plane API removes it from future provisioning, but unless the product specifically claims live revocation, it should **not** retroactively strip the key from `authorized_keys` on already-running machines. Test this and record which behavior you observe — treat a mismatch against the product's documented claim as the bug, not the behavior itself.

## Pitfalls

- **Trusting "key accepted" as proof of injection.** Always attempt an actual login with the injected key.
- **Never testing a wrong key/password.** Without a negative case, "auth works" is unverified — you've only shown the happy path doesn't crash.
- **Skipping the host-key-after-rebuild check.** This is where golden-image reuse and broken re-imaging pipelines hide.
- **No weak-cipher checklist.** `sshd_config` drift (an OS update re-enabling a legacy algorithm) goes unnoticed without an explicit denylist assertion.
- **Assuming a single root-login policy across all image families.** Detect it per instance instead of hardcoding an expectation.
- **Treating key-revocation-not-propagating-to-live-machines as a bug by default.** It's expected behavior in most designs — verify against what the product actually promises before flagging it.

## Checklist

- [ ] Key uploaded via API, then an actual login attempted with the matching private key (not just checked for API acceptance)
- [ ] Login with an unregistered key confirmed rejected
- [ ] Host key scanned twice consecutively; fingerprint identical
- [ ] Host key identical across reboot; different after rebuild/re-image
- [ ] Cipher/KEX/MAC audit: known-weak list absent, at least one modern cipher and KEX offered
- [ ] Root-login policy detected (permitted vs. blocked + default user); wrong password rejected either way
- [ ] Key-deletion-from-API tested against live-machine access and compared to the product's documented revocation claim
- [ ] Basic in-session sanity commands (OS identity, disk, memory, outbound connectivity) run as corroboration once auth is proven
