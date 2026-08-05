---
name: routing-filters
description: Use when users define rules that route incoming items (emails, events, tickets, alerts, webhooks) to one or more destinations — priority-ordered match/ignore rules, stop-processing, catch-all, and the operability layer (overlap detection, simulator, matrix) that keeps a rule set maintainable instead of silently wrong.
---

# Routing Filters

A rule set that decides, for each incoming item, which destination(s) it goes to. Easy to build and easy to get quietly wrong: **a filter set fails silently.** Two rules claim the same item and nobody notices; a high-priority rule with *stop* swallows a lower one, and the lower rule's card still looks healthy while its destination receives nothing. Nothing errors. The cost of getting this right is mostly in the tooling around the matcher, not the matcher itself.

## When to use

Any user-configurable "when X, send to Y" surface: notification routing, alert dispatch, ticket assignment, webhook fan-out, inbox rules, log shipping. Not for a handful of hard-coded branches — reach for it when non-engineers edit the rules.

## Data model

```
rule:
  id, name, description, enabled
  priority          -- higher evaluated first
  conditions: { field -> [pattern, ...] }    -- ANY match fires the rule
  ignore:     { field -> [pattern, ...] }    -- ANY match skips the rule
  actions:
    destinations: [id, ...]
    stop_processing: bool     -- halt lower-priority rules
    discard: bool             -- halt, and route nowhere at all
    <per-rule flags>
catch_all:  { destinations, discard, <flags> }   -- when nothing matched
```

A pattern is either a plain string (regex) or `{value, mode}` where mode is `regex` or `substr`. Offer `substr` — most rules people write are "contains", and making them write a regex to say so is how invalid regexes get into the config.

## Evaluation

1. Sort by priority descending. **Use a stable sort and keep the original index** — equal priorities then fall back to config order, and that tiebreak is real precedence that somebody will depend on. Carry the index with the rule for anything that reasons about order.
2. Ignore patterns first: a match skips the rule without evaluating conditions.
3. Conditions: **OR within a rule** across fields and patterns. Don't add AND — express it as two rules. AND inside a rule doubles the model's complexity for a case that rarely comes up.
4. A firing rule contributes its destinations to an accumulating set.
5. `stop_processing` halts the loop; `discard` halts it *and* routes nowhere, not even the catch-all.
6. If nothing fired, the catch-all decides.
7. Filter destinations down to the ones that are actually deliverable.

**One match function, shared by everything.** The runtime, the simulator and the analyzer must all call the same `match(pattern, value)`. Any reimplementation drifts on case folding, the string-vs-object pattern shape, or how a malformed pattern is treated — and then your analysis tool confidently describes routing that isn't what happens.

### Return a trace, not just a result

The result set loses the cases that matter most. A rule that *matched, had no deliverable destination, and still stopped every rule below it* is invisible in "which destinations got it" — and that is exactly the failure an operator needs to see. Have the core evaluate into a trace and make the plain result a thin projection of it:

```
outcome per rule:
  disabled | ignored | no_match | matched
  matched_no_destination | discarded | not_reached
```

Record `not_reached` too — the simulator's most useful output is "this rule *would* have matched, but the one above stopped the loop."

### Deliverable ≠ exists

A destination can exist and still not be usable (channel with no chat id, address with no mailbox, webhook with no URL). Keep the two apart everywhere: "routes to a destination that does not exist" and "routes to a destination that cannot receive" have different fixes.

## Tolerate a hand-edited config

If the store accepts whatever it is given, a non-numeric priority and a non-object rule entry are both reachable. Coerce and skip rather than raise — an exception here stops *all* routing, for every item, because one field is a string. Log it, keep flowing, and let the analyzer report it so it still gets fixed.

Malformed patterns: treat as non-matching, and **warn once per unique pattern, not once per item.** A broken regex is a one-time config edit, not a per-event incident; logging it per event buries everything else.

## The operability layer

This is the part that gets skipped and the part that pays.

**Overlap detection.** Report duplicate patterns, rules that overlap each other, patterns cancelled by their own ignore rules, and rules that can never fire because something upstream already consumed the item.

**Simulator.** Enter a hypothetical item, get every rule annotated in evaluation order — matched, skipped by ignore, matched with no usable destination, discarded here, never reached. It must call the *same* evaluate the live path calls, so it reports what would really happen rather than a second opinion about it. Nothing is sent, nothing is changed.

**Matrix.** Rules × destinations, filled cells for active routing. Answers "where *could* this go" at a glance; the simulator answers "where would *this one* go".

## Overlap detection: no false positives

The governing rule. Telling someone a working rule is broken is worse than saying nothing, because the next thing they do is "fix" something that was fine. Missing a finding only costs a finding.

So every claim is **proved**, one of two ways:

- **Tier 1 — subsumption.** For patterns with no unescaped metacharacters, parse to a literal plus its anchors (`exact` / `prefix` / `suffix` / `contains`) and decide language inclusion outright.
- **Tier 2 — witness.** For anything with real metacharacters, generate candidate strings, then **verify with the real match function** that both patterns match one. Report the witness in the finding. A concrete value beats any amount of explanation.

Where neither proves anything, say nothing — and let the wording follow: *"no overlap was found"*, never *"these do not overlap"*.

Two subtleties that decide whether witnesses are believable:

- **Normalize the probe the way the runtime normalizes the value.** If the live path extracts a bare address out of a `Name <addr>` header before matching, a witness with text around the address collapses to something else and can never be the value a pattern sees. Rank probes by realism and show the most believable one.
- **Apply the runtime's empty-value guard.** If the live path returns early on an empty value, the analyzer must too — otherwise `a*` and `b*` both match `""` and get reported as overlapping on a value that cannot occur.

The generator can be approximate: every candidate is checked against the real engine before it becomes evidence, so a bug there costs recall, never correctness.

## Running it safely on a request thread

Probing user-supplied regexes is a ReDoS surface aimed at your own server.

- **Refuse to probe pathological shapes at all.** A quantified group whose body is itself quantified or alternated — the `(a+)+` family — is rejected outright. Measured on one engine, `(a+)+$` against 18 characters takes 0.02s and 24 takes 1.29s: clean exponential, so short probes alone are not a defence.
- Cap quantifier counts, pattern length, probe length, total match calls, and a wall-clock deadline shared across the whole analysis.
- In a threaded server you usually **cannot** use signal-based timeouts (signals are main-thread only). Budgets are the defence.
- **Report truncation.** Silently analysing less than everything reads as "all clear" — the most dangerous output this tool can produce.
- Memoize on a hash of exactly the inputs (rules + catch-all + destination deliverability). The tab re-renders on every toggle.

## Pitfalls

- **The editor silently dropping fields it has no UI for.** If the form rebuilds `ignore` from its own inputs and the config supports an ignore field the form doesn't render, opening a rule and pressing Save deletes it. Either render every field or preserve unknown keys — and have the analyzer flag the gap until you do.
- **Deleting a destination leaves dangling references.** Rules keep pointing at an id that no longer exists, which is not an error anywhere — it is a rule that quietly stops delivering. Name what points at a destination before deleting it, and repoint on rename.
- **Reimplementing matching in the analyzer or simulator.** They drift, and then your tooling lies with confidence.
- **Sorting by priority without a defined tiebreak** — equal priorities reorder between renders and routing changes with no config change.
- **AND semantics inside a rule** — resist it; two rules are clearer and compose.
- **Regex-only patterns** — most authors want "contains"; forcing regex is how `.` and `+` end up unescaped in production rules.
- **Per-item logging of a config-level problem** — a bad pattern warns on every single item forever.

## Checklist

- [ ] Priority sort is stable, and the tiebreak is documented as precedence
- [ ] Ignore evaluated before conditions; OR within a rule; catch-all last
- [ ] `stop_processing` and `discard` are distinct, and `discard` skips the catch-all
- [ ] One shared match function for runtime, simulator and analyzer
- [ ] Evaluation produces a trace with an outcome per rule, including `not_reached`
- [ ] "Does not exist" and "cannot receive" are separate findings
- [ ] Malformed priority/rule/pattern degrade instead of stopping routing; warned once
- [ ] Overlap findings are proved by subsumption or by a verified witness — never a hunch
- [ ] Probes normalized like real values; runtime's empty-value guard applied
- [ ] Pathological patterns refused; call + wall-clock budget; truncation reported
- [ ] Simulator runs the live evaluate; matrix shows the whole rule × destination grid
- [ ] Destination rename repoints references; delete names what breaks
