# ADR-002: Shared-Schema Multi-Tenancy with Explicit Tenant Isolation

- **Status:** Accepted baseline; security controls must be verified in implementation
- **Date:** 2026-09-15
- **Decision owners:** Project team

## Context

The platform serves multiple jewelry companies. Each company owns users, branches, customers, suppliers, catalog records, inventory, sales, settings and reports. Cross-tenant exposure would be a critical security defect. The first release needs an affordable and maintainable deployment model.

## Decision

Use a shared PostgreSQL database and shared schema with tenant-scoped rows for the initial release. Each tenant-owned table carries `tenant_id` directly or follows a documented aggregate-child rule. Tenant context is established from validated authentication/membership or trusted domain mapping, not from an untrusted request field.

Enforce tenant ownership through application use cases, authorization, EF Core query filters as defense in depth, tenant-aware database constraints/foreign keys where practical, and automated negative tests. Branch authorization is an additional scope check, not a replacement for tenant isolation.

## Consequences

### Positive

- Lower infrastructure and migration complexity than database-per-tenant.
- Shared reporting and platform operations are easier.
- One migration path and shared schema simplify the MVP.

### Trade-offs

- Requires discipline for every query, background job, report, import and external callback.
- A bug can have wider impact, so isolation tests and monitoring are mandatory.
- Per-tenant backup/restore or data residency needs may require additional design later.

## Alternatives considered

1. **Database per tenant:** deferred due to operational/migration overhead for the initial product stage.
2. **Schema per tenant:** deferred because schema lifecycle and cross-tenant platform operations become more complex.
3. **Client-supplied TenantId without trusted membership checks:** rejected as insecure.

## Required safeguards

- Validate the tenant and all linked row ownership on every command.
- Never trust `tenantId` in body/query/header as authorization evidence.
- Add composite uniqueness/FK patterns for high-risk relationships where practical.
- Test cross-tenant reads, writes, relationship injection and report filters.
- Do not use `IgnoreQueryFilters()` without a reviewed privileged use case.
- Background jobs must carry explicit tenant/system context.
- Define an audited privileged support path for cross-tenant operations.

## Revisit when

Legal/data-residency commitments, tenant-specific backup/restore, very large tenant isolation needs or custom operational requirements justify schema/database-per-tenant or database row-level security as an additional layer.
