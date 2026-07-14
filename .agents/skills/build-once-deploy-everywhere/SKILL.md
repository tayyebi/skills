---
name: build-once-deploy-everywhere
description: Use when deploying a containerized app to multiple self-managed hosts — build each image exactly once in CI, tag it with the commit SHA, push to a registry, then pull-and-run the identical artifact on every host. Covers layer-cache warming, an atomic config tarball for host-side bind mounts, safe .env generation with secret carry-forward, and applying config-only changes that `compose up` alone silently misses. Deterministic rollouts and clean rollback from one immutable tag per pipeline.
---

# Build Once, Deploy Everywhere

Build the image a single time, tag it with the commit SHA, and ship that exact artifact to every host. No host ever builds; every host runs byte-identical code. A single immutable tag per pipeline gives deterministic pulls and trivial rollback (re-point to the previous SHA).

## When to use

- Containerized deploys to N self-managed hosts (VMs, bare metal) via a registry.
- Multi-region / multi-host fleets where "did every host build the same thing?" must never be a question.
- Anywhere you want rollback to be "deploy the previous tag" rather than "rebuild an old commit and hope it's reproducible."

Pairs with `host-auto-provisioning` (readying the host) and `rolling-fleet-deploy` (sequencing across the fleet).

## Core pattern

**1. Build once in CI**, tag `registry/app:$SHORT_SHA`, and warm the layer cache from a stable `:cache` tag:

```
docker pull registry/app:cache || true          # best-effort; first run is a no-op
docker build --cache-from registry/app:cache \
  -t registry/app:$SHA -t registry/app:cache ./app
docker push registry/app:$SHA && docker push registry/app:cache
```

**2. Bundle host-side config into one atomic tarball** (compose file, reverse-proxy config, TLS certs) as a build artifact — "everything the host needs" in one transfer. Fail the build if any required file is missing.

**3. Ship to each host**: scp the config tarball, extract it, then `docker login && docker pull registry/app:$SHA && docker logout` on the host. The host pulls the pre-built image — it never builds.

**4. Write the runtime `.env`** with the image tags + config, then `docker compose up -d` (compose recreates only containers whose image tag changed).

## Design decisions

- **SHA tag for immutability + a `:cache` tag for speed.** The SHA tag is what runs and what you roll back to; the `:cache` tag is only a BuildKit cache source (`--build-arg BUILDKIT_INLINE_CACHE=1`). Keep them separate.
- **Atomic config delivery.** Extract the tarball over the existing tree (`tar` overwrites in place). First purge any wrong-type bind-mount source (see `host-auto-provisioning`) so extraction always lands the right shape, then post-verify every file-typed source is a regular file — failing here beats a cryptic OCI mount error at `up`.
- **Secret carry-forward in `.env`.** For secrets that must be identical across hosts and stable across deploys (JWT signing key, cookie domain): prefer a CI-level global variable; else reuse the host's previous `.env` value; else generate once and warn. Keep `.env` at `chmod 600` and snapshot it as `.env.prev` for the next deploy's fallback.
- **Config-only changes need a container recreate.** `compose up` recreates a container only when its image/service definition changes — a bind-mounted config edit is invisible to it. Worse, if you deliver the file by `tar` (new inode) it replaces the file a running single-file mount is pinned to, so even `nginx -s reload` re-reads the *old* inode. The reliable fix: **validate the new config in a throwaway container, then `up -d --force-recreate` that one service** so it re-binds the current file. Validate first so a bad config can never take the site down.
- **CI-side vs host-side variable expansion.** When writing the `.env` over SSH, be deliberate about which `$VAR`s expand on the CI runner (registry creds, image tags) versus land verbatim for the host — quoting the heredoc (`<<'EOF'`) controls this.

## Pitfalls

- **Letting hosts build** → "works on host A, breaks on host B" from differing base layers, toolchain versions, or network state at build time. Build once, pull everywhere.
- **`latest` / mutable tags** → non-deterministic pulls and no clean rollback. One SHA per pipeline.
- **Config-only edit + plain `compose up`** → the change silently doesn't apply; the running container keeps the old file (inode-pinned). Force-recreate the affected service.
- **Regenerating a shared secret every deploy** → sessions/tokens break across hosts, or invalidate on each rollout. Carry it forward.
- **Non-atomic config placement** → a half-written or wrong-type mount source produces the OCI `not a directory` / missing-file error at `up`. Purge, extract, verify.
- **World-readable `.env`** with secrets in it → tighten to `600`.

## Checklist

- [ ] Build once in CI; tag `:$SHA` (immutable) + `:cache` (BuildKit cache source)
- [ ] Host-side config bundled as one atomic tarball artifact; missing-file check
- [ ] Host pulls `:$SHA` from the registry — never builds
- [ ] `.env` written with image tags + config; secrets carried forward; `chmod 600` + `.env.prev`
- [ ] Purge wrong-type mount sources → extract → verify files before `up`
- [ ] Config-only changes applied via validate-then-`--force-recreate` of that service
- [ ] Rollback = deploy the previous SHA tag
