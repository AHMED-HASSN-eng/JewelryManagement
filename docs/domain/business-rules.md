<!-- Converted from Jewelry_Management_System_Business_Rules_Specification.docx. Keep rule IDs stable unless reviewed. -->

**Jewelry Management System**

**Business Rules & Domain Behavior Specification**

Version 1.0 — Team Baseline

**Multi-tenant • Multi-branch • Multi-market • Scalable SaaS**

![High-level Jewelry Management System ERD](../assets/jewelry-management-system-erd.png)

Reference ERD — high-level domain model

# 1. Purpose and Scope

This document defines the business rules that govern the Jewelry Management System. It is the bridge between the high-level ERD and implementation: domain entities, application use cases, database constraints, APIs, desktop screens, mobile clients, and tests should all conform to these rules.

The platform is designed as a multi-tenant SaaS product. A tenant represents one jewelry business/company, can operate one or more branches, and can use market-specific configuration for Egypt, Saudi Arabia, or future markets without creating separate codebases.

# 2. Global Business Rules

| **ID**  | **Rule**                      | **Decision / Requirement**                                                                                                               | **Rationale / Notes**                                                              |
|---------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| GBR-001 | Tenant isolation              | Every tenant-owned record is scoped to exactly one TenantId. A user must never read, modify, or delete another tenant's data.            | Cross-tenant access is a security defect, not a normal authorization failure.      |
| GBR-002 | Branch ownership              | A branch belongs to exactly one tenant. Business transactions that occur physically in a branch reference that branch.                   | Some platform records may be tenant-level and branch-less.                         |
| GBR-003 | Historical integrity          | Posted sales, purchases, payments, inventory transactions, accounting transactions, and audit records are not hard-deleted.              | Use reversal/void/correction workflows instead.                                    |
| GBR-004 | Auditability                  | Material business changes must record actor, timestamp, tenant, branch where relevant, and before/after values when appropriate.         | Especially pricing, discounts, stock adjustments, refunds, and permission changes. |
| GBR-005 | Immutable posted transactions | Once a financial or inventory transaction is posted, its core economic facts are immutable.                                              | Corrections are represented by explicit reversal or adjustment transactions.       |
| GBR-006 | Money precision               | Money is stored using a fixed-precision decimal type with currency context; floating-point arithmetic is prohibited for monetary values. | Exact precision will be finalized in database design.                              |
| GBR-007 | Weight precision              | Jewelry and gold weights use fixed-precision decimal values appropriate to the business scale.                                           | Do not use binary floating-point for weights.                                      |
| GBR-008 | Time handling                 | System timestamps are stored consistently and business-day calculations use the tenant/branch timezone.                                  | Display can be localized while preserving an unambiguous timestamp.                |
| GBR-009 | Config before code            | Market and tenant behavior that is expected to vary is represented as configuration or strategy, not duplicated country-specific code.   | Avoid country-specific if/else branches spread across modules.                     |
| GBR-010 | Idempotency                   | Commands that can be retried safely should support an idempotency key or equivalent duplicate-prevention mechanism.                      | Important for payments, sale posting, sync, imports, and external integrations.    |

# 3. Tenant, Market, Subscription and Feature Rules

| **ID**  | **Rule**             | **Decision / Requirement**                                                                                                                     | **Rationale / Notes**                                                                                                              |
|---------|----------------------|------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| TMS-001 | Tenant lifecycle     | Tenant statuses are Pending, Active, Suspended, and Closed.                                                                                    | Closed tenants remain available for historical/reporting/retention operations unless removed by a separate data-retention process. |
| TMS-002 | Default market       | Each tenant has one default Market.                                                                                                            | Market supplies defaults; tenant-level overrides may exist.                                                                        |
| TMS-003 | Market configuration | Currency, terminology, tax defaults, invoice defaults, payment methods, and market rules come from Market configuration plus tenant overrides. | Core business entities remain market-neutral.                                                                                      |
| TMS-004 | Subscription history | Subscription changes create historical subscription records; history is never overwritten as the only record.                                  | Supports upgrades, downgrades, renewals, and audits.                                                                               |
| TMS-005 | Feature gating       | Premium functionality is enabled through Feature / TenantFeature / Plan configuration.                                                         | Business services should check feature access centrally.                                                                           |
| TMS-006 | Usage limits         | Plan limits are enforced at the application boundary before the operation that would exceed a quota.                                           | Examples: branches, users, storage, API calls.                                                                                     |
| TMS-007 | Suspended tenant     | A suspended tenant cannot create normal business transactions. Read access is limited to approved recovery/support operations.                 | Exact grace-period behavior is a product decision.                                                                                 |
| TMS-008 | Owner continuity     | A tenant must always have at least one active owner/admin capable of managing the tenant, unless the tenant is in a terminal state.            | Do not allow the last owner to remove themselves without replacement.                                                              |

# 4. Identity, Users, Roles and Permissions

| **ID**  | **Rule**                | **Decision / Requirement**                                                                                                                     | **Rationale / Notes**                                                            |
|---------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| IAM-001 | User membership         | A user must belong to a tenant through an explicit tenant membership model or equivalent.                                                      | Keep authentication identity separate from tenant business membership.           |
| IAM-002 | Least privilege         | Permissions are granted through roles and, where required, scoped assignments.                                                                 | Do not rely on UI hiding for authorization.                                      |
| IAM-003 | Branch scope            | A role may be restricted to one or more branches.                                                                                              | A branch-scoped user cannot operate on another branch unless explicitly granted. |
| IAM-004 | Sensitive actions       | Changing gold prices, large discounts, stock adjustments, refunds, invoice cancellation, and permission changes require dedicated permissions. | Optional approval workflows can add a second authorization step.                 |
| IAM-005 | Self-protection         | A user cannot disable/delete the last active tenant owner.                                                                                     | Prevents lockout.                                                                |
| IAM-006 | Authentication security | Passwords are never stored in plaintext; access tokens and refresh/session mechanisms follow the chosen identity architecture.                 | Security implementation is a separate technical standard.                        |

# 5. Customer and Supplier Rules

| **ID** | **Rule**                | **Decision / Requirement**                                                                        | **Rationale / Notes**                                                              |
|--------|-------------------------|---------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| CS-001 | Tenant ownership        | Customer and Supplier records belong to exactly one tenant.                                       | They may later be linked to branch activity without duplicating the master record. |
| CS-002 | Customer identity       | The system must support individual and company customers.                                         | Required fields depend on market and workflow.                                     |
| CS-003 | Duplicate control       | Customer creation should detect likely duplicates using configurable identity/contact matching.   | Do not silently merge records.                                                     |
| CS-004 | Historical preservation | Customers with historical transactions cannot be physically deleted through normal UI operations. | Archive/deactivate instead.                                                        |
| CS-005 | Supplier continuity     | A supplier linked to purchase history remains available for history even if no longer active.     | Use an active/inactive state.                                                      |

# 6. Catalog, Product and Jewelry Item Rules

| **ID**  | **Rule**                    | **Decision / Requirement**                                                                                                             | **Rationale / Notes**                                             |
|---------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| CAT-001 | Product definition          | Product describes the commercial definition of an item/category of jewelry.                                                            | It is not itself the physical inventory piece.                    |
| CAT-002 | Variant definition          | ProductVariant captures commercially meaningful differences such as gold type, purity, stone, or other attributes.                     | Variant structure should avoid excessive duplicated product rows. |
| CAT-003 | Physical inventory identity | An InventoryItem represents one physical piece when the item requires serialized/weight-level tracking.                                | This is central to jewelry inventory accuracy.                    |
| CAT-004 | Barcode uniqueness          | A barcode assigned to a physical inventory item must be unique within the required scope, normally the tenant.                         | Scope can be broadened later for global identifiers.              |
| CAT-005 | Weight validity             | Gross weight, gold weight, and stone weight cannot be negative.                                                                        | Domain validation is required before persistence.                 |
| CAT-006 | Weight consistency          | Where the business workflow uses the components, gross weight must be logically consistent with stone/non-gold weight and gold weight. | Exact formula must match the shop's operational convention.       |
| CAT-007 | Pricing snapshot            | A sale line must preserve the actual pricing inputs/results used at sale time.                                                         | Later gold-price changes must not rewrite old sales.              |
| CAT-008 | Certificate/reference data  | Certificate numbers and external references are treated as identifiers/attributes, not as the source of financial truth.               | Validation rules depend on certificate type.                      |
| CAT-009 | Product archival            | Products and variants with transaction history are archived/inactivated, not deleted.                                                  | History must remain reportable.                                   |
| CAT-010 | Custom fields               | Tenant-specific custom attributes are allowed for extensibility, but core operational data must remain strongly typed.                 | Do not hide critical business facts only inside JSON.             |

# 7. Gold, Purity, Stone and Gold Price Rules

| **ID**   | **Rule**                | **Decision / Requirement**                                                                                                                 | **Rationale / Notes**                                                                                   |
|----------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| GOLD-001 | Purity reference        | Purity is a reference value with a defined karat/fineness representation.                                                                  | Examples can include 18K, 21K, 22K, 24K, but the actual allowed list is market/configuration data.      |
| GOLD-002 | Gold type               | GoldType identifies the material/category used for gold pricing and reporting.                                                             | Do not encode market terminology directly into code.                                                    |
| GOLD-003 | Effective price periods | Gold prices are effective-dated and should not overlap for the same market/gold type/purity dimension when overlap would create ambiguity. | Use deterministic selection by effective timestamp.                                                     |
| GOLD-004 | Price history           | Historical gold prices are immutable after use in a posted transaction.                                                                    | Correct an incorrect price with a new effective record or explicit correction workflow.                 |
| GOLD-005 | Price source            | Each price should be traceable to its source: manual, imported, API, or another configured source.                                         | Useful for audit and dispute resolution.                                                                |
| GOLD-006 | Market context          | Gold price rules may differ by market and tenant.                                                                                          | Resolve in a deterministic order: explicit tenant rule → market default → system default where allowed. |

# 8. Pricing Engine Rules

| **ID**    | **Rule**                     | **Decision / Requirement**                                                                                                                   | **Rationale / Notes**                                          |
|-----------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| PRICE-001 | Single calculation authority | Final sale price must be produced by a single pricing service/rule engine, not recomputed independently in UI, API controllers, and reports. | Clients display the server result.                             |
| PRICE-002 | Inputs                       | Pricing may use weight, purity, gold price, making charge, stone value, discounts, taxes, and other configured charges.                      | Not every product uses every component.                        |
| PRICE-003 | Rule precedence              | When multiple pricing rules match, the system uses an explicit deterministic precedence/order.                                               | No implicit database ordering.                                 |
| PRICE-004 | Rounding                     | Rounding is defined by market/tenant/currency rules and is applied at documented stages.                                                     | Avoid repeated rounding throughout calculations.               |
| PRICE-005 | Discount authorization       | Discounts are subject to configured maximums and permission/approval rules.                                                                  | Exceeding threshold may require manager approval.              |
| PRICE-006 | Price snapshot               | The sale stores the pricing result and relevant rule/version identifiers used at posting time.                                               | Allows historical reconstruction.                              |
| PRICE-007 | Manual override              | A manual price override requires an authorized role and reason when configured.                                                              | Override is auditable.                                         |
| PRICE-008 | Tax isolation                | Tax calculation is a distinct step and market-aware.                                                                                         | Do not hard-code a single tax percentage into the sale module. |

# 9. Inventory Rules

| **ID**  | **Rule**                 | **Decision / Requirement**                                                                                                                                                  | **Rationale / Notes**                                                                    |
|---------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| INV-001 | Ledger-based stock       | Inventory state is derived from controlled inventory transactions or an equivalent ledger-backed model.                                                                     | Avoid relying on a single mutable quantity without traceability.                         |
| INV-002 | Transaction types        | Inventory movements must identify the business cause: purchase, sale, return, transfer, adjustment, repair, exchange, buyback, opening balance, or future configured types. | Type drives validation and reporting.                                                    |
| INV-003 | Physical ownership       | An inventory item is owned by one tenant and is located at one branch/location at a time unless the design explicitly represents in-transit state.                          | In-transit is preferable for stock transfers.                                            |
| INV-004 | Negative stock           | Negative stock is disallowed by default.                                                                                                                                    | A tenant may explicitly enable negative-stock behavior if the business case is approved. |
| INV-005 | Sold item protection     | A physical serialized/unique item cannot be sold twice unless a return/reversal makes it available again.                                                                   | Enforce through transactional checks and database constraints where possible.            |
| INV-006 | Adjustment authorization | Manual stock adjustments require an authorized permission and a reason.                                                                                                     | High-risk operation.                                                                     |
| INV-007 | Stock count              | Physical stock count differences create adjustment transactions rather than silently overwriting balances.                                                                  | Count event should be auditable.                                                         |
| INV-008 | Branch transfer          | A transfer moves ownership/location through explicit transfer state and receiving confirmation.                                                                             | Do not simply change BranchId with no history.                                           |

# 10. Purchasing Rules

| **ID**  | **Rule**            | **Decision / Requirement**                                                                                          | **Rationale / Notes**                                      |
|---------|---------------------|---------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| PUR-001 | Purchase lifecycle  | Purchase orders, receiving, and purchase invoices are distinct concepts when the business needs them.               | Not every shop must use purchase orders.                   |
| PUR-002 | Receiving           | Inventory is increased only when goods are actually received/posted according to the configured receiving workflow. | Do not increase stock merely because an order was created. |
| PUR-003 | Supplier reference  | Supplier and supplier document/reference must be captured where required.                                           | Supports reconciliation.                                   |
| PUR-004 | Cost capture        | Acquisition cost is preserved at receipt/purchase line level.                                                       | Later price changes do not rewrite historical cost.        |
| PUR-005 | Purchase correction | Posted purchases are corrected through returns, debit/credit adjustments, or other explicit workflows.              | Never silently mutate posted historical values.            |

# 11. Sales and POS Rules

| **ID**   | **Rule**                 | **Decision / Requirement**                                                                                               | **Rationale / Notes**                                  |
|----------|--------------------------|--------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| SALE-001 | Draft vs posted          | A sale can be in draft/editing state before posting; once posted, its financial and inventory effects are committed.     | Exact status set is finalized in state-machine design. |
| SALE-002 | Transaction atomicity    | Posting a sale must atomically create/commit the sale, sale lines, payment records as applicable, and inventory effects. | All-or-nothing transaction boundary.                   |
| SALE-003 | Availability check       | Before posting, the system verifies that required inventory items are available and not already sold/reserved.           | Concurrency control required.                          |
| SALE-004 | Payment validation       | Paid amount and payment methods must satisfy the configured payment policy.                                              | Support single or mixed payments as configured.        |
| SALE-005 | Invoice numbering        | Invoice numbers are generated from a tenant/branch/market-aware numbering policy and are not reused after posting.       | Exact sequence format is configuration.                |
| SALE-006 | Discount                 | Discounts cannot exceed configured boundaries unless approved.                                                           | Every discount remains part of the audit trail.        |
| SALE-007 | Return authorization     | Returns reference the original sale and returned item/quantity.                                                          | Avoid unrestricted refund creation.                    |
| SALE-008 | Return limits            | A return cannot exceed the quantity/value available for return under the original sale and prior returns.                | State must be tracked.                                 |
| SALE-009 | Cancellation             | Posted sales are not physically deleted. Cancellation creates an explicit reversal/cancellation event.                   | Inventory/payment reversal must be auditable.          |
| SALE-010 | Receipt/invoice snapshot | The printed/displayed invoice is based on the transaction snapshot at posting time.                                      | Template changes should not rewrite historical facts.  |

# 12. Payment Rules

| **ID**  | **Rule**            | **Decision / Requirement**                                                                            | **Rationale / Notes**                                                 |
|---------|---------------------|-------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| PAY-001 | Payment ownership   | Every payment belongs to a tenant and is linked to the business transaction it settles.               | Do not allow orphan financial records except controlled prepayments.  |
| PAY-002 | Supported methods   | Payment methods are configurable per market/tenant rather than hard-coded.                            | Examples: cash, card, bank transfer, wallet.                          |
| PAY-003 | Mixed payments      | A transaction may use multiple payment records if mixed payment is enabled.                           | Sum of valid payment allocations must satisfy the transaction policy. |
| PAY-004 | Refund traceability | Refunds reference the original payment/sale where possible.                                           | Protects reconciliation.                                              |
| PAY-005 | No silent edits     | Posted payment records are not silently edited; corrections use reversal/refund/adjustment workflows. | Audit sensitive.                                                      |

# 13. Gold Buyback Rules

| **ID**  | **Rule**           | **Decision / Requirement**                                                                                           | **Rationale / Notes**                                         |
|---------|--------------------|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| BUY-001 | Customer ownership | A gold buyback is linked to the customer when identity/business policy requires it.                                  | Anonymous buyback can be disabled by market or tenant policy. |
| BUY-002 | Measurement        | Weight and purity are mandatory inputs to valuation.                                                                 | Additional test/verification fields may be configured.        |
| BUY-003 | Valuation snapshot | The buyback records the gold price/rule/version used for valuation.                                                  | Historical price changes do not change an old buyback.        |
| BUY-004 | Payment            | Buyback payment follows configured payment methods, limits, and approval rules.                                      | May require manager approval depending on amount.             |
| BUY-005 | Inventory effect   | If the purchased material enters shop inventory, an explicit inventory transaction is created with traceable source. | Do not hide the stock effect.                                 |

# 14. Exchange / Trade-in Rules

| **ID**  | **Rule**               | **Decision / Requirement**                                                                                                                          | **Rationale / Notes**                                  |
|---------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| EXC-001 | Two-sided transaction  | An exchange combines a customer trade-in/old item side with a new sale side.                                                                        | Both sides remain individually traceable.              |
| EXC-002 | Trade-in valuation     | Old jewelry value uses an explicit valuation rule and snapshot.                                                                                     | Do not reuse the new item selling price logic blindly. |
| EXC-003 | Difference calculation | Customer payable/refundable difference equals new-side obligations minus accepted old-side value, subject to configured charges/tax/discount rules. | Sign and refund behavior must be deterministic.        |
| EXC-004 | Inventory effects      | Old item/material and new item inventory movements are posted as explicit transactions.                                                             | Atomic operation.                                      |
| EXC-005 | Approval               | Large-value exchanges or manual overrides may require approval.                                                                                     | Threshold configurable.                                |

# 15. Repair and Service Rules

| **ID**  | **Rule**           | **Decision / Requirement**                                                                                                   | **Rationale / Notes**                                                                             |
|---------|--------------------|------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| REP-001 | Repair intake      | A repair order records the customer, received item/reference, problem description, and intake timestamp.                     | Photos/certificates can be attached later.                                                        |
| REP-002 | Status lifecycle   | Repair status follows a controlled state machine; invalid backwards/sideways transitions are blocked.                        | Example: Received → Inspection → Approved → In Repair → Completed → Ready for Pickup → Delivered. |
| REP-003 | Cost approval      | Estimated cost changes above configured thresholds may require customer approval.                                            | Approval must be recorded.                                                                        |
| REP-004 | Inventory handling | If the repaired item is tracked as physical inventory, its movement/status is recorded without losing the original identity. | Do not create duplicate physical items unnecessarily.                                             |
| REP-005 | Delivery           | A repair cannot be marked delivered without a completion state and required payment/authorization conditions.                | Configurable by tenant.                                                                           |

# 16. Accounting and Finance Rules

| **ID**  | **Rule**              | **Decision / Requirement**                                                                                                          | **Rationale / Notes**                                             |
|---------|-----------------------|-------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| ACC-001 | Posting boundary      | Operational transactions can generate accounting postings according to configured accounting rules.                                 | Sales, purchases, payments, refunds, buybacks, etc.               |
| ACC-002 | Balanced transactions | Every financial journal/AccountTransaction must satisfy the platform's balance requirement when double-entry accounting is enabled. | Exact debit/credit schema will be finalized in accounting design. |
| ACC-003 | Immutable postings    | Posted accounting transactions are immutable; corrections are reversals/adjustments.                                                | Preserves financial history.                                      |
| ACC-004 | Account ownership     | Financial accounts are tenant-scoped and cannot cross tenants.                                                                      | Branch dimension can be included where needed.                    |
| ACC-005 | Reconciliation        | Cash/card/bank/wallet movements should support reconciliation against source transactions.                                          | Detailed reconciliation workflow is a future design item.         |

# 17. Audit and Security Rules

| **ID**  | **Rule**            | **Decision / Requirement**                                                                    | **Rationale / Notes**                                                |
|---------|---------------------|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| AUD-001 | Audited operations  | Create/update/delete/reverse/approve operations on sensitive entities are audited.            | Exact list should be configurable but critical events are mandatory. |
| AUD-002 | Before/after values | For sensitive changes, audit stores old and new values or a structured diff.                  | Do not log secrets/passwords/tokens.                                 |
| AUD-003 | Actor context       | Audit identifies the user/service that caused the action and the tenant.                      | Branch/device/IP metadata may be captured where lawful and useful.   |
| AUD-004 | Audit retention     | Audit logs follow the product retention policy and are protected from ordinary user deletion. | Administrative purge, if any, is separate and audited.               |

# 18. Multi-Market / Localization Rules

| **ID**  | **Rule**                | **Decision / Requirement**                                                                                  | **Rationale / Notes**                                                                         |
|---------|-------------------------|-------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| MKT-001 | Market-neutral core     | Core entities and use cases use neutral domain concepts rather than Egypt/Saudi-specific table names.       | Market-specific differences are policies/configuration.                                       |
| MKT-002 | Terminology             | UI labels and business terminology can be localized by language and market.                                 | Do not rename database entities per market.                                                   |
| MKT-003 | Currency                | A tenant has a default currency; transactions may use configured transaction currency support when enabled. | Exchange-rate rules are a separate concern.                                                   |
| MKT-004 | Tax                     | Tax rules are market-aware and versioned/effective-dated where required.                                    | Historical transactions keep their applied tax snapshot.                                      |
| MKT-005 | Invoice rules           | Numbering, required fields, invoice formatting, and legal fields are configurable by market/tenant.         | Implementation must keep regulatory rules outside core sales calculation code where possible. |
| MKT-006 | Measurement conventions | Weight/unit conventions are explicit per market/tenant configuration.                                       | Display formatting may differ without changing stored facts.                                  |

# 19. Reporting Rules

| **ID**  | **Rule**            | **Decision / Requirement**                                                                                           | **Rationale / Notes**                                                                      |
|---------|---------------------|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| RPT-001 | Posted data only    | Financial and operational reports use posted/valid business transactions unless a report explicitly includes drafts. | Prevents draft contamination.                                                              |
| RPT-002 | Historical accuracy | Reports reconstruct historical value from transaction snapshots, not current product/gold-price settings.            | Critical for sales and profit analysis.                                                    |
| RPT-003 | Branch filtering    | Users see only branches they are authorized to access.                                                               | Reporting is subject to authorization.                                                     |
| RPT-004 | Tenant filtering    | All reports are tenant-scoped.                                                                                       | No cross-tenant aggregation except privileged platform analytics with explicit safeguards. |

# 20. Concurrency and Transaction Rules

| **ID**  | **Rule**                   | **Decision / Requirement**                                                                          | **Rationale / Notes**                          |
|---------|----------------------------|-----------------------------------------------------------------------------------------------------|------------------------------------------------|
| CON-001 | Atomic posting             | Sale, transfer receipt, purchase receipt, and similar operations complete atomically or not at all. | Use database transactions.                     |
| CON-002 | Concurrent sale protection | Two terminals cannot successfully sell the same serialized inventory item.                          | Requires concurrency control/locking strategy. |
| CON-003 | Concurrent pricing         | A sale selects a deterministic gold-price/pricing snapshot at a defined point in time.              | Do not allow price changes mid-posting.        |
| CON-004 | Retry safety               | Retrying the same command must not create duplicate sale/payment/inventory effects.                 | Use idempotency or command keys.               |

# 21. Offline and Synchronization Rules (Future Phase)

| **ID**   | **Rule**             | **Decision / Requirement**                                                                        | **Rationale / Notes**                                                    |
|----------|----------------------|---------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| SYNC-001 | Local continuity     | Offline mode, when introduced, must preserve the same business invariants locally as online mode. | Local database is not a weaker rules engine.                             |
| SYNC-002 | Conflict policy      | Every synchronized operation must have a deterministic conflict policy.                           | Inventory/sales conflicts cannot be resolved by last-write-wins blindly. |
| SYNC-003 | Unique IDs           | Offline-created records require globally unique identifiers.                                      | Supports safe synchronization.                                           |
| SYNC-004 | Event/order metadata | Syncable commands/events need causation/idempotency metadata.                                     | Detailed protocol is a later design.                                     |

# 22. Data Lifecycle and Deletion Rules

| **ID** | **Rule**          | **Decision / Requirement**                                                                                             | **Rationale / Notes**                                   |
|--------|-------------------|------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| DL-001 | Reference data    | Inactive/archive is preferred to physical deletion for records referenced by transactions.                             | Examples: Product, Customer, Supplier, User, Branch.    |
| DL-002 | Financial records | Sales, payments, purchases, accounting transactions are never hard-deleted through normal application paths.           | Use reversal/void/correction.                           |
| DL-003 | Tenant closure    | Closing a tenant disables normal operations while preserving retained historical data.                                 | Retention/legal policy applies separately.              |
| DL-004 | Attachments       | Files such as images/documents follow a retention policy and must remain linked to their business record while needed. | Storage deletion must not silently break audit/history. |

# 23. Core Application Invariants

1.  No command may operate on a tenant-owned record outside the caller's authorized tenant context.

2.  No posted sale may exist without valid sale lines and a valid inventory/payment outcome according to the configured workflow.

3.  No serialized jewelry item may have two simultaneously valid ownership/sale states.

4.  No inventory quantity change may occur without a traceable inventory transaction.

5.  No historical transaction may be recalculated from today's gold price or today's product configuration.

6.  No privileged financial or inventory correction may occur without authorization and audit evidence.

7.  No market-specific business behavior may bypass the central market/tenant policy mechanism.

8.  No subscription/feature rule may be enforced only in the frontend; the backend is authoritative.

# 24. Initial State Machines

The following are the initial recommended states. Exact names can be adjusted during use-case design, but transitions must remain explicit and testable.

**Tenant:** Pending → Active → Suspended → Active; Active → Closed

**Sale:** Draft → Confirmed → Posted; Draft → Cancelled; Posted → Reversed/Returned according to workflow

**Purchase:** Draft → Ordered → Received → Posted; Ordered → Cancelled

**Stock Transfer:** Draft → Approved → Shipped → Received → Completed; Draft/Approved/Shipped → Cancelled where valid

**Repair:** Received → Inspection → Approved → In Repair → Completed → Ready for Pickup → Delivered; Waiting Approval may branch from Inspection

**Subscription:** Trial/Active → PastDue → Active/Suspended/Cancelled according to billing policy

# 25. Business Rule Acceptance Checklist

- **Tenant isolation:** No automated or manual integration test can access another tenant's business data through supported APIs.

- **Inventory integrity:** Every stock-affecting workflow creates the expected inventory transaction and cannot create impossible stock states.

- **Pricing reproducibility:** A historical sale can be reconstructed from its stored pricing snapshot without depending on current gold price.

- **Authorization:** Sensitive actions fail server-side without permission even when the client UI is bypassed.

- **Auditability:** A reviewer can determine who changed a sensitive business fact, when, and what changed.

- **Localization:** Switching market configuration changes required terminology/rules without requiring a new codebase.

- **Concurrency:** Two simultaneous sale attempts cannot successfully sell the same unique physical jewelry item.

- **Correction:** Historical posted transactions are corrected through explicit reversal/adjustment workflows, not destructive updates.

# 26. Decisions Still Requiring Product Confirmation

- Exact accounting model and whether full double-entry accounting is part of MVP or a later module.

- Exact tax/legal invoice requirements for each initial market and how frequently regulatory rules change.

- Whether anonymous customer transactions are permitted and under what thresholds.

- Whether negative inventory is ever allowed and who can enable it.

- Exact jewelry weight formula/convention (gross, stone, non-gold, net gold) for each supported workflow.

- Whether a physical item can be split/merged/reworked and how that affects identity and inventory history.

- Whether installment sales and customer credit are in MVP and the exact accounting treatment.

- Offline POS scope and synchronization/conflict policy.

- Gold-price provider strategy and fallback/manual override policy.

# 27. Implementation Guidance for the .NET Team

- **Domain layer:** own invariants, value objects, statuses, and domain behavior that must always be true.

- **Application layer:** orchestrate use cases, authorization, idempotency, transactions, and policy resolution.

- **Infrastructure layer:** EF Core mappings, PostgreSQL, external providers, file storage, background jobs, and persistence concerns.

- **API layer:** authentication/authorization, request validation, tenant context, versioning, error contracts, and transport DTOs.

- **Frontend/Desktop:** display and interaction only; never become the source of truth for pricing, permissions, inventory, or financial calculations.

- **Tests:** encode every rule ID as one or more automated tests, especially tenant isolation, inventory integrity, pricing snapshots, and transaction atomicity.

# 28. Revision History

| **Version** | **Date**   | **Change**                                                    |
|-------------|------------|---------------------------------------------------------------|
| 1.0         | 2026-09-12 | Initial team baseline for business rules and domain behavior. |
