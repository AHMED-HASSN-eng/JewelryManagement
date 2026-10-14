# Jewelry Management System — Solution Architecture

**Status:** Architecture baseline  
**Version:** 1.0  
**Updated:** 14 October 2026

## 1. Purpose

This document defines the .NET solution, project responsibilities, dependency direction, module layout, API boundary, database strategy, testing approach, and deployment principles. It is the implementation guide for the first cloud-backed release.

## 2. Architecture decision

Build a **modular monolith** first: one deployable ASP.NET Core API containing explicitly separated business modules. Keep domain behavior independent of frameworks. Extract services only when scale, reliability, deployment independence, or team ownership demonstrates a real need.

The backend is API-first. Desktop, web, and future mobile clients use the same supported API rather than connecting directly to PostgreSQL or duplicating business logic.

## 3. Technology baseline

| Area | Baseline | Notes |
|---|---|---|
| Runtime and language | .NET 10 LTS / C# 14 | Keep security and servicing patches current |
| HTTP API | ASP.NET Core 10 | REST API, middleware, health checks and dependency injection |
| ORM | Entity Framework Core 10 | Mappings, transactions and migrations |
| Database | PostgreSQL | Primary cloud system of record |
| API contract | REST + OpenAPI | Versioned, client-independent contract |
| Logging/observability | Structured logs + OpenTelemetry | Correlation IDs, traces, metrics and health |
| Frontend | React + TypeScript + Vite | Web/admin and Electron renderer |
| Desktop | Electron + React + TypeScript | API client; no business rules in renderer |
| Cache | Redis only when justified | Do not make MVP depend on it without a measured need |
| Background work | Hosted services initially | Add durable job/queue infrastructure when workload requires it |
| Object storage | S3-compatible abstraction when needed | Store image/document bytes outside relational rows |
| Containers | Docker | Repeatable local and deployment environments |
| Mobile | React Native later | Uses the same versioned API |

Packages and patch versions should be pinned in project files and upgraded through normal dependency review. Reconfirm current supported versions before creating the solution if implementation starts after this baseline becomes stale.

## 4. Repository layout

```text
JewelryManagement/
├── JewelryManagement.sln
├── README.md
├── Directory.Build.props
├── .editorconfig
├── .gitignore
├── global.json                     # optional: pin SDK version for team reproducibility
├── src/
│   ├── JewelryManagement.Api/
│   ├── JewelryManagement.Application/
│   ├── JewelryManagement.Domain/
│   └── JewelryManagement.Infrastructure/
├── tests/
│   ├── JewelryManagement.UnitTests/
│   ├── JewelryManagement.IntegrationTests/
│   └── JewelryManagement.ArchitectureTests/
├── docs/
└── infrastructure/
    ├── docker/
    ├── deployment/
    └── scripts/
```

Each project directory contains its own `.csproj`; do not leave the actual API `.csproj`, `Program.cs`, `appsettings.json`, or API launch settings at the repository root. Root files should describe or coordinate the solution, not act as a hidden fifth application project.

## 5. Project responsibilities

### `JewelryManagement.Domain`

Owns entities, aggregate roots, child entities, value objects, domain enums, invariant checks, domain services and domain events. It must not reference ASP.NET Core, EF Core, PostgreSQL providers, web clients, logging infrastructure, payment SDKs, or file-system APIs.

Suggested folders:

```text
Domain/
├── Shared/              # Money, Weight, Result/error primitives if adopted
├── Tenancy/
├── Identity/
├── Catalog/
├── Inventory/
├── Pricing/
├── Sales/
├── Purchasing/
├── Customers/
├── Repairs/
├── Payments/
├── Accounting/
├── Subscriptions/
└── Auditing/
```

Create files only when implementing a real domain concept. Avoid empty folders and generic `Helpers`/`Managers` dumping grounds.

### `JewelryManagement.Application`

Owns use cases and orchestration: commands, queries, handlers, request/response DTOs, validation, authorization requirements, transaction coordination and ports/interfaces for external capabilities. Examples include `CreateTenant`, `CreateProduct`, `ReceivePurchase`, `CalculatePrice`, `CompleteSale`, `CompleteReturn` and `ReceiveStockTransfer`.

It may reference Domain. It should not contain EF Core mapping classes, HTTP controller concerns, or provider-specific code.

### `JewelryManagement.Infrastructure`

Implements persistence and external adapters: EF Core `DbContext`, configurations, repositories where they add value, migrations, database seeding, gold-price providers, storage, external payment adapters, notification adapters, caching and background workers. It references Application and Domain.

Suggested folders:

```text
Infrastructure/
├── Persistence/
│   ├── JewelryManagementDbContext.cs
│   ├── Configurations/       # IEntityTypeConfiguration<T>, grouped by module
│   ├── Migrations/
│   ├── Interceptors/         # audit/concurrency only where appropriate
│   └── Seeding/
├── Identity/
├── Pricing/
├── Payments/
├── Files/
├── Notifications/
├── BackgroundJobs/
└── Observability/
```

Do not make provider-specific behavior part of Domain.

### `JewelryManagement.Api`

Owns the HTTP surface: endpoint/controller definitions, authentication middleware integration, HTTP contracts, API versioning, OpenAPI, ProblemDetails, rate limiting, health endpoints and composition root. `Program.cs` registers Application and Infrastructure implementations. Endpoints should call application use cases; they should not query `DbContext` directly or implement business rules.

Suggested folders:

```text
Api/
├── Endpoints/ or Controllers/    # choose one style and use it consistently
├── Contracts/
├── Middleware/
├── Authorization/
├── OpenApi/
├── Configuration/
├── Health/
├── Program.cs
├── appsettings.json
├── appsettings.Development.json  # local overrides only; no secrets committed
└── Properties/launchSettings.json
```

Use either minimal API endpoints or controllers based on a documented team convention; don't mix styles casually.

## 6. Project reference direction

```text
JewelryManagement.Api  ───────> JewelryManagement.Application
        │                              │
        │ composition root              ▼
        └──────────────────────> JewelryManagement.Infrastructure
                                       │
                                       ├────────> Application
                                       └────────> Domain

JewelryManagement.Application ────────────────> Domain
```

- Domain references no other solution project.
- Application references Domain.
- Infrastructure references Application and Domain.
- Api references Application and Infrastructure only to wire dependencies at startup. API business endpoints must not depend on Infrastructure implementation types.
- Tests reference only the projects they need.
- Add ArchitectureTests to prevent forbidden references and cycles.

## 7. Bounded contexts

| Module | Primary responsibility |
|---|---|
| Tenancy & Markets | Tenant, branch, market defaults and overrides |
| Identity & Access | User, role, permission and authorization |
| Catalog | Category, brand, product, variants and media references |
| Inventory | Physical items, inventory ledger, transfers and adjustments |
| Gold & Pricing | Gold type, purity, market price, pricing rules/calculation |
| Sales & POS | Sale lifecycle, sale items, invoices and returns |
| Purchasing | Suppliers, purchase orders, receiving and purchase costs |
| Customers & CRM | Customer profile and customer history references |
| Repairs & Services | Repair intake, item/service lines, status and handoff |
| Payments | Payment records, allocations, refunds and provider contracts |
| Accounting | Accounts, postings and reconciliations when enabled |
| Subscriptions | Plans, features, entitlements and limits |
| Reporting | Read models and authorized reports; later expand if needed |
| Auditing | Append-only security and sensitive business audit records |

Modules may share Domain primitives but must not bypass another module's use cases to mutate its data. Start with one transactional database and one primary DbContext; maintain logical ownership through namespace, application interfaces, configurations and tests.

## 8. Persistence conventions

- PostgreSQL is the cloud system of record.
- Use explicit EF Core configurations rather than relying on conventions for monetary, weight, status, delete and relationship behavior.
- Physical table/column names use `snake_case`; C# types/properties use PascalCase.
- UUID primary keys; use a time-ordered UUID strategy if consistently supported and adopted.
- `timestamptz` for event timestamps, stored consistently in UTC; use tenant/branch IANA timezone for business-day interpretation.
- `numeric(19,4)` for money amounts and `numeric(19,6)` for gold price values in the current database baseline; physical weights/quantities use `numeric(18,6)`. Confirm every column precision in EF configuration.
- Use `jsonb` only for vetted extensibility/configuration, snapshots, and custom fields; do not store core relational concepts in JSON.
- Apply tenant-aware unique indexes and composite constraints to prevent cross-tenant references where appropriate.
- Do not hard-delete posted sales, purchases, payments, inventory movements, accounting postings or audit records; use explicit reversals or lifecycle states.

See `../database/database-schema.md`, `../database/final-tables.md` and `../database/migrations.md`.

## 9. Transaction and cross-module workflows

A single business operation must commit all of its local database effects atomically. For example, completing a sale must validate the physical item and price, write the posted sale and line snapshots, create payment records when applicable, append inventory movement records, and write audit/outbox records as designed.

Application orchestration coordinates multiple aggregate roots. Each aggregate owns its invariants; a sales handler must ask Inventory to perform a stock operation rather than mutating InventoryItem internals directly. For external effects (email, provider calls, integrations), persist a durable intent/outbox record inside the transaction and process it asynchronously when required. Domain events express facts that have already happened; they are not a substitute for commands.

## 10. Configuration, errors and observability

- Bind configuration to validated options and fail fast for required configuration.
- Keep secrets in environment variables or a secret manager; never commit production credentials.
- Use centralized exception handling and consistent ProblemDetails responses.
- Use structured logs with correlation/trace IDs. Do not log passwords, tokens, payment secrets, or unnecessary personal data.
- Add liveness/readiness health checks with dependency checks appropriate to each endpoint.
- Propagate `CancellationToken` through I/O operations.
- Use OpenTelemetry for traces/metrics when the instrumentation is ready; logs alone are not full observability.

## 11. Testing strategy

- Unit tests validate domain invariants and application behavior without a real external provider.
- Integration tests use PostgreSQL-compatible infrastructure, preferably a real disposable PostgreSQL instance, to verify EF mappings, constraints, transactions, tenant isolation and migrations. Do not assume SQLite tests prove PostgreSQL behavior.
- Architecture tests enforce dependency direction and module boundaries.
- API tests verify authorization, validation, ProblemDetails and OpenAPI output.
- Critical end-to-end scenarios include tenant isolation, two terminals competing to sell one physical item, sale posting atomicity, return limits, transfer idempotency, price snapshots and role permissions.

## 12. Deployment and future evolution

Build and test each change in CI. Apply database migrations through a controlled deployment step, not as an unreviewed side effect of every API startup in production. Deploy the API and PostgreSQL independently; use backups and restore drills. Add Redis, durable jobs, object storage, reporting database, offline synchronization, mobile app or service extraction only when a demonstrated requirement exists.

## 13. Definition of architecture compliance

A feature is architecture-compliant when business invariants are in Domain, orchestration is in Application, EF Core/external adapters are in Infrastructure, HTTP concerns stay in Api, tenant/branch scopes are enforced, all persistence precision/constraints are explicit, tests cover the important invariants, and the relevant docs/ADR are updated when design changes.
