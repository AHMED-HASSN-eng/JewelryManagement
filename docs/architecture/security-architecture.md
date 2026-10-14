# Security Architecture

**Status:** Initial implementation guide; review before production • Version 1.0

## 1. Security goals

Protect tenant separation, branch permissions, financial and inventory actions, customer personal data, credentials, and audit history. Security controls must be enforced on the server and database boundary; hiding a button in the UI is not authorization.

## 2. Authentication baseline

- Prefer an OAuth 2.0 / OpenID Connect-compatible identity design. The production identity provider choice remains an explicit implementation decision; do not invent a custom token format or password protocol.
- The domain actor/user identity should remain conceptually separable from tenant membership so future users can be members of multiple tenants if product requirements demand it.
- If local credentials are used, store only modern password hashes produced by a maintained identity framework; never store raw passwords or reversible password encryption.
- Access tokens must be short-lived and validated for issuer, audience, signature, expiry and required claims. Refresh-token/session lifecycle, revocation and rotation must be designed with the selected identity provider.
- Use HTTPS for all non-local environments. Do not put access tokens in URLs or logs.

## 3. Tenant context and isolation

### Trusted tenant resolution

Resolve tenant context from a validated authenticated membership/claim and/or trusted tenant-domain mapping. A request body, query parameter or arbitrary `X-Tenant-Id` header must never be accepted as proof that the caller may act for that tenant.

The request pipeline should establish:

1. Authentication and token validation.
2. User/tenant membership validation.
3. Tenant context and allowed branch scope.
4. Endpoint permission and feature entitlement checks.
5. Application use case execution with tenant-scoped data access.

### Defense in depth

- Carry `TenantId` on tenant-owned records or through a documented aggregate boundary.
- Enforce tenant scope in Application queries and commands.
- Use EF Core global query filters as a safeguard where useful, but do not rely on them as the only security control; raw SQL, background jobs and `IgnoreQueryFilters()` must be reviewed.
- Use tenant-aware unique constraints and composite foreign keys for high-risk cross-tenant references where practical.
- Validate tenant ownership for every related identifier submitted to an operation (branch, customer, product variant, inventory item, role, subscription, account).
- Include automated negative tests that try to read, edit, attach, transfer, sell or delete another tenant's records.
- Use explicit system/platform administrative pathways for cross-tenant support; never silently bypass tenant scope.

## 4. Branch-scoped authorization

A user may be restricted to one or more branches. Every branch-level operation checks both tenant ownership and branch permission. The user's default `BranchId` is a convenience, not a grant of access to all branches.

Sensitive permissions should be explicit, including:

- `pricing.change-gold-price`
- `pricing.override-sale-price`
- `sales.apply-discount-over-limit`
- `sales.cancel-posted-sale`
- `sales.refund`
- `inventory.adjust`
- `inventory.transfer.approve`
- `purchasing.post-receipt`
- `repairs.approve-cost` / `repairs.deliver`
- `users.manage` / `roles.manage`
- `reports.view-financial`

Permission keys are stable identifiers; localized labels belong in UI resources.

## 5. Tenant lifecycle and subscription enforcement

- `Pending`, `Suspended` and `Closed` tenants must have explicitly defined access policies.
- A suspended tenant cannot create normal business transactions; allow only documented recovery/support reads and administrative actions.
- Feature entitlements and quota enforcement happen server-side in a central application service/policy. Client-side feature hiding is only a user-experience aid.
- Always preserve history needed for audit, support, business reporting and legally required retention. Data deletion/export must be a separate controlled workflow.

## 6. Financial and inventory protection

- Posted sales, payments, purchases, returns, inventory transactions, buybacks, exchanges, accounting postings and audit events must not be hard-deleted through normal application workflows.
- Correct posted facts with reversals, refunds or explicit adjustments.
- Use database transactions, concurrency tokens/locks, idempotency keys and unique constraints to prevent duplicate payments, duplicate sale posting, duplicate transfer receipt and double-selling a physical item.
- Record actor, tenant, branch, correlation/idempotency key where applicable, reason/approval, and before/after values for sensitive actions.
- Never accept a client-supplied sale total as authoritative; server-side pricing and tax calculations determine the posted total.

## 7. Secrets, configuration and data handling

- No credentials, signing keys, production connection strings or provider secrets in Git.
- Use environment variables for local development and a secret manager in hosted environments.
- Keep development configuration safe to commit; local secret files should be ignored by Git.
- Validate configuration at startup and fail fast for missing mandatory secrets/settings.
- Minimize and redact personal data in logs and audit payloads. Never log passwords, access/refresh tokens, payment credentials or complete sensitive records indiscriminately.
- Define retention and deletion/export policies for customer personal data before production; preserve financial/audit evidence according to applicable market requirements.
- Limit file-upload size/type and scan/validate untrusted files before making them available.

## 8. API and runtime controls

- Apply request size limits, input validation, rate limiting and safe CORS policy.
- Use consistent ProblemDetails errors without stack traces or database internals in production responses.
- Separate readiness/liveness checks; do not expose diagnostic details to anonymous users.
- Use structured security logs and correlation IDs; alert on repeated authentication failures, tenant-scope violations and unusual sensitive operations.
- Run dependency and container vulnerability checks in CI. Patch supported runtime and libraries regularly.

## 9. Minimum security tests

1. Anonymous caller cannot access protected business endpoints.
2. Tenant A token cannot access Tenant B's resources by changing IDs, URLs, filters or payloads.
3. A branch-restricted user cannot access an unauthorized branch.
4. Missing feature entitlement blocks server-side operation.
5. Cashier cannot change gold price, bypass discount thresholds, or perform unapproved refunds.
6. Concurrent attempts cannot sell the same physical item twice.
7. Retried commands with the same idempotency key do not duplicate financial/inventory effects.
8. Audit entries are append-only and redact secrets.
9. Production errors do not disclose secrets, SQL, file paths or stack traces.
10. Background jobs cannot accidentally execute without a tenant/system context explicitly supplied by the job payload.

## 10. Decisions still requiring approval

- Identity provider and exact OIDC flow.
- Whether PostgreSQL row-level security will supplement application-level tenant isolation.
- Tenant membership model for MVP (one tenant per user record versus separate identity and membership tables).
- Customer-data retention/deletion policy for Egypt, Saudi Arabia and future markets.
- Operational security alerting and audit retention periods.
