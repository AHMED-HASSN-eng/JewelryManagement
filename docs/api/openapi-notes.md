# OpenAPI Notes

**Status:** Initial implementation guide • Version 1.0

## 1. Purpose

OpenAPI is the machine-readable API contract for frontend developers, QA, future mobile applications and external integrations. It should be generated from the actual ASP.NET Core endpoints and DTOs, reviewed as part of API changes, and tested in CI.

## 2. Generation and hosting

- Use ASP.NET Core OpenAPI generation supported by the chosen .NET version.
- Keep the generated document available in development/test. Expose production OpenAPI/interactive UI only according to the deployment security policy.
- The team may choose Swagger UI, Scalar or another viewer; the viewer is separate from the OpenAPI document itself and should be recorded in an ADR if it affects tooling.
- Use a stable document name such as `v1` and title such as `Jewelry Management API`.
- Do not manually maintain a completely separate OpenAPI file that drifts from the actual endpoint contracts. Add hand-authored descriptions/examples or overlays only when the build process verifies consistency.

## 3. Required document metadata

- API title, version and concise description.
- Server URLs for local development and documented deployed environments without credentials.
- Global security scheme when most endpoints require authentication.
- Tags aligned with modules (Tenancy, Identity, Catalog, Inventory, Pricing, Sales, Purchasing, Customers, Repairs, Payments, Accounting, Subscriptions, Reports, Audit).
- Stable operation IDs for client generation.
- Request/response schemas and field requirements.
- Success responses plus validation, authorization, not-found, conflict, rate-limit and server-error responses as appropriate.

## 4. Security scheme

Document the selected OAuth 2.0/OIDC or bearer-token scheme accurately after the identity provider and flow are chosen. Never publish a dummy flow as if it were production-ready. Mark endpoints as anonymous only when intentionally public. Document the required permissions/scopes in endpoint descriptions or vendor extensions if the tooling standard supports them.

## 5. Reusable schemas

Standardize reusable schemas for:

- `ProblemDetails` and validation errors.
- Pagination metadata / page or cursor result wrapper.
- Money and precision-sensitive Weight.
- IDs and timestamps.
- Idempotency conflicts and state-transition errors.
- Concurrency/version tokens where used.

Keep enum string values stable. Do not expose internal class names or database fields as a public promise unless that is an intentional API contract.

## 6. Examples and descriptions

Every non-trivial command should have at least one valid request example and a success response example. For jewelry transactions, examples should clearly show weight units, currency, gold purity, applied price snapshot, tax/discount components and the posting result without using real customer data.

Mark nullable fields explicitly. Distinguish a field that is omitted, a field that can be null, and a field that must be supplied. Use consistent names, casing and date/time formats.

## 7. CI quality checks

- Generate the OpenAPI document during CI and fail if generation fails.
- Validate the document with an OpenAPI validator.
- Detect unintended breaking changes against the previously released document when possible.
- Ensure each protected endpoint is documented with security requirements.
- Ensure every operation has an operation ID and appropriate success/error responses.
- Review client generator output when an API change affects web/desktop/mobile integrations.

## 8. Versioning and breaking changes

Prefer additive compatible changes within `/api/v1`. Renaming/removing fields, changing decimal representation, changing enum keys or altering requiredness may be breaking. Require an explicit migration/versioning strategy for breaking changes; do not rely on release notes alone to make an incompatible schema safe.

## 9. Remaining decisions

Before first public API release, approve the UI viewer, API versioning package/approach, auth provider/flow, decimal wire format, pagination convention, and client-generation tool.
