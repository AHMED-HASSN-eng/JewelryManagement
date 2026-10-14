<!-- Converted from Jewelry_Management_System_Enums_and_Value_Objects_Specification.docx. -->

**JEWELRY MANAGEMENT SYSTEM**

Enums & Value Objects Specification

Domain foundation for ASP.NET Core 10 / EF Core 10 / PostgreSQL

| **Document** | **Value**                                                  |
|--------------|------------------------------------------------------------|
| Document ID  | DOMAIN-FOUND-ENUM-VO-001                                   |
| Version      | 1.0                                                        |
| Status       | Team Baseline                                              |
| Audience     | Backend, Frontend, QA, Architecture, DevOps                |
| Scope        | Core enums, value objects, persistence and API conventions |

**Purpose:** Define the finite states and immutable domain concepts shared by the tenant, catalog, inventory, pricing, sales, purchasing, repairs, accounting and platform modules. These definitions are intended to prevent primitive obsession, inconsistent validation and cross-module terminology drift.

# 1. Design Principles

> • Enums represent a closed, stable set of states or categories controlled by the platform code.
>
> • Value objects represent a concept defined by its value rather than by an identity; they should be immutable and validated at construction.
>
> • Tenant-configurable concepts such as payment methods, gold types, product types, pricing rules and feature definitions remain entities/configuration records, not enums.
>
> • Domain values must not depend on UI labels. API labels and localized text are presentation concerns.
>
> • Core domain calculations must use explicit decimal precision and must never depend on binary floating-point values.
>
> • Enums and value objects are shared domain language. Changes require architecture review because they can affect API contracts, database values, integrations and reports.

# 2. Enum Strategy

| **Rule**          | **Decision**                                                                                                                                               |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Naming            | PascalCase singular enum type; PascalCase members.                                                                                                         |
| Persistence       | Prefer string values in PostgreSQL through EF Core value conversion for business-facing enums; this keeps data readable and safer across inserted members. |
| API               | Serialize enum values as stable strings, not numeric ordinals.                                                                                             |
| Unknown values    | API clients should tolerate unknown enum values; server validation remains authoritative.                                                                  |
| Adding a member   | Backward compatible in most cases, but any state machine must define transitions explicitly.                                                               |
| Renaming a member | Breaking change; requires data migration and API/version review.                                                                                           |
| Removing a member | Do not remove while historical rows reference it; deprecate first.                                                                                         |

# 3. Core Enum Catalog

| **Enum**                 | **Members**                                                                                                                                                                        | **Purpose**                                                                                       |
|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| TenantStatus             | Pending, Active, Suspended, Closed                                                                                                                                                 | Tenant lifecycle. Closed is terminal for normal operations.                                       |
| UserStatus               | Invited, Active, Suspended, Locked, Deactivated                                                                                                                                    | User access lifecycle; authentication state is separate from authorization.                       |
| SubscriptionStatus       | Trialing, Active, PastDue, GracePeriod, Suspended, Cancelled, Expired                                                                                                              | Subscription lifecycle used by entitlement checks and billing workflows.                          |
| FeatureAvailability      | Available, Deprecated, Retired                                                                                                                                                     | Platform-level lifecycle for a feature definition; tenant enablement is separate.                 |
| ProductStatus            | Draft, Active, Inactive, Discontinued                                                                                                                                              | Catalog lifecycle. Discontinued does not delete historical references.                            |
| InventoryItemStatus      | Available, Reserved, Sold, Transferred, InRepair, Damaged, Lost, Returned, Retired                                                                                                 | Physical item lifecycle. Some transitions are workflow-controlled.                                |
| InventoryTransactionType | OpeningBalance, Purchase, Sale, SaleReturn, TransferOut, TransferIn, AdjustmentIncrease, AdjustmentDecrease, RepairOut, RepairIn, ExchangeOut, ExchangeIn, BuybackIn, Damage, Loss | Authoritative inventory movement classification.                                                  |
| StockTransferStatus      | Draft, Requested, Approved, Dispatched, Received, Cancelled, Rejected                                                                                                              | Branch transfer workflow.                                                                         |
| PurchaseOrderStatus      | Draft, Submitted, Approved, PartiallyReceived, Received, Cancelled                                                                                                                 | Optional procurement workflow before a Purchase.                                                  |
| PurchaseStatus           | Draft, Posted, PartiallyPaid, Paid, Cancelled                                                                                                                                      | Purchase financial/inventory posting lifecycle.                                                   |
| SaleStatus               | Draft, Quoted, Confirmed, Completed, PartiallyRefunded, Refunded, Cancelled                                                                                                        | POS/sales lifecycle.                                                                              |
| ReturnStatus             | Requested, Approved, Received, Refunded, Rejected, Cancelled                                                                                                                       | Return workflow; the financial refund is represented separately by Payment/refund records.        |
| PaymentStatus            | Pending, Authorized, Captured, PartiallyRefunded, Refunded, Failed, Voided                                                                                                         | Payment provider / settlement lifecycle.                                                          |
| RepairStatus             | Received, Inspected, AwaitingApproval, Approved, InRepair, ReadyForPickup, Delivered, Cancelled                                                                                    | Repair order lifecycle.                                                                           |
| RepairItemStatus         | Pending, Accepted, Rejected, InRepair, Completed, Returned, Cancelled                                                                                                              | Per-item repair state when a repair order contains multiple items.                                |
| GoldPriceSource          | Manual, Imported, ProviderApi, MarketFeed                                                                                                                                          | Origin of a gold price record; exact providers are configuration, not enum members.               |
| GoldPriceStatus          | Scheduled, Active, Superseded, Cancelled                                                                                                                                           | Lifecycle of a price record.                                                                      |
| PricingRuleStatus        | Draft, Active, Inactive, Archived                                                                                                                                                  | Pricing rule lifecycle.                                                                           |
| PricingRuleType          | GoldValue, MakingCharge, StoneValue, Discount, Tax, Rounding, MinimumPrice, MaximumDiscount                                                                                        | Classifies a configurable pricing rule. Rule configuration remains data-driven.                   |
| GoldBuybackStatus        | Draft, Quoted, Approved, Purchased, Paid, Cancelled                                                                                                                                | Old-gold buyback workflow.                                                                        |
| ExchangeStatus           | Draft, Quoted, Approved, Completed, Cancelled                                                                                                                                      | Old-for-new exchange workflow.                                                                    |
| AccountType              | Cash, Bank, Receivable, Payable, Revenue, Expense, Inventory, Tax, Clearing, Equity                                                                                                | Chart-of-accounts classification; account records remain tenant data.                             |
| AccountTransactionType   | Debit, Credit                                                                                                                                                                      | Low-level accounting posting direction.                                                           |
| AuditAction              | Create, Update, Delete, SoftDelete, Restore, Approve, Reject, Post, Cancel, Refund, Login, Logout, PriceChange, PermissionChange, Transfer, StatusChange                           | Audit event classification. Additional action types may be added without changing entity history. |
| AddressType              | Billing, Shipping, Branch, Residential, Business, Other                                                                                                                            | Address purpose; multiple addresses per party are supported.                                      |
| ContactType              | Email, Phone, WhatsApp, SMS, Other                                                                                                                                                 | Generic contact channel classification.                                                           |
| WeightUnit               | Gram, Kilogram, Milligram, Ounce, Tola                                                                                                                                             | Measurement unit. Tenant/market configuration may restrict which units are allowed operationally. |
| DocumentType             | SaleInvoice, PurchaseInvoice, CreditNote, DebitNote, RepairReceipt, TransferNote, BuybackReceipt, ExchangeReceipt, Quotation                                                       | Business document classification for numbering/templates.                                         |
| DocumentStatus           | Draft, Issued, Voided, Archived                                                                                                                                                    | Generic document lifecycle; module-specific states remain authoritative for business workflows.   |
| TaxTreatment             | Taxable, ZeroRated, Exempt, OutOfScope, NotApplicable                                                                                                                              | Tax treatment classification; actual rates/rules are market configuration.                        |

# 4. State-Machine Rules

Enums alone do not define valid transitions. Every workflow aggregate must enforce an explicit transition map in domain/application services.

| **Aggregate** | **Valid example transitions**                                                          | **Forbidden examples**                                                                  |
|---------------|----------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------|
| Tenant        | Pending → Active → Suspended → Active; Active → Closed                                 | Closed → Active; Pending → Closed without activation/closure policy                     |
| InventoryItem | Available → Reserved → Sold; Available → InRepair → Available; Available → Transferred | Sold → Available without a return workflow; Lost → Sold                                 |
| Sale          | Draft → Confirmed → Completed; Completed → PartiallyRefunded → Refunded                | Cancelled after financial settlement unless cancellation policy creates reversal/refund |
| Payment       | Pending → Authorized → Captured → Refunded; Pending → Failed                           | Captured → Pending; Refunded → Captured                                                 |
| Repair        | Received → Inspected → Approved → InRepair → ReadyForPickup → Delivered                | Delivered → InRepair except via a new repair order                                      |

# 5. Value Objects Overview

| **Value Object** | **Suggested type**                                     | **Core validation / invariant**                                                                          |
|------------------|--------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Money            | record struct Money(decimal Amount, Currency Currency) | Amount finite; currency required; no cross-currency arithmetic without explicit conversion.              |
| Currency         | readonly value object / code wrapper                   | ISO-style 3-letter uppercase code; supported currencies validated against market/platform configuration. |
| Weight           | record struct Weight(decimal Value, WeightUnit Unit)   | Value \>= 0; conversion explicit; internal calculations use a canonical base unit.                       |
| Purity           | readonly value object (Karat + fineness)               | Karat/fineness must be valid; the two representations must agree.                                        |
| Percentage       | record struct Percentage(decimal Value)                | 0 \<= value \<= 100 unless explicitly marked as signed rate.                                             |
| TaxRate          | record struct TaxRate(decimal Percentage)              | 0 \<= rate \<= 100; tax computation supplied by market/tax policy.                                       |
| Quantity         | record struct Quantity(decimal Value)                  | Value \>= 0 unless negative quantities are explicitly allowed by a correction command.                   |
| Barcode          | readonly value object string                           | Trimmed, normalized, non-empty; format and length policy configurable.                                   |
| Sku              | readonly value object string                           | Trimmed, normalized, tenant-unique at persistence boundary.                                              |
| SerialNumber     | readonly value object string                           | Trimmed, normalized, non-empty when serial tracking is required.                                         |
| DocumentNumber   | readonly value object string                           | Immutable issued identifier; uniqueness scoped by tenant + document type/numbering policy.               |
| PhoneNumber      | readonly value object string                           | Normalized representation; country/market-aware validation.                                              |
| EmailAddress     | readonly value object string                           | Normalized/canonicalized for comparisons; syntactic validation only in domain.                           |
| Address          | immutable object                                       | Country/region/city/street/postal fields consistent with market configuration.                           |
| DateRange        | readonly value object                                  | Start \<= End; timezone semantics explicitly defined for business dates.                                 |
| GoldPrice        | record struct MoneyPerWeightUnit                       | Positive when active; currency and weight unit are explicit.                                             |
| KaratConversion  | readonly policy/value object                           | Converts karat/fineness to a common fine-gold basis using a single documented formula.                   |

## 5.1 Money

**Purpose:** Represents a monetary amount with its currency; prevents accidental mixing of EGP, SAR, USD, etc.

**Representation:** C# readonly record struct with decimal Amount and Currency Currency.

**Invariants:**

> • Amount uses decimal; Currency is required; addition/subtraction requires the same currency; multiplication/division by a scalar is allowed.

**Operations:** Add, Subtract, Multiply, Allocate, IsZero, IsNegative, Round(policy).

**Persistence:** Persist Amount as numeric(19,4) (or approved precision) and CurrencyCode as char(3) / varchar(3). Never persist Money as JSON for core transactional tables.

**Example:** Money(1250.50, SAR)

## 5.2 Weight

**Purpose:** Represents physical mass for jewelry, gold and stones.

**Representation:** C# readonly record struct with decimal Value and WeightUnit Unit.

**Invariants:**

> • Value \>= 0; conversions must be explicit; database persistence uses a canonical operational unit for critical quantities, preferably grams.

**Operations:** ToGrams, To(Unit), Add, Subtract, Multiply, Compare.

**Persistence:** Persist canonical value as numeric(18,6) grams where precision supports weighing workflow; retain original unit only where business display requires it.

**Example:** Weight(5.240, Gram)

## 5.3 Purity

**Purpose:** Represents precious-metal purity using karat and/or fineness without allowing contradictory values.

**Representation:** Readonly record struct/class with decimal Karat and decimal Fineness, or one canonical fineness plus derived karat.

**Invariants:**

> • Karat between 0 and 24; fineness between 0 and 1000; values must match the configured conversion rule.

**Operations:** FromKarat, FromFineness, ToFineGoldWeight, Compare.

**Persistence:** Persist canonical fineness or a reference to a Purity entity; do not store independent free-form karat and fineness columns without a consistency constraint.

**Example:** 21K ≈ 875 fineness

## 5.4 Percentage and TaxRate

**Purpose:** Represents a bounded percentage used for discount, tax, or markup calculations.

**Representation:** Readonly record struct around decimal.

**Invariants:**

> • Standard percentage is 0..100; signed rates are a separate concept and must not reuse this VO.

**Operations:** Of(amount), ApplyTo(amount), ToFraction.

**Persistence:** Persist as numeric(9,4) percentage rather than integer basis points unless the accounting model explicitly standardizes on basis points.

**Example:** 15.00%

## 5.5 Quantity

**Purpose:** Represents an amount/count of something being moved, sold, received or adjusted.

**Representation:** Readonly record struct around decimal.

**Invariants:**

> • Non-negative by default; quantity unit is defined by context.

**Operations:** Add, Subtract, IsZero, Compare.

**Persistence:** Persist as numeric(18,6) where decimal quantity is needed; database constraints enforce \>= 0 for contexts that prohibit negatives.

**Example:** Quantity(1)

## 5.6 Barcode / SKU / SerialNumber

**Purpose:** Provides validated identifiers instead of raw strings.

**Representation:** Readonly value object wrapping normalized string.

**Invariants:**

> • Trim whitespace; reject blank values; preserve significant zeros; uniqueness enforced at the appropriate tenant scope.

**Operations:** Normalize, EqualsNormalized, AsString.

**Persistence:** Persist as varchar with unique indexes scoped to tenant/business rules.

**Example:** SKU-21K-RING-001 / 240001923 / SN-00021

## 5.7 DocumentNumber

**Purpose:** Represents an issued business document identifier.

**Representation:** Readonly value object wrapping immutable string.

**Invariants:**

> • Non-empty, normalized, numbering scheme compliant; once issued, the value must not be edited in place.

**Operations:** Parse, Format, IsValidForType.

**Persistence:** Persist as varchar with a unique index over the approved numbering scope.

**Example:** INV-RIY-2026-000123

## 5.8 Address

**Purpose:** Represents a structured address used by a branch, customer or supplier.

**Representation:** Immutable record/class containing structured fields plus optional free-form lines.

**Invariants:**

> • Country is required; local validation varies by market; presentation formatting belongs outside the domain object.

**Operations:** Normalize, Format(country/locale), IsEquivalent.

**Persistence:** Prefer structured columns or an owned type. Do not hide operationally important address fields in a JSON blob.

**Example:** Riyadh, Saudi Arabia

## 5.9 DateRange

**Purpose:** Represents a bounded interval for validity, price effectivity, subscription or reporting windows.

**Representation:** Readonly record struct with DateTimeOffset Start and End.

**Invariants:**

> • Start \<= End; inclusive/exclusive semantics must be declared by the consuming policy.

**Operations:** Contains, Overlaps, IsOpenEnded, Duration.

**Persistence:** Persist start/end columns; open-ended end is nullable only when the business rule explicitly allows it.

**Example:** 2026-09-01 → 2026-09-30

## 5.10 GoldPrice

**Purpose:** Represents a gold market price paired with a purity, currency and weight basis.

**Representation:** Value object or domain record composed of Money + Weight basis + Purity context.

**Invariants:**

> • Price \> 0 for active quotes; basis must be explicit (per gram, per kilogram, etc.); purity must be known.

**Operations:** PriceFor(weight), NormalizeToGram, ConvertPurity.

**Persistence:** In persistence, use GoldPrice entity/history for effective-dated records; use the VO inside pricing calculations and application commands.

**Example:** SAR 512.50 / gram for 21K

# 6. Recommended C# Enum Declarations

public enum TenantStatus  
{  
Pending, Active, Suspended, Closed  
}  
  
public enum InventoryItemStatus  
{  
Available, Reserved, Sold, Transferred, InRepair,  
Damaged, Lost, Returned, Retired  
}  
  
public enum SaleStatus  
{  
Draft, Quoted, Confirmed, Completed,  
PartiallyRefunded, Refunded, Cancelled  
}

**Guideline:** Keep enum members stable. Do not use their numeric ordinal values as persisted business data or API contracts.

# 7. What Must NOT Be an Enum

| **Concept**     | **Use**                                          | **Reason**                                                                                                       |
|-----------------|--------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| PaymentMethod   | Entity / tenant configuration                    | Shops can add Card, Wallet, Bank Transfer, Installment, local gateways, or custom methods without redeploying.   |
| GoldType        | Entity / market reference data                   | Different markets and businesses may define gold categories differently; names, units and active flags are data. |
| ProductType     | Entity / configuration                           | The catalog will evolve; customers may need custom categories/types.                                             |
| Currency        | Reference data + value object wrapper            | Currency codes are standardized external data, but supported currencies are configuration.                       |
| Market          | Entity                                           | Carries language, currency, tax, terminology and other configuration.                                            |
| PricingRuleType | Enum is acceptable for broad rule classification | Actual rules, formulas and parameters must remain data-driven.                                                   |
| Feature         | Entity                                           | Subscription/plan features are platform data and need descriptions, limits and lifecycle.                        |

# 8. Numeric & Calculation Conventions

> • Money: decimal only. Never float/double for monetary values or gold weights in the core domain.
>
> • Weight precision: use enough scale for the shop workflow; six decimal places in grams is a safe initial baseline, subject to real scale calibration requirements.
>
> • Rounding: centralized in a pricing/calculation policy; individual screens must not invent their own rounding.
>
> • Tax: calculate from market/tenant tax policy, not from a hard-coded enum member.
>
> • Currency conversion: never silently convert. Require an explicit exchange rate and conversion policy.
>
> • Gold purity conversion: define one canonical formula and test it with approved market examples.
>
> • Percentage calculations: keep full precision through intermediate steps and round only at the documented business boundary.

# 9. EF Core & PostgreSQL Mapping

protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)  
{  
configurationBuilder.Properties\<decimal\>()  
.HavePrecision(19, 4);  
  
// Example enum policy:  
// modelBuilder.Entity\<Sale\>()  
// .Property(x =\> x.Status)  
// .HasConversion\<string\>();  
}

> • Do not apply one global decimal precision blindly if weight precision and financial precision differ; configure precision by property/category.
>
> • Use owned types or EF Core complex types for multi-column value objects where they improve querying and indexing.
>
> • Use JSON only for genuinely dynamic configuration, not for Money, Weight, Address or other core concepts that require constraints and querying.
>
> • For immutable value objects, expose private constructors/factories where EF Core requirements allow, and keep domain validation in the public creation path.

# 10. API & Serialization Rules

| **Concern**            | **Rule**                                                                                                                    |
|------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Enum JSON              | Serialize stable string values (e.g. "Completed"), not numbers.                                                             |
| Localized labels       | UI/API presentation layer may map enum/code values to Arabic/English labels; do not store localized labels as domain state. |
| Value objects          | API DTOs may flatten value objects for usability, e.g. { amount: 1250.50, currency: "SAR" }.                                |
| Validation errors      | Return field-level validation errors using stable machine-readable codes plus human-readable messages.                      |
| Backward compatibility | Unknown enum values from newer servers should not crash clients; use tolerant parsing where appropriate.                    |
| Versioning             | Renaming enum members or changing value-object serialization shape is an API contract change.                               |

# 11. Unit Test Requirements

> • Every value object must have constructor/factory validation tests for minimum, maximum, zero, negative and malformed inputs as applicable.
>
> • Money tests must cover same-currency arithmetic, cross-currency rejection, allocation and rounding.
>
> • Weight tests must cover unit conversion, precision and arithmetic safety.
>
> • Purity tests must cover valid karats/fineness and inconsistent inputs.
>
> • Percentage/TaxRate tests must cover 0%, 100%, out-of-range values and calculation behavior.
>
> • Identifier value objects must cover normalization, whitespace, empty input and equality.
>
> • Every aggregate state machine must have explicit allowed-transition and forbidden-transition tests.
>
> • Serialization tests must verify stable enum strings and value-object JSON contracts.

# 12. Relationship to Aggregates & Domain Events

> • Enums describe state inside aggregates; they do not orchestrate workflows.
>
> • Value objects should not publish events themselves. Aggregate roots publish domain events when a meaningful business change occurs.
>
> • A status change that matters outside the aggregate should produce a domain event where appropriate, e.g. SaleCompleted, PaymentCaptured, InventoryTransferred, RepairCompleted.
>
> • Event payloads should use stable IDs and value objects, but avoid leaking EF Core entities into event contracts.

# 13. Decisions to Keep Stable

| **Decision ID** | **Decision**                                                               | **Rationale**                                                               |
|-----------------|----------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| ENUM-001        | Persist business enums as strings, with explicit EF conversions.           | Readable data, safer migrations, easier support/debugging.                  |
| ENUM-002        | Do not use enums for tenant-configurable/reference data.                   | Supports multi-market customization and subscription-based features.        |
| VO-001          | Money includes currency; arithmetic requires same currency.                | Prevents silent currency bugs.                                              |
| VO-002          | Weight is explicit about unit and calculations have a canonical basis.     | Prevents gram/kg and stone/gold-weight mistakes.                            |
| VO-003          | Purity is a first-class value concept.                                     | Gold pricing and buyback depend on purity correctness.                      |
| VO-004          | Core value objects are immutable.                                          | Reduces hidden mutation and makes domain invariants easier to reason about. |
| API-001         | Enum APIs use stable strings and localized labels stay outside the domain. | Keeps integrations resilient and market/language independent.               |

# 14. Implementation Checklist

> Create Domain/Common/Enums and Domain/Common/ValueObjects folders/namespaces.
>
> Implement enum declarations and centralized transition policies.
>
> Implement Money, Currency, Weight, Purity, Percentage, TaxRate and Quantity first.
>
> Implement identifier value objects next: Barcode, Sku, SerialNumber and DocumentNumber.
>
> Implement Address, DateRange and GoldPrice policies.
>
> Add EF Core conversions and precision conventions per module.
>
> Add JSON converters or DTO mappings for API contracts where needed.
>
> Add unit tests before the first transactional module is implemented.
>
> Review this document whenever a new market or business workflow introduces a new status or domain concept.

**Next document:** After this baseline, the solution should move to concrete .NET project implementation and module contracts, starting with Tenancy/Identity and the shared domain primitives defined here.
