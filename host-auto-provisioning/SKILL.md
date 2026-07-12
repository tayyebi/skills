---
name: host-auto-provisioning
description: Use when your deploy pipeline must turn a bare Linux host (nothing but SSH) into a ready target — idempotently installing the runtime (Docker/Compose, git, curl), creating the deploy tree, self-healing wrong-type mount sources, and running preflight checks — so a freshly-spun server converges automatically on the next deploy with no manual setup. Runs every deploy; skips whatever is already in place.
---

# Host Auto-Provisioning

Make "provision the box" a step of the pipeline, not a runbook. Given only SSH access, the provisioning stage installs everything the app needs, prepares the filesystem, and validates the host — idempotently, so it's safe to run on every deploy. A new host added to the fleet becomes deployable automatically; a drifted host self-corrects.

## When to use

- Deploying to VMs / bare hosts you control over SSH (as opposed to a managed platform that provisions for you).
- Fleets where hosts are added over time and you refuse to hand-configure each one.
- Any deploy where "works on the host that's been up for a year" and "works on a host spun up this morning" must be the same code path.

Pairs with `build-once-deploy-everywhere` (what runs *after* the host is ready) and `rolling-fleet-deploy` (doing this across a matrix).

## Core pattern

A `provision` stage, run per host, that SSHes in and converges the box:

**1. Prove SSH first.** Run a trivial `ssh host true` before anything else. Without this gate, an auth failure on the first real command (`command -v docker`) is misread as "docker not found", and the script tries to "install" it via more failing SSH calls — flooding the log with `Permission denied` instead of one clear "SSH auth is misconfigured" error.

**2. Install the runtime idempotently.** For each dependency: check `command -v`, install only if absent, detect the OS (`/etc/os-release`) to pick the right package repo, enable+start the service. Re-running is a fast series of "already installed" checks.

**3. Create the deploy tree.** `mkdir -p` the deploy path and any directories the app bind-mounts.

**4. Self-heal mount sources** (the subtle one). If a container bind-mounts a *file* that doesn't exist yet, Docker silently creates it as an empty **directory** — and the next run dies with `not a directory: are you trying to mount a directory onto a file?`. Before deploy, walk the file-typed mount sources and `rm -rf` any that are wrongly a directory; ensure dir-typed sources are directories. The shipping step repopulates the files.

**5. Preflight checks.** Verify Docker actually runs (`docker run --rm hello-world`), report free disk, warn on ports already in use. Fail loudly on anything that would break the deploy.

## Design decisions

- **Idempotent by construction.** Every action is "check, then act only if needed." The stage runs on *every* deploy so drift and new hosts both converge — that only works if re-running is cheap and safe.
- **Preflight the credential, not just the tools.** The single most confusing class of provisioning failure is auth misread as missing software. One explicit SSH-connectivity check up front turns a 200-line cascade into a one-line, actionable error naming the likely cause (key not in `authorized_keys`, wrong user, stale secret, wrong port).
- **Provision depends only on validation, not on build.** Host prep is independent of which artifact you'll ship, so it can run in parallel with the build; the ship stage waits for both.
- **Support key *or* password auth** with fast-fail SSH options (`BatchMode=yes` for keys, `NumberOfPasswordPrompts=1` for passwords) so a bad credential fails once instead of looping prompts against an EOF stdin in CI.
- **Self-heal, don't just detect.** Wrong-type mount sources are recoverable — fix them in place and let the shipping step re-place the real files, rather than aborting and paging a human.

## Pitfalls

- **No SSH preflight** → auth failures masquerade as missing dependencies and the log becomes unreadable.
- **Docker's file-vs-directory mount trap** — the classic OCI `not a directory` error. Purge wrong-type sources before every `up`.
- **Non-idempotent installs** (unconditional `apt-get install`, re-adding repos) → slow, noisy, and occasionally breaking on re-run.
- **Assuming one OS/codename** — detect it; map testing/unstable codenames to the nearest stable one for third-party repos.
- **Looping password prompts** in non-TTY CI when auth is wrong — force single-prompt / batch mode.
- **Skipping the `hello-world` smoke test** → you discover Docker is broken only when the real deploy fails cryptically.

## Checklist

- [ ] SSH-connectivity preflight before any provisioning command, with an actionable error
- [ ] Idempotent installs: `command -v` guard, OS-detected repos, enable+start services
- [ ] Deploy tree + all bind-mount directories created
- [ ] Wrong-type mount sources self-healed (file-typed dirs removed, dir-typed files fixed)
- [ ] Preflight: `docker run hello-world`, disk space, port availability
- [ ] Key-or-password SSH auth with fast-fail options
- [ ] Runs every deploy; safe and cheap to re-run
