# Module Boundaries

**Status:** Initial implementation guide • Version 1.0

## 1. Objective

Define what each module owns and how modules interact in the modular monolith. A module boundary is a business ownership boundary, not merely a folder. Code in one module must not directly mutate another module's aggregate or treat its database tables as public API.

## 2. Module catalog

| Module | Owns | Primary aggregate roots / records | Public capabilities |
|---|---|---|---|
| Tenancy & Markets | Tenant lifecycle, branches, market defaults and overrides | Tenant, Branch, Market | Resolve tenant, validate branch ownership, retrieve market configuration |
| Identity & Access | Users, roles, permissions, authentication integration | User, Role, Permission and membership links | Authenticate/identify actor, authorize capability and branch scope |
| Subscriptions | Plans, features, plan entitlements and tenant overrides | Plan, Subscription, TenantFeature | Check feature entitlement and quota |
| Catalog | Category, brand, product, translation, image, variant/reference assignment | Product, ProductVariant, Category, Brand | Find and manage products/variants/reference data |
| Inventory | Physical item state, movement ledger, stock transfers and adjustments | InventoryItem, StockTransfer | Reserve/validate item, post stock movement, transfer/receive stock |
| Gold & Pricing | Gold types, purity references, gold price history and pricing rules | GoldPrice, PricingRule | Resolve effective price and calculate a price quote/snapshot |
| Customers & CRM | Customer and customer-address profiles | Customer | Create/update/deactivate customer and perform allowed lookup |
| Purchasing | Supplier records, purchase orders, purchase receiving/posting | PurchaseOrder, Purchase, Supplier | Validate order and post a receipt/cost with inventory integration |
| Sales & POS | Sale lifecycle, sale lines, invoice snapshot and returns | Sale, Return | Create draft, calculate totals through Pricing, post/cancel via reversal |
| Payments | Payment lifecycle, allocations, refund/provider contracts | Payment and allocation records | Record/confirm/reverse payment and reconcile settlement result |
| Repairs & Services | Repair intake, item lines, status history and release | Repair | Receive, progress, price/approve and deliver repair work |
| Accounting | Account chart, posted financial movements and reconciliation | Account, AccountTransaction | Apply accounting postings from operational facts |
| Reporting | Read models, projections, report queries and exports | Read models; not a write authority | Return authorized tenant/branch-scoped reporting data |
| Auditing | Append-only audit events | AuditLog | Record sensitive actor/action/context/change facts |

## 3. Rules for module ownership

1. Each aggregate root and its mutable children have one owning module.
2. A module changes its aggregate through its own application use cases/domain methods.
3. Other modules may hold IDs/references and request behavior through application-level interfaces/use cases.
4. Shared reference data is not automatically a shared writable model. For example, Catalog owns Product and ProductVariant; Inventory owns InventoryItem and its lifecycle.
5. Reporting reads approved projections or queries; it must not be used to write business data.
6. Audit records describe actions but do not decide whether those actions are authorized.
7. Shared primitives (`Money`, `Weight`, `DateRange`, `Result` if adopted) belong in a small `Domain.Shared` area. Do not move module-specific rules into Shared.
8. Database tables are not a module interface. Direct cross-module `DbContext` queries that bypass a use case require architectural review.

## 4. Cross-module workflow ownership

### Complete a sale

- Sales owns the Sale aggregate and sale lifecycle.
- Pricing returns a validated quote with input/version snapshots.
- Inventory validates/reserves/releases the physical item and appends the required movement.
- Payments records payment effects through the defined payment capability.
- Auditing records the sensitive operation.
- The Application handler coordinates the transaction; the UI never coordinates these writes itself.

### Receive a purchase

- Purchasing owns Purchase/PurchaseOrder lifecycle and supplier cost snapshots.
- Catalog supplies the product variant definition.
- Inventory creates/updates physical item state and appends movement entries.
- Accounting consumes the posted purchasing fact if accounting is enabled.

### Transfer between branches

- Inventory owns StockTransfer and its `StockTransferItem` rows.
- Tenancy validates that both branches belong to the same tenant and are operational.
- Shipment/receipt operations are idempotent and create exactly-once effective movement records.

### Gold buyback/exchange

- The customer-service transaction owns the buyback/exchange workflow and valuation snapshots.
- Pricing supplies the valuation policy/price where appropriate.
- Inventory records acquired/returned physical items.
- Payments and Accounting own financial effects.
- One completed operation must not double-record stock or money.

## 5. Dependency rules

```text
Api -> Application -> Domain
Api -> Infrastructure (startup composition only)
Infrastructure -> Application + Domain
Domain -> no other solution project
```

Do not add `Catalog -> Sales -> Inventory` project references. Keep project references at the layer level and use module interfaces/use cases inside the Application layer to preserve ownership.

## 6. When to add a new module

Create a module when it has a distinct business language, lifecycle/invariants, data ownership, or change cadence. Do not create modules merely because a class has a different technical responsibility. Notifications, integrations, background jobs and reporting can start as infrastructure/application capabilities until their domain complexity justifies a dedicated module.

## 7. Architecture tests

Tests should verify at least:

- Domain does not reference API or Infrastructure assemblies.
- Application does not reference Infrastructure or ASP.NET Core API assemblies.
- Module contracts are the only supported cross-module write path.
- Tenant-owned writes require tenant context.
- Read/report queries are tenant/branch scoped.
- Inventory and financial ledgers do not expose ordinary destructive delete paths.
