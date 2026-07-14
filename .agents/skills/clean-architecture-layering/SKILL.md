---
name: clean-architecture-layering
description: Use when structuring an application into layers so business logic doesn't depend on frameworks, databases, or transport — domain and use-case code stay pure and testable while infrastructure stays swappable. Covers the dependency rule, layer responsibilities, and repository ports. Complements object-oriented-design-principles and solid-principles at the module-boundary level.
---

# Clean Architecture / Layering

Draw a boundary between *what the software does* (business rules) and *how it's wired up* (framework, database, HTTP, filesystem). Dependencies point one way — inward — so the core logic can be tested, reused, and understood without any of the outer machinery running.

## When to use

Any codebase where business rules are worth protecting from churn in the framework, database, or transport layer — which is most applications past a prototype. Skip the ceremony for a genuine CRUD-only app with no real business rules; layering has a cost, and a thin wrapper around a datastore doesn't need four folders to say so.

## The dependency rule

Dependencies only point inward. An inner layer must never import, reference, or know about an outer one.

```
[Frameworks & Drivers] → [Interface Adapters] → [Application] → [Domain]
```

- **Domain** — entities, value objects, domain events, and the *interfaces* (ports) other layers implement. No framework imports, no ORM base classes, no HTTP types. This code should compile and run with zero infrastructure present.
- **Application** — use cases that orchestrate domain objects to satisfy one specific operation ("place an order," "reset a password"). Depends on domain and on ports (interfaces), never on a concrete database or HTTP client.
- **Interface adapters** — controllers, CLI handlers, repository *implementations*, presenters. Translates between the outside world's shapes (HTTP request, CLI args, DB row) and the application layer's shapes.
- **Frameworks & drivers** — the web framework, database driver, queue client, filesystem. Thin glue only; if this layer contains business rules, the boundary has already leaked.

The test for whether a layer is placed correctly: could you delete every outer layer and still compile/run the inner one in isolation (with a fake or in-memory implementation of its ports)? If domain code fails to compile without the database library present, the dependency rule is already broken.

## Entities and value objects

- **Entities** have identity (an ID) and encapsulate the state transitions that are legal for that concept — not just data, but the rules about how the data may change.

```
Order
  id, status, total
  cancel():
    if status == SHIPPED: raise CannotCancelShippedOrder
    status = CANCELLED
```

- **Value objects** are identity-less, immutable, and self-validating: money, an email address, a URL. Construction is the only place invalidity can enter — once constructed, callers trust it.

```
Email(raw):
  if not looks_like_email(raw): raise InvalidEmail
  value = raw
  domain(): return substring after "@"
```

Modeling a field as a primitive string/number everywhere it appears means every call site re-derives its own validation and formatting; a value object centralizes it once.

## Use cases (application layer)

One use case = one class/module = one operation, with a single entry point (`execute`/`run`/`handle`). It takes an input shape, returns an output shape, and depends only on domain objects and ports — no request/response objects, no session, no framework types leak in.

```
PlaceOrder(orderRepo: OrderRepository, gateway: PaymentGateway, events: EventBus)
  execute(input):
    order = Order.create(...)
    orderRepo.save(order)
    events.publish(OrderPlaced(order.id))
    return { orderId: order.id }
```

A use case that imports an HTTP request object or reads a session directly has quietly become an interface adapter wearing an application-layer name.

## Ports (repository pattern)

The *interface* for talking to infrastructure lives in domain/application; the *implementation* lives in the outer layer. Callers depend on the interface only.

```
interface OrderRepository:
  save(order)
  findById(id) -> Order | null
  findPending() -> Order[]
```

Return domain entities, fully hydrated — never leak a query builder, an ORM cursor, or a raw row map across the boundary. The moment a caller starts chaining query-builder methods on what a repository returned, the abstraction has been defeated.

## Boundaries between modules

- Give each bounded context (order management, billing, identity) its own domain/application/infrastructure split — don't let one giant `Domain/` folder mix concepts that don't share a ubiquitous language.
- Cross-context communication goes through events (async) or a narrow, explicit public interface (sync) — never by importing another context's internal entities directly. If module A needs module B's internals, that's a sign they're really one context.

## Pitfalls

- Anemic layering: folders named `Domain`/`Application`/`Infrastructure` exist, but business rules actually live in controllers or in the ORM models — the structure is cosplay without the dependency rule enforced.
- Framework types (request objects, ORM base classes, decorators tied to a specific framework) imported into domain or application code — the whole point was to keep this code portable.
- Repository interfaces defined in the infrastructure layer instead of domain/application — that inverts the dependency the pattern exists to fix.
- Over-applying this to a genuinely simple CRUD service — four layers for a service with no business rules just adds indirection with no payoff.
- Use cases that return infrastructure objects (raw DB rows, HTTP response builders) instead of plain output data.

## Checklist

- [ ] Domain compiles/runs with zero infrastructure dependencies present
- [ ] Use cases depend on ports (interfaces), not concrete infrastructure
- [ ] Repository interfaces live inward; implementations live outward
- [ ] Entities enforce their own invariants; value objects validate on construction
- [ ] No framework/HTTP/ORM types appear in domain or application code
- [ ] Bounded contexts talk through events or an explicit public interface, never internals
