<!-- Converted from Jewelry_Management_System_Database_Schema_Specification.docx. -->

PostgreSQL + Entity Framework Core \| Multi-tenant \| Multi-branch \| Multi-market \| Scalable SaaS

**Version 1.0 \| Engineering Baseline**

# Purpose

This document converts the approved domain model and aggregate boundaries into an implementation-ready relational database design. It defines table ownership, PostgreSQL data types, primary and foreign keys, constraints, indexes, tenancy rules, monetary and weight precision, status fields, deletion strategy, and EF Core mapping guidance.

## Scope

- Core platform and tenancy

- Identity, roles and permissions

- Catalog and jewelry-specific inventory

- Gold, purity, stones and pricing

- Purchasing, sales, payments and returns

- Stock transfers, repairs, buyback and exchange

- Accounting, subscriptions, feature management and auditing

- Cross-cutting constraints and indexing conventions

## Database principle

PostgreSQL is the system of record for the cloud platform. Every tenant-owned business table carries TenantId directly unless ownership is inherited only inside a tightly controlled aggregate. Historical financial, inventory and audit data is preserved rather than physically deleted.

# 1. Database Architecture & Conventions

| **Topic**         | **Decision**                                                                                                     |
|-------------------|------------------------------------------------------------------------------------------------------------------|
| Database          | PostgreSQL                                                                                                       |
| ORM               | Entity Framework Core                                                                                            |
| Primary keys      | UUID (Guid); prefer time-ordered UUID strategy such as UUIDv7 where supported                                    |
| Tenant isolation  | TenantId on tenant-owned tables + application enforcement + database constraints/indexes                         |
| Money             | numeric(19,4) for amounts; currency code stored per transactional aggregate where multi-currency is supported    |
| Weight            | numeric(18,6) grams; exact scale may be narrowed per business validation                                         |
| Rates/percentages | numeric(12,6)                                                                                                    |
| Timestamps        | timestamptz                                                                                                      |
| JSON              | jsonb only for intentionally flexible configuration/custom fields; not for core relational data                  |
| Deletes           | Restrict/cancel/close for historical records; physical delete limited to safe reference/config data              |
| Concurrency       | rowversion equivalent via xmin/concurrency token strategy or explicit Version bigint; use optimistic concurrency |
| Naming            | snake_case database identifiers; PascalCase C# properties; EF Core maps explicitly                               |
| Schema versioning | EF Core migrations reviewed and applied through CI/CD                                                            |

## 1.1 Common audit columns

Transactional/domain entities should use CreatedAt and UpdatedAt where mutable. Append-only ledgers may only require CreatedAt/OccurredAt. AuditLog keeps its own immutable event timestamp. Do not duplicate UserId fields where an entity already has a clear actor/created-by concept unless the business meaning is different.

## 1.2 Tenant ownership pattern

> Tenant-scoped table -\> tenant_id NOT NULL -\> index (tenant_id, ...business lookup...)
>
> Cross-tenant FK rule -\> composite uniqueness/FK patterns where necessary to prevent cross-tenant references

For high-risk entities such as Product, Customer, Sale, InventoryItem and StockTransfer, prefer tenant-aware unique indexes and composite foreign-key relationships where they materially strengthen isolation.

# 2. Core Tables

The following sections are the database contract. Field names are logical names; the physical PostgreSQL implementation should use snake_case. Every foreign key must be reviewed for tenant isolation and delete behavior before migration approval.

## 1. market

Platform / Localization

Defines the default country/market configuration used as a starting point for tenant behavior.

| **Column**    | **PostgreSQL type** | **Constraints / meaning** |
|---------------|---------------------|---------------------------|
| market_id     | uuid                | PK                        |
| name          | varchar(100)        | NOT NULL                  |
| country_code  | char(2)             | NOT NULL                  |
| language_code | varchar(10)         | NOT NULL                  |
| currency_code | char(3)             | NOT NULL                  |
| timezone      | varchar(64)         | NOT NULL                  |
| settings      | jsonb               | NOT NULL DEFAULT {}       |
| is_active     | boolean             | NOT NULL DEFAULT true     |
| created_at    | timestamptz         | NOT NULL                  |

### Constraints

- UNIQUE(country_code)

### Indexes

- ix_market_active(is_active)

### Relationships

- 1:N Tenant

### Implementation notes

- Reference/config table; changes should be audited if used by active tenants.

## 2. tenant

Platform / Tenancy

Represents a business/company using the SaaS platform.

| **Column**       | **PostgreSQL type** | **Constraints / meaning** |
|------------------|---------------------|---------------------------|
| tenant_id        | uuid                | PK                        |
| name             | varchar(200)        | NOT NULL                  |
| code             | varchar(50)         | NOT NULL                  |
| domain           | varchar(255)        | NULL                      |
| market_id        | uuid                | FK -\> market             |
| default_currency | char(3)             | NOT NULL                  |
| timezone         | varchar(64)         | NOT NULL                  |
| status           | varchar(30)         | NOT NULL                  |
| created_at       | timestamptz         | NOT NULL                  |
| updated_at       | timestamptz         | NOT NULL                  |

### Constraints

- UNIQUE(code)

- UNIQUE(domain) WHERE domain IS NOT NULL

### Indexes

- ix_tenant_market(market_id)

- ix_tenant_status(status)

### Relationships

- N:1 Market

- 1:N Branch/User/Product/Customer/Supplier/Subscription/TenantFeature/AuditLog

### Implementation notes

- Closed/Suspended tenants remain stored.

## 3. branch

Platform / Tenancy

A physical or operational shop location owned by a tenant.

| **Column** | **PostgreSQL type** | **Constraints / meaning** |
|------------|---------------------|---------------------------|
| branch_id  | uuid                | PK                        |
| tenant_id  | uuid                | FK                        |
| name       | varchar(150)        | NOT NULL                  |
| code       | varchar(50)         | NOT NULL                  |
| location   | varchar(255)        | NULL                      |
| address    | varchar(500)        | NULL                      |
| phone      | varchar(50)         | NULL                      |
| is_active  | boolean             | NOT NULL DEFAULT true     |
| created_at | timestamptz         | NOT NULL                  |
| updated_at | timestamptz         | NOT NULL                  |

### Constraints

- UNIQUE(tenant_id, code)

### Indexes

- ix_branch_tenant_active(tenant_id,is_active)

### Relationships

- N:1 Tenant

- 1:N User/Sale/InventoryItem/Repair/StockTransfer

### Implementation notes

- Do not hard-delete a branch with historical transactions.

## 4. user

Identity / Security

Tenant member/application user. Authentication credentials may later be delegated to an external identity provider.

| **Column**    | **PostgreSQL type** | **Constraints / meaning** |
|---------------|---------------------|---------------------------|
| user_id       | uuid                | PK                        |
| tenant_id     | uuid                | FK                        |
| branch_id     | uuid                | FK NULL                   |
| name          | varchar(150)        | NOT NULL                  |
| email         | varchar(320)        | NOT NULL                  |
| password_hash | text                | NULL                      |
| phone         | varchar(50)         | NULL                      |
| is_active     | boolean             | NOT NULL DEFAULT true     |
| last_login_at | timestamptz         | NULL                      |
| created_at    | timestamptz         | NOT NULL                  |
| updated_at    | timestamptz         | NOT NULL                  |

### Constraints

- UNIQUE(tenant_id, lower(email))

### Indexes

- ix_user_tenant_branch(tenant_id,branch_id)

- ix_user_active(tenant_id,is_active)

### Relationships

- N:1 Tenant

- N:1 Branch

- M:N Role via UserRole

### Implementation notes

- Do not use email as the immutable identity key.

## 5. role

Identity / Security

A named authorization role.

| **Column**  | **PostgreSQL type** | **Constraints / meaning** |
|-------------|---------------------|---------------------------|
| role_id     | uuid                | PK                        |
| tenant_id   | uuid                | FK NULL                   |
| name        | varchar(100)        | NOT NULL                  |
| description | varchar(500)        | NULL                      |
| is_system   | boolean             | NOT NULL DEFAULT false    |

### Constraints

- UNIQUE(tenant_id, lower(name))

### Indexes

- ix_role_tenant(tenant_id)

### Relationships

- M:N User via UserRole

- M:N Permission

### Implementation notes

- tenant_id NULL supports platform/system roles; application authorization must distinguish platform roles from tenant roles.

## 6. permission

Identity / Security

Fine-grained authorization capability.

| **Column**    | **PostgreSQL type** | **Constraints / meaning** |
|---------------|---------------------|---------------------------|
| permission_id | uuid                | PK                        |
| name          | varchar(150)        | NOT NULL                  |
| module        | varchar(100)        | NOT NULL                  |
| action        | varchar(100)        | NOT NULL                  |
| description   | varchar(500)        | NULL                      |
| is_system     | boolean             | NOT NULL DEFAULT true     |

### Constraints

- UNIQUE(name)

### Indexes

- ix_permission_module(module)

### Relationships

- M:N Role

### Implementation notes

- Permissions are reference/configuration data.

## 7. user_role

Identity / Security

Join table assigning roles to users.

| **Column** | **PostgreSQL type** | **Constraints / meaning** |
|------------|---------------------|---------------------------|
| user_id    | uuid                | PK/FK                     |
| role_id    | uuid                | PK/FK                     |

### Constraints

- PRIMARY KEY(user_id, role_id)

### Relationships

- N:1 User

- N:1 Role

## 8. role_permission

Identity / Security

Join table assigning permissions to roles.

| **Column**    | **PostgreSQL type** | **Constraints / meaning** |
|---------------|---------------------|---------------------------|
| role_id       | uuid                | PK/FK                     |
| permission_id | uuid                | PK/FK                     |

### Constraints

- PRIMARY KEY(role_id, permission_id)

### Relationships

- N:1 Role

- N:1 Permission

## 9. subscription

Subscription / SaaS

One historical subscription period for a tenant.

| **Column**      | **PostgreSQL type** | **Constraints / meaning** |
|-----------------|---------------------|---------------------------|
| subscription_id | uuid                | PK                        |
| tenant_id       | uuid                | FK                        |
| plan_id         | uuid                | FK                        |
| start_date      | date                | NOT NULL                  |
| end_date        | date                | NULL                      |
| status          | varchar(30)         | NOT NULL                  |
| features        | jsonb               | NULL                      |
| usage_limits    | jsonb               | NULL                      |
| created_at      | timestamptz         | NOT NULL                  |

### Constraints

- CHECK(end_date IS NULL OR end_date \>= start_date)

### Indexes

- ix_subscription_tenant_dates(tenant_id,start_date,end_date)

- ix_subscription_status(status)

### Relationships

- N:1 Tenant

- N:1 Plan

### Implementation notes

- Preserve history; active interval overlap rules belong in application/database exclusion strategy if required.

## 10. plan

Subscription / SaaS

Commercial subscription plan definition.

| **Column**    | **PostgreSQL type** | **Constraints / meaning** |
|---------------|---------------------|---------------------------|
| plan_id       | uuid                | PK                        |
| name          | varchar(100)        | NOT NULL                  |
| description   | varchar(500)        | NULL                      |
| price         | numeric(19,4)       | NOT NULL                  |
| billing_cycle | varchar(30)         | NOT NULL                  |
| features      | jsonb               | NOT NULL DEFAULT {}       |
| is_active     | boolean             | NOT NULL DEFAULT true     |

### Constraints

- CHECK(price \>= 0)

- UNIQUE(lower(name))

### Indexes

- ix_plan_active(is_active)

### Relationships

- 1:N Subscription

### Implementation notes

- Pricing for plans can later be externalized to a billing provider while preserving plan identity.

## 11. feature

Subscription / SaaS

Catalog of platform features/add-ons.

| **Column**  | **PostgreSQL type** | **Constraints / meaning** |
|-------------|---------------------|---------------------------|
| feature_id  | uuid                | PK                        |
| name        | varchar(120)        | NOT NULL                  |
| description | varchar(500)        | NULL                      |
| module      | varchar(100)        | NOT NULL                  |
| is_active   | boolean             | NOT NULL DEFAULT true     |

### Constraints

- UNIQUE(lower(name))

### Indexes

- ix_feature_module(module)

### Relationships

- 1:N TenantFeature

## 12. tenant_feature

Subscription / SaaS

Tenant-specific activation and limits for a feature.

| **Column**        | **PostgreSQL type** | **Constraints / meaning** |
|-------------------|---------------------|---------------------------|
| tenant_feature_id | uuid                | PK                        |
| tenant_id         | uuid                | FK                        |
| feature_id        | uuid                | FK                        |
| is_enabled        | boolean             | NOT NULL                  |
| limit_value       | integer             | NULL                      |
| created_at        | timestamptz         | NOT NULL                  |
| updated_at        | timestamptz         | NOT NULL                  |

### Constraints

- UNIQUE(tenant_id,feature_id)

- CHECK(limit_value IS NULL OR limit_value \>= 0)

### Indexes

- ix_tenant_feature_tenant(tenant_id)

### Relationships

- N:1 Tenant

- N:1 Feature

## 13. customer

CRM

Tenant-owned customer/business profile.

| **Column**    | **PostgreSQL type** | **Constraints / meaning** |
|---------------|---------------------|---------------------------|
| customer_id   | uuid                | PK                        |
| tenant_id     | uuid                | FK                        |
| name          | varchar(200)        | NOT NULL                  |
| phone         | varchar(50)         | NULL                      |
| email         | varchar(320)        | NULL                      |
| customer_type | varchar(30)         | NOT NULL                  |
| address       | varchar(500)        | NULL                      |
| notes         | text                | NULL                      |
| is_active     | boolean             | NOT NULL DEFAULT true     |
| created_at    | timestamptz         | NOT NULL                  |
| updated_at    | timestamptz         | NOT NULL                  |

### Constraints

- CHECK(customer_type IN (..configured values..))

### Indexes

- ix_customer_tenant_name(tenant_id,name)

- ix_customer_tenant_phone(tenant_id,phone)

### Relationships

- N:1 Tenant

- 1:N Sale/Repair/GoldBuyback/Exchange

### Implementation notes

- Do not require globally unique customer phone/email.

## 14. supplier

Purchasing

Tenant-owned supplier.

| **Column**     | **PostgreSQL type** | **Constraints / meaning** |
|----------------|---------------------|---------------------------|
| supplier_id    | uuid                | PK                        |
| tenant_id      | uuid                | FK                        |
| name           | varchar(200)        | NOT NULL                  |
| contact_person | varchar(150)        | NULL                      |
| phone          | varchar(50)         | NULL                      |
| email          | varchar(320)        | NULL                      |
| address        | varchar(500)        | NULL                      |
| is_active      | boolean             | NOT NULL DEFAULT true     |
| created_at     | timestamptz         | NOT NULL                  |
| updated_at     | timestamptz         | NOT NULL                  |

### Indexes

- ix_supplier_tenant_name(tenant_id,name)

### Relationships

- N:1 Tenant

- 1:N Purchase/PurchaseOrder

## 15. category

Catalog

Hierarchical product category.

| **Column**         | **PostgreSQL type** | **Constraints / meaning** |
|--------------------|---------------------|---------------------------|
| category_id        | uuid                | PK                        |
| tenant_id          | uuid                | FK                        |
| name               | varchar(150)        | NOT NULL                  |
| description        | varchar(500)        | NULL                      |
| parent_category_id | uuid                | FK NULL                   |
| is_active          | boolean             | NOT NULL DEFAULT true     |

### Constraints

- UNIQUE(tenant_id, lower(name), parent_category_id)

### Indexes

- ix_category_parent(tenant_id,parent_category_id)

### Relationships

- N:1 Tenant

- N:1 Category parent

- 1:N Product

## 16. brand

Catalog

Optional product brand/designer.

| **Column**  | **PostgreSQL type** | **Constraints / meaning** |
|-------------|---------------------|---------------------------|
| brand_id    | uuid                | PK                        |
| tenant_id   | uuid                | FK                        |
| name        | varchar(150)        | NOT NULL                  |
| description | varchar(500)        | NULL                      |
| is_active   | boolean             | NOT NULL DEFAULT true     |

### Constraints

- UNIQUE(tenant_id, lower(name))

### Indexes

- ix_brand_tenant(tenant_id)

### Relationships

- N:1 Tenant

- 1:N Product

## 17. product

Catalog

Product definition: the commercial identity shared by one or more variants/physical items.

| **Column**    | **PostgreSQL type** | **Constraints / meaning** |
|---------------|---------------------|---------------------------|
| product_id    | uuid                | PK                        |
| tenant_id     | uuid                | FK                        |
| category_id   | uuid                | FK                        |
| brand_id      | uuid                | FK NULL                   |
| sku           | varchar(80)         | NOT NULL                  |
| name          | varchar(200)        | NOT NULL                  |
| description   | text                | NULL                      |
| image_url     | text                | NULL                      |
| custom_fields | jsonb               | NULL                      |
| is_active     | boolean             | NOT NULL DEFAULT true     |
| created_at    | timestamptz         | NOT NULL                  |
| updated_at    | timestamptz         | NOT NULL                  |

### Constraints

- UNIQUE(tenant_id, sku)

### Indexes

- ix_product_tenant_category(tenant_id,category_id)

- ix_product_tenant_name(tenant_id,name)

### Relationships

- N:1 Tenant/Category/Brand

- 1:N ProductVariant

### Implementation notes

- SKU belongs to tenant.

## 18. gold_type

Reference / Jewelry

Defines the gold type/material family.

| **Column**   | **PostgreSQL type** | **Constraints / meaning** |
|--------------|---------------------|---------------------------|
| gold_type_id | uuid                | PK                        |
| name         | varchar(100)        | NOT NULL                  |
| unit         | varchar(20)         | NOT NULL                  |
| is_active    | boolean             | NOT NULL DEFAULT true     |

### Constraints

- UNIQUE(lower(name))

### Indexes

- ix_gold_type_active(is_active)

### Relationships

- 1:N ProductVariant/GoldPrice/GoldBuyback

### Implementation notes

- May be tenant-configurable in a later extension if businesses require custom gold types.

## 19. purity

Reference / Jewelry

Defines gold purity/karat.

| **Column** | **PostgreSQL type** | **Constraints / meaning** |
|------------|---------------------|---------------------------|
| purity_id  | uuid                | PK                        |
| name       | varchar(50)         | NOT NULL                  |
| karat      | numeric(6,2)        | NOT NULL                  |
| fineness   | numeric(8,5)        | NULL                      |
| is_active  | boolean             | NOT NULL DEFAULT true     |

### Constraints

- UNIQUE(lower(name))

- UNIQUE(karat)

### Indexes

- ix_purity_active(is_active)

### Relationships

- 1:N ProductVariant/GoldPrice/GoldBuyback

### Implementation notes

- Karat/purity interpretation must be consistent across markets.

## 20. stone

Reference / Jewelry

Defines a stone/gem/diamond type or reference.

| **Column** | **PostgreSQL type** | **Constraints / meaning** |
|------------|---------------------|---------------------------|
| stone_id   | uuid                | PK                        |
| name       | varchar(100)        | NOT NULL                  |
| type       | varchar(50)         | NOT NULL                  |
| unit       | varchar(20)         | NOT NULL                  |
| is_active  | boolean             | NOT NULL DEFAULT true     |

### Constraints

- UNIQUE(lower(name), lower(type))

### Indexes

- ix_stone_type(type)

### Relationships

- 1:N ProductVariant

## 21. product_variant

Catalog / Jewelry

Variant containing the specific jewelry characteristics shared by physical items.

| **Column**         | **PostgreSQL type** | **Constraints / meaning** |
|--------------------|---------------------|---------------------------|
| product_variant_id | uuid                | PK                        |
| product_id         | uuid                | FK                        |
| gold_type_id       | uuid                | FK NULL                   |
| purity_id          | uuid                | FK NULL                   |
| stone_id           | uuid                | FK NULL                   |
| sku_suffix         | varchar(50)         | NULL                      |
| weight             | numeric(18,6)       | NULL                      |
| stone_weight       | numeric(18,6)       | NULL                      |
| default_price      | numeric(19,4)       | NULL                      |
| attributes         | jsonb               | NULL                      |
| is_active          | boolean             | NOT NULL DEFAULT true     |
| created_at         | timestamptz         | NOT NULL                  |

### Constraints

- CHECK(weight IS NULL OR weight \>= 0)

- CHECK(stone_weight IS NULL OR stone_weight \>= 0)

- CHECK(default_price IS NULL OR default_price \>= 0)

### Indexes

- ix_variant_product(product_id)

- ix_variant_purity(purity_id)

- ix_variant_gold(gold_type_id)

### Relationships

- N:1 Product/GoldType/Purity/Stone

- 1:N InventoryItem

### Implementation notes

- Fields that vary per physical piece belong on InventoryItem, not Variant.

## 22. inventory_item

Inventory

One traceable physical jewelry piece or stock unit.

| **Column**         | **PostgreSQL type** | **Constraints / meaning** |
|--------------------|---------------------|---------------------------|
| inventory_item_id  | uuid                | PK                        |
| tenant_id          | uuid                | FK                        |
| branch_id          | uuid                | FK                        |
| product_variant_id | uuid                | FK                        |
| serial_number      | varchar(100)        | NULL                      |
| barcode            | varchar(120)        | NOT NULL                  |
| gross_weight       | numeric(18,6)       | NULL                      |
| gold_weight        | numeric(18,6)       | NULL                      |
| stone_weight       | numeric(18,6)       | NULL                      |
| acquisition_cost   | numeric(19,4)       | NULL                      |
| current_value      | numeric(19,4)       | NULL                      |
| status             | varchar(30)         | NOT NULL                  |
| location           | varchar(120)        | NULL                      |
| created_at         | timestamptz         | NOT NULL                  |
| updated_at         | timestamptz         | NOT NULL                  |
| notes              | text                | NULL                      |

### Constraints

- UNIQUE(tenant_id, barcode)

- CHECK(gross_weight IS NULL OR gross_weight \>= 0)

- CHECK(gold_weight IS NULL OR gold_weight \>= 0)

- CHECK(stone_weight IS NULL OR stone_weight \>= 0)

- CHECK(gold_weight IS NULL OR gross_weight IS NULL OR gold_weight \<= gross_weight)

### Indexes

- ix_inventory_tenant_branch_status(tenant_id,branch_id,status)

- ix_inventory_variant(product_variant_id)

- ix_inventory_serial(tenant_id,serial_number)

### Relationships

- N:1 Tenant/Branch/ProductVariant

- 1:N InventoryTransaction

- 1:N SaleItem/RepairItem/ExchangeItem via references

### Implementation notes

- Physical item is the unit of traceability for high-value jewelry.

## 23. gold_price

Pricing

Historical gold price by market/material/purity.

| **Column**     | **PostgreSQL type** | **Constraints / meaning** |
|----------------|---------------------|---------------------------|
| gold_price_id  | uuid                | PK                        |
| market_id      | uuid                | FK                        |
| gold_type_id   | uuid                | FK                        |
| purity_id      | uuid                | FK                        |
| price_per_gram | numeric(19,6)       | NOT NULL                  |
| effective_date | timestamptz         | NOT NULL                  |
| end_date       | timestamptz         | NULL                      |
| source         | varchar(100)        | NULL                      |
| is_active      | boolean             | NOT NULL DEFAULT true     |

### Constraints

- CHECK(price_per_gram \>= 0)

- CHECK(end_date IS NULL OR end_date \> effective_date)

### Indexes

- ix_gold_price_lookup(market_id,gold_type_id,purity_id,effective_date DESC)

- ix_gold_price_active(is_active)

### Relationships

- N:1 Market/GoldType/Purity

### Implementation notes

- Gold price history is append-oriented; corrections should be new records or controlled adjustments.

## 24. pricing_rule

Pricing

Tenant-specific pricing formula/configuration.

| **Column**      | **PostgreSQL type** | **Constraints / meaning** |
|-----------------|---------------------|---------------------------|
| pricing_rule_id | uuid                | PK                        |
| tenant_id       | uuid                | FK                        |
| rule_type       | varchar(50)         | NOT NULL                  |
| configuration   | jsonb               | NOT NULL                  |
| effective_date  | timestamptz         | NOT NULL                  |
| end_date        | timestamptz         | NULL                      |
| is_active       | boolean             | NOT NULL DEFAULT true     |

### Constraints

- CHECK(end_date IS NULL OR end_date \> effective_date)

### Indexes

- ix_pricing_rule_tenant_type(tenant_id,rule_type,effective_date DESC)

- ix_pricing_rule_active(tenant_id,is_active)

### Relationships

- N:1 Tenant

### Implementation notes

- Configuration must be schema-validated by application/domain code.

## 25. purchase_order

Purchasing

Optional procurement order before receiving a purchase.

| **Column**        | **PostgreSQL type** | **Constraints / meaning** |
|-------------------|---------------------|---------------------------|
| purchase_order_id | uuid                | PK                        |
| tenant_id         | uuid                | FK                        |
| supplier_id       | uuid                | FK                        |
| order_date        | timestamptz         | NOT NULL                  |
| expected_date     | timestamptz         | NULL                      |
| status            | varchar(30)         | NOT NULL                  |
| total_amount      | numeric(19,4)       | NOT NULL DEFAULT 0        |
| notes             | text                | NULL                      |

### Constraints

- CHECK(total_amount \>= 0)

### Indexes

- ix_po_tenant_supplier(tenant_id,supplier_id)

- ix_po_status(tenant_id,status)

### Relationships

- N:1 Tenant/Supplier

- 1:N PurchaseOrderItem

- 0..1 Purchase

## 26. purchase_order_item

Purchasing

Line item in a purchase order.

| **Column**             | **PostgreSQL type** | **Constraints / meaning** |
|------------------------|---------------------|---------------------------|
| purchase_order_item_id | uuid                | PK                        |
| purchase_order_id      | uuid                | FK                        |
| product_variant_id     | uuid                | FK                        |
| quantity               | numeric(18,6)       | NOT NULL                  |
| unit_price             | numeric(19,4)       | NOT NULL                  |
| total_price            | numeric(19,4)       | NOT NULL                  |
| weight                 | numeric(18,6)       | NULL                      |

### Constraints

- CHECK(quantity \> 0)

- CHECK(unit_price \>= 0)

- CHECK(total_price \>= 0)

### Indexes

- ix_po_item_order(purchase_order_id)

### Relationships

- N:1 PurchaseOrder/ProductVariant

## 27. purchase

Purchasing

Posted/received supplier purchase that creates inventory/cost entries.

| **Column**        | **PostgreSQL type** | **Constraints / meaning** |
|-------------------|---------------------|---------------------------|
| purchase_id       | uuid                | PK                        |
| tenant_id         | uuid                | FK                        |
| supplier_id       | uuid                | FK                        |
| purchase_order_id | uuid                | FK NULL                   |
| purchase_date     | timestamptz         | NOT NULL                  |
| receive_date      | timestamptz         | NULL                      |
| total_amount      | numeric(19,4)       | NOT NULL                  |
| status            | varchar(30)         | NOT NULL                  |
| notes             | text                | NULL                      |

### Constraints

- CHECK(total_amount \>= 0)

### Indexes

- ix_purchase_tenant_date(tenant_id,purchase_date DESC)

- ix_purchase_supplier(tenant_id,supplier_id)

### Relationships

- N:1 Tenant/Supplier/PurchaseOrder

- 1:N PurchaseItem

### Implementation notes

- Posting a purchase should create inventory/accounting side effects atomically where applicable.

## 28. purchase_item

Purchasing

Line item within a posted purchase.

| **Column**         | **PostgreSQL type** | **Constraints / meaning** |
|--------------------|---------------------|---------------------------|
| purchase_item_id   | uuid                | PK                        |
| purchase_id        | uuid                | FK                        |
| product_variant_id | uuid                | FK                        |
| inventory_item_id  | uuid                | FK NULL                   |
| quantity           | numeric(18,6)       | NOT NULL                  |
| unit_price         | numeric(19,4)       | NOT NULL                  |
| total_price        | numeric(19,4)       | NOT NULL                  |
| weight             | numeric(18,6)       | NULL                      |

### Constraints

- CHECK(quantity \> 0)

- CHECK(unit_price \>= 0)

- CHECK(total_price \>= 0)

### Indexes

- ix_purchase_item_purchase(purchase_id)

- ix_purchase_item_inventory(inventory_item_id)

### Relationships

- N:1 Purchase/ProductVariant

- N:1 InventoryItem optional

## 29. sale

Sales

Posted customer sale header.

| **Column**     | **PostgreSQL type** | **Constraints / meaning** |
|----------------|---------------------|---------------------------|
| sale_id        | uuid                | PK                        |
| tenant_id      | uuid                | FK                        |
| branch_id      | uuid                | FK                        |
| customer_id    | uuid                | FK NULL                   |
| user_id        | uuid                | FK                        |
| sale_number    | varchar(60)         | NOT NULL                  |
| sale_date      | timestamptz         | NOT NULL                  |
| total_amount   | numeric(19,4)       | NOT NULL                  |
| discount       | numeric(19,4)       | NOT NULL DEFAULT 0        |
| tax            | numeric(19,4)       | NOT NULL DEFAULT 0        |
| final_amount   | numeric(19,4)       | NOT NULL                  |
| payment_status | varchar(30)         | NOT NULL                  |
| status         | varchar(30)         | NOT NULL                  |
| notes          | text                | NULL                      |

### Constraints

- UNIQUE(tenant_id,sale_number)

- CHECK(discount \>= 0)

- CHECK(tax \>= 0)

- CHECK(final_amount \>= 0)

### Indexes

- ix_sale_tenant_date(tenant_id,sale_date DESC)

- ix_sale_branch_date(tenant_id,branch_id,sale_date DESC)

- ix_sale_customer(tenant_id,customer_id)

### Relationships

- N:1 Tenant/Branch/Customer/User

- 1:N SaleItem/Payment/Return

### Implementation notes

- Sale posting is an aggregate transaction; inventory and payment effects must be consistent with status.

## 30. sale_item

Sales

Line item in a sale.

| **Column**         | **PostgreSQL type** | **Constraints / meaning** |
|--------------------|---------------------|---------------------------|
| sale_item_id       | uuid                | PK                        |
| sale_id            | uuid                | FK                        |
| product_variant_id | uuid                | FK                        |
| inventory_item_id  | uuid                | FK NULL                   |
| quantity           | numeric(18,6)       | NOT NULL                  |
| unit_price         | numeric(19,4)       | NOT NULL                  |
| discount           | numeric(19,4)       | NOT NULL DEFAULT 0        |
| tax                | numeric(19,4)       | NOT NULL DEFAULT 0        |
| total_price        | numeric(19,4)       | NOT NULL                  |
| weight             | numeric(18,6)       | NULL                      |
| notes              | text                | NULL                      |

### Constraints

- CHECK(quantity \> 0)

- CHECK(unit_price \>= 0)

- CHECK(discount \>= 0)

- CHECK(tax \>= 0)

- CHECK(total_price \>= 0)

### Indexes

- ix_sale_item_sale(sale_id)

- ix_sale_item_inventory(inventory_item_id)

### Relationships

- N:1 Sale/ProductVariant/InventoryItem

### Implementation notes

- For serialized jewelry, quantity is normally 1 and InventoryItemId is required by application rules.

## 31. payment

Sales / Finance

Payment allocation against a sale.

| **Column**     | **PostgreSQL type** | **Constraints / meaning** |
|----------------|---------------------|---------------------------|
| payment_id     | uuid                | PK                        |
| sale_id        | uuid                | FK                        |
| payment_method | varchar(50)         | NOT NULL                  |
| amount         | numeric(19,4)       | NOT NULL                  |
| reference_no   | varchar(120)        | NULL                      |
| payment_date   | timestamptz         | NOT NULL                  |
| status         | varchar(30)         | NOT NULL                  |

### Constraints

- CHECK(amount \> 0)

### Indexes

- ix_payment_sale(sale_id)

- ix_payment_date(payment_date DESC)

- ix_payment_method(payment_method)

### Relationships

- N:1 Sale

### Implementation notes

- Mixed payment is represented as multiple payment rows.

## 32. return

Sales

Return transaction linked to an original sale.

| **Column**    | **PostgreSQL type** | **Constraints / meaning** |
|---------------|---------------------|---------------------------|
| return_id     | uuid                | PK                        |
| tenant_id     | uuid                | FK                        |
| sale_id       | uuid                | FK                        |
| user_id       | uuid                | FK                        |
| return_date   | timestamptz         | NOT NULL                  |
| reason        | varchar(500)        | NULL                      |
| status        | varchar(30)         | NOT NULL                  |
| refund_amount | numeric(19,4)       | NOT NULL DEFAULT 0        |
| notes         | text                | NULL                      |

### Constraints

- CHECK(refund_amount \>= 0)

### Indexes

- ix_return_tenant_date(tenant_id,return_date DESC)

- ix_return_sale(sale_id)

### Relationships

- N:1 Tenant/Sale/User

- 1:N ReturnItem

### Implementation notes

- Return should reference the original sale item and never mutate historical SaleItem data.

## 33. return_item

Sales

Returned sale line.

| **Column**        | **PostgreSQL type** | **Constraints / meaning** |
|-------------------|---------------------|---------------------------|
| return_item_id    | uuid                | PK                        |
| return_id         | uuid                | FK                        |
| sale_item_id      | uuid                | FK                        |
| inventory_item_id | uuid                | FK NULL                   |
| quantity          | numeric(18,6)       | NOT NULL                  |
| refund_amount     | numeric(19,4)       | NOT NULL                  |

### Constraints

- CHECK(quantity \> 0)

- CHECK(refund_amount \>= 0)

### Indexes

- ix_return_item_return(return_id)

- ix_return_item_sale_item(sale_item_id)

### Relationships

- N:1 Return/SaleItem/InventoryItem

## 34. inventory_transaction

Inventory

Append-only stock movement entry for a physical item or quantity-based stock.

| **Column**               | **PostgreSQL type** | **Constraints / meaning** |
|--------------------------|---------------------|---------------------------|
| inventory_transaction_id | uuid                | PK                        |
| tenant_id                | uuid                | FK                        |
| branch_id                | uuid                | FK                        |
| inventory_item_id        | uuid                | FK                        |
| transaction_type         | varchar(50)         | NOT NULL                  |
| quantity                 | numeric(18,6)       | NOT NULL                  |
| weight                   | numeric(18,6)       | NULL                      |
| unit_cost                | numeric(19,4)       | NULL                      |
| reference_type           | varchar(50)         | NULL                      |
| reference_id             | uuid                | NULL                      |
| transaction_date         | timestamptz         | NOT NULL                  |
| user_id                  | uuid                | FK NULL                   |
| notes                    | text                | NULL                      |

### Constraints

- CHECK(quantity \<\> 0)

- CHECK(weight IS NULL OR weight \<\> 0)

- CHECK(unit_cost IS NULL OR unit_cost \>= 0)

### Indexes

- ix_inv_tx_item_date(inventory_item_id,transaction_date DESC)

- ix_inv_tx_branch_date(tenant_id,branch_id,transaction_date DESC)

- ix_inv_tx_reference(reference_type,reference_id)

### Relationships

- N:1 Tenant/Branch/InventoryItem/User

### Implementation notes

- Append-only. Do not update/delete posted ledger entries; corrections are compensating transactions.

## 35. stock_transfer

Inventory

Header for moving inventory between branches.

| **Column**        | **PostgreSQL type** | **Constraints / meaning** |
|-------------------|---------------------|---------------------------|
| stock_transfer_id | uuid                | PK                        |
| tenant_id         | uuid                | FK                        |
| from_branch_id    | uuid                | FK                        |
| to_branch_id      | uuid                | FK                        |
| user_id           | uuid                | FK                        |
| transfer_date     | timestamptz         | NOT NULL                  |
| status            | varchar(30)         | NOT NULL                  |
| requested_at      | timestamptz         | NULL                      |
| approved_at       | timestamptz         | NULL                      |
| shipped_at        | timestamptz         | NULL                      |
| received_at       | timestamptz         | NULL                      |
| notes             | text                | NULL                      |

### Constraints

- CHECK(from_branch_id \<\> to_branch_id)

### Indexes

- ix_transfer_tenant_date(tenant_id,transfer_date DESC)

- ix_transfer_status(tenant_id,status)

- ix_transfer_from_to(tenant_id,from_branch_id,to_branch_id)

### Relationships

- N:1 Tenant/Branch/User

- 1:N StockTransferItem

### Implementation notes

- Cross-branch movement changes ownership/location only through posted workflow.

## 36. stock_transfer_item

Inventory

Physical or quantity line contained in a stock transfer.

| **Column**             | **PostgreSQL type** | **Constraints / meaning** |
|------------------------|---------------------|---------------------------|
| stock_transfer_item_id | uuid                | PK                        |
| stock_transfer_id      | uuid                | FK                        |
| inventory_item_id      | uuid                | FK                        |
| quantity               | numeric(18,6)       | NOT NULL                  |
| weight                 | numeric(18,6)       | NULL                      |
| notes                  | text                | NULL                      |

### Constraints

- CHECK(quantity \> 0)

### Indexes

- ix_transfer_item_transfer(stock_transfer_id)

- ix_transfer_item_inventory(inventory_item_id)

### Relationships

- N:1 StockTransfer/InventoryItem

## 37. repair

Repairs / Services

Repair/service work order for customer jewelry.

| **Column**          | **PostgreSQL type** | **Constraints / meaning** |
|---------------------|---------------------|---------------------------|
| repair_id           | uuid                | PK                        |
| tenant_id           | uuid                | FK                        |
| customer_id         | uuid                | FK                        |
| branch_id           | uuid                | FK                        |
| received_by_user_id | uuid                | FK NULL                   |
| received_date       | timestamptz         | NOT NULL                  |
| expected_date       | timestamptz         | NULL                      |
| completed_date      | timestamptz         | NULL                      |
| status              | varchar(40)         | NOT NULL                  |
| estimated_cost      | numeric(19,4)       | NULL                      |
| actual_cost         | numeric(19,4)       | NULL                      |
| notes               | text                | NULL                      |

### Constraints

- CHECK(estimated_cost IS NULL OR estimated_cost \>= 0)

- CHECK(actual_cost IS NULL OR actual_cost \>= 0)

### Indexes

- ix_repair_customer(tenant_id,customer_id)

- ix_repair_status(tenant_id,status)

- ix_repair_branch_date(tenant_id,branch_id,received_date DESC)

### Relationships

- N:1 Tenant/Customer/Branch/User

- 1:N RepairItem

## 38. repair_item

Repairs / Services

Jewelry item included in a repair order.

| **Column**         | **PostgreSQL type** | **Constraints / meaning** |
|--------------------|---------------------|---------------------------|
| repair_item_id     | uuid                | PK                        |
| repair_id          | uuid                | FK                        |
| inventory_item_id  | uuid                | FK NULL                   |
| product_variant_id | uuid                | FK NULL                   |
| description        | text                | NOT NULL                  |
| cost               | numeric(19,4)       | NOT NULL DEFAULT 0        |
| status             | varchar(30)         | NOT NULL                  |

### Constraints

- CHECK(cost \>= 0)

### Indexes

- ix_repair_item_repair(repair_id)

- ix_repair_item_inventory(inventory_item_id)

### Relationships

- N:1 Repair/InventoryItem/ProductVariant

## 39. gold_buyback

Gold / Pricing / Sales

Purchase of old gold/jewelry from a customer.

| **Column**        | **PostgreSQL type** | **Constraints / meaning** |
|-------------------|---------------------|---------------------------|
| buyback_id        | uuid                | PK                        |
| tenant_id         | uuid                | FK                        |
| customer_id       | uuid                | FK                        |
| branch_id         | uuid                | FK                        |
| user_id           | uuid                | FK                        |
| gold_type_id      | uuid                | FK NULL                   |
| purity_id         | uuid                | FK                        |
| gross_weight      | numeric(18,6)       | NOT NULL                  |
| stone_weight      | numeric(18,6)       | NULL                      |
| gold_weight       | numeric(18,6)       | NOT NULL                  |
| gold_price        | numeric(19,6)       | NOT NULL                  |
| value             | numeric(19,4)       | NOT NULL                  |
| payment_method    | varchar(50)         | NOT NULL                  |
| payment_reference | varchar(120)        | NULL                      |
| status            | varchar(30)         | NOT NULL                  |
| transaction_date  | timestamptz         | NOT NULL                  |
| notes             | text                | NULL                      |

### Constraints

- CHECK(gross_weight \> 0)

- CHECK(gold_weight \> 0)

- CHECK(gold_price \>= 0)

- CHECK(value \>= 0)

- CHECK(gold_weight \<= gross_weight)

### Indexes

- ix_buyback_customer_date(tenant_id,customer_id,transaction_date DESC)

- ix_buyback_date(tenant_id,transaction_date DESC)

### Relationships

- N:1 Tenant/Customer/Branch/User/GoldType/Purity

### Implementation notes

- Buyback valuation should snapshot the price used, not depend on future GoldPrice rows.

## 40. exchange

Sales / Jewelry

Trade-in/exchange transaction combining old jewelry value and new jewelry purchase.

| **Column**       | **PostgreSQL type** | **Constraints / meaning** |
|------------------|---------------------|---------------------------|
| exchange_id      | uuid                | PK                        |
| tenant_id        | uuid                | FK                        |
| customer_id      | uuid                | FK                        |
| branch_id        | uuid                | FK                        |
| user_id          | uuid                | FK                        |
| old_item_value   | numeric(19,4)       | NOT NULL                  |
| new_item_value   | numeric(19,4)       | NOT NULL                  |
| net_amount       | numeric(19,4)       | NOT NULL                  |
| gold_price       | numeric(19,6)       | NULL                      |
| payment_method   | varchar(50)         | NULL                      |
| status           | varchar(30)         | NOT NULL                  |
| transaction_date | timestamptz         | NOT NULL                  |
| notes            | text                | NULL                      |

### Constraints

- CHECK(old_item_value \>= 0)

- CHECK(new_item_value \>= 0)

- CHECK(net_amount \>= 0)

### Indexes

- ix_exchange_customer_date(tenant_id,customer_id,transaction_date DESC)

- ix_exchange_date(tenant_id,transaction_date DESC)

### Relationships

- N:1 Tenant/Customer/Branch/User

- 1:N ExchangeItem

### Implementation notes

- Use line items so multiple old/new pieces can participate.

## 41. exchange_item

Sales / Jewelry

Line belonging to an exchange transaction.

| **Column**         | **PostgreSQL type** | **Constraints / meaning** |
|--------------------|---------------------|---------------------------|
| exchange_item_id   | uuid                | PK                        |
| exchange_id        | uuid                | FK                        |
| inventory_item_id  | uuid                | FK NULL                   |
| product_variant_id | uuid                | FK NULL                   |
| direction          | varchar(20)         | NOT NULL                  |
| quantity           | numeric(18,6)       | NOT NULL                  |
| weight             | numeric(18,6)       | NULL                      |
| value              | numeric(19,4)       | NOT NULL                  |

### Constraints

- CHECK(direction IN (IN,OUT))

- CHECK(quantity \> 0)

- CHECK(value \>= 0)

### Indexes

- ix_exchange_item_exchange(exchange_id)

- ix_exchange_item_inventory(inventory_item_id)

### Relationships

- N:1 Exchange/InventoryItem/ProductVariant

### Implementation notes

- Direction semantics: IN = item acquired by shop; OUT = item sold to customer.

## 42. account

Accounting

Chart-of-accounts ledger account.

| **Column** | **PostgreSQL type** | **Constraints / meaning** |
|------------|---------------------|---------------------------|
| account_id | uuid                | PK                        |
| tenant_id  | uuid                | FK                        |
| name       | varchar(150)        | NOT NULL                  |
| code       | varchar(50)         | NOT NULL                  |
| type       | varchar(30)         | NOT NULL                  |
| balance    | numeric(19,4)       | NOT NULL DEFAULT 0        |
| is_active  | boolean             | NOT NULL DEFAULT true     |

### Constraints

- UNIQUE(tenant_id,code)

- CHECK(balance \>= 0) OR allow signed balance based on accounting policy

### Indexes

- ix_account_tenant_type(tenant_id,type)

### Relationships

- N:1 Tenant

- 1:N AccountTransaction

### Implementation notes

- Balance policy must match accounting model; in double-entry systems derived balances are preferable to stored mutable balances.

## 43. account_transaction

Accounting

Financial posting entry related to a source business transaction.

| **Column**             | **PostgreSQL type** | **Constraints / meaning** |
|------------------------|---------------------|---------------------------|
| account_transaction_id | uuid                | PK                        |
| tenant_id              | uuid                | FK                        |
| account_id             | uuid                | FK                        |
| type                   | varchar(30)         | NOT NULL                  |
| amount                 | numeric(19,4)       | NOT NULL                  |
| transaction_date       | timestamptz         | NOT NULL                  |
| reference_id           | uuid                | NULL                      |
| reference_type         | varchar(50)         | NULL                      |
| description            | varchar(500)        | NULL                      |

### Constraints

- CHECK(amount \> 0)

### Indexes

- ix_account_tx_account_date(account_id,transaction_date DESC)

- ix_account_tx_reference(reference_type,reference_id)

- ix_account_tx_tenant_date(tenant_id,transaction_date DESC)

### Relationships

- N:1 Tenant/Account

### Implementation notes

- If implementing full double-entry accounting, replace single amount with debit/credit lines or a journal-entry model.

## 44. audit_log

System / Audit

Immutable audit trail for security-sensitive and material business changes.

| **Column**   | **PostgreSQL type** | **Constraints / meaning** |
|--------------|---------------------|---------------------------|
| audit_log_id | uuid                | PK                        |
| tenant_id    | uuid                | FK NULL                   |
| user_id      | uuid                | FK NULL                   |
| entity_name  | varchar(150)        | NOT NULL                  |
| entity_id    | uuid                | NULL                      |
| action       | varchar(50)         | NOT NULL                  |
| old_value    | jsonb               | NULL                      |
| new_value    | jsonb               | NULL                      |
| timestamp    | timestamptz         | NOT NULL                  |
| ip_address   | inet                | NULL                      |
| device_info  | varchar(500)        | NULL                      |

### Indexes

- ix_audit_tenant_time(tenant_id,timestamp DESC)

- ix_audit_entity(entity_name,entity_id)

- ix_audit_user_time(user_id,timestamp DESC)

### Relationships

- N:1 Tenant/User

### Implementation notes

- Append-only; protect access more strongly than normal business data.

# 3. Enumerations and Status Reference

Use C# enums/domain value objects at the application layer. The PostgreSQL representation can be varchar with CHECK constraints or native Postgres enums. For a SaaS product with evolving workflows, varchar + validated constants is usually easier to migrate than native enums.

| **Concept**              | **Recommended values (initial baseline)**                                                                                                           | **Notes**                                              |
|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| TenantStatus             | Pending, Active, Suspended, Closed                                                                                                                  | Closed is a lifecycle state, not delete                |
| User/Reference active    | Active/Inactive via boolean                                                                                                                         | Prefer status enum only where more states are required |
| InventoryItemStatus      | Available, Reserved, InRepair, Sold, Transferred, Damaged, Lost, Archived                                                                           | Exact states may be refined during workflow design     |
| InventoryTransactionType | Purchase, Sale, Return, TransferOut, TransferIn, AdjustmentIn, AdjustmentOut, RepairIn, RepairOut, Buyback, ExchangeIn, ExchangeOut, OpeningBalance | Ledger entries are append-only                         |
| SaleStatus               | Draft, Posted, Cancelled, Refunded, PartiallyReturned                                                                                               | Posted is financially/inventory relevant               |
| PaymentStatus            | Pending, Completed, Failed, Refunded, PartiallyRefunded                                                                                             |                                                        |
| PurchaseStatus           | Draft, Received, Posted, Cancelled                                                                                                                  |                                                        |
| TransferStatus           | Draft, Requested, Approved, Shipped, Received, Cancelled                                                                                            |                                                        |
| RepairStatus             | Received, Inspection, WaitingApproval, InRepair, Completed, ReadyForPickup, Delivered, Cancelled                                                    |                                                        |
| ExchangeItemDirection    | IN, OUT                                                                                                                                             | IN acquired by shop; OUT sold to customer              |
| SubscriptionStatus       | Pending, Active, PastDue, Cancelled, Expired                                                                                                        |                                                        |
| CustomerType             | Individual, Company                                                                                                                                 | May be extended per market                             |
| AccountType              | Asset, Liability, Equity, Revenue, Expense                                                                                                          | For proper double-entry accounting                     |

# 4. Keys, Constraints and Integrity Rules

- Tenant-scoped identifiers must never be resolvable across tenants. Application services must derive TenantId from authenticated tenant context, not from untrusted request body fields.

- Unique business identifiers should normally be scoped to TenantId: SKU, barcode, sale number, purchase/order numbers, branch code, customer-facing reference numbers.

- Historical business records must be preserved. Use status transitions, cancellation, reversal, or compensating ledger entries instead of DELETE for sales, payments, purchases, inventory transactions, accounting transactions and audit logs.

- Use CHECK constraints for impossible numeric states: negative money where disallowed, negative weights, zero/negative quantities, invalid date ranges, and branch transfer source = destination.

- Foreign-key delete behavior should default to RESTRICT/NO ACTION for historical transactional data. SET NULL is appropriate only when the relationship is optional and retaining the transaction without the referenced actor/config record is valid.

- Prefer composite uniqueness/FK constraints when they help prove tenant ownership at the database level, especially for relations among Tenant, Branch, Product, InventoryItem, Sale and Customer.

- Use timestamptz for all persisted timestamps. The tenant timezone is presentation/business-context data, not a replacement for timezone-aware timestamps.

- Money calculations must use decimal/numeric, never floating point. Weight and purity calculations must use decimal/numeric with explicitly chosen precision and scale.

- JSONB is reserved for settings, custom fields and versioned rule configuration. Core entities that participate in joins, constraints, reporting or financial logic should use typed columns/tables.

# 5. Indexing Strategy

| **Pattern**            | **Example**                                 | **Reason**                         |
|------------------------|---------------------------------------------|------------------------------------|
| Tenant + date          | (tenant_id, sale_date DESC)                 | Fast tenant-scoped recent activity |
| Tenant + branch + date | (tenant_id, branch_id, sale_date DESC)      | Branch dashboards/reports          |
| Tenant + business key  | UNIQUE (tenant_id, sku)                     | Isolation + lookup + uniqueness    |
| Reference lookup       | (reference_type, reference_id)              | Cross-module transaction tracing   |
| Inventory lookup       | (tenant_id, barcode)                        | POS/barcode scanning               |
| Inventory history      | (inventory_item_id, transaction_date DESC)  | Piece audit trail                  |
| Active config          | (tenant_id, is_active)                      | Current feature/config lookups     |
| Effective dating       | (tenant_id, rule_type, effective_date DESC) | Resolve current pricing/rules      |

Do not index every column by default. Add indexes based on actual query paths and explain plans. PostgreSQL can carry many indexes, but every additional index increases write cost, storage, and migration time.

# 6. Multi-Tenant Database Strategy

Recommended starting model: shared PostgreSQL database, shared schema, tenant_id on tenant-owned tables. This keeps operations and migrations simple while allowing strong logical isolation. The architecture should keep a future path open for dedicated databases for very large enterprise tenants without changing domain APIs.

> API request -\> authenticated tenant context -\> application authorization -\> tenant-aware query -\> database

For especially sensitive tables, consider PostgreSQL Row Level Security (RLS) once operational practices are ready. RLS can become a defense-in-depth layer, but the application must still enforce tenant context and authorization.

# 7. EF Core Mapping Guidance

- Keep entity configuration in dedicated IEntityTypeConfiguration\<T\> classes grouped by module.

- Configure all decimal precision explicitly with HasPrecision(19,4), HasPrecision(18,6), or the exact domain-approved scale.

- Map database names explicitly to snake_case rather than depending on conventions.

- Use shadow properties only for infrastructure concerns such as concurrency where appropriate; keep domain-significant data explicit.

- Use owned types/value converters selectively for Money, Weight, Address and similar value objects. Avoid overcomplicating persistence for the first production release.

- Use optimistic concurrency tokens on mutable aggregates such as Product, InventoryItem, Sale draft state, StockTransfer and PricingRule.

- Do not expose EF entities directly from controllers. Application DTOs/contracts should define API boundaries.

- Keep migrations reviewable. A migration that changes financial/inventory history should be treated as a controlled production change, not a routine generated file.

# 8. Transaction Boundaries

| **Operation**     | **Atomic database boundary**                                                                          | **Must stay consistent**                             |
|-------------------|-------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Post Sale         | Sale + SaleItem + Payment(s) + InventoryTransaction(s) + status updates + audit event/outbox entry    | Sale totals, payment allocation, inventory state     |
| Post Purchase     | Purchase + PurchaseItem + InventoryItem creation/update + InventoryTransaction(s) + accounting/outbox | Received inventory, cost, payable/accounting         |
| Complete Transfer | StockTransfer + StockTransferItem + transfer inventory movements + statuses                           | Source and destination stock                         |
| Post Return       | Return + ReturnItem + refund/payment record + inventory movement                                      | Refund amount and returned stock                     |
| Complete Buyback  | GoldBuyback + payment + inventory/accounting effect                                                   | Price snapshot, weight, valuation, payment           |
| Complete Exchange | Exchange + ExchangeItem + sale/buyback/stock effects as modeled                                       | Old item intake, new item sale, net customer payment |

# 9. Reporting and Read Models

Operational transactions stay normalized. Reporting requirements may justify SQL views, materialized views, or a dedicated read model later. Do not denormalize financial/inventory source tables prematurely. Build dashboards from indexed queries first and introduce reporting infrastructure when measured workload justifies it.

# 10. Data Lifecycle, Backup and Security

- Use automated PostgreSQL backups with tested restore procedures.

- Encrypt data in transit; use managed secrets for database credentials and signing keys.

- Limit direct database access in production. Application services should be the normal access path.

- Audit security-sensitive actions and material business changes, especially price overrides, gold-price changes, discounts, refunds, inventory adjustments, permission changes and tenant configuration changes.

- PII such as customer contact data must be access-controlled and retained according to the business/privacy policy for the deployment market.

- When a tenant is closed, retain data according to the retention policy; a purge process, if ever introduced, must be explicit, audited and separated from the normal application delete flow.

# 11. Open Decisions Before First Migration

These items should be decided explicitly before generating the first production migration. They are deliberately left as architecture decisions rather than hidden assumptions in the schema.

- Whether authentication users are fully platform-owned or delegated to an external identity provider; this affects the final User schema.

- Exact accounting model. The current AccountTransaction is a baseline; full double-entry accounting should use JournalEntry + JournalLine if accounting becomes a first-class module.

- Exact market/tax tables. Tax rules should be separate typed entities once jurisdiction-specific tax behavior is implemented.

- Whether gold types and purities are global reference data or tenant-configurable reference data. A hybrid model may be best.

- Whether all jewelry inventory is serialized. Some consumables/accessories may require quantity-based stock rather than one-row-per-piece.

- Invoice/numbering sequences and whether they reset by branch, year, market, or tenant.

- Offline synchronization. If enabled later, additional sync metadata, idempotency keys, conflict strategy and device identity tables will be required.

- Outbox/event infrastructure. Recommended before integrating reliable notifications, cloud sync, analytics or external billing.

# 12. Implementation Order

- Create PostgreSQL database and base schemas/extensions.

- Implement Market, Tenant, Branch, User/Role/Permission foundation.

- Implement subscription/feature configuration.

- Implement catalog: Category, Brand, Product, ProductVariant, GoldType, Purity, Stone.

- Implement InventoryItem and InventoryTransaction.

- Implement GoldPrice and PricingRule.

- Implement Sales, SaleItem, Payment, Return/ReturnItem.

- Implement Purchasing entities.

- Implement StockTransfer and Repair.

- Implement GoldBuyback and Exchange.

- Add accounting model at the agreed scope.

- Add AuditLog, outbox and reporting read models as required.

- Create integration tests that prove tenant isolation, inventory integrity, pricing snapshot behavior and financial transaction atomicity.

# 13. Database Naming Examples

> Table: inventory_item  
> PK: inventory_item_id uuid  
> FK: tenant_id -\> tenant.tenant_id  
> FK: branch_id -\> branch.branch_id  
> UNIQUE: (tenant_id, barcode)  
> INDEX: (tenant_id, branch_id, status)
>
> Table: sale  
> PK: sale_id uuid  
> FK: tenant_id -\> tenant.tenant_id  
> FK: branch_id -\> branch.branch_id  
> UNIQUE: (tenant_id, sale_number)  
> INDEX: (tenant_id, sale_date DESC)

# 14. Final Baseline

This schema is the engineering baseline derived from the current ERD, business rules, aggregate boundaries and domain model. It is intentionally implementation-oriented but not yet a generated EF Core migration. The next engineering step is to turn this specification into the .NET solution structure, DbContext modules, entity configurations, and the first migration while validating the remaining open decisions above.
