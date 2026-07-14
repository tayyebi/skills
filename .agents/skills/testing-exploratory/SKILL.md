---
name: testing-exploratory
description: Use when there's no test script yet and you need structured discovery — Session-Based Exploratory Testing (SBET) with charters, a timebox, real-time note-taking, and a debrief, plus tour patterns (landmark, lifecycle/FedEx, garbage-collector, saboteur, anti-tour) and the HICCUPPS consistency heuristics. The hat worn when you don't yet know what you're looking for.
---

# Testing Exploratory (Session-Based)

Session-based exploratory testing (SBET): no script, no predetermined pass/fail. The job is to *learn* the system while simultaneously *testing* it — noticing surprises, inconsistencies, and anything that doesn't feel right, then following interesting threads. Distinct from scripted testing: you're not checking a known behavior against a spec, you're discovering behaviors and forming hypotheses.

## When to use

- Early on a new system, or any area with no existing test coverage to script against.
- After scripted/functional testing has run dry — exploratory finds the bugs a checklist wasn't written to look for.
- Whenever "explore X and see what you find" is the actual ask, not "verify X does Y."

Scripted regression lives in [[testing-functional]]; adversarial/security probing in [[testing-security]]; spec conformance in [[testing-contract]].

## Core pattern

The SBET loop: **write a charter → timebox it → explore and take notes → debrief → write the next charter from what you learned.**

**Charter** — one sentence: *"Explore [area] with [resources/techniques] to discover [risks/questions]."* A good charter names an area and a concern, not a step-by-step script. Example shapes worth seeding on most products:
- New-user orientation: explore first-run onboarding to discover friction points and dead ends.
- Error territory: explore invalid input, expired sessions, and network failure to discover how the system communicates and recovers from failure.
- Edge-case data: explore pricing/currency/quantity/state-transition edge cases in whatever subsystem handles money or scarce resources.
- Privileged surface: explore admin-only capabilities to discover whether they're properly scoped and don't leak data to lower-privileged views.
- Full object lifecycle: create, configure, and destroy one entity end-to-end to discover stale state and missing confirmations.
- Curious-user security: unusual inputs, direct URL navigation, modified form values, DevTools — discover what breaks without actually attacking anything.

**Timebox** 45–90 minutes per session — long enough to get past the surface, short enough to keep notes sharp and force a debrief before context is lost.

**Tour patterns** — mental models to rotate through so a session doesn't just retread the happy path:
- **Landmark tour** — visit every link/button/menu, especially the ones you'd normally skip.
- **Lifecycle ("FedEx") tour** — follow one object through its entire life (create → modify → archive/delete). Watch for stale labels, actions available in the wrong state, or state that doesn't update.
- **Garbage-collector tour** — try to create orphaned state: abandon a checkout mid-flow, interrupt an async operation, create-then-immediately-delete. Does the system clean up, or do ghosts appear in lists?
- **Supermodel tour** — judge purely on appearance: consistent layout, consistent error styling, aligned tables. Surface quality is a proxy for code quality.
- **Saboteur tour** — act like a bad actor without attacking: reuse URLs across sessions, hit "back" mid-wizard, bookmark a mid-flow step and return later.
- **Anti-tour** — deliberately avoid the happy path: click secondary actions, fill forms out of order, submit before required fields are filled.

**Note-taking** — capture in real time, don't polish mid-session:

```
[TIME]  [TYPE]  [OBSERVATION]        Types: N note, B bug, Q question, I idea-for-next-charter

10:03  N   Onboarding wizard has 5 steps but back button only goes to step 1
10:07  B   Removing the last cart item leaves checkout clickable; submits 400 "cart is empty"
10:12  Q   Does the spend cap actually block checkout, or just warn?
```

**Debrief** at the end of every session, even a short one:

```
Charter:       [what you set out to explore]
Duration:      [actual time]
Coverage:      [what you actually explored — often differs from the charter]
Bugs found:    [with severity]
Questions:     [unknowns that affect risk assessment]
Follow-up:     [next charter(s) this session suggests]
Areas skipped: [and why]
```

## Heuristics: HICCUPPS

Consistency oracles — expect the system to be consistent with:

**H**istory (behaves as it did before) · **I**mage (matches intended brand/quality) · **C**omparable products (how similar products behave) · **C**laims (matches marketing/docs/tooltip text) · **U**ser expectations (does what a user would predict) · **P**roduct (consistent with itself — same action, same effect, everywhere) · **P**urpose (fits the intended use case) · **S**tandards (web/platform conventions — keyboard nav, HTTP semantics).

Extend with **FEW**: **F**amiliar (feels like the rest of the web/platform), **E**xplicit claims (matches on-screen copy), **W**orld (consistent with how the real world works — e.g., no negative quantities).

## Design decisions

- **Exploratory tools worth having open**: DevTools Network/Application/Console tabs, a way to manipulate URLs and IDs directly, multi-tab (mutate in one, watch if the other updates), and aggressive back/forward navigation especially after mutations.
- **Exploratory bug reports carry extra context** since there's no script to point to — always include the charter/session, exact steps observed, expected vs. actual, and severity, not just a one-line description.

## Pitfalls

- **Charters too broad to timebox** ("explore the whole app") — narrow to an area + a concern, or the session wanders without producing a debrief worth reading.
- **Skipping the debrief** — the next charter should come from what you actually learned, not from a pre-made list; skipping the debrief loses that feedback loop.
- **Treating exploratory notes as the final bug report** — raw session notes are for you; convert anything real into a proper report with repro steps before handing it off.
- **Only running the landmark tour** — it's the natural default and finds the least; rotate tours deliberately to cover different risk dimensions.
- **No timebox** — sessions that run indefinitely lose focus and rarely get debriefed at all.

## Checklist

- [ ] Charter written: area + technique + what it's meant to discover
- [ ] Session timeboxed (45–90 min)
- [ ] Notes taken in real time with N/B/Q/I typing
- [ ] At least one non-landmark tour pattern applied per session
- [ ] Debrief completed before starting the next charter
- [ ] Findings checked against HICCUPPS for "is this actually inconsistent, and with what"
- [ ] Bugs converted to full reports with repro steps, not left as raw notes
