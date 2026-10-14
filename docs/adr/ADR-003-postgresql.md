# ADR-003: PostgreSQL as the Primary Database

- **Status:** Accepted baseline
- **Date:** 2026-09-15
- **Decision owners:** Project team

## Context

The platform needs reliable relational constraints and transactions for tenant data, weighted/serialized jewelry items, stock movements, sales, purchases, payments, subscriptions and audit history. It also needs flexible but bounded configuration fields and a mature EF Core provider.

## Decision

Use PostgreSQL as the cloud platform's primary transactional database and Entity Framework Core as the primary ORM/migration tool. Keep the schema relational for core business concepts and use `jsonb` only for intentionally flexible configurations/snapshots/custom attributes.

## Baseline conventions

- UUID primary keys.
- `timestamptz` for timestamps; persist consistently in UTC.
- `numeric(19,4)` for money amounts and `numeric(19,6)` for gold price values in the current schema baseline.
- `numeric(18,6)` for physical weights/quantities in the current schema baseline.
- `snake_case` tables/columns.
- Explicit foreign keys, unique/check constraints, indexes and delete behavior.
- EF Core migrations generated and reviewed through CI/release workflows.

Column-level precision and relationships must be explicit in EF Core configuration. If multi-currency transactions are enabled, transaction currency is stored alongside amounts; it must not be inferred from today's tenant settings.

## Consequences

### Positive

- Strong transactional integrity and relational constraints.
- Good fit for tenant-scoped relational data and complex queries.
- `jsonb` provides extensibility without moving core data out of relational tables.
- Suitable foundation for one modular monolith and a future multi-instance API.

### Trade-offs

- The team must learn PostgreSQL operational practices, backup/restore, indexing, query plans and migration safety.
- Provider-specific behavior must be covered by PostgreSQL integration tests; SQLite-only tests are insufficient for persistence correctness.

## Alternatives considered

1. **SQLite as the primary SaaS database:** rejected for the shared cloud transactional system; it may be considered for a future offline desktop cache.
2. **Document database:** rejected as primary store because inventory/financial consistency relies heavily on relational references and transactional constraints.

## Revisit when

Capacity, operational constraints, data residency, offline-first design or workload measurements prove that a complementary store or partitioning strategy is necessary. Do not add a second database simply for anticipated scale.
