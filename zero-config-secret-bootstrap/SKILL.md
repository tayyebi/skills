---
name: zero-config-secret-bootstrap
description: Use when a companion service needs a shared secret/token to talk to your app but you don't want operators to generate, set, and sync it — have the service mint the secret itself on first boot, persist it to a shared volume, and let the other side read it from there. No MOCK_TOKEN/API_KEY to configure; the secret exists before anyone needs it and survives restarts. For trusted co-located services sharing a filesystem, not for secrets crossing a trust boundary.
---

# Zero-Config Secret Bootstrap

Eliminate the "generate a token, set it in two places, keep them in sync" chore. The service that *owns* a control-plane secret generates it itself on first boot, writes it to a **shared volume**, and persists it there. Any co-located peer reads the secret from the volume instead of being handed one. There is nothing to configure and nothing to drift.

## When to use

- A sidecar / companion service (admin panel, mock/test target, internal tool) that authenticates callers with a token, deployed alongside the app **on the same host, sharing a filesystem/volume**.
- When you want "it just works after deploy" — no secret to add to CI variables, no two-place sync, no rotation runbook for a low-stakes internal credential.

Do **not** use it for secrets that cross a trust boundary or leave the host (customer keys, cross-org auth). Those need real secret management. This pattern trades isolation for zero-config and is only appropriate between mutually-trusted, co-located processes.

## Core pattern

**Owner generates on first boot, persists to a shared path:**

```
on boot:
  if TOKEN_FILE exists and non-empty:   token = read(TOKEN_FILE)      # restart → reuse
  elif env override set (dev):          token = env.TOKEN            # optional pin
  else:                                 token = randomSecret(); write(TOKEN_FILE, token)
```

Mount `TOKEN_FILE` on a volume shared with the consumer. The consumer mounts the same volume **read-only** and reads the token when it needs to call the owner — or, better, never exposes it: the app reads the file server-side and proxies authenticated requests, so the token never reaches the browser/client at all.

**Deliver to the human via existing auth.** If an operator opens the tool's UI, have it fetch the token from the app through the operator's *already-authenticated* admin session (`GET /internal/tool-token`, admin-only), rather than showing it or asking them to paste it. The tool "connects automatically."

## Design decisions

- **First-boot generation + persistence** means the secret exists before anything needs it and is stable across restarts — no chicken-and-egg, no regeneration invalidating live sessions.
- **The owner owns the secret.** Exactly one process generates it; everyone else reads. Two generators = two secrets = broken auth.
- **Read-only mount for the consumer.** The app that reads the token mounts the volume `:ro` so only the owner can (re)write it.
- **Prefer proxying over exposing.** Reading the token server-side and relaying calls keeps it off the wire and out of client code — the consumer authenticates the human with *its own* session and uses the shared secret internally.
- **Keep an env override for local dev** so a developer can pin a known token without a volume, while production stays zero-config.

## Pitfalls

- **Using it across a trust boundary.** Anything on the host that can read the volume gets the secret. Fine for co-located trusted services; unacceptable for anything sensitive or remote.
- **Two owners / regenerating on every boot** → the peer's cached or previously-read token stops matching. Generate once, persist, reuse.
- **Race on first boot** — consumer reads the file before the owner has written it. Have the consumer read lazily (on first use) and retry, or gate startup on the file existing.
- **Leaking the token to clients.** If the UI needs it, fetch it server-side behind existing auth; don't ship it to the browser or log it.
- **Writable mount on the consumer side** → the peer can clobber the secret. Mount read-only.
- **No restart persistence** (writing to a tmpfs / container-local path) → a new random token every restart breaks every existing peer. Persist on the shared volume.

## Checklist

- [ ] Exactly one owner generates the secret on first boot
- [ ] Persisted to a path on a **shared volume**; reused if already present
- [ ] Consumer mounts the volume **read-only**
- [ ] Human delivery via existing authenticated session (proxy/fetch), not exposure
- [ ] Optional env override for local dev
- [ ] Used only between co-located, mutually-trusted processes
