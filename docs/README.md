# Jewelry Management System — Engineering Documentation

Version: 1.0 baseline • Updated: 14 October 2026

This folder is the Git-versioned engineering documentation for the Jewelry Management System. It supports the approved plan: a multi-tenant, multi-branch, multi-market jewelry platform using ASP.NET Core/.NET, Entity Framework Core, and PostgreSQL, initially implemented as a modular monolith.

## Documentation map

| Path | Purpose | Source/status |
|---|---|---|
| `architecture/solution-architecture.md` | Solution/project structure, dependencies, runtime and hosting choices | Reconstructed from the previously approved .NET architecture baseline |
| `architecture/module-boundaries.md` | Module ownership and allowed interactions | New implementation guide based on domain model and architecture baseline |
| `architecture/security-architecture.md` | Authentication, authorization, tenant/branch isolation and security controls | Initial implementation guide; review before production |
| `domain/business-rules.md` | Rules identified by stable IDs (GBR, TMS, IAM, CAT, INV, SALE, etc.) | Converted from approved Business Rules Specification DOCX |
| `domain/aggregates.md` | Aggregate roots, owned children, invariants and cross-aggregate workflows | Consolidated from Aggregate Boundaries and Domain Model baselines |
| `domain/domain-model.md` | Bounded contexts, entities, aggregates, value objects and invariants | Converted from approved Domain Model Specification DOCX |
| `domain/enums-value-objects.md` | Enum catalog, value objects, persistence and serialization conventions | Converted from approved Enums & Value Objects Specification DOCX |
| `database/database-schema.md` | Relational design principles, types, keys, constraints, indexes and tenancy | Converted from approved Database Schema Specification DOCX |
| `database/final-tables.md` | Detailed 52-table baseline | Converted from approved Final Database Tables DOCX |
| `database/migrations.md` | EF Core migration authoring, review, deployment and rollback rules | Initial implementation guide |
| `api/api-guidelines.md` | HTTP contract, endpoint, error, pagination and idempotency conventions | Initial implementation guide |
| `api/openapi-notes.md` | OpenAPI document and generated-client conventions | Initial implementation guide |
| `adr/ADR-001-modular-monolith.md` | Why the first release is a modular monolith | Initial ADR reflecting approved architecture decision |
| `adr/ADR-002-multi-tenancy.md` | Tenant isolation model | Initial ADR reflecting approved architecture decision |
| `adr/ADR-003-postgresql.md` | Primary database choice | Initial ADR reflecting approved architecture decision |
| `assets/jewelry-management-system-erd.png` | ERD visual reference | Diagram previously generated for this project |

## Source-of-truth policy

1. These Markdown files are the version-controlled engineering references for the repository.
2. The previous Word documents remain useful review/export artifacts, but implementation changes should be reflected in Markdown through pull requests.
3. If a design decision changes, update the relevant file and add/update an ADR for a material architectural change.
4. `database/final-tables.md` is a proposed 52-table baseline, not an instruction to create all tables in the first migration. Use Jira acceptance criteria and MVP phase labels to sequence implementation.
5. Documents marked “initial implementation guide” need team review. They define a practical starting standard rather than claiming external legal or compliance certification.

## Suggested review order

1. `architecture/solution-architecture.md`
2. `architecture/module-boundaries.md` and `architecture/security-architecture.md`
3. `domain/business-rules.md`, `domain/aggregates.md`, `domain/domain-model.md`
4. `domain/enums-value-objects.md`
5. `database/database-schema.md`, then `database/final-tables.md`, then `database/migrations.md`
6. API guidelines/OpenAPI and ADRs

## How to use

Unzip this package at the repository root so the files land under `docs/`. Review with Git, edit Markdown directly, and submit changes through pull requests. Do not check secrets, credentials, live customer data, or production connection strings into this directory.
