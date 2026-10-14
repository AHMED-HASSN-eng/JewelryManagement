<!-- Converted from Jewelry_Management_System_Final_Database_Tables.docx. Baseline lists 52 tables; implement in Jira-approved phases, not one unreviewed migration. -->

**JEWELRY MANAGEMENT SYSTEM**

**Final Database Tables Specification**

PostgreSQL • EF Core • Multi-tenant • Multi-branch • Multi-market • Scalable SaaS

![High-level Jewelry Management System ERD](../assets/jewelry-management-system-erd.png)

Reference ERD (high-level). This document is the implementation table baseline.

Document status: Engineering baseline • Version 1.0 • 15 September 2026

Purpose: This document consolidates the previously approved entity definitions, business rules, aggregate boundaries, domain model, database schema, and enums/value-object decisions into one final database-table inventory. It is the table-level source of truth for the first PostgreSQL/EF Core implementation unless an ADR explicitly supersedes it.

# 1. Final Table Count

**Final baseline:** 52 PostgreSQL tables.

The baseline contains the 44 tables already present in the detailed database schema plus 8 normalized/extensibility tables promoted from the earlier domain recommendations: plan_feature, payment_allocation, repair_status_history, customer_address, product_image, product_translation, market_terminology, and tax_rule.

Enums, value objects, domain events, DTOs, EF Core owned types, and JSON configuration structures are not separate tables unless explicitly listed below.

# 2. Master Table Inventory

| \#  | Table                 | Area                        | Phase |
|-----|-----------------------|-----------------------------|-------|
| 1   | market                | Platform / Localization     | MVP   |
| 2   | tenant                | Platform / Tenancy          | MVP   |
| 3   | branch                | Platform / Tenancy          | MVP   |
| 4   | user                  | Identity / Security         | MVP   |
| 5   | role                  | Identity / Security         | MVP   |
| 6   | permission            | Identity / Security         | MVP   |
| 7   | user_role             | Identity / Security         | MVP   |
| 8   | role_permission       | Identity / Security         | MVP   |
| 9   | plan                  | Subscription / SaaS         | MVP   |
| 10  | feature               | Subscription / SaaS         | MVP   |
| 11  | plan_feature          | Subscription / SaaS         | MVP   |
| 12  | subscription          | Subscription / SaaS         | MVP   |
| 13  | tenant_feature        | Subscription / SaaS         | MVP   |
| 14  | customer              | CRM                         | MVP   |
| 15  | customer_address      | CRM                         | V1+   |
| 16  | supplier              | Purchasing                  | MVP   |
| 17  | purchase_order        | Purchasing                  | MVP   |
| 18  | purchase_order_item   | Purchasing                  | MVP   |
| 19  | purchase              | Purchasing                  | MVP   |
| 20  | purchase_item         | Purchasing                  | MVP   |
| 21  | category              | Catalog                     | MVP   |
| 22  | brand                 | Catalog                     | MVP   |
| 23  | product               | Catalog                     | MVP   |
| 24  | product_translation   | Catalog                     | MVP   |
| 25  | product_image         | Catalog                     | MVP   |
| 26  | gold_type             | Jewelry Reference           | MVP   |
| 27  | purity                | Jewelry Reference           | MVP   |
| 28  | stone                 | Jewelry Reference           | MVP   |
| 29  | product_variant       | Jewelry Reference           | MVP   |
| 30  | gold_price            | Pricing                     | MVP   |
| 31  | pricing_rule          | Pricing                     | MVP   |
| 32  | market_terminology    | Localization / Compliance   | MVP   |
| 33  | tax_rule              | Localization / Compliance   | MVP   |
| 34  | inventory_item        | Inventory                   | MVP   |
| 35  | inventory_transaction | Inventory                   | MVP   |
| 36  | stock_transfer        | Inventory                   | MVP   |
| 37  | stock_transfer_item   | Inventory                   | MVP   |
| 38  | sale                  | Sales & POS                 | MVP   |
| 39  | sale_item             | Sales & POS                 | MVP   |
| 40  | payment               | Sales & POS                 | MVP   |
| 41  | payment_allocation    | Sales & POS                 | V1+   |
| 42  | return                | Sales & POS                 | MVP   |
| 43  | return_item           | Sales & POS                 | MVP   |
| 44  | repair                | Repairs & Services          | MVP   |
| 45  | repair_item           | Repairs & Services          | MVP   |
| 46  | repair_status_history | Repairs & Services          | V1+   |
| 47  | gold_buyback          | Gold / Pricing Transactions | MVP   |
| 48  | exchange              | Gold / Pricing Transactions | MVP   |
| 49  | exchange_item         | Gold / Pricing Transactions | MVP   |
| 50  | account               | Accounting & Finance        | V1+   |
| 51  | account_transaction   | Accounting & Finance        | V1+   |
| 52  | audit_log             | Audit / Compliance          | MVP   |

# 3. Database-Wide Conventions

- Physical table and column names use snake_case. C# entity/property names use PascalCase.

- Primary keys use uuid. Time values use timestamptz and are stored consistently in UTC at the persistence boundary.

- Money uses numeric(19,4). Gold prices use numeric(19,6). Physical weights/quantities use numeric(18,6).

- All tenant-owned tables carry tenant_id directly except tightly controlled aggregate children where tenant ownership is safely inherited through the aggregate root.

- Historical sales, purchases, inventory movements, buybacks, exchanges, audit logs, and accounting postings are preserved; normal hard deletion is prohibited for historical records.

- Financial and inventory operations are transactionally atomic. Inventory changes are represented through inventory_transaction rows rather than hidden quantity mutations.

- Business statuses are stored with string/enum-backed values and validated in the domain/application layer. Status transition rules are not delegated solely to the database.

- JSONB is used only for intentionally extensible/configurable data (custom fields, validated rule configuration, market settings, snapshots); it is not a replacement for core relational entities.

- High-risk tenant relationships should use composite FK/unique strategies where they materially strengthen isolation.

- Indexes should always begin with tenant_id for tenant-scoped high-volume lookups, followed by the actual business lookup columns.

# 4. Final Table Definitions

## 1. market

*Area: Platform / Localization \| Phase: MVP*

Platform-level market defaults for country, language, currency, timezone, and market settings.

| **Column**    | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                    |
|---------------|---------------------|-----------------------|--------------------------------|
| market_id     | uuid                | PK                    | Market identifier              |
| name          | varchar(100)        | NOT NULL              | Market name                    |
| country_code  | char(2)             | NOT NULL              | ISO country code               |
| language_code | varchar(10)         | NOT NULL              | Default language               |
| currency_code | char(3)             | NOT NULL              | ISO 4217 currency code         |
| timezone      | varchar(64)         | NOT NULL              | IANA timezone                  |
| settings      | jsonb               | NOT NULL DEFAULT {}   | Validated market configuration |
| is_active     | boolean             | NOT NULL DEFAULT true | Whether market is active       |
| created_at    | timestamptz         | NOT NULL              | Creation timestamp             |

**Key constraints:** UNIQUE(country_code)

**Relationships:** 1:N tenant

**Implementation notes:** Reference/configuration table; changes affecting active tenants should be audited.

## 2. tenant

*Area: Platform / Tenancy \| Phase: MVP*

Business/company account using the SaaS platform.

| **Column**       | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                     |
|------------------|---------------------|-----------------------|---------------------------------|
| tenant_id        | uuid                | PK                    | Tenant identifier               |
| name             | varchar(200)        | NOT NULL              | Business name                   |
| code             | varchar(50)         | NOT NULL              | Stable unique tenant code       |
| domain           | varchar(255)        | NULL                  | Optional custom/platform domain |
| market_id        | uuid                | FK, NOT NULL          | Default market                  |
| default_currency | char(3)             | NOT NULL              | Tenant operational currency     |
| timezone         | varchar(64)         | NOT NULL              | Tenant business timezone        |
| status           | varchar(30)         | NOT NULL              | Pending/Active/Suspended/Closed |
| created_at       | timestamptz         | NOT NULL              | Creation timestamp              |
| updated_at       | timestamptz         | NOT NULL              | Last modification               |

**Key constraints:** UNIQUE(code); UNIQUE(domain) WHERE domain IS NOT NULL

**Relationships:** N:1 market; 1:N branch/user/customer/supplier/product/...

**Implementation notes:** Tenant is never normally hard-deleted.

## 3. branch

*Area: Platform / Tenancy \| Phase: MVP*

Physical or operational shop location owned by a tenant.

| **Column** | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                      |
|------------|---------------------|-----------------------|----------------------------------|
| branch_id  | uuid                | PK                    | Branch identifier                |
| tenant_id  | uuid                | FK, NOT NULL          | Owning tenant                    |
| name       | varchar(150)        | NOT NULL              | Branch name                      |
| code       | varchar(50)         | NOT NULL              | Branch code unique within tenant |
| location   | varchar(255)        | NULL                  | Short location label             |
| address    | varchar(500)        | NULL                  | Branch address                   |
| phone      | varchar(50)         | NULL                  | Branch phone                     |
| is_active  | boolean             | NOT NULL DEFAULT true | Branch availability              |
| created_at | timestamptz         | NOT NULL              | Creation timestamp               |
| updated_at | timestamptz         | NOT NULL              | Last modification                |

**Key constraints:** UNIQUE(tenant_id, code)

**Relationships:** N:1 tenant; 1:N operational documents

**Implementation notes:** Do not hard-delete branches with history.

## 4. user

*Area: Identity / Security \| Phase: MVP*

Tenant application user/member.

| **Column**    | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                                                              |
|---------------|---------------------|-----------------------|--------------------------------------------------------------------------|
| user_id       | uuid                | PK                    | User identifier                                                          |
| tenant_id     | uuid                | FK, NOT NULL          | Owning tenant                                                            |
| branch_id     | uuid                | FK, NULL              | Default/primary branch                                                   |
| name          | varchar(150)        | NOT NULL              | Display name                                                             |
| email         | varchar(320)        | NOT NULL              | Login/contact email                                                      |
| password_hash | text                | NULL                  | Local auth credential hash; null when external identity provider is used |
| phone         | varchar(50)         | NULL                  | Phone                                                                    |
| is_active     | boolean             | NOT NULL DEFAULT true | Whether login is active                                                  |
| last_login_at | timestamptz         | NULL                  | Last successful login                                                    |
| created_at    | timestamptz         | NOT NULL              | Creation timestamp                                                       |
| updated_at    | timestamptz         | NOT NULL              | Last modification                                                        |

**Key constraints:** UNIQUE(tenant_id, lower(email))

**Relationships:** N:1 tenant/branch; M:N role

**Implementation notes:** Do not use email as immutable identity key.

## 5. role

*Area: Identity / Security \| Phase: MVP*

Authorization role.

| **Column**  | **PostgreSQL Type** | **Key / Constraints**  | **Meaning**                        |
|-------------|---------------------|------------------------|------------------------------------|
| role_id     | uuid                | PK                     | Role identifier                    |
| tenant_id   | uuid                | FK, NULL               | Tenant owner; null for system role |
| name        | varchar(100)        | NOT NULL               | Role name                          |
| description | varchar(500)        | NULL                   | Description                        |
| is_system   | boolean             | NOT NULL DEFAULT false | System-managed flag                |

**Key constraints:** UNIQUE(tenant_id, lower(name)) (tenant roles)

**Relationships:** M:N user; M:N permission

**Implementation notes:** System/platform roles need explicit authorization handling.

## 6. permission

*Area: Identity / Security \| Phase: MVP*

Fine-grained permission capability.

| **Column**    | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                                      |
|---------------|---------------------|-----------------------|--------------------------------------------------|
| permission_id | uuid                | PK                    | Permission identifier                            |
| name          | varchar(150)        | NOT NULL              | Stable permission key                            |
| module        | varchar(100)        | NOT NULL              | Module grouping                                  |
| action        | varchar(100)        | NOT NULL              | Action such as view/create/update/approve/refund |
| description   | varchar(500)        | NULL                  | Description                                      |
| is_system     | boolean             | NOT NULL DEFAULT true | System-managed                                   |

**Key constraints:** UNIQUE(name)

**Relationships:** M:N role

**Implementation notes:** Reference/configuration data.

## 7. user_role

*Area: Identity / Security \| Phase: MVP*

User-to-role assignment.

| **Column** | **PostgreSQL Type** | **Key / Constraints** | **Meaning**    |
|------------|---------------------|-----------------------|----------------|
| user_id    | uuid                | PK/FK                 | User reference |
| role_id    | uuid                | PK/FK                 | Role reference |

**Key constraints:** PRIMARY KEY(user_id, role_id)

**Relationships:** N:1 user; N:1 role

**Implementation notes:** No payload beyond assignment unless later effective dating is required.

## 8. role_permission

*Area: Identity / Security \| Phase: MVP*

Role-to-permission assignment.

| **Column**    | **PostgreSQL Type** | **Key / Constraints** | **Meaning**          |
|---------------|---------------------|-----------------------|----------------------|
| role_id       | uuid                | PK/FK                 | Role reference       |
| permission_id | uuid                | PK/FK                 | Permission reference |

**Key constraints:** PRIMARY KEY(role_id, permission_id)

**Relationships:** N:1 role; N:1 permission

**Implementation notes:** Permissions are assigned through roles.

## 9. plan

*Area: Subscription / SaaS \| Phase: MVP*

Commercial subscription plan definition.

| **Column**    | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                                                                           |
|---------------|---------------------|-----------------------|---------------------------------------------------------------------------------------|
| plan_id       | uuid                | PK                    | Plan identifier                                                                       |
| name          | varchar(100)        | NOT NULL              | Plan name                                                                             |
| description   | varchar(500)        | NULL                  | Description                                                                           |
| price         | numeric(19,4)       | NOT NULL              | Recurring plan price                                                                  |
| billing_cycle | varchar(30)         | NOT NULL              | Monthly/yearly/etc.                                                                   |
| features      | jsonb               | NOT NULL DEFAULT {}   | Optional display/snapshot configuration; canonical entitlements are plan_feature rows |
| is_active     | boolean             | NOT NULL DEFAULT true | Whether plan is available                                                             |

**Key constraints:** UNIQUE(lower(name)); CHECK(price \>= 0)

**Relationships:** 1:N subscription; M:N feature via plan_feature

**Implementation notes:** JSON is not the authoritative entitlement model.

## 10. feature

*Area: Subscription / SaaS \| Phase: MVP*

Catalog of platform features and add-ons.

| **Column**  | **PostgreSQL Type** | **Key / Constraints** | **Meaning**             |
|-------------|---------------------|-----------------------|-------------------------|
| feature_id  | uuid                | PK                    | Feature identifier      |
| name        | varchar(120)        | NOT NULL              | Stable feature key/name |
| description | varchar(500)        | NULL                  | Description             |
| module      | varchar(100)        | NOT NULL              | Functional module       |
| is_active   | boolean             | NOT NULL DEFAULT true | Availability            |

**Key constraints:** UNIQUE(lower(name))

**Relationships:** M:N plan; 1:N tenant_feature

**Implementation notes:** Feature keys should be stable once in production.

## 11. plan_feature

*Area: Subscription / SaaS \| Phase: MVP*

Normalized mapping of features included in a plan.

| **Column**  | **PostgreSQL Type** | **Key / Constraints** | **Meaning**               |
|-------------|---------------------|-----------------------|---------------------------|
| plan_id     | uuid                | PK/FK                 | Plan reference            |
| feature_id  | uuid                | PK/FK                 | Feature reference         |
| included    | boolean             | NOT NULL DEFAULT true | Whether included          |
| limit_value | integer             | NULL                  | Optional plan-level limit |

**Key constraints:** PRIMARY KEY(plan_id, feature_id); CHECK(limit_value IS NULL OR limit_value \>= 0)

**Relationships:** N:1 plan; N:1 feature

**Implementation notes:** Replaces large plan feature lists as authoritative data.

## 12. subscription

*Area: Subscription / SaaS \| Phase: MVP*

Historical subscription period for a tenant.

| **Column**      | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                                        |
|-----------------|---------------------|-----------------------|----------------------------------------------------|
| subscription_id | uuid                | PK                    | Subscription identifier                            |
| tenant_id       | uuid                | FK, NOT NULL          | Tenant reference                                   |
| plan_id         | uuid                | FK, NOT NULL          | Plan reference                                     |
| start_date      | date                | NOT NULL              | Start date                                         |
| end_date        | date                | NULL                  | End date                                           |
| status          | varchar(30)         | NOT NULL              | Subscription status                                |
| features        | jsonb               | NULL                  | Optional entitlement snapshot at subscription time |
| usage_limits    | jsonb               | NULL                  | Optional usage snapshot/configuration              |
| created_at      | timestamptz         | NOT NULL              | Creation timestamp                                 |

**Key constraints:** CHECK(end_date IS NULL OR end_date \>= start_date)

**Relationships:** N:1 tenant/plan

**Implementation notes:** Preserve subscription history; prevent overlapping active periods at application/db level.

## 13. tenant_feature

*Area: Subscription / SaaS \| Phase: MVP*

Tenant-specific feature activation and limits.

| **Column**        | **PostgreSQL Type** | **Key / Constraints** | **Meaning**           |
|-------------------|---------------------|-----------------------|-----------------------|
| tenant_feature_id | uuid                | PK                    | Assignment identifier |
| tenant_id         | uuid                | FK, NOT NULL          | Tenant reference      |
| feature_id        | uuid                | FK, NOT NULL          | Feature reference     |
| is_enabled        | boolean             | NOT NULL              | Effective activation  |
| limit_value       | integer             | NULL                  | Tenant-specific limit |
| created_at        | timestamptz         | NOT NULL              | Creation timestamp    |
| updated_at        | timestamptz         | NOT NULL              | Last modification     |

**Key constraints:** UNIQUE(tenant_id, feature_id); CHECK(limit_value IS NULL OR limit_value \>= 0)

**Relationships:** N:1 tenant/feature

**Implementation notes:** Effective feature state should be resolved through subscription + overrides.

## 14. customer

*Area: CRM \| Phase: MVP*

Tenant-owned customer/business profile.

| **Column**    | **PostgreSQL Type** | **Key / Constraints** | **Meaning**            |
|---------------|---------------------|-----------------------|------------------------|
| customer_id   | uuid                | PK                    | Customer identifier    |
| tenant_id     | uuid                | FK, NOT NULL          | Owning tenant          |
| name          | varchar(200)        | NOT NULL              | Customer name          |
| phone         | varchar(50)         | NULL                  | Phone                  |
| email         | varchar(320)        | NULL                  | Email                  |
| customer_type | varchar(30)         | NOT NULL              | Individual/Company     |
| address       | varchar(500)        | NULL                  | Default/simple address |
| notes         | text                | NULL                  | Notes                  |
| is_active     | boolean             | NOT NULL DEFAULT true | Active flag            |
| created_at    | timestamptz         | NOT NULL              | Creation timestamp     |
| updated_at    | timestamptz         | NOT NULL              | Last modification      |

**Key constraints:** CHECK(customer_type IN configured values)

**Relationships:** N:1 tenant; 1:N sale/repair/buyback/exchange/customer_address

**Implementation notes:** Phone/email not globally unique.

## 15. customer_address

*Area: CRM \| Phase: V1+*

Multiple customer addresses when the business needs them.

| **Column**          | **PostgreSQL Type** | **Key / Constraints**  | **Meaning**            |
|---------------------|---------------------|------------------------|------------------------|
| customer_address_id | uuid                | PK                     | Address identifier     |
| tenant_id           | uuid                | FK, NOT NULL           | Owning tenant          |
| customer_id         | uuid                | FK, NOT NULL           | Customer reference     |
| label               | varchar(50)         | NULL                   | Home/work/billing/etc. |
| address_line1       | varchar(250)        | NOT NULL               | Address line           |
| address_line2       | varchar(250)        | NULL                   | Additional line        |
| city                | varchar(100)        | NULL                   | City                   |
| state_region        | varchar(100)        | NULL                   | Region/state           |
| postal_code         | varchar(30)         | NULL                   | Postal code            |
| country_code        | char(2)             | NULL                   | Country                |
| is_default          | boolean             | NOT NULL DEFAULT false | Default address        |
| created_at          | timestamptz         | NOT NULL               | Creation timestamp     |
| updated_at          | timestamptz         | NOT NULL               | Last modification      |

**Key constraints:** INDEX(tenant_id, customer_id)

**Relationships:** N:1 tenant/customer

**Implementation notes:** Promote from extension to final table because the platform is multi-market.

## 16. supplier

*Area: Purchasing \| Phase: MVP*

Tenant-owned supplier.

| **Column**     | **PostgreSQL Type** | **Key / Constraints** | **Meaning**         |
|----------------|---------------------|-----------------------|---------------------|
| supplier_id    | uuid                | PK                    | Supplier identifier |
| tenant_id      | uuid                | FK, NOT NULL          | Owning tenant       |
| name           | varchar(200)        | NOT NULL              | Supplier name       |
| contact_person | varchar(150)        | NULL                  | Primary contact     |
| phone          | varchar(50)         | NULL                  | Phone               |
| email          | varchar(320)        | NULL                  | Email               |
| address        | varchar(500)        | NULL                  | Address             |
| is_active      | boolean             | NOT NULL DEFAULT true | Active flag         |
| created_at     | timestamptz         | NOT NULL              | Creation timestamp  |
| updated_at     | timestamptz         | NOT NULL              | Last modification   |

**Key constraints:** INDEX(tenant_id, name)

**Relationships:** N:1 tenant; 1:N purchase/purchase_order

**Implementation notes:** Preserve suppliers referenced by historical documents.

## 17. category

*Area: Catalog \| Phase: MVP*

Hierarchical product category.

| **Column**         | **PostgreSQL Type** | **Key / Constraints** | **Meaning**              |
|--------------------|---------------------|-----------------------|--------------------------|
| category_id        | uuid                | PK                    | Category identifier      |
| tenant_id          | uuid                | FK, NOT NULL          | Owning tenant            |
| name               | varchar(150)        | NOT NULL              | Category name            |
| description        | varchar(500)        | NULL                  | Description              |
| parent_category_id | uuid                | FK, NULL              | Optional parent category |
| is_active          | boolean             | NOT NULL DEFAULT true | Active flag              |

**Key constraints:** UNIQUE(tenant_id, lower(name), parent_category_id)

**Relationships:** N:1 tenant/parent; 1:N product

**Implementation notes:** Prevent category cycles in application/domain logic.

## 18. brand

*Area: Catalog \| Phase: MVP*

Optional product brand/designer.

| **Column**  | **PostgreSQL Type** | **Key / Constraints** | **Meaning**      |
|-------------|---------------------|-----------------------|------------------|
| brand_id    | uuid                | PK                    | Brand identifier |
| tenant_id   | uuid                | FK, NOT NULL          | Owning tenant    |
| name        | varchar(150)        | NOT NULL              | Brand name       |
| description | varchar(500)        | NULL                  | Description      |
| is_active   | boolean             | NOT NULL DEFAULT true | Active flag      |

**Key constraints:** UNIQUE(tenant_id, lower(name))

**Relationships:** N:1 tenant; 1:N product

**Implementation notes:** Optional reference data.

## 19. product

*Area: Catalog \| Phase: MVP*

Commercial product definition shared by variants.

| **Column**    | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                          |
|---------------|---------------------|-----------------------|--------------------------------------|
| product_id    | uuid                | PK                    | Product identifier                   |
| tenant_id     | uuid                | FK, NOT NULL          | Owning tenant                        |
| category_id   | uuid                | FK, NOT NULL          | Category reference                   |
| brand_id      | uuid                | FK, NULL              | Optional brand                       |
| sku           | varchar(80)         | NOT NULL              | Tenant-unique product SKU            |
| name          | varchar(200)        | NOT NULL              | Product name                         |
| description   | text                | NULL                  | Description                          |
| image_url     | text                | NULL                  | Primary image/legacy field           |
| custom_fields | jsonb               | NULL                  | Validated tenant-specific attributes |
| is_active     | boolean             | NOT NULL DEFAULT true | Active flag                          |
| created_at    | timestamptz         | NOT NULL              | Creation timestamp                   |
| updated_at    | timestamptz         | NOT NULL              | Last modification                    |

**Key constraints:** UNIQUE(tenant_id, sku)

**Relationships:** N:1 tenant/category/brand; 1:N product_variant; 1:N product_image/translation

**Implementation notes:** SKU belongs to tenant.

## 20. product_translation

*Area: Catalog \| Phase: MVP*

Localized product name/description.

| **Column**             | **PostgreSQL Type** | **Key / Constraints** | **Meaning**              |
|------------------------|---------------------|-----------------------|--------------------------|
| product_translation_id | uuid                | PK                    | Translation identifier   |
| tenant_id              | uuid                | FK, NOT NULL          | Owning tenant            |
| product_id             | uuid                | FK, NOT NULL          | Product reference        |
| language_code          | varchar(10)         | NOT NULL              | BCP-47/ISO language code |
| name                   | varchar(200)        | NULL                  | Localized name           |
| description            | text                | NULL                  | Localized description    |

**Key constraints:** UNIQUE(product_id, language_code)

**Relationships:** N:1 product

**Implementation notes:** Supports Arabic/English and market-specific language presentation.

## 21. product_image

*Area: Catalog \| Phase: MVP*

Multiple ordered product images.

| **Column**       | **PostgreSQL Type** | **Key / Constraints**  | **Meaning**                                  |
|------------------|---------------------|------------------------|----------------------------------------------|
| product_image_id | uuid                | PK                     | Image identifier                             |
| tenant_id        | uuid                | FK, NOT NULL           | Owning tenant                                |
| product_id       | uuid                | FK, NOT NULL           | Product reference                            |
| storage_key      | text                | NOT NULL               | Object-storage key or stable asset reference |
| alt_text         | varchar(250)        | NULL                   | Accessibility text                           |
| sort_order       | integer             | NOT NULL DEFAULT 0     | Display order                                |
| is_primary       | boolean             | NOT NULL DEFAULT false | Primary image flag                           |
| created_at       | timestamptz         | NOT NULL               | Creation timestamp                           |

**Key constraints:** UNIQUE(product_id, sort_order)

**Relationships:** N:1 product

**Implementation notes:** Store files outside PostgreSQL; DB stores metadata.

## 22. gold_type

*Area: Jewelry Reference \| Phase: MVP*

Gold/material family.

| **Column**   | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                 |
|--------------|---------------------|-----------------------|-----------------------------|
| gold_type_id | uuid                | PK                    | Gold type identifier        |
| name         | varchar(100)        | NOT NULL              | Gold/material name          |
| unit         | varchar(20)         | NOT NULL              | Default unit, normally gram |
| is_active    | boolean             | NOT NULL DEFAULT true | Active flag                 |

**Key constraints:** UNIQUE(lower(name))

**Relationships:** 1:N product_variant/gold_price/gold_buyback

**Implementation notes:** Reference data; later tenant-specific extensions are possible.

## 23. purity

*Area: Jewelry Reference \| Phase: MVP*

Gold purity/karat definition.

| **Column** | **PostgreSQL Type** | **Key / Constraints** | **Meaning**              |
|------------|---------------------|-----------------------|--------------------------|
| purity_id  | uuid                | PK                    | Purity identifier        |
| name       | varchar(50)         | NOT NULL              | Display name such as 21K |
| karat      | numeric(6,2)        | NOT NULL              | Karat representation     |
| fineness   | numeric(8,5)        | NULL                  | Optional fineness ratio  |
| is_active  | boolean             | NOT NULL DEFAULT true | Active flag              |

**Key constraints:** UNIQUE(lower(name)); UNIQUE(karat)

**Relationships:** 1:N product_variant/gold_price/gold_buyback

**Implementation notes:** Use one interpretation consistently across markets.

## 24. stone

*Area: Jewelry Reference \| Phase: MVP*

Stone/gem/diamond type.

| **Column** | **PostgreSQL Type** | **Key / Constraints** | **Meaning**       |
|------------|---------------------|-----------------------|-------------------|
| stone_id   | uuid                | PK                    | Stone identifier  |
| name       | varchar(100)        | NOT NULL              | Stone name        |
| type       | varchar(50)         | NOT NULL              | Diamond/ruby/etc. |
| unit       | varchar(20)         | NOT NULL              | Carat/piece/etc.  |
| is_active  | boolean             | NOT NULL DEFAULT true | Active flag       |

**Key constraints:** UNIQUE(lower(name), lower(type))

**Relationships:** 1:N product_variant

**Implementation notes:** Can later be extended with quality/certificate reference data.

## 25. product_variant

*Area: Jewelry Reference \| Phase: MVP*

Variant describing shared jewelry characteristics.

| **Column**         | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                          |
|--------------------|---------------------|-----------------------|--------------------------------------|
| product_variant_id | uuid                | PK                    | Variant identifier                   |
| product_id         | uuid                | FK, NOT NULL          | Parent product                       |
| gold_type_id       | uuid                | FK, NULL              | Gold/material                        |
| purity_id          | uuid                | FK, NULL              | Purity/karat                         |
| stone_id           | uuid                | FK, NULL              | Primary stone type                   |
| sku_suffix         | varchar(50)         | NULL                  | Variant-specific SKU suffix          |
| weight             | numeric(18,6)       | NULL                  | Default/shared weight                |
| stone_weight       | numeric(18,6)       | NULL                  | Default/shared stone weight          |
| default_price      | numeric(19,4)       | NULL                  | Default/static price when applicable |
| attributes         | jsonb               | NULL                  | Validated variant attributes         |
| is_active          | boolean             | NOT NULL DEFAULT true | Active flag                          |
| created_at         | timestamptz         | NOT NULL              | Creation timestamp                   |

**Key constraints:** CHECK(nonnegative numeric fields)

**Relationships:** N:1 product/gold_type/purity/stone; 1:N inventory_item

**Implementation notes:** Physical-item-specific values belong on inventory_item.

## 26. gold_price

*Area: Pricing \| Phase: MVP*

Historical gold price per gram by market/type/purity.

| **Column**     | **PostgreSQL Type** | **Key / Constraints** | **Meaning**             |
|----------------|---------------------|-----------------------|-------------------------|
| gold_price_id  | uuid                | PK                    | Price record identifier |
| market_id      | uuid                | FK, NOT NULL          | Market reference        |
| gold_type_id   | uuid                | FK, NOT NULL          | Gold/material type      |
| purity_id      | uuid                | FK, NOT NULL          | Purity                  |
| price_per_gram | numeric(19,6)       | NOT NULL              | Price per gram          |
| effective_date | timestamptz         | NOT NULL              | Effective start         |
| end_date       | timestamptz         | NULL                  | Effective end           |
| source         | varchar(100)        | NULL                  | Source/provider/manual  |
| is_active      | boolean             | NOT NULL DEFAULT true | Active record           |

**Key constraints:** CHECK(price_per_gram \>= 0); CHECK(end_date IS NULL OR end_date \> effective_date)

**Relationships:** N:1 market/gold_type/purity

**Implementation notes:** Append-oriented history; corrections should preserve auditability.

## 27. pricing_rule

*Area: Pricing \| Phase: MVP*

Tenant-specific configurable pricing formula.

| **Column**      | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                                       |
|-----------------|---------------------|-----------------------|---------------------------------------------------|
| pricing_rule_id | uuid                | PK                    | Rule identifier                                   |
| tenant_id       | uuid                | FK, NOT NULL          | Owning tenant                                     |
| rule_type       | varchar(50)         | NOT NULL              | GoldValue/MakingCharge/Discount/Tax/Rounding/etc. |
| configuration   | jsonb               | NOT NULL              | Validated configuration payload                   |
| effective_date  | timestamptz         | NOT NULL              | Effective start                                   |
| end_date        | timestamptz         | NULL                  | Effective end                                     |
| is_active       | boolean             | NOT NULL DEFAULT true | Active flag                                       |

**Key constraints:** CHECK(end_date IS NULL OR end_date \> effective_date)

**Relationships:** N:1 tenant

**Implementation notes:** Application/domain must validate the JSON schema by rule type.

## 28. market_terminology

*Area: Localization / Compliance \| Phase: MVP*

Market-specific terminology labels without changing domain meaning.

| **Column**            | **PostgreSQL Type** | **Key / Constraints** | **Meaning**            |
|-----------------------|---------------------|-----------------------|------------------------|
| market_terminology_id | uuid                | PK                    | Terminology identifier |
| market_id             | uuid                | FK, NOT NULL          | Market reference       |
| language_code         | varchar(10)         | NOT NULL              | Language               |
| term_key              | varchar(150)        | NOT NULL              | Stable semantic key    |
| term_value            | varchar(250)        | NOT NULL              | Localized term         |
| created_at            | timestamptz         | NOT NULL              | Creation timestamp     |
| updated_at            | timestamptz         | NOT NULL              | Last modification      |

**Key constraints:** UNIQUE(market_id, language_code, term_key)

**Relationships:** N:1 market

**Implementation notes:** Presentation/localization data only; never use to change the domain model.

## 29. tax_rule

*Area: Localization / Compliance \| Phase: MVP*

Market default or tenant override for tax calculation.

| **Column**     | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                             |
|----------------|---------------------|-----------------------|-----------------------------------------|
| tax_rule_id    | uuid                | PK                    | Tax rule identifier                     |
| market_id      | uuid                | FK, NOT NULL          | Market reference                        |
| tenant_id      | uuid                | FK, NULL              | Optional tenant override                |
| code           | varchar(80)         | NOT NULL              | Stable tax rule code                    |
| name           | varchar(150)        | NOT NULL              | Display name                            |
| rate           | numeric(9,6)        | NOT NULL              | Tax rate representation                 |
| rule_type      | varchar(50)         | NOT NULL              | Standard/reduced/exempt/etc.            |
| configuration  | jsonb               | NULL                  | Validated exemption/category conditions |
| effective_date | timestamptz         | NOT NULL              | Effective start                         |
| end_date       | timestamptz         | NULL                  | Effective end                           |
| is_active      | boolean             | NOT NULL DEFAULT true | Active flag                             |

**Key constraints:** CHECK(rate \>= 0); CHECK(end_date IS NULL OR end_date \> effective_date)

**Relationships:** N:1 market; optional N:1 tenant

**Implementation notes:** Use market defaults plus tenant overrides; validate overlaps and precedence.

## 30. inventory_item

*Area: Inventory \| Phase: MVP*

Traceable physical jewelry piece or stock unit.

| **Column**         | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                         |
|--------------------|---------------------|-----------------------|-------------------------------------|
| inventory_item_id  | uuid                | PK                    | Physical item identifier            |
| tenant_id          | uuid                | FK, NOT NULL          | Owning tenant                       |
| branch_id          | uuid                | FK, NOT NULL          | Current branch                      |
| product_variant_id | uuid                | FK, NOT NULL          | Variant reference                   |
| serial_number      | varchar(100)        | NULL                  | Optional serial identifier          |
| barcode            | varchar(120)        | NOT NULL              | Scannable barcode                   |
| gross_weight       | numeric(18,6)       | NULL                  | Measured gross weight               |
| gold_weight        | numeric(18,6)       | NULL                  | Gold weight                         |
| stone_weight       | numeric(18,6)       | NULL                  | Stone weight                        |
| acquisition_cost   | numeric(19,4)       | NULL                  | Acquisition cost                    |
| current_value      | numeric(19,4)       | NULL                  | Current/valuation snapshot          |
| status             | varchar(30)         | NOT NULL              | InStock/Reserved/Sold/InRepair/etc. |
| location           | varchar(120)        | NULL                  | Shelf/vault/location                |
| created_at         | timestamptz         | NOT NULL              | Creation timestamp                  |
| updated_at         | timestamptz         | NOT NULL              | Last modification                   |
| notes              | text                | NULL                  | Notes                               |

**Key constraints:** UNIQUE(tenant_id, barcode); nonnegative weight checks; gold_weight \<= gross_weight when both set

**Relationships:** N:1 tenant/branch/product_variant; 1:N inventory_transaction; references from sale/repair/exchange/transfer

**Implementation notes:** Physical item is the primary traceability unit for high-value jewelry.

## 31. inventory_transaction

*Area: Inventory \| Phase: MVP*

Append-only inventory movement ledger.

| **Column**               | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                        |
|--------------------------|---------------------|-----------------------|------------------------------------|
| inventory_transaction_id | uuid                | PK                    | Transaction identifier             |
| tenant_id                | uuid                | FK, NOT NULL          | Owning tenant                      |
| branch_id                | uuid                | FK, NOT NULL          | Branch at transaction time         |
| inventory_item_id        | uuid                | FK, NOT NULL          | Physical item                      |
| transaction_type         | varchar(50)         | NOT NULL              | Purchase/Sale/Return/Transfer/etc. |
| quantity                 | numeric(18,6)       | NOT NULL              | Quantity moved                     |
| weight                   | numeric(18,6)       | NULL                  | Weight affected                    |
| unit_cost                | numeric(19,4)       | NULL                  | Unit cost when applicable          |
| reference_type           | varchar(50)         | NULL                  | Source document type               |
| reference_id             | uuid                | NULL                  | Source document id                 |
| transaction_date         | timestamptz         | NOT NULL              | Business transaction timestamp     |
| user_id                  | uuid                | FK, NULL              | Actor                              |
| notes                    | text                | NULL                  | Notes                              |

**Key constraints:** INDEX(tenant_id, branch_id, transaction_date); append-oriented

**Relationships:** N:1 tenant/branch/item/user

**Implementation notes:** Never update history to “fix” stock; use a compensating transaction.

## 32. stock_transfer

*Area: Inventory \| Phase: MVP*

Branch-to-branch transfer header and lifecycle.

| **Column**        | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                                             |
|-------------------|---------------------|-----------------------|---------------------------------------------------------|
| stock_transfer_id | uuid                | PK                    | Transfer identifier                                     |
| tenant_id         | uuid                | FK, NOT NULL          | Owning tenant                                           |
| from_branch_id    | uuid                | FK, NOT NULL          | Source branch                                           |
| to_branch_id      | uuid                | FK, NOT NULL          | Destination branch                                      |
| user_id           | uuid                | FK, NOT NULL          | Initiator/owner                                         |
| transfer_date     | timestamptz         | NOT NULL              | Transfer creation date                                  |
| status            | varchar(30)         | NOT NULL              | Requested/Approved/Shipped/Received/Completed/Cancelled |
| requested_at      | timestamptz         | NULL                  | Requested timestamp                                     |
| approved_at       | timestamptz         | NULL                  | Approval timestamp                                      |
| shipped_at        | timestamptz         | NULL                  | Shipment timestamp                                      |
| received_at       | timestamptz         | NULL                  | Receipt timestamp                                       |
| notes             | text                | NULL                  | Notes                                                   |

**Key constraints:** CHECK(from_branch_id \<\> to_branch_id)

**Relationships:** N:1 tenant/branches/user; 1:N stock_transfer_item

**Implementation notes:** Transfer inventory through explicit lifecycle.

## 33. stock_transfer_item

*Area: Inventory \| Phase: MVP*

Physical item lines belonging to a stock transfer.

| **Column**             | **PostgreSQL Type** | **Key / Constraints** | **Meaning**     |
|------------------------|---------------------|-----------------------|-----------------|
| stock_transfer_item_id | uuid                | PK                    | Line identifier |
| stock_transfer_id      | uuid                | FK, NOT NULL          | Transfer header |
| inventory_item_id      | uuid                | FK, NOT NULL          | Physical item   |
| quantity               | numeric(18,6)       | NOT NULL              | Quantity moved  |
| weight                 | numeric(18,6)       | NULL                  | Weight moved    |
| notes                  | text                | NULL                  | Line notes      |

**Key constraints:** UNIQUE(stock_transfer_id, inventory_item_id)

**Relationships:** N:1 stock_transfer/inventory_item

**Implementation notes:** Usually quantity=1 for unique serialized jewelry.

## 34. purchase_order

*Area: Purchasing \| Phase: MVP*

Optional procurement order before receipt.

| **Column**        | **PostgreSQL Type** | **Key / Constraints** | **Meaning**         |
|-------------------|---------------------|-----------------------|---------------------|
| purchase_order_id | uuid                | PK                    | PO identifier       |
| tenant_id         | uuid                | FK, NOT NULL          | Owning tenant       |
| supplier_id       | uuid                | FK, NOT NULL          | Supplier            |
| order_date        | timestamptz         | NOT NULL              | Order timestamp     |
| expected_date     | timestamptz         | NULL                  | Expected receipt    |
| status            | varchar(30)         | NOT NULL              | PO lifecycle status |
| total_amount      | numeric(19,4)       | NOT NULL DEFAULT 0    | Order total         |
| notes             | text                | NULL                  | Notes               |

**Key constraints:** CHECK(total_amount \>= 0)

**Relationships:** N:1 tenant/supplier; 1:N items; 0..1:N purchase

**Implementation notes:** Order may remain unreceived or partially received.

## 35. purchase_order_item

*Area: Purchasing \| Phase: MVP*

Purchase order line.

| **Column**             | **PostgreSQL Type** | **Key / Constraints** | **Meaning**            |
|------------------------|---------------------|-----------------------|------------------------|
| purchase_order_item_id | uuid                | PK                    | PO line identifier     |
| purchase_order_id      | uuid                | FK, NOT NULL          | PO reference           |
| product_variant_id     | uuid                | FK, NOT NULL          | Ordered variant        |
| quantity               | numeric(18,6)       | NOT NULL              | Ordered quantity       |
| unit_price             | numeric(19,4)       | NOT NULL              | Expected unit price    |
| total_price            | numeric(19,4)       | NOT NULL              | Line total             |
| weight                 | numeric(18,6)       | NULL                  | Expected/shared weight |

**Key constraints:** CHECK(quantity \> 0); CHECK(unit_price \>= 0); CHECK(total_price \>= 0)

**Relationships:** N:1 purchase_order/product_variant

**Implementation notes:** Physical receipt details belong to purchase/purchase_item/inventory_item.

## 36. purchase

*Area: Purchasing \| Phase: MVP*

Supplier purchase receipt/document.

| **Column**        | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                          |
|-------------------|---------------------|-----------------------|--------------------------------------|
| purchase_id       | uuid                | PK                    | Purchase identifier                  |
| tenant_id         | uuid                | FK, NOT NULL          | Owning tenant                        |
| supplier_id       | uuid                | FK, NOT NULL          | Supplier                             |
| purchase_order_id | uuid                | FK, NULL              | Optional PO                          |
| purchase_date     | timestamptz         | NOT NULL              | Purchase timestamp                   |
| receive_date      | timestamptz         | NULL                  | Inventory receipt timestamp          |
| total_amount      | numeric(19,4)       | NOT NULL              | Purchase total                       |
| status            | varchar(30)         | NOT NULL              | Draft/Received/Posted/Cancelled etc. |
| notes             | text                | NULL                  | Notes                                |

**Key constraints:** CHECK(total_amount \>= 0)

**Relationships:** N:1 tenant/supplier/PO; 1:N purchase_item

**Implementation notes:** Posting creates inventory and accounting effects.

## 37. purchase_item

*Area: Purchasing \| Phase: MVP*

Supplier purchase line and physical item association.

| **Column**         | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                      |
|--------------------|---------------------|-----------------------|----------------------------------|
| purchase_item_id   | uuid                | PK                    | Purchase line identifier         |
| purchase_id        | uuid                | FK, NOT NULL          | Purchase reference               |
| product_variant_id | uuid                | FK, NOT NULL          | Variant reference                |
| inventory_item_id  | uuid                | FK, NULL              | Created/associated physical item |
| quantity           | numeric(18,6)       | NOT NULL              | Received quantity                |
| unit_price         | numeric(19,4)       | NOT NULL              | Actual unit price                |
| total_price        | numeric(19,4)       | NOT NULL              | Line total                       |
| weight             | numeric(18,6)       | NULL                  | Received/shared weight           |

**Key constraints:** CHECK(quantity \> 0); CHECK(unit_price \>= 0); CHECK(total_price \>= 0)

**Relationships:** N:1 purchase/product_variant; optional N:1 inventory_item

**Implementation notes:** Unique physical jewelry may create one inventory item per received piece.

## 38. sale

*Area: Sales & POS \| Phase: MVP*

Sales invoice/transaction header.

| **Column**     | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                             |
|----------------|---------------------|-----------------------|-----------------------------------------|
| sale_id        | uuid                | PK                    | Sale identifier                         |
| tenant_id      | uuid                | FK, NOT NULL          | Owning tenant                           |
| branch_id      | uuid                | FK, NOT NULL          | Selling branch                          |
| customer_id    | uuid                | FK, NULL              | Customer; nullable for anonymous retail |
| user_id        | uuid                | FK, NOT NULL          | Sales user                              |
| sale_number    | varchar(60)         | NOT NULL              | Human-facing invoice number             |
| sale_date      | timestamptz         | NOT NULL              | Sale timestamp                          |
| total_amount   | numeric(19,4)       | NOT NULL              | Pre-discount/pre-tax total              |
| discount       | numeric(19,4)       | NOT NULL DEFAULT 0    | Discount                                |
| tax            | numeric(19,4)       | NOT NULL DEFAULT 0    | Tax                                     |
| final_amount   | numeric(19,4)       | NOT NULL              | Final payable                           |
| payment_status | varchar(30)         | NOT NULL              | Unpaid/PartiallyPaid/Paid/Refunded etc. |
| status         | varchar(30)         | NOT NULL              | Draft/Posted/Cancelled/etc.             |
| notes          | text                | NULL                  | Notes                                   |

**Key constraints:** UNIQUE(tenant_id, sale_number); nonnegative monetary checks

**Relationships:** N:1 tenant/branch/customer/user; 1:N sale_item/payment/return

**Implementation notes:** Totals stored as transaction snapshots; derive/detail-check from lines.

## 39. sale_item

*Area: Sales & POS \| Phase: MVP*

Sale line linked to a variant and optionally a physical inventory item.

| **Column**         | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                         |
|--------------------|---------------------|-----------------------|-------------------------------------|
| sale_item_id       | uuid                | PK                    | Sale line                           |
| sale_id            | uuid                | FK, NOT NULL          | Sale reference                      |
| product_variant_id | uuid                | FK, NOT NULL          | Variant                             |
| inventory_item_id  | uuid                | FK, NULL              | Specific physical item when tracked |
| quantity           | numeric(18,6)       | NOT NULL              | Quantity sold                       |
| unit_price         | numeric(19,4)       | NOT NULL              | Unit selling price snapshot         |
| discount           | numeric(19,4)       | NOT NULL DEFAULT 0    | Line discount                       |
| tax                | numeric(19,4)       | NOT NULL DEFAULT 0    | Line tax                            |
| total_price        | numeric(19,4)       | NOT NULL              | Line total                          |
| weight             | numeric(18,6)       | NULL                  | Weight snapshot                     |
| notes              | text                | NULL                  | Notes                               |

**Key constraints:** CHECK(quantity \> 0); nonnegative monetary checks

**Relationships:** N:1 sale/product_variant; optional N:1 inventory_item

**Implementation notes:** Capture final commercial values at sale time; do not recalculate historical invoices from current prices.

## 40. payment

*Area: Sales & POS \| Phase: MVP*

Payment received against a sale.

| **Column**     | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                            |
|----------------|---------------------|-----------------------|----------------------------------------|
| payment_id     | uuid                | PK                    | Payment identifier                     |
| sale_id        | uuid                | FK, NOT NULL          | Sale reference                         |
| payment_method | varchar(50)         | NOT NULL              | Cash/Card/Transfer/Wallet/etc.         |
| amount         | numeric(19,4)       | NOT NULL              | Payment amount                         |
| reference_no   | varchar(120)        | NULL                  | External/payment reference             |
| payment_date   | timestamptz         | NOT NULL              | Payment timestamp                      |
| status         | varchar(30)         | NOT NULL              | Pending/Completed/Failed/Refunded/etc. |

**Key constraints:** CHECK(amount \> 0)

**Relationships:** N:1 sale

**Implementation notes:** Multiple payments per sale are supported.

## 41. payment_allocation

*Area: Sales & POS \| Phase: V1+*

Optional normalization for applying a payment across one or more financial documents.

| **Column**            | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                   |
|-----------------------|---------------------|-----------------------|-------------------------------|
| payment_allocation_id | uuid                | PK                    | Allocation identifier         |
| payment_id            | uuid                | FK, NOT NULL          | Payment reference             |
| document_type         | varchar(50)         | NOT NULL              | Sale/Purchase/Credit/etc.     |
| document_id           | uuid                | NOT NULL              | Allocated document identifier |
| amount                | numeric(19,4)       | NOT NULL              | Allocated amount              |
| allocated_at          | timestamptz         | NOT NULL              | Allocation timestamp          |

**Key constraints:** CHECK(amount \> 0); UNIQUE(payment_id, document_type, document_id)

**Relationships:** N:1 payment; logical reference to document

**Implementation notes:** Optional extension; generic document reference preserves future finance flexibility.

## 42. return

*Area: Sales & POS \| Phase: MVP*

Return/refund header tied to an original sale.

| **Column**    | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                        |
|---------------|---------------------|-----------------------|------------------------------------|
| return_id     | uuid                | PK                    | Return identifier                  |
| tenant_id     | uuid                | FK, NOT NULL          | Owning tenant                      |
| sale_id       | uuid                | FK, NOT NULL          | Original sale                      |
| user_id       | uuid                | FK, NOT NULL          | Processing user                    |
| return_date   | timestamptz         | NOT NULL              | Return timestamp                   |
| reason        | varchar(500)        | NULL                  | Reason                             |
| status        | varchar(30)         | NOT NULL              | Draft/Approved/Completed/Cancelled |
| refund_amount | numeric(19,4)       | NOT NULL DEFAULT 0    | Refund total                       |
| notes         | text                | NULL                  | Notes                              |

**Key constraints:** CHECK(refund_amount \>= 0)

**Relationships:** N:1 tenant/sale/user; 1:N return_item

**Implementation notes:** Partial returns supported through return_item.

## 43. return_item

*Area: Sales & POS \| Phase: MVP*

Returned sale line and inventory effect reference.

| **Column**        | **PostgreSQL Type** | **Key / Constraints** | **Meaning**            |
|-------------------|---------------------|-----------------------|------------------------|
| return_item_id    | uuid                | PK                    | Return line            |
| return_id         | uuid                | FK, NOT NULL          | Return header          |
| sale_item_id      | uuid                | FK, NOT NULL          | Original sale line     |
| inventory_item_id | uuid                | FK, NULL              | Returned physical item |
| quantity          | numeric(18,6)       | NOT NULL              | Returned quantity      |
| refund_amount     | numeric(19,4)       | NOT NULL              | Refund for this line   |

**Key constraints:** CHECK(quantity \> 0); CHECK(refund_amount \>= 0)

**Relationships:** N:1 return/sale_item; optional N:1 inventory_item

**Implementation notes:** Must not exceed eligible sold quantity.

## 44. repair

*Area: Repairs & Services \| Phase: MVP*

Repair/service order.

| **Column**          | **PostgreSQL Type** | **Key / Constraints** | **Meaning**             |
|---------------------|---------------------|-----------------------|-------------------------|
| repair_id           | uuid                | PK                    | Repair identifier       |
| tenant_id           | uuid                | FK, NOT NULL          | Owning tenant           |
| customer_id         | uuid                | FK, NOT NULL          | Customer                |
| branch_id           | uuid                | FK, NOT NULL          | Handling branch         |
| received_by_user_id | uuid                | FK, NULL              | Receiving user          |
| received_date       | timestamptz         | NOT NULL              | Received timestamp      |
| expected_date       | timestamptz         | NULL                  | Expected completion     |
| completed_date      | timestamptz         | NULL                  | Actual completion       |
| status              | varchar(40)         | NOT NULL              | Repair lifecycle status |
| estimated_cost      | numeric(19,4)       | NULL                  | Estimate                |
| actual_cost         | numeric(19,4)       | NULL                  | Actual cost             |
| notes               | text                | NULL                  | Notes                   |

**Key constraints:** nonnegative cost checks

**Relationships:** N:1 tenant/customer/branch/user; 1:N repair_item/status_history

**Implementation notes:** Repair item may point to the customer’s physical inventory item.

## 45. repair_item

*Area: Repairs & Services \| Phase: MVP*

Repair line describing work on one item.

| **Column**         | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                |
|--------------------|---------------------|-----------------------|----------------------------|
| repair_item_id     | uuid                | PK                    | Repair line                |
| repair_id          | uuid                | FK, NOT NULL          | Repair header              |
| inventory_item_id  | uuid                | FK, NULL              | Physical item when known   |
| product_variant_id | uuid                | FK, NULL              | Variant fallback           |
| description        | text                | NOT NULL              | Repair/service description |
| cost               | numeric(19,4)       | NOT NULL DEFAULT 0    | Line cost                  |
| status             | varchar(30)         | NOT NULL              | Line status                |

**Key constraints:** CHECK(cost \>= 0)

**Relationships:** N:1 repair; optional N:1 inventory_item/product_variant

**Implementation notes:** Do not require product catalog for externally sourced/non-catalog repair items.

## 46. repair_status_history

*Area: Repairs & Services \| Phase: V1+*

Immutable repair status timeline.

| **Column**               | **PostgreSQL Type** | **Key / Constraints** | **Meaning**          |
|--------------------------|---------------------|-----------------------|----------------------|
| repair_status_history_id | uuid                | PK                    | History row          |
| repair_id                | uuid                | FK, NOT NULL          | Repair reference     |
| from_status              | varchar(40)         | NULL                  | Previous status      |
| to_status                | varchar(40)         | NOT NULL              | New status           |
| changed_by_user_id       | uuid                | FK, NULL              | Actor                |
| changed_at               | timestamptz         | NOT NULL              | Change timestamp     |
| notes                    | text                | NULL                  | Optional status note |

**Key constraints:** INDEX(repair_id, changed_at)

**Relationships:** N:1 repair/user

**Implementation notes:** Append-only history; excellent for SLA/service analysis.

## 47. gold_buyback

*Area: Gold / Pricing Transactions \| Phase: MVP*

Customer sale of old gold/jewelry to the shop.

| **Column**        | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                    |
|-------------------|---------------------|-----------------------|--------------------------------|
| buyback_id        | uuid                | PK                    | Buyback identifier             |
| tenant_id         | uuid                | FK, NOT NULL          | Owning tenant                  |
| customer_id       | uuid                | FK, NOT NULL          | Customer                       |
| branch_id         | uuid                | FK, NOT NULL          | Branch                         |
| user_id           | uuid                | FK, NOT NULL          | Processing user                |
| gold_type_id      | uuid                | FK, NULL              | Gold/material                  |
| purity_id         | uuid                | FK, NOT NULL          | Purity                         |
| gross_weight      | numeric(18,6)       | NOT NULL              | Gross weight                   |
| stone_weight      | numeric(18,6)       | NULL                  | Stone weight                   |
| gold_weight       | numeric(18,6)       | NOT NULL              | Gold weight used for valuation |
| gold_price        | numeric(19,6)       | NOT NULL              | Locked price per gram          |
| value             | numeric(19,4)       | NOT NULL              | Buyback value                  |
| payment_method    | varchar(50)         | NOT NULL              | Payment method                 |
| payment_reference | varchar(120)        | NULL                  | Payment reference              |
| status            | varchar(30)         | NOT NULL              | Draft/Completed/Cancelled      |
| transaction_date  | timestamptz         | NOT NULL              | Transaction time               |
| notes             | text                | NULL                  | Notes                          |

**Key constraints:** nonnegative weights/money; gold_weight \<= gross_weight when both set

**Relationships:** N:1 tenant/customer/branch/user/gold_type/purity

**Implementation notes:** Historical price and weight values are snapshots; do not recalculate old buybacks from current price.

## 48. exchange

*Area: Gold / Pricing Transactions \| Phase: MVP*

Exchange/trade-in transaction combining old and new jewelry.

| **Column**       | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                                        |
|------------------|---------------------|-----------------------|----------------------------------------------------|
| exchange_id      | uuid                | PK                    | Exchange identifier                                |
| tenant_id        | uuid                | FK, NOT NULL          | Owning tenant                                      |
| customer_id      | uuid                | FK, NOT NULL          | Customer                                           |
| branch_id        | uuid                | FK, NOT NULL          | Branch                                             |
| user_id          | uuid                | FK, NOT NULL          | Processing user                                    |
| old_item_value   | numeric(19,4)       | NOT NULL              | Trade-in value                                     |
| new_item_value   | numeric(19,4)       | NOT NULL              | New jewelry value                                  |
| net_amount       | numeric(19,4)       | NOT NULL              | Net customer payable/refund depending on direction |
| gold_price       | numeric(19,6)       | NULL                  | Locked gold price when used                        |
| payment_method   | varchar(50)         | NULL                  | Payment method for net amount                      |
| status           | varchar(30)         | NOT NULL              | Draft/Completed/Cancelled                          |
| transaction_date | timestamptz         | NOT NULL              | Transaction time                                   |
| notes            | text                | NULL                  | Notes                                              |

**Key constraints:** nonnegative value checks as applicable

**Relationships:** N:1 tenant/customer/branch/user; 1:N exchange_item

**Implementation notes:** Multi-item exchange supported via exchange_item.

## 49. exchange_item

*Area: Gold / Pricing Transactions \| Phase: MVP*

Exchange line indicating old-in vs new-out direction.

| **Column**         | **PostgreSQL Type** | **Key / Constraints** | **Meaning**             |
|--------------------|---------------------|-----------------------|-------------------------|
| exchange_item_id   | uuid                | PK                    | Exchange line           |
| exchange_id        | uuid                | FK, NOT NULL          | Exchange header         |
| inventory_item_id  | uuid                | FK, NULL              | Physical item reference |
| product_variant_id | uuid                | FK, NULL              | Variant reference       |
| direction          | varchar(20)         | NOT NULL              | IN/OUT or equivalent    |
| quantity           | numeric(18,6)       | NOT NULL              | Quantity                |
| weight             | numeric(18,6)       | NULL                  | Weight                  |
| value              | numeric(19,4)       | NOT NULL              | Line value              |

**Key constraints:** CHECK(quantity \> 0); CHECK(value \>= 0)

**Relationships:** N:1 exchange; optional N:1 inventory_item/product_variant

**Implementation notes:** Direction must be constrained by application/domain enum.

## 50. account

*Area: Accounting & Finance \| Phase: V1+*

Tenant financial account.

| **Column** | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                       |
|------------|---------------------|-----------------------|-----------------------------------|
| account_id | uuid                | PK                    | Account identifier                |
| tenant_id  | uuid                | FK, NOT NULL          | Owning tenant                     |
| name       | varchar(150)        | NOT NULL              | Account name                      |
| code       | varchar(50)         | NOT NULL              | Account code                      |
| type       | varchar(30)         | NOT NULL              | Cash/Bank/Receivable/Payable/etc. |
| balance    | numeric(19,4)       | NOT NULL DEFAULT 0    | Current balance snapshot          |
| is_active  | boolean             | NOT NULL DEFAULT true | Active flag                       |

**Key constraints:** UNIQUE(tenant_id, code); CHECK(balance can be signed)

**Relationships:** N:1 tenant; 1:N account_transaction

**Implementation notes:** Balance is a projection; account_transaction is the audit source.

## 51. account_transaction

*Area: Accounting & Finance \| Phase: V1+*

Accounting movement/posting.

| **Column**             | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                   |
|------------------------|---------------------|-----------------------|-------------------------------|
| account_transaction_id | uuid                | PK                    | Transaction identifier        |
| tenant_id              | uuid                | FK, NOT NULL          | Owning tenant                 |
| account_id             | uuid                | FK, NOT NULL          | Account                       |
| type                   | varchar(30)         | NOT NULL              | Debit/Credit or posting type  |
| amount                 | numeric(19,4)       | NOT NULL              | Posting amount                |
| transaction_date       | timestamptz         | NOT NULL              | Posting time                  |
| reference_id           | uuid                | NULL                  | Source transaction identifier |
| reference_type         | varchar(50)         | NULL                  | Source transaction type       |
| description            | varchar(500)        | NULL                  | Description                   |

**Key constraints:** CHECK(amount \> 0)

**Relationships:** N:1 tenant/account

**Implementation notes:** Accounting rules are intentionally isolated from operational sales/purchase entities.

## 52. audit_log

*Area: Audit / Compliance \| Phase: MVP*

Immutable audit record for important business/security actions.

| **Column**   | **PostgreSQL Type** | **Key / Constraints** | **Meaning**                             |
|--------------|---------------------|-----------------------|-----------------------------------------|
| audit_log_id | uuid                | PK                    | Audit record identifier                 |
| tenant_id    | uuid                | FK, NULL              | Owning tenant; null for platform events |
| user_id      | uuid                | FK, NULL              | Actor                                   |
| entity_name  | varchar(150)        | NOT NULL              | Affected entity type                    |
| entity_id    | uuid                | NULL                  | Affected entity id                      |
| action       | varchar(50)         | NOT NULL              | Action performed                        |
| old_value    | jsonb               | NULL                  | Relevant previous state                 |
| new_value    | jsonb               | NULL                  | Relevant new state                      |
| timestamp    | timestamptz         | NOT NULL              | Event timestamp                         |
| ip_address   | inet                | NULL                  | Source IP                               |
| device_info  | varchar(500)        | NULL                  | Device/client details                   |

**Key constraints:** INDEX(tenant_id, timestamp DESC)

**Relationships:** N:1 tenant/user

**Implementation notes:** Append-only; do not expose raw logs to normal tenant users.

# 5. Relationship Summary

| From            | Cardinality           | To  | Purpose                        |
|-----------------|-----------------------|-----|--------------------------------|
| market          | tenant                | 1:N | Default regional configuration |
| tenant          | branch                | 1:N | Tenant owns branches           |
| tenant          | user                  | 1:N | Tenant owns members            |
| user            | role                  | N:M | Via user_role                  |
| role            | permission            | N:M | Via role_permission            |
| plan            | feature               | N:M | Via plan_feature               |
| tenant          | subscription          | 1:N | Subscription history           |
| tenant          | tenant_feature        | 1:N | Feature overrides              |
| customer        | customer_address      | 1:N | Optional multiple addresses    |
| tenant          | supplier              | 1:N | Suppliers                      |
| tenant          | category              | 1:N | Categories                     |
| tenant          | brand                 | 1:N | Brands                         |
| tenant          | product               | 1:N | Products                       |
| product         | product_variant       | 1:N | Variants                       |
| product         | product_translation   | 1:N | Translations                   |
| product         | product_image         | 1:N | Images                         |
| product_variant | inventory_item        | 1:N | Physical items                 |
| market          | gold_price            | 1:N | Market gold prices             |
| tenant          | pricing_rule          | 1:N | Pricing rules                  |
| market          | market_terminology    | 1:N | Localized terminology          |
| market          | tax_rule              | 1:N | Tax defaults                   |
| tenant          | tax_rule              | 1:N | Optional overrides             |
| inventory_item  | inventory_transaction | 1:N | Stock movement history         |
| stock_transfer  | stock_transfer_item   | 1:N | Transfer lines                 |
| purchase_order  | purchase_order_item   | 1:N | PO lines                       |
| purchase        | purchase_item         | 1:N | Purchase lines                 |
| sale            | sale_item             | 1:N | Sale lines                     |
| sale            | payment               | 1:N | Payments                       |
| payment         | payment_allocation    | 1:N | Optional allocations           |
| return          | return_item           | 1:N | Return lines                   |
| repair          | repair_item           | 1:N | Repair lines                   |
| repair          | repair_status_history | 1:N | Status history                 |
| exchange        | exchange_item         | 1:N | Exchange lines                 |
| account         | account_transaction   | 1:N | Accounting postings            |

# 6. Final Module-to-Table Map

**Platform / Localization**: market

**Platform / Tenancy**: tenant, branch

**Identity / Security**: user, role, permission, user_role, role_permission

**Subscription / SaaS**: plan, feature, plan_feature, subscription, tenant_feature

**CRM**: customer, customer_address

**Purchasing**: supplier, purchase_order, purchase_order_item, purchase, purchase_item

**Catalog**: category, brand, product, product_translation, product_image

**Jewelry Reference**: gold_type, purity, stone, product_variant

**Pricing**: gold_price, pricing_rule

**Localization / Compliance**: market_terminology, tax_rule

**Inventory**: inventory_item, inventory_transaction, stock_transfer, stock_transfer_item

**Sales & POS**: sale, sale_item, payment, payment_allocation, return, return_item

**Repairs & Services**: repair, repair_item, repair_status_history

**Gold / Pricing Transactions**: gold_buyback, exchange, exchange_item

**Accounting & Finance**: account, account_transaction

**Audit / Compliance**: audit_log

# 7. Tables That Are Required Before the First Migration

These should be implemented as part of the first production-quality schema baseline: market, tenant, branch, user, role, permission, user_role, role_permission, plan, feature, plan_feature, subscription, tenant_feature, customer, supplier, category, brand, product, product_translation, product_image, gold_type, purity, stone, product_variant, gold_price, pricing_rule, market_terminology, tax_rule, inventory_item, inventory_transaction, stock_transfer, stock_transfer_item, purchase_order, purchase_order_item, purchase, purchase_item, sale, sale_item, payment, return, return_item, repair, repair_item, gold_buyback, exchange, exchange_item, and audit_log.

Accounting tables (account, account_transaction) and the optional support tables payment_allocation, customer_address, and repair_status_history can be included in the same schema migration set or activated in the first feature release; their structures are fixed here so later adoption does not require redesigning core transactional tables.

# 8. What Is Deliberately Not a Table

- Enums: TenantStatus, CustomerType, InventoryItemStatus, InventoryTransactionType, StockTransferStatus, SaleStatus, PaymentStatus, PurchaseStatus, RepairStatus, BuybackStatus, ExchangeStatus, SubscriptionStatus, BillingCycle, PaymentMethodType, PricingRuleType, AccountType/TransactionType, and related domain enums.

- Value objects: Money, Weight, Quantity, Purity/Karat, Barcode, SKU, DocumentNumber, Address, DateRange, GoldPrice and other immutable domain concepts defined by the domain model.

- Domain events and outbox messages are not part of the core business-table list; if an outbox persistence table is adopted by the infrastructure implementation, it should be treated as a cross-cutting infrastructure table and documented separately.

- Object files/images are not stored as database blobs; PostgreSQL stores asset metadata and storage keys.

# 9. Final Implementation Rules

- Create schema through EF Core migrations only after an architecture review confirms all cross-tenant foreign keys and unique indexes.

- Use private setters/constructors in domain entities and mutate aggregates through domain/application operations.

- Use explicit indexes for barcode, SKU, sale number, tenant+code, time-series ledgers, and common tenant+status lookups.

- Use optimistic concurrency where multiple terminals/users can update the same aggregate; inventory sale/transfer operations must verify current state before commit.

- Historical monetary and weight snapshots on completed documents are immutable. New business changes create new documents or compensating entries.

- All migrations must be backward-aware for deployed environments and must include a rollback/forward-recovery strategy documented in release notes.

- Any addition or removal of a table from this baseline requires an ADR and an update to this specification.

# 10. Recommended Next Step

The next engineering artifact is the EF Core implementation mapping: DbContext boundaries, entity configurations (IEntityTypeConfiguration\<T\>), owned/value-object mappings, global tenant filters where appropriate, indexes/constraints, enum conversions, migration ordering, and the first migration plan. The application should not start generating migrations until that mapping is reviewed against this final table contract.
