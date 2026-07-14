---
name: design-patterns-catalog
description: Use when choosing a design pattern for object creation, composition, or behavior coordination — a language-agnostic catalog of the GoF patterns that still earn their keep (factory, builder, strategy, decorator, observer, command, repository, specification), plus when a pattern is over-engineering versus the right call.
---

# Design Patterns Catalog

Patterns are names for recurring *shapes* of problem, not a checklist to apply everywhere. Their real value is vocabulary — "make this a Strategy" tells another engineer the intent in two words. Reach for one because the problem recurred, never to prove sophistication.

## When to use

When object creation, composition, or behavior coordination keeps hitting the same shape of friction: a growing `if/switch` picking between variants, a constructor with a dozen optional parameters, one event needing several independent side effects. Don't reach for a pattern before a second, concrete use case demands the flexibility.

## Creational patterns

**Factory** — object construction is conditional or has non-trivial setup, and callers shouldn't know the concrete type.

```
function createGateway(provider):
  match provider:
    "stripe" -> new StripeGateway(stripeClient)
    "paypal" -> new PaypalGateway(paypalClient)
    else -> raise UnknownProvider(provider)
```

**Builder** — many optional parameters or multi-step construction; a fluent, readable assembly beats a ten-argument constructor.

```
query = QueryBuilder()
  .where("status", "active")
  .orderBy("created_at", DESC)
  .limit(50)
  .build()
```

Decide once whether each builder call returns a new instance (immutable) or mutates in place, and stay consistent — mixing the two styles produces surprising aliasing bugs.

## Structural patterns

**Adapter** — integrating a third-party client whose shape doesn't match your domain. Wrap it behind your own interface so the rest of the codebase depends on your abstraction, not the vendor's.

**Decorator** — adding behavior (caching, retries, logging) around an existing implementation without touching it or subclassing.

```
interface ProductRepository:
  findById(id) -> Product

class CachedProductRepository implements ProductRepository:
  field inner: ProductRepository
  field cache: Cache
  findById(id):
    return cache.remember("product:" + id, ttl=3600, () -> inner.findById(id))
```

**Composite** — treating a single item and a collection of items uniformly; common in trees (menus, permission hierarchies, nested comments).

## Behavioral patterns

**Strategy** — an algorithm varies and the caller should pick/swap the variant, replacing an ever-growing conditional.

```
interface DiscountStrategy: apply(order) -> order

function calculatePrice(order, discounts: DiscountStrategy[]):
  return discounts.reduce(order, (o, d) -> d.apply(o)).total()
```

**Observer** — one event should trigger several independent side effects (notify, update inventory, log analytics) without the trigger knowing about its listeners. Most frameworks already provide this as an event bus/pub-sub mechanism — use that before hand-rolling one.

**Command** — encapsulating a request as an object you can queue, log, retry, or undo.

```
class CancelOrderCommand:
  field orders: OrderRepository
  execute(orderId):
    order = orders.findById(orderId) or raise NotFound(orderId)
    order.cancel()
    orders.save(order)
```

**Specification** — business rules that combine with AND/OR/NOT and get reused across filtering, validation, and querying.

```
interface Specification: isSatisfiedBy(candidate) -> bool

class ActiveUser implements Specification:
  isSatisfiedBy(user): return user.isActive()
```

**Repository** — isolating persistence behind a domain-shaped interface so business logic never depends on the storage engine's query language directly. See `clean-architecture-layering` for where repositories sit relative to domain/application layers.

## Patterns to avoid by default

- **Singleton** — makes testing hard by hiding global state. Use your dependency-injection container's singleton *scope* instead of a language-level singleton class.
- **Service locator** (a global "resolve this type" call buried inside business code) — hides a class's real dependencies. Prefer constructor/parameter injection so dependencies are visible in the signature.
- **Active Record with business logic baked in** — couples persistence and domain rules together. Keep query/persistence code thin; once the domain gets non-trivial, move business rules into plain domain objects (Data Mapper style).

## Pitfalls

- Introducing a pattern "just in case" before the second use case exists — that's speculative complexity, not design.
- Misapplying a pattern to code whose shape doesn't match its intent — a misapplied pattern adds indirection without buying flexibility, which is worse than no pattern at all.
- Calling every interface a "Strategy" — Strategy specifically means a swappable *algorithm* selected by the caller, not any old abstraction.
- Decorator chains or Observer fan-out deep enough that control flow can no longer be traced by reading the code — cap the chain, or make the flow explicit in one coordinating function.

## Checklist

- [ ] Pattern adopted because the problem recurred, not preemptively
- [ ] Vocabulary matches intent (Strategy = swappable algorithm, Decorator = behavior wrapping, Factory = conditional construction, …)
- [ ] No Singleton, service-locator, or logic-bearing Active Record introduced
- [ ] A reviewer can trace the resulting control flow in one read
