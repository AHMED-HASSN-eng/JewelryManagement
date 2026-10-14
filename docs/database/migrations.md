# Database Migrations — EF Core + PostgreSQL

**Status:** Initial implementation guide • Version 1.0

## 1. Purpose

This file governs how schema changes are created, reviewed, tested and deployed. `final-tables.md` describes the intended table baseline; it does not mean all 52 tables must be created in the first migration. Build schema incrementally according to Jira/MVP dependencies.

## 2. Migration ownership and location

- EF Core configurations and migrations live in `JewelryManagement.Infrastructure` (or a dedicated migrations project if deployment later requires it).
- Use one clearly named context for the initial business database, for example `JewelryManagementDbContext`.
- The API project is the startup project for design-time configuration unless an explicit design-time factory is adopted.
- Group `IEntityTypeConfiguration<T>` files by domain module and keep them separate from domain entity classes.

## 3. Local commands

Run commands from the repository root after the projects and context have been created. Adjust the context name only if a reviewed architecture decision changes it.

```bash
# Create a migration in Infrastructure
 dotnet ef migrations add InitialPlatformSchema \
   --project src/JewelryManagement.Infrastructure \
   --startup-project src/JewelryManagement.Api \
   --context JewelryManagementDbContext \
   --output-dir Persistence/Migrations

# Apply to the configured local development database only
 dotnet ef database update \
   --project src/JewelryManagement.Infrastructure \
   --startup-project src/JewelryManagement.Api \
   --context JewelryManagementDbContext

# Generate SQL for review / CI deployment
 dotnet ef migrations script --idempotent \
   --project src/JewelryManagement.Infrastructure \
   --startup-project src/JewelryManagement.Api \
   --context JewelryManagementDbContext \
   --output artifacts/migrations.sql
```

Do not run a production database update from a developer workstation as the normal release procedure. Exact commands can be added to repository scripts after the solution is scaffolded.

## 4. Naming and authoring rules

- Use clear imperative names such as `AddTenantAndBranch`, `AddCatalogTables`, `AddInventoryLedger`, `AddSalePostingConstraints`.
- One migration should represent a coherent, reviewable change.
- Do not edit a migration that has been applied to any shared environment. Add a new forward migration.
- Keep generated model snapshot changes committed with the migration.
- Avoid mixing unrelated schema changes merely to reduce migration count.
- Keep seed data deterministic and idempotent. System permissions/features and baseline market/reference data should be seeded with stable keys and safe upsert semantics.
- Use explicit PostgreSQL types, precision, default values, unique indexes, check constraints and FK delete behavior.

## 5. Required review checklist

For every migration, review:

- Are all new fields nullability/defaults safe for existing rows?
- Are money/weight/rate columns using the defined precision?
- Do indexes match actual tenant-scoped lookup patterns?
- Are tenant ownership and branch ownership preserved by foreign keys/constraints?
- Could a new unique constraint fail because existing data contains duplicates?
- Does the change lock/rewrite a large table or cause significant downtime?
- Is any destructive operation (drop/rename/type narrowing) handled with expand-and-contract?
- Are historical/posted records preserved?
- Does the migration run against an empty database and an upgraded prior version?
- Is rollback/forward-recovery documented and tested?

## 6. Expand-and-contract for breaking changes

For production data, prefer a phased change:

1. Add a new column/table or compatible representation.
2. Deploy code that can read the old and new representations, if necessary.
3. Backfill data in a controlled, observable process.
4. Switch writes/reads to the new representation.
5. Verify data and metrics.
6. Remove the old representation in a later release, only after all deployed code no longer uses it.

Avoid combining rename/drop and application code change into one risky deployment where old/new application versions cannot both operate safely.

## 7. Production deployment policy

1. CI builds, runs tests and generates migration SQL.
2. Review generated SQL for destructive/locking operations.
3. Back up the database and verify recent backup success before risky migrations.
4. Test upgrade on a production-like database with representative data size.
5. Apply through a controlled, serialized deployment job using least-privilege credentials.
6. Deploy compatible application code in a planned order.
7. Verify schema, health checks, error rates and key workflows.
8. If failure occurs, stop rollout and use a tested recovery path. Forward-fix is usually safer than assuming schema rollback is lossless.

## 8. Concurrency and schema state

- Use unique constraints and idempotency records to prevent duplicate commands.
- Use optimistic concurrency tokens or an explicit version column for mutable aggregates where competing updates are possible.
- Never rely on application checks alone for critical uniqueness such as tenant-scoped SKU/barcode where a database unique index is appropriate.
- Avoid runtime automatic schema mutation in production on every API startup. Migrations should be an explicit release step.

## 9. Tenant and data lifecycle

The initial model uses shared PostgreSQL database/schema with tenant-scoped rows, as described in `../adr/ADR-002-multi-tenancy.md`. Migration scripts must not assume a database per tenant. New tenant-owned tables need the appropriate tenant ownership path, indexes and isolation tests. Historical transactional/audit tables must not gain casual cascade-delete behavior.

## 10. Definition of Done for a schema task

- Entity configuration compiles and has explicit precision/required relationships.
- Migration is generated, inspected and committed.
- Upgrade test passes from empty and previous supported schema.
- Constraints/indexes and tenant isolation are integration-tested against PostgreSQL.
- Seed data is repeatable and has no environment-specific secrets.
- Documentation and `final-tables.md` are updated if the approved logical schema changed.
