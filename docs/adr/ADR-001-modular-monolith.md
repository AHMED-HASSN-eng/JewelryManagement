# ADR-001: Start with a Modular Monolith

- **Status:** Accepted baseline
- **Date:** 2026-09-15
- **Decision owners:** Project team

## Context

The product must support multi-tenant jewelry operations, market configuration, desktop/web clients and future mobile/e-commerce capability. Early requirements will evolve while the team learns the real shop workflows. Microservices would introduce distributed transactions, deployment complexity, service observability and operational overhead before module boundaries and workload are validated.

## Decision

Build one ASP.NET Core deployable backend, divided into explicit bounded-context modules, with a clean dependency direction:

- Domain contains framework-independent business concepts and invariants.
- Application orchestrates use cases and defines ports.
- Infrastructure implements persistence and external adapters.
- API exposes versioned HTTP contracts.

Keep module ownership clear even though modules initially share one PostgreSQL database and deployment unit.

## Consequences

### Positive

- Simple local development, deployment, debugging and transaction management.
- Easier atomic handling of sales, inventory and payments in one database transaction.
- Clear boundaries allow later extraction if justified.
- Lower infrastructure cost while the MVP validates product-market fit.

### Trade-offs

- Requires architecture tests and code review to prevent a “big ball of mud.”
- Modules share process and deploy together initially.
- A future extraction requires explicit integration contracts, data ownership and deployment planning.

## Alternatives considered

1. **Microservices from day one:** rejected due to high operational and distributed-consistency costs for the initial team/product stage.
2. **Unstructured single project:** rejected because domain ownership and dependency direction would erode as features grow.

## Revisit when

A module requires independent scaling, independent release cadence, a different reliability boundary, or a separate team ownership model—and metrics/operational needs justify the additional complexity.
