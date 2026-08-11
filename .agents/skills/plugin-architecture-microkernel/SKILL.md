---
name: plugin-architecture-microkernel
description: Use when splitting an application into a minimal kernel plus optional feature plugins, or when retrofitting that split onto a monolith — the explicit plugin contract (interface + value objects + boot-time validation), hook gating policy, disabled-state semantics, UI extension registries, and the caller-first migration order that keeps every intermediate commit clean. Complements clean-architecture-layering (the dependency rule) and design-patterns-catalog at the whole-application level.
---

# Plugin Architecture (Microkernel)

A minimal **kernel** owns the platform — identity, storage, transport, scheduling, the admin shell — and every business domain lives in a **plugin** the kernel knows only through a contract. Done right, disabling a plugin makes its feature vanish completely (commands, routes, UI, background work) with all data kept; deleting its directory leaves a working platform.

## When to use

- An application with one core domain today but more coming — build the seam before the second domain, not after.
- A monolith where optional features (an integration, a scanner, a vanity feature) are tangled through core files and can't be turned off cleanly.
- Any codebase where "turn it off" is a product requirement: per-deployment feature sets, paid tiers, kill switches.

Skip it for a single-domain tool that will never grow a second one — a kernel with one plugin is ceremony without payoff.

## The kernel/plugin split

**Kernel**: config, DB access, auth, the send/deliver primitives, generic capture/recording, the scheduler loop, the web shell, health checks, and the plugin registry itself. The test: every kernel module compiles and greps clean of any domain's vocabulary.

**Plugin**: everything about one domain — its SQL, parsing, rendering, notifications, commands, routes, background ticks, UI, MCP/tool definitions. Migrations can stay in one global sequence (simpler, ordering preserved) as long as each plugin *owns* its tables' SQL.

Prefer an **explicit registry** (`PLUGINS = [a, b, c]`, order = dispatch order) over filesystem discovery or entry points. Explicit wiring is one line per plugin and eliminates a whole class of "why did this load" surprises.

## The contract is code, not a docstring

Three pieces, whatever the language:

1. **An explicit interface** (Protocol / interface / trait) declaring every hook with its exact signature. All hooks optional — a plugin implements only the segments it needs (interface segregation); the kernel depends on the abstraction, never on whatever attributes a module happens to grow (dependency inversion).
2. **Value objects for wire shapes.** Anything a plugin declares as data — routes, commands, menu entries — is a frozen, self-validating construct, not a raw dict/map. Construction is the only place invalidity can enter, so dispatch never re-checks shape per request:

```
Route(method, handler, path|pattern, auth = user|admin|public, gated = true)
  validate at construction: one of path/pattern, known auth, callable handler
```

3. **Fail-fast validation at the composition root.** The registry validates every plugin at load — required attributes, hook arities, table shapes — and refuses to boot on a violation, naming the plugin and the problem. The contract tests call the *same* validator (one source of truth), plus tests that the validator itself rejects bad shapes.

Keep a prose companion of the contract for humans, but say explicitly which one is normative.

Dispatch discipline: the kernel wraps **every** plugin call in its own try/except-and-log, so one broken plugin can't take down the recorder, the scheduler, or its siblings.

## Gating: one written policy

Classify every hook once, in the contract:

- **Feature behavior** — commands, routes, ticks, UI-facing payload contributions: gated on the plugin's `enabled` flag. Disabled = fully inert; a disabled feature is indistinguishable from an absent one (404, not 403 or an apologetic banner).
- **Data hygiene** — delete cascades, retention pruning: **never** gated. Hygiene is not feature behavior; rows must not leak because a feature is off.
- **Management surfaces** — the admin pages/routes that configure or clean up the plugin: explicitly ungated (`gated: false`) — you must be able to manage what you turned off.
- **Historical obligations** — permission checks on already-sent artifacts (old messages' buttons): consulted even while disabled; old artifacts keep their rules.
- Unauthenticated endpoints (webhooks) are `auth: public`: dispatched before platform auth, the handler owns its own gate (shared secret).

Config: per-plugin blob under one key convention (`plugin_<name>`), always containing `enabled`, writes coerced against declared defaults. The generic admin editor should render **every** declared knob from the defaults schema — if a plugin declares a setting, it's editable without a deploy, or the declaration is a lie.

## Namespacing shared identifiers

Any identifier that crosses plugins through a shared channel (callback data, dismiss/permission refs) carries its owner: `<plugin>:<id>`, routed only to the named plugin. A bare legacy format stays honored forever (artifacts already in the wild), and an id nobody claims defaults to the harmless behavior — a leftover nobody owns must never become unusable.

Beware first-opinion-wins hook chains where one plugin never abstains: it silently monopolizes the channel for everyone registered after it. Namespacing is the structural fix.

## UI plugins: shell registries + fragments

The web/app shell exposes **registries**, and plugin UI ships as **fragments** beside the plugin's code, spliced in at serve time:

- Registries for every extension point: views/tabs (with label + order), URL routes, admin sections, settings/profile/page sections, list-row enrichments, back-navigation handlers. Each registration names its `plugin` so the shell hides it when disabled — gating lives in the registry, in one place, not in per-fragment banners.
- Wrap each fragment in its own function scope at splice time: closure access to shell helpers, but fragment-local state never leaks into the shell or a sibling.
- The shell holds **no handles to plugin DOM** — plugin markup, styles, element lookups and wiring live in the fragment; the shell touches at most a lazy, null-safe visibility toggle.
- Server-side payload hooks mirror the UI hooks: the plugin that renders a section also contributes its payload keys, so disabling removes data and UI in one motion.

## Retrofitting a monolith: the migration method

The trap: moving a feature's **entry points** (routes, commands, screens) into the plugin while its implementation stays in core. That's worse than not starting — the plugin now re-imports core privates and cycles appear. Completion criterion is the kernel test above: core clean of the domain's vocabulary.

Method that keeps every intermediate commit shippable:

1. **Contract and seams first**: registry, hooks, validation, and an automated boundary test (AST/grep ban on the forbidden import direction, with a shrink-only allowlist for known offenders that empties as you go).
2. **Move callers before callees** — top-down by dependency (handlers → notification composers → renderers → parsers → data layer last). Every intermediate state then has only plugin→core edges; moving callees first creates temporary core→plugin edges and cycles.
3. **Tests move in the same commit as their code**, and monkeypatch the plugin's own seams, not core module attributes that from-imports no longer see.
4. **Grandfather external wire formats** — URLs, webhook paths, stored keys — as explicit plugin declarations. Zero client churn; only *new* endpoints adopt the namespaced scheme.
5. Give each plugin **one facade module per concern** (data, rendering, notifications, parsing) that re-exports the few platform names siblings also need — every plugin module imports exactly one module per concern, and call sites don't reveal which side of the boundary a helper lives on.

## Pitfalls

- **Dead hooks**: a declaration hook exists but core still hardcodes the values and no plugin declares any — worse than no hook, because the seam looks done. Wire it or delete it.
- **Split-brain features**: half a small feature in core (its admin actions), half in the plugin. Smallest features drift into this first; move them whole.
- Generic kernel mechanisms with a **default pointing at one plugin's consumer** (a prompt flow defaulting to the tasks handler) — the mechanism isn't generic until every consumer names itself.
- Plugin worker threads and queues created at **import time** — build them in the `start` hook, register them with health monitoring, and re-check `enabled` per job for long-lived workers.
- Validating the contract **only in tests**: tests use stubs and drift; run the same validator over the real registry at boot.
- Frontend fragments calling shell functions that were quietly moved/renamed — every shell↔fragment touchpoint goes through a registry, and a composed-page syntax check (even just `node --check` per inline script) in the verification loop catches splice-time breakage cheaply when there's no JS test harness.

## Checklist

- [ ] Kernel modules compile and grep clean of every domain's vocabulary
- [ ] Contract exists as code (interface + value objects) and is validated over the real registry at boot, fail-fast with named errors
- [ ] Every hook classified: gated feature / ungated hygiene / ungated management / historical obligation — policy written once in the contract
- [ ] Disabled plugin: commands hint, routes 404, ticks stop, UI vanishes, data kept; management surface still reachable
- [ ] Every plugin call site individually guarded so one broken plugin can't down its siblings
- [ ] Cross-plugin identifiers namespaced by owner; legacy formats honored
- [ ] UI extension points are plugin-gated registries; fragments own their DOM and scope
- [ ] Boundary enforced by an automated import-direction test, not convention
- [ ] Migration commits move callers before callees, tests travel with code, external URLs grandfathered
