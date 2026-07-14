---
name: solid-principles
description: Use when designing or reviewing object-oriented code and a class is hard to change, test, or extend without breaking something else — the five SOLID principles (SRP, OCP, LSP, ISP, DIP) with language-agnostic pseudocode per principle and guidance on when violating one is the pragmatic choice. Covers class/interface-level design; see `object-oriented-design-principles` for broader OO fundamentals and `design-patterns-catalog` for the concrete patterns (Strategy, Adapter, …) that implement these principles.
---

# SOLID Principles

Five heuristics for object-oriented design that keep classes easy to change in isolation. They apply in any language with interfaces or structural typing — the syntax differs, the shape of the problem does not.

## When to use

- A class keeps changing for unrelated reasons (a bug fix in billing logic touches the same file as a notification tweak) — that's SRP calling.
- Adding a new case (a new payment method, a new shipping option) means editing a `switch`/`match` buried in existing, already-tested code — that's OCP.
- A subclass needs `if (instanceof X)` checks in code that consumes the base type — that's an LSP violation waiting to be found.
- An interface has methods most implementers stub out with "not supported" — that's ISP.
- A high-level module imports a concrete database/HTTP client/mailer directly instead of an interface — that's DIP.

## S — Single Responsibility Principle

A class (or module) should have **one reason to change** — one actor or concern it answers to.

```
# Bad: persistence, notification, and pricing share one class — a change
# to any of the three risks breaking the other two.
class OrderProcessor:
    process(order):
        save_to_database(order); send_confirmation_email(order); apply_discount(order)

# Good: each concern owns its own reason to change.
class OrderRepository: save(order)
class OrderNotifier:   send_confirmation(order)
class OrderPricer:     apply_discount(order)
```

"Responsibility" means an axis of change, not "does one thing" taken literally — a class with five tiny methods that all change together for the same reason is fine; a class with one method that two unrelated stakeholders both want to change is not.

## O — Open/Closed Principle

Classes should be **open for extension, closed for modification** — add new behavior without editing code that already works and is already tested.

```
# Bad: every new shipping method means editing this function and re-testing
# all the existing branches along with the new one.
function shipping_cost(order):
    match order.shipping_method:
        "standard" -> Money(500)
        "express"  -> Money(1500)
        # adding "international" means modifying this function

# Good: new behavior is a new implementation, not a new branch.
interface ShippingMethod:
    cost(order) -> Money

class StandardShipping      implements ShippingMethod: cost(order) -> Money(500)
class ExpressShipping       implements ShippingMethod: cost(order) -> Money(1500)
class InternationalShipping implements ShippingMethod: cost(order) -> Money(3000)  # added, nothing else touched
```

This is the Strategy pattern — see `design-patterns-catalog` for the general form and other patterns that satisfy OCP the same way (Decorator, Chain of Responsibility).

## L — Liskov Substitution Principle

A subtype must be **substitutable** for its base type — code written against the base type keeps working, unmodified, when handed any subtype.

```
# Violation: Square "is-a" Rectangle in the type system, but setting its width
# silently changes its height too — behavior a Rectangle caller never expects.
class Rectangle:
    set_width(w):  width = w
    set_height(h): height = h
    area(): return width * height

class Square extends Rectangle:
    set_width(w):  width = w; height = w   # surprises any Rectangle-typed caller
    set_height(h): width = h; height = h

# Fix: don't force an inheritance relationship the behavior doesn't support —
# model both as independent implementations of a Shape interface instead.
interface Shape:
    area() -> Number
```

If a subtype needs its consumer to special-case it (`if (shape instanceof Square)`), the inheritance models an "is-a-in-name-only" relationship — replace it with a shared interface, or reshape the base contract so both can honor it.

## I — Interface Segregation Principle

**No client should be forced to depend on methods it doesn't use.** A fat interface forces every implementer to fake support for capabilities it doesn't have.

```
# Bad: a fat interface — Robot is forced to implement methods that make no
# sense for it, usually as empty/throwing stubs.
interface Worker: work(); sleep(); eat()

class Robot implements Worker:
    work(): ...
    sleep(): pass   # doesn't apply — dead code that still has to compile
    eat():   pass

# Good: segregated interfaces — implement only what applies.
interface Workable:  work()
interface Sleepable: sleep()
interface Eatable:   eat()

class Robot implements Workable: work(): ...
class Human implements Workable, Sleepable, Eatable: ...
```

## D — Dependency Inversion Principle

**Depend on abstractions, not concretions.** High-level policy (business logic) shouldn't import low-level detail (a specific database driver, a specific payment provider) directly.

```
# Bad: high-level logic is welded to a concrete implementation — switching
# database or payment provider means editing OrderService itself.
class OrderService:
    constructor(): repository = MySQLOrderRepository(); gateway = StripePaymentGateway()

# Good: depend on the interface; the concrete choice is injected from outside.
interface OrderRepository: save(order)
interface PaymentGateway:  charge(amount) -> PaymentResult

class OrderService:
    constructor(repository: OrderRepository, gateway: PaymentGateway): ...
```

Now `OrderService` is testable against a fake/in-memory repository and gateway, and swapping the concrete provider never touches this class.

## When violating one is the pragmatic choice

SOLID is a direction to lean, not a law to satisfy unconditionally. SRP taken too far produces a maze of one-method classes where following a single business operation means jumping through ten files — group responsibilities that genuinely change together. OCP applied to something that will never grow a second case (an internal script, a one-off migration) is speculative abstraction — a two-branch `switch` doesn't need a Strategy hierarchy yet. DIP on a leaf module with one implementation and no plausible second one just adds an interface nobody will ever swap — introduce the abstraction when a second implementation becomes real, not before.

Reach for each principle when a class is already causing pain — hard to test, hard to extend, hard to change safely — not as a checklist to satisfy on every class regardless of size or stability.

## Checklist

- [ ] Each class changes for one identifiable reason/actor
- [ ] New cases are added as new implementations, not new branches in existing logic
- [ ] No subtype requires callers to type-check it before using it safely
- [ ] Interfaces are small and role-specific; no stubbed-out "not supported" methods
- [ ] High-level modules depend on interfaces; concrete choices are injected, not constructed inline
- [ ] Abstraction introduced because a second real case exists, not speculatively
