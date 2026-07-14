---
name: testing-data-fuzz
description: Use when probing an API's full input space for correctness — string boundaries, Unicode/encoding edge cases, numeric boundaries, type confusion, malformed identifiers, JSON structure attacks (deep nesting, huge arrays, duplicate keys), prototype pollution, ReDoS, and content-type/pagination abuse. Every bad input should degrade to a clean 4xx, never a 500 or silent corruption. Distinct from testing-pentest-injection (which owns exploitation).
---

# Data Validation, Boundary & Encoding Testing

Probe the full input space of an API — not the happy path, the edges, the weird, and the malformed. The bar for every field is: valid input succeeds, invalid input produces a clean, informative 4xx, and nothing ever crashes the process or silently corrupts data.

> Owns input-boundary/encoding/schema-enforcement correctness. For turning a finding into an exploit see [[testing-pentest-injection]]; for the broader vulnerability methodology see [[testing-security]]; for the general HTTP contract see [[testing-api]].

## When to use

- Any field that accepts free-text, numbers, or identifiers from an untrusted client.
- Before trusting a new endpoint's validation layer, or after any change to request parsing/deserialization.

## Core pattern

Work field-by-field, category-by-category. For each category, the failure signature to watch for is always the same shape: **200 with corrupted/truncated/oversized data stored silently**, or **500 instead of 400**. Both are bugs; only a clean 4xx with a descriptive error is a pass.

**String boundaries.** Empty string, whitespace-only, exactly-at-limit, one-over-limit, and orders-of-magnitude over limit (100 → 100k+ chars). Confirm the server enforces its own documented (or inferred) length limits rather than accepting and storing anything.

**Unicode & encoding.** Emoji, RTL scripts, CJK, zero-width characters, null bytes, BOM, unpaired surrogates, the maximum valid codepoint, homoglyphs (a lookalike character from another script — relevant anywhere display names are compared or rendered), and two Unicode-normalization-equivalent byte sequences (precomposed vs combining-mark). Watch for encoding crashes, silent truncation at a null byte (can bypass a length check performed pre-truncation), and inconsistent normalization between storage and comparison.

**Numeric boundaries.** Zero, one, negative, int32/int64 min/max, a bignum beyond int64, floats, and the JSON-illegal special floats (`Infinity`, `NaN` — most JSON encoders reject these, which is itself worth confirming). Apply to every numeric field: quantities, amounts, pagination params.

**Type confusion.** Send the wrong JSON type for every field — a number where a string is expected, an array, an object, a boolean, explicit `null`. In dynamically-typed backends and in statically-typed backends that panic on failed type assertions, this is a common 500 source; it must always degrade to 400.

**Identifier handling.** Malformed IDs in path and body: not-the-right-format, all-zeros, all-max-value, path-traversal-shaped, SQL-injection-shaped, percent-encoded null, and the literal strings `"null"`/`"undefined"`. Expect 400 or 404; never 500.

**JSON structure attacks.** Deeply nested objects (stack-overflow/recursion-limit probe), very large arrays, and duplicate keys (confirm defined, consistent last-wins-or-error behavior rather than undefined parser behavior).

**Prototype pollution and ReDoS — gate before testing.** Prototype pollution only applies where a JS/template-consuming path actually processes the field (SSR template, Node microservice) — if the whole stack has no such path, mark the check not-applicable rather than reporting a false negative. Likewise, ReDoS only applies where a non-linear-time regex engine is in the path (PCRE, JS regex, hand-rolled backtracking) — if the runtime's regex engine is linear-time (e.g. RE2), a slow response points to network/DB latency, not catastrophic backtracking. Confirm the engine before flagging timing anomalies.

**Async-endpoint validation.** Endpoints that enqueue background work must still validate synchronously — a malformed submission should 400 *at submit time*, never accepted with a 202 and left to fail invisibly in a worker with no actionable error returned to the caller.

## Pitfalls

- Reporting a slow response as ReDoS without first confirming a backtracking-capable regex engine is actually on the input's path.
- Reporting prototype pollution against a backend with no prototype-based object model.
- Treating a 400 with a vague message as a pass — the message should actually describe what's wrong, or the API is unusable for real clients.
- Skipping the explicit-`null` case because "omitted field" was already tested — they must be verified to behave identically.
- Fuzzing only string fields and skipping numeric/enum fields, where type confusion is just as likely to panic a handler.

## Checklist

- [ ] String boundaries: empty, whitespace, at-limit, over-limit, orders-of-magnitude over
- [ ] Unicode: emoji, RTL, CJK, zero-width, null byte, BOM, surrogate, homoglyph, normalization pair
- [ ] Numeric boundaries: zero, negative, int32/int64 edges, bignum, special floats
- [ ] Type confusion on every field (string/number/array/object/bool/null swapped)
- [ ] Malformed identifiers in path and body → 400/404 never 500
- [ ] JSON structure attacks: deep nesting, huge arrays, duplicate keys
- [ ] Prototype-pollution and ReDoS checks explicitly gated on a vulnerable code path existing
- [ ] Async/enqueue endpoints validate synchronously before accepting 202
- [ ] Every failure mode above produces a clean 4xx with a descriptive message, never a 5xx
