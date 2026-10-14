# API Guidelines

**Status:** Initial implementation guide • Version 1.0

## 1. API contract principles

- REST is the public API style for web, desktop, future mobile and integrations.
- Use versioned URLs beginning with `/api/v1/` for the first stable public contract.
- Backend use cases own business rules. API endpoints validate transport input, establish actor/tenant context, call Application, and map results to HTTP responses.
- Do not expose EF Core entities or Domain aggregate classes as request/response contracts. Use explicit DTOs.
- Document every supported endpoint in OpenAPI.

## 2. Resource and method conventions

Use plural resource names and standard HTTP semantics:

| Intent | Example | Expected method |
|---|---|---|
| List/search | `/api/v1/products` | `GET` |
| Get details | `/api/v1/products/{productId}` | `GET` |
| Create draft/resource | `/api/v1/sales` | `POST` |
| Update editable resource | `/api/v1/customers/{customerId}` | `PATCH` or `PUT` by chosen convention |
| Execute a named transition | `/api/v1/sales/{saleId}/complete` | `POST` |
| Cancel/reverse | `/api/v1/sales/{saleId}/cancel` | `POST` command; do not physically delete posted sale |

Choose and document whether partial updates use `PATCH` or full replacement uses `PUT`; do not use `PUT` for a business action with irreversible effects.

## 3. HTTP status conventions

- `200 OK`: successful read/update or operation returning a representation.
- `201 Created`: resource created; include `Location` where appropriate.
- `202 Accepted`: asynchronous process accepted but not complete.
- `204 No Content`: successful action with no response body.
- `400 Bad Request`: malformed request or failed transport-level validation.
- `401 Unauthorized`: authentication is absent/invalid.
- `403 Forbidden`: authenticated caller lacks permission/feature access.
- `404 Not Found`: resource does not exist in the caller's allowed scope. Avoid leaking cross-tenant resource existence.
- `409 Conflict`: business state/concurrency/idempotency conflict.
- `422 Unprocessable Entity`: optional convention for domain validation failure; use consistently if adopted.
- `429 Too Many Requests`: rate limit reached.
- `500`/`503`: unexpected/internal/dependency failures with safe ProblemDetails.

## 4. Error format

Use ASP.NET Core `ProblemDetails` consistently. Error responses should include stable machine-readable `type`/`code`, human-readable `title`/`detail`, HTTP status and trace/correlation identifier. Validation errors should identify input fields where safe. Do not return stack traces, SQL text, connection strings, secrets or private cross-tenant details.

## 5. Tenant and branch context

- Resolve tenant from validated membership/claims or trusted domain mapping; don't accept request `tenantId` as proof of authorization.
- Tenant-owned route identifiers are always checked against the active tenant.
- Branch-bound operations validate both branch ownership and caller branch permissions.
- If tenant/branch identifiers appear in route bodies for explicit selection, treat them as requested context and validate against authorized scope.
- Keep a consistent endpoint convention: tenant is implicit in the authenticated context for most business routes; use explicit platform-admin endpoints only for platform-level administration.

## 6. Validation and business commands

- Validate request shape and basic fields at the boundary; enforce business invariants again in Domain/Application.
- Posted sales, purchases, returns, buybacks, exchanges and transfers should use explicit command endpoints/use cases rather than generic entity updates.
- All important command requests should define required permissions, feature entitlement, tenant/branch scope, transaction boundary, idempotency behavior and audit requirements.
- Client-provided totals, tax values and pricing amounts are never authoritative. Server calculation produces the committed result.
- Posted transaction corrections use reversal/return/adjustment workflows.

## 7. Pagination, filtering and sorting

- List endpoints must paginate once record counts can grow. Prefer an explicit page size default and maximum; reject or clamp unreasonably large sizes.
- Return a consistent wrapper with items and pagination metadata or use cursor pagination; pick one convention per API version.
- Allow-list sortable/filterable fields. Never interpolate arbitrary user-provided sort values into SQL.
- Queries must always be tenant-scoped; branch scoping applies when relevant.
- Use opaque continuation tokens for cursor pagination if implementation requires stable paging under concurrent writes.

## 8. Idempotency and concurrency

- Support an `Idempotency-Key` header or equivalent request token for commands where retries could duplicate effects: sale completion, payment confirmation, return completion, buyback/exchange completion, transfer shipment/receipt and external callbacks.
- Scope idempotency keys by tenant and operation, store the result/operation reference durably, and reject key reuse with a different request payload.
- Use database constraints and concurrency tokens in addition to checks in application code.
- Return `409 Conflict` for state transition/concurrency conflicts and include a stable error code.

## 9. Money, weight, dates and localization

- The server owns authoritative financial and gold-pricing calculations; frontends are display/input clients.
- Recommended wire representation for precision-sensitive decimals is a decimal string in an explicit object, e.g. `{"amount":"1234.5000","currency":"EGP"}` and `{"value":"5.123400","unit":"g"}`. Avoid JavaScript binary floating-point arithmetic for money/weight. If the implementation instead serializes JSON numbers, require decimal-safe client libraries and document the choice consistently before first release.
- Timestamps use ISO-8601/RFC3339 with UTC offset; the database stores UTC `timestamptz`. Business-day calculations use configured tenant/branch timezone.
- Currency codes use ISO 4217 and language codes follow the agreed locale convention. Localize display labels, not persisted enum keys or database identifiers.
- Persist the currency, gold price, purity, tax and pricing-rule snapshots actually used by posted transactions.

## 10. Security and observability

- Protected endpoints require authentication and explicit authorization policies/permissions.
- Feature entitlement and usage limit checks occur on the server.
- Apply appropriate request size limits, rate limits and CORS configuration.
- Propagate trace/correlation ID; logs should not contain credentials, tokens or full sensitive payloads.
- API handlers and downstream I/O honor `CancellationToken`.

## 11. Testing requirements

Every endpoint/use case should include relevant tests for success, validation failure, unauthenticated access, forbidden permission, cross-tenant access, wrong-branch access, concurrent duplicate requests and safe error output. Critical financial/inventory endpoints need integration tests against PostgreSQL.
