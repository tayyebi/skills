---
name: static-typing-discipline
description: Use when deciding how strictly to enforce a language's type system, or reviewing code where a type-related bug slipped through — strict-mode type checking (PHP `strict_types`, TypeScript `strict`, Python type hints + mypy/pyright), why implicit coercion is a correctness bug magnet, static analysis in CI as a cheap gate, null-safety via optional/nullable types instead of sentinel values, and immutability via readonly/final/const/frozen. Applies to any language with an optional, gradual, or inherently static type system.
---

# Static Typing Discipline

Every mainstream language now offers some form of static or gradual typing — the discipline is switching it to maximum strictness and treating it as a correctness tool, not decoration. A type error caught by a compiler or analyzer is a bug that never reaches a user; the same bug caught only by a test is a bug someone had to think to test for.

## When to use

- Starting a new codebase or module in a language with optional typing (PHP, TypeScript, Python) — set strictness to maximum from the first commit; retrofitting it onto a large lenient codebase later is slow and contentious.
- Reviewing a bug post-mortem and noticing "a type checker would have caught this" — that's a signal to raise the strictness knob project-wide, not just patch the one call site.
- Designing any function/module boundary where a caller could plausibly pass the wrong shape of data.

## Core pattern: turn strictness to maximum, everywhere

Optional and gradual type systems default to lenient — coercion, `any`, and untyped fallbacks are opt-out, not opt-in. Explicitly promote to the strict mode and enforce it, rather than hoping people annotate voluntarily:

- **PHP**: `declare(strict_types=1)` at the top of every file. Without it, PHP silently coerces `"123"` to `int(123)` at a call boundary, turning a caller bug into a runtime surprise instead of a `TypeError`.
- **TypeScript**: `"strict": true` in `tsconfig.json` (bundles `strictNullChecks`, `noImplicitAny`, and friends). Without it, `any` leaks in anywhere inference fails, and the type system silently stops protecting that code.
- **Python**: type hints on every function signature, checked by `mypy --strict` or `pyright` in CI. A hint with no checker running against it is a comment, not a guarantee — it's the checker that makes it load-bearing.

The common thread: the language lets you opt out, and the lenient path is usually the default. Flip the switch deliberately and keep it flipped.

## Why implicit coercion is a bug magnet

Loose comparison and silent coercion make the same code mean different things depending on what happens to be in a variable at runtime — a query-string value arriving as the string `"0"` can compare falsy in one language, truthy in another, and equal to integer `0` in a third, all "by design." A strict type system forces the conversion to be explicit, so a caller passing the wrong shape fails at the call site instead of three functions downstream:

```
# loose: "0" and 0 and false may all compare equal — behavior depends on
# implicit coercion rules the reader has to memorize
if amount == 0: ...

# strict: the type of `amount` is fixed (integer), so a caller passing a
# string is a compile-time / lint-time error, not a silent runtime bug
function apply_discount(amount: Integer) -> Money
```

## Static analysis in CI as a cheap correctness gate

A type checker — `tsc`, `mypy`/`pyright`, PHPStan/Psalm, or the compiler itself for Go/Rust/Java — run in CI and blocking merge on new errors catches whole classes of bugs (null dereference, wrong argument count, incompatible type) without writing a single test case. It is almost always cheaper than the equivalent test coverage.

- On a codebase that isn't clean yet, **baseline** existing errors and forbid new ones (PHPStan/Psalm baseline file, `mypy` scoped to changed files, TypeScript `strict` enabled per-directory) rather than blocking all progress until the whole tree is fixed.
- Ratchet the baseline down over time instead of letting it grow — a baseline that only ever expands is a static analyzer in name only.
- See `ci-cd-pipeline-discipline` for wiring this gate into the pipeline itself.

## Null-safety: optional types over sentinel values

A sentinel value — `-1`, `""`, a magic string like `"N/A"`, or `null` used ambiguously for both "not loaded yet" and "genuinely absent" — forces every caller to remember the special case by convention. A type system that makes "may be absent" part of the declared type forces the compiler or analyzer to remind them instead:

- **PHP**: `?Order` return type plus the null-safe operator `$order?->getTotal()` — the caller must acknowledge null before touching the object.
- **TypeScript**: `Order | undefined` return type plus optional chaining `order?.total` — same shape, enforced by `strictNullChecks`.
- **Python**: `Order | None` return type plus an explicit `if order is not None:` guard — `mypy` flags a missed check as an error, not a warning.

Languages with inherent static typing bake this in further: Rust's `Option<T>` forces a `match` (or an explicit `.unwrap()` that reads as a deliberate risk); Go's explicit `value, ok :=` idiom and nil-pointer checks make "did I actually get something" a first-class question at every call site. The mechanism differs; the discipline is identical — never let "missing" collapse into a value the type system can't tell apart from real data.

## Immutability where the language supports it

Preventing an object from being mutated after construction removes a whole class of aliasing bugs, where one caller's edit leaks into another caller's view of what it assumed was "the same" data untouched:

- **PHP**: `readonly` properties on a class (PHP 8.1+).
- **TypeScript**: `readonly` fields, or wrapping a type in `Readonly<T>`.
- **Python**: `@dataclass(frozen=True)`, or an immutable `NamedTuple`.
- Elsewhere: Rust's default-immutable bindings (`let` vs. `let mut`), Java/Kotlin `final`/`val`, Go's convention of returning copies instead of exposing mutable internals.

Default value objects (money, coordinates, dates, identifiers) to immutable — there's rarely a reason for them to change in place. Prefer immutable entities wherever the domain allows it; where mutation is unavoidable, keep the mutable surface small and explicit rather than letting any caller reach in and change a shared object.

## Pitfalls

- Enabling strict mode only for new files — old files keep silently coercing, and the codebase now has two dialects with no plan to converge; schedule the migration instead of leaving it open-ended.
- Treating `any` / `mixed` / an untyped dict as a routine escape hatch rather than a rare, commented exception — it's a hole in the type system exactly where a bug is most likely to hide.
- Type hints with no checker wired into CI — decoration that silently drifts out of sync with the actual code, since nothing enforces it.
- Modeling "missing" with a sentinel (`-1`, `""`, a magic string) instead of an optional/nullable type — the type system can't protect a value it can't distinguish from data.
- Passing a mutable object (a config, a request context) across a boundary and letting a downstream function mutate it in place — prefer an immutable copy or a builder that returns a new instance.

## Checklist

- [ ] Strict mode enabled from day one: `strict_types=1` / `"strict": true` / `mypy --strict` (or the language's inherent static typing)
- [ ] Static analyzer wired into CI, blocking merge on new errors; existing debt baselined and ratcheted down, not ignored
- [ ] No bare `any`/`mixed`/untyped fallback without a comment justifying it
- [ ] Absence modeled as `Optional`/`?Type`/`| undefined`/`Option<T>`, never a sentinel value
- [ ] Value objects and cross-boundary data marked immutable (`readonly`/`final`/`frozen`/`const`)
- [ ] Every new or changed function signature is fully typed, including the return type
