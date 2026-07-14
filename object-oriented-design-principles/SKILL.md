---
name: object-oriented-design-principles
description: Use when designing classes/objects and deciding how they collaborate — encapsulation, composition vs. inheritance, programming to an interface, and coupling/cohesion as the metrics that predict whether a design will be painful to change. Covers OOP fundamentals broadly; see solid-principles for the SOLID acronym specifically.
---

# Object-Oriented Design Principles

OOP bundles state and behavior so the two evolve together, and hides *how* something works behind a stable *what it does*. The value isn't inheritance hierarchies or getters/setters — it's controlling coupling so a change in one place doesn't cascade everywhere else.

## When to use

Any codebase organized around types that carry both data and behavior — most non-trivial codebases, regardless of whether the language calls the unit a "class," a "struct with methods," or a "type implementing an interface."

## Encapsulation

- Keep internal state private/unexported; expose behavior through methods that enforce invariants, not raw field access. Reaching in to set a balance to a negative number directly is exactly the bug encapsulation exists to prevent.
- Validate at the boundary where state changes, not after the fact — the method that changes an email address should be the only path that can ever produce an invalid one.
- Favor immutable value objects (money, dates, IDs) where the value's identity is its content: construct once, never mutate, produce a new instance for a "changed" version. This eliminates a whole class of aliasing bugs where two references silently see each other's edits.

```
class Money:
  private amount, currency   # not directly settable

  add(other: Money) -> Money:
    assert currency == other.currency
    return Money(amount + other.amount, currency)   # returns new, doesn't mutate self
```

## Composition over inheritance

- Prefer "has-a" (a class holds a reference to a collaborator and delegates to it) over "is-a" (subclassing) whenever the relationship isn't a strict, stable specialization. Composition can be reassembled at runtime; inheritance is fixed at definition time and couples every subclass to the parent's implementation, not just its contract.
- The classic tell: a subclass that overrides an inherited method to reject it ("not supported") is a substitutability violation waiting to break callers. Model the behavior as a separate, composable collaborator instead.

```
# Inheritance: rigid — Penguin inherits fly() it must reject
class Bird: fly() ...
class Penguin extends Bird: fly() -> raise Unsupported   # breaks substitutability

# Composition: flexible — assemble only the behaviors that apply
class Penguin:
  field swim: SwimBehavior
  # no FlyBehavior composed in — penguins don't fly, and callers never ask
```

- Use inheritance only for genuine is-a relationships where the subtype can stand in anywhere the supertype is expected without surprising callers (Liskov substitutability). Use mixins/traits, where the language offers them, for pure code reuse that isn't really a type relationship.

## Interfaces and abstraction

- Depend on an interface (a behavior contract), not a concrete implementation — inject the interface type, and any conforming implementation becomes swappable without touching the caller.
- Start with an interface; add an abstract/base class only once there's real shared state or shared implementation to factor out. An abstract class you never subclass twice is speculative complexity.
- A good interface describes a *role* the caller depends on (`PaymentProcessor`, `Notifier`), not just one concrete class's method list — if a second, unrelated implementation is easy to imagine, the interface is cut at the right seam.

```
interface PaymentProcessor:
  process(payment) -> PaymentResult

function checkout(cart, processor: PaymentProcessor) -> Order:
  return processor.process(cart.toPayment())
  # works identically whether processor is backed by one provider, another, or a test double
```

## Coupling and cohesion

These two metrics predict how painful a design will be to change — optimize for both together, not either alone.

- **Cohesion** — how tightly a class's own methods use its own fields. A class where most methods touch most fields is cohesive; one with disjoint method/field clusters is really two classes wearing one name. Split it.
- **Coupling** — how much one class needs to know about another's internals to work with it. Depending on another type's public interface is normal coupling; reaching into its internal state, or depending on its concrete class instead of an abstraction, is tight coupling that breaks the moment that class changes.
- Small, focused public surfaces reduce coupling by construction — the fewer public methods a class exposes, the fewer ways other code can become entangled with its internals.
- Avoid global mutable state and static/class-level methods that do real work. They can't be substituted or isolated in a test, and every caller becomes implicitly coupled to whatever else touches that global. Inject dependencies instead of reaching for one.

## Pitfalls

- Deep inheritance chains (3+ levels) — a change to a grandparent method has effects nobody can fully trace; prefer flatter composition.
- "God" interfaces with a dozen methods, forcing every implementer to stub methods it doesn't need — split by role instead (see Interface Segregation in `solid-principles`).
- Public setters for every field, which defeats the point of encapsulation — expose intent-revealing methods (`cancel()`, `applyDiscount()`) instead of `setStatus()`.
- Reaching for inheritance purely for code reuse when there's no real is-a relationship — composition or mixins fit that need better.
- Anemic objects (all public fields, no behavior) paired with separate "service" classes that do everything — this pushes logic out of the object that owns the data and re-litigates encapsulation everywhere else instead of solving it once.

## Checklist

- [ ] State is private; changes go through methods that enforce invariants
- [ ] Composition used for optional/variant behavior; inheritance reserved for true is-a + substitutability
- [ ] Collaborators depended on via interface, injected rather than constructed inline
- [ ] Each class is cohesive (methods use its own fields) and narrowly coupled (small public surface)
- [ ] No global/static mutable state standing in for a real dependency
- [ ] SOLID-specific guidance (single responsibility, open/closed, etc.) lives in `solid-principles`, not duplicated here
