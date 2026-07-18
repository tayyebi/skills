---
name: mock-data
description: Use when seeding a database with fake data for development, testing, or demos — generating realistic mock records that obey the same schema, constraints, and normalization as the production database.
---

# Mock Data

Mock data is not throwaway — it's the development-time stand-in for every production record the app will ever handle. When the mock schema drifts from the real schema, features develop against a phantom and tests pass on a lie.

## When to use

Any time you populate a non-production database with rows a developer, tester, or demo viewer will interact with. Covers seeding scripts, factory/fixture generators, and migration-adjacent sample data.

## The cardinal rule: same schema, same normalization

**Mock data tables, columns, types, constraints, foreign keys, indexes, and relationships must be identical to production.** The mock data script must produce rows that could be `INSERT`ed into the production schema without a single DDL change.

### What this means concretely

- **Run the same migrations.** Do not create a separate "mock schema" or hand-write `CREATE TABLE` statements that mirror production. Apply the real migrations to the mock target, then populate — never define the schema twice.
- **Same column types, same widths.** A `varchar(255)` that is `TEXT` in the mock target hides truncation bugs. A `DECIMAL(10,2)` replaced with `FLOAT` in the mock target hides floating-point drift bugs.
- **Same nullable constraints.** A column marked `NOT NULL` in production that is nullable in the mock dataset means code paths that accidentally rely on a value being present won't blow up until production.
- **Same foreign key relationships.** If `order_items.order_id` references `orders.id` in production, every mock `order_item` must reference an existing mock `order`. No orphaned child records that production's FK would reject.
- **Same unique constraints and indexes.** A mock dataset without the production unique index on `(tenant_id, email)` won't catch the duplicate-email bug that production will hit on the second insert.
- **Same check constraints.** If production has `CHECK (status IN ('pending', 'active', 'closed'))`, mock data that inserts `'cancelled'` loads fine locally and fails in production.

### Anti-pattern: "convenience" mock schemas

```
# BAD: a separate "seed" table definition that drifted from production
CREATE TABLE mock_users (
    id TEXT,              -- production uses UUID with a default
    name TEXT,            -- production has NOT NULL
    email TEXT            -- production has UNIQUE + NOT NULL
);
```

```
# GOOD: migration first, populate second
$ migrate up                           # apply production migrations
$ seed --reset --count=1000            # populate the migrated schema
```

## Generating mock data

### Approach: factories over hand-crafted INSERTs

A factory is a function that produces a valid row with sensible defaults, letting the caller override only the fields they care about. This scales far better than maintaining multi-row `INSERT` scripts.

```
factory("user", {
    id       : () => ulid(),
    email    : (i) => `user-${i}@example.test`,
    name     : () => fake.name(),
    role     : () => "member",
    createdAt: () => now()
})
```

### Realism vs. determinism

- **Realistic enough to exercise the app** — edge-case names (Unicode, very long, very short), past and future dates, varied but valid enum values, amounts that hit rounding boundaries.
- **Deterministic enough to reproduce a bug** — seed the random generator so the same seed produces the same dataset every time. A non-deterministic mock dataset means a test that passes today might fail tomorrow on the same code.
- **Visibly fake** — never use real-looking PII (names, emails, phone numbers that could belong to actual people). Use `@example.test` domains, `555-` phone prefixes, or generators that explicitly produce synthetic output.

### Respect production invariants

- If production has a per-tenant uniqueness constraint, mock data spread across tenants must satisfy it per-tenant, not just globally.
- If production has a state machine (e.g., an order must be `paid` before it can be `shipped`), mock data must respect the same valid transitions — an order inserted directly as `shipped` without a `paid_at` timestamp will hide validation gaps that production's workflow enforces.
- If production cascades deletes, mock data generators that write child records must account for the fact that deleting a parent row in a test also deletes the children.

## Volume and performance

- Generate enough data to surface N+1 queries and pagination bugs — if the production table is expected to hold 100K rows, a mock dataset of 20 rows won't reveal that the list endpoint fetches every row without a `LIMIT`.
- But don't make seeding slow in the common case. Default to a small dataset (enough for a smoke test), with a flag or env var to scale up (`--count=100000`) for performance-sensitive work.
- Use bulk inserts, not row-by-row single-statement inserts, when generating more than a few hundred records — the I/O difference is orders of magnitude.

## Seeding lifecycle

- **Idempotent**: running the seed command twice produces the same final state, not duplicate rows. Use `ON CONFLICT` upserts keyed on a stable natural key, or truncate-and-reload for small datasets.
- **Clean separation**: seeding is a distinct command from migration. Migrations run first; seeding populates what migrations created. Never embed seed data inside a migration file.
- **Environment-gated**: seed commands that truncate tables must refuse to run against a production database (check `DATABASE_URL` or an explicit `--i-know-what-im-doing` flag).

## Pitfalls

- **Maintaining a parallel schema** — the most common and most destructive mistake. Every DDL change in production must be manually mirrored in the mock schema; one will eventually be missed, and from that point forward the mock environment is testing a different database than the one your users use.
- **Deterministic IDs that collide** — hardcoded IDs (`id: 1`, `id: 2`) collide with auto-increment sequences or cause FK violations when run against a non-empty database. Use a factory that generates per-run IDs or use `ON CONFLICT` upsert logic.
- **Mock data that masks validation gaps** — a mock dataset where every email is `user@example.com` and every phone is `555-0000` won't reveal that the app accepts invalid email formats or phone numbers, because the mock data never exercises the validation code paths.
- **Too small to surface performance bugs** — 10 rows in every table. Every query is fast; pagination code is never exercised; indexes are never needed.
- **Too slow to run routinely** — 10 million rows generated synchronously on `npm run dev`. Developers stop running it; the mock data becomes a once-per-quarter ritual that drifts.
- **Production-adjacent PII** — using real names, real email addresses, or real phone numbers in mock data. Someone will eventually export the mock database, email the mock users, or trigger a real notification to a real person.

## Checklist

- [ ] Mock schema is the production schema, applied via the same migrations — no parallel DDL
- [ ] Factories produce rows that would pass every `NOT NULL`, `UNIQUE`, `CHECK`, and FK constraint in production
- [ ] State-machine invariants are respected (valid transitions, required predecessor states)
- [ ] Per-tenant uniqueness is honored, not just global uniqueness
- [ ] RNG is seeded for reproducibility; `--count` flag controls dataset size
- [ ] Seeding is idempotent (safe to re-run without duplication)
- [ ] Seeding uses bulk inserts above a few hundred rows
- [ ] Mock data is visibly synthetic — no real PII
- [ ] Seed command is gated from running against production
