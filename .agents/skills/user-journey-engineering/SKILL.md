---
name: user-journey-engineering
description: Use when designing, reviewing, or improving a feature or flow — optimize the user's end-to-end journey toward a goal rather than individual screens. Covers actor/goal mapping, step-by-step friction analysis, common journey anti-patterns (dead ends, loops, forced registration), and a redesign heuristic set.
---

# User Journey Engineering

Users don't care about features — they care about completing a goal. Optimize for **goal → outcome**, never for **screen → screen**. A screen that doesn't move the user toward their goal is friction, no matter how well it's built.

## When to use

Any time you design a new flow, review an existing one, or get asked "how do we improve this feature" — before touching individual screens, forms, or components. Also useful as a review lens on a PR that adds a multi-step flow (checkout, onboarding, approval, signup).

## Process

### 1. Identify actors and goals

List every distinct user type (visitor, customer, admin, support agent, seller, etc.) and, per actor, write their goal in plain language: "the user wants to buy a product," "approve an invoice," "reset a password." Avoid technical wording — the goal is what the user is trying to accomplish, not what the system does.

### 2. Map the journey

Represent the goal as an ordered sequence of steps from entry to completion (e.g. open app → login → search → compare → add to cart → checkout → payment → confirmation). For each step, capture the user's intention, the action they take, and the system's response.

### 3. Analyze every step

For each step, ask:

- **Why does this exist?** Can it be removed or merged into another step?
- **Cost** — how much thinking, clicking, typing, waiting, or reading does it demand?
- **Failure modes** — what breaks here (network failure, validation error, missing permission, payment rejection, timeout, accidental navigation), and can the user recover without losing progress?
- **Alternatives** — can this step be automated, prefilled, skipped, inferred, or remembered from a prior visit?

Rate each step's friction (low/medium/high) based on complexity, confusion, and manual effort. Anything high needs redesign before shipping.

### 4. Detect journey anti-patterns

- **Dead ends** — no obvious next action after a step completes.
- **Loops** — the user repeats work they already did.
- **Decision overload** — too many choices presented at once.
- **Context switching** — the user has to leave the app (email, another tab) to continue.
- **Duplicate information** — the same data entered twice across steps.
- **Hidden actions** — a critical action exists but is hard to discover.
- **Long forms** — fields collected before they're actually needed.
- **Forced registration** — an account required before any value has been demonstrated.
- **Unnecessary interruptions** — notifications, popups, or dialogs that break the primary goal without being essential to it.

### 5. Optimize

Reduce clicks, typing, waiting, navigation, and confirmations. Increase automation, smart defaults, suggestions, previews, inline editing, and progressive disclosure. Concretely:

- One action over three confirmations.
- Editing over delete-then-recreate.
- Inline over modal.
- Preview over trial-and-error.
- Undo over a confirmation dialog.
- Autocomplete over manual typing.
- Progressive disclosure over one large form up front.
- Smart defaults over empty fields.

### 6. Validate the redesign

Check that a first-time user can complete the journey, an expert can complete it quickly, mistakes are recoverable, work can be resumed later, and the next action is obvious at every step.

## Deliverables

When reviewing or designing a flow, produce: the actor and their goal, the current step-by-step journey, a friction analysis per step, improvement suggestions ordered by impact, and a simplified/optimized journey (also step-by-step, so the before/after is directly comparable).

## Pitfalls

- Optimizing an individual screen before understanding the full journey it sits inside — a beautifully designed step that shouldn't exist is still waste.
- Adding a screen to "clarify" something instead of removing the ambiguity that made clarification necessary — every screen must answer "why does this exist," or it should be removed or merged.
- Treating step count as the only metric — a shorter journey with a confusing decision point is worse than a slightly longer one with none.
- Redesigning in isolation from failure modes — a journey that looks clean on the happy path but has no recovery story for a timeout or validation error just moves the friction to the worst possible moment.
- Adding features to compensate for a broken journey instead of fixing the journey — more features is not the goal; less total effort to reach the outcome is.

## Checklist

- [ ] Actor and goal stated in plain language, not technical terms
- [ ] Full journey mapped step-by-step before any individual screen is redesigned
- [ ] Every step has a friction rating and a documented failure mode + recovery path
- [ ] Anti-patterns checked: dead ends, loops, decision overload, context switching, duplicate input, hidden actions, forced registration, unnecessary interruptions
- [ ] Redesign reduces total user effort, not just moves it around or adds a feature
- [ ] First-time and expert completion both validated, along with resume-later and mistake-recovery
