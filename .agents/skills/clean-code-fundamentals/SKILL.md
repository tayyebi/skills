---
name: clean-code-fundamentals
description: Use when writing or reviewing everyday code for readability and maintainability — naming, function size and shape, comments, duplication, and error handling. Line-level and function-level discipline; complements clean-architecture-layering (module boundaries) and solid-principles (design forces) which operate one level up.
---

# Clean Code Fundamentals

Code is read far more often than it's written. Optimize for the next reader — often you, in six months, with no memory of why a decision was made. Apply these pragmatically; a rule that makes a specific piece of code worse is a rule to break, not follow blindly.

## When to use

Every time you write or review code. This is baseline discipline, not a special-occasion pattern — but treat it as a floor, not a ceiling: these rules make code readable, they don't substitute for getting the design right first.

## Naming

- Names should answer *why it exists* and *what it does*, not require a comment to explain. `daysSinceLastLogin` beats `d` plus a comment.
- Variables are nouns (`userCount`, `isActive`); functions are verbs or verb phrases (`calculateTotal`, `sendNotification`); types are nouns (`OrderRepository`, `PaymentGateway`).
- Avoid names that lie: don't call something a `list` if it's not ordered/indexable; don't call a function `getX` if it also mutates state.
- Prefer a searchable name over a magic literal: a named constant for `5` (e.g. `MAX_LOGIN_ATTEMPTS`) is greppable and self-documenting; a bare `5` scattered across the codebase is neither.
- Single-letter names are acceptable only for a tight, few-line loop index — never for anything that outlives the loop body.

## Functions

- Keep functions small — as a rule of thumb, if you can't see the whole thing without scrolling, it's doing too much.
- One function, one level of abstraction. Don't interleave "orchestrate the workflow" with "parse this substring" in the same body — extract the low-level step and name it.
- Argument count is a smell gauge: zero is ideal, one or two are fine, three or more usually means the arguments belong together as a single named structure (a request/parameter object) instead of a positional list.
- A boolean parameter that changes what the function does (`process(order, true)`) means it's secretly two functions; split them and give each a name that says what it does.
- Separate command from query: a function that both mutates state and returns a meaningful value invites bugs where callers use it for one purpose and get surprised by the other. Prefer `save(x)` (returns nothing/an error) and `find(x)` (returns data, mutates nothing) as separate calls over `saveAndReturn(x)`.

## Comments

- A comment that restates what the code already says (`# increment i` above `i += 1`) is noise; delete it.
- A comment that explains *why* — a non-obvious constraint, a workaround for a specific bug, a business rule that isn't visible in the code shape — earns its place.
- Don't comment out dead code "just in case." Version control already has the history; delete it.
- A `TODO` without an issue/ticket reference is a comment that will outlive everyone's memory of what it meant. Link it or remove it.

```
// weak: restates the code
// check if user is admin and can edit
if user.role == "admin" and user.hasPermission("edit"): ...

// better: extract and name the concept
if user.canEdit(): ...
```

## Duplication

- Every piece of business knowledge should have exactly one place it's expressed. Two functions computing the same tax rule independently will eventually disagree.
- Duplication in test code is more tolerable than in production code — tests favor clarity-per-test over shared setup.
- Don't abstract on the first repetition. A guideline like "wait for the third occurrence" avoids building an abstraction around a coincidence that turns out not to be the same concept after all — a *wrong* abstraction, forcing unrelated cases through one shared function, costs more than living with duplication for a while.

## Structured data over loose maps

- When a shape recurs — the same set of fields passed around, checked, and transformed together — give it a name (a struct/record/class) instead of passing an untyped map/dict/array. A typed shape documents which fields exist and lets tooling catch a missing or misspelled field; an untyped bag only fails at the point of use, often far from the mistake.

```
// weaker: shape is implicit, only discoverable by reading every call site
function process(data) -> result

// stronger: shape is explicit and enforced
function process(input: OrderInput) -> OrderResult
```

## Guard clauses and early returns

Handle the exceptional/invalid case first and return, instead of nesting the main logic inside a deep conditional. Flat code with early exits is easier to scan than a pyramid of nested ifs.

```
function processOrder(order):
    if order.isCancelled: return
    if not order.isPaid: raise UnpaidOrderError
    // main logic, unindented, easy to follow
```

## Error handling

- Signal failure through the language's real error mechanism (exceptions, `Result`/`Either` types, explicit multi-return errors) — not through sentinel return values (`-1`, `null`, a magic string) that callers can silently ignore.
- Don't return `null`/`nil` to mean "nothing here" if the caller then has to null-check everywhere; prefer an empty collection, an explicit optional/`Maybe` type, or throwing when absence is actually an error.
- Only catch what you can meaningfully handle (retry, fall back, add context) — a catch block that swallows the error and continues silently turns a loud failure into a mysterious one later.

## Pitfalls

- Consistency violations are worse than any individual rule broken alone — a codebase with one dominant (imperfect) style beats one with five competing "correct" styles.
- Over-engineering small code out of a desire to look clean — a 4-line function doesn't need a strategy pattern.
- Treating this skill as a substitute for the actual design work — clean naming and small functions can't fix a fundamentally wrong module boundary (see `clean-architecture-layering`, `solid-principles`).

## Checklist

- [ ] Names say why/what without needing a comment
- [ ] Functions are short, single-purpose, single abstraction level
- [ ] 3+ positional arguments become a named parameter object
- [ ] No boolean flags that silently switch behavior
- [ ] Comments explain *why*, not *what*; no dead code, no unlinked TODOs
- [ ] Business rules expressed in exactly one place
- [ ] Recurring loose data shapes promoted to a named type
- [ ] Guard clauses handle exceptional cases first
- [ ] Failures signaled through real error types, not sentinel values
