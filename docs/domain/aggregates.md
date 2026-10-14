# Aggregate Boundaries

**Status:** Consolidated domain baseline • Version 1.0

## 1. Purpose

An aggregate is a cluster of domain objects that must remain consistent together. The aggregate root is the public entry point for changing invariant-protected state. These boundaries guide domain methods, repositories, EF Core loading, transaction scope and tests; they are not a one-to-one copy of database tables.

## 2. Aggregate map

| Aggregate root | Owned children / contained data | Invariants that must be enforced |
|---|---|---|
| `Tenant` | Tenant-owned configuration references and lifecycle state | Code/identity stable; valid lifecycle; no cross-tenant ownership |
| `Branch` | Branch profile/address values | Branch belongs to one tenant; inactive branch cannot accept normal new operations |
| `User` | User profile/auth link; role assignments may be explicit link entities | User/tenant membership valid; last owner cannot be removed without replacement |
| `Role` | Role metadata and role-permission links | Tenant-safe role assignment; only known permission keys may be granted |
| `Category` | Category metadata; parent reference | Same-tenant parent; no hierarchy cycles |
| `Product` | Catalog metadata, translations, images and controlled custom attributes | SKU unique within tenant; archive rather than delete when referenced |
| `ProductVariant` | Variant-specific gold/purity/stone references and attributes | Variant identity describes commercial configuration, not each physical piece |
| `InventoryItem` | Physical item identity/state and stock-lifecycle facts | Barcode uniqueness; valid weights; one effective location/status; sold item cannot be sold twice |
| `InventoryTransaction` | One immutable stock ledger entry; normally append-only record | Never edited/deleted after posting; source operation and movement commit atomically |
| `StockTransfer` | **`StockTransferItem`** lines and transfer lifecycle | Source != destination; both branches in same tenant; shipment/receipt idempotent; movements occur exactly once |
| `GoldPrice` | One effective-dated price record | Explicit market/tenant scope; effective period and source are traceable; no ambiguous overlapping active price |
| `PricingRule` | Versioned rule configuration | Configuration validates before activation; precedence deterministic; evaluation auditable |
| `Sale` | `SaleItem` lines and immutable sale/price/tax snapshots; references to payment effects | Totals reconcile; item available; posting atomic; completed sale corrected only via controlled reversal |
| `Payment` | Payment state and external settlement reference | Positive valid amount, tenant/source ownership, duplicate prevention, no silent edits after completion |
| `Return` | **`ReturnItem`** lines | Must reference original sale/line; quantity/value cannot exceed returnable balance; completion idempotent |
| `PurchaseOrder` | `PurchaseOrderItem` lines | Valid supplier/variant ownership; valid order lifecycle; order creation alone does not increase stock |
| `Purchase` | `PurchaseItem` receipt lines and acquisition-cost snapshots | Posted receipt creates inventory exactly once; posted purchase is immutable except reversal |
| `Repair` | **`RepairItem`** lines and `RepairStatusHistory` records | Controlled lifecycle; item/customer identity preserved; delivery requirements met |
| `GoldBuyback` | Valuation snapshot and transaction details | Weight/purity/price required; payment and stock effects are atomic and traceable |
| `Exchange` | **`ExchangeItem`** lines, old/new valuations and difference calculation | Both sides traceable; eligible items only; no duplicate stock or payment effect |
| `Account` | Account metadata and balance projection if used | Tenant scope and account type valid; balance remains consistent with postings |
| `AccountTransaction` | One immutable accounting posting/entry | Posted entries immutable; source traceable; debit/credit model reconciles if double-entry is enabled |
| `Subscription` | Plan reference and lifecycle/period | Preserve history; prevent invalid or conflicting active periods |
| `TenantFeature` | One feature entitlement override | Effective state and limits are deterministic and checked server-side |
| `AuditLog` | One append-only event | Normal users cannot edit/delete audit history; sensitive values redacted |

`Market`, `Feature`, `Permission`, `Purity`, `GoldType`, `Stone`, `Brand` and similar reference catalogs can be independently managed according to their lifecycle rules. Their stable identifiers must remain valid for historical records.

## 3. Aggregate design rules

1. **One root for one invariant boundary.** External code must call root methods or an owning module's use case rather than editing child state directly.
2. **Keep aggregates small.** Do not load every referenced entity into one giant graph merely to complete an operation.
3. **Reference other aggregates by ID.** `Sale` can reference `CustomerId` and `BranchId`; it does not own the Customer or Branch.
4. **Own only true lifecycle children.** `SaleItem` belongs to Sale; `StockTransferItem` belongs to StockTransfer; they should not be managed independently.
5. **Use a single local database transaction when required by business invariants.** Cross-module orchestration may coordinate several aggregate methods in one transaction, but each module still owns its invariants.
6. **Use concurrency control where two operations can race.** In particular physical item sale/reservation, transfer receipt, payments and usage-limit consumption.
7. **Posted records are immutable.** Corrections use reversal, refund or adjustment workflows.
8. **Domain events describe facts.** Durable cross-process effects use an outbox/job pattern when delivery must survive process failure.

## 4. Key transaction boundaries

### Complete sale

Validate branch, tenant, item availability, current pricing rules, discounts/approval and tax. Persist Sale + SaleItems + price/tax snapshots and required payment effects, append the inventory movement, audit the operation, and record durable integration intent in one transaction when these effects share the same PostgreSQL database. Failure means none of the posting effects commit.

### Receive purchase

Validate supplier/order and received quantities. Create Purchase/PurchaseItems, assign acquisition cost to physical items, create inventory records and movement entries, and make the purchase post exactly once.

### Receive stock transfer

Validate transfer state, source/destination tenant and item status. Receiving is idempotent; a repeated request must return the prior outcome rather than create duplicate movements.

### Complete return

Validate the original sale and remaining returnable line/value, record Return/ReturnItems, issue inventory movements if the items are accepted back, and create traceable refund/payment effects atomically.

### Complete gold buyback/exchange

Preserve purity, weight, gold price, valuation formula/version, currency and approval snapshots. Inventory and financial effects must happen once and have a traceable originating operation.

## 5. What is not an aggregate boundary

- A database table is not automatically an aggregate.
- A module is not automatically one aggregate; Inventory contains multiple aggregate types.
- A report/read model should not own business writes.
- A cross-reference such as `CustomerId` on Sale does not mean Sale owns Customer.
- A value object (Money, Weight, DateRange) has value semantics and no independent identity.

## 6. Persistence guidance

- Repositories, if used, should normally be defined around aggregate roots rather than every table.
- EF Core entity configurations remain in Infrastructure. Domain entities must not depend on lazy loading or EF-specific collection behaviors.
- Use restrictive delete behavior for historical/business entities.
- Use unique/check/foreign-key constraints as a second line of defense, especially tenant consistency and duplicate posting.
- Test invariants at domain level and concurrency/constraint behavior against PostgreSQL integration tests.

## 7. Open implementation choices

Before coding each module, confirm whether Payment is an independent aggregate root, whether the accounting model is simple posting or full double-entry, and whether the tenant identity model requires separate user membership tables. These choices must be recorded in an ADR if they change the baseline schema or cross-module behavior.
