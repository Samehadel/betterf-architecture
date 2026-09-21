# Database Schema

## Overview

PostgreSQL is the relational database used in this application. All schema changes are managed through Liquibase migrations for version control and reversibility.

**Database**: PostgreSQL with Liquibase for schema versioning.
**Configuration**: See `application.properties` for connection details.
**Migrations**: Automatic on application startup from `src/main/resources/db/changelog/`

## Liquibase Migrations

### Directory Structure

```
src/main/resources/db/changelog/
├── db.changelog-master.yaml          # Master changelog (explicitly includes all changesets)
├── 20260101-001-initial-schema.sql   # Initial tables (SQL format)
├── 20260115-002-add-indexes.sql      # Add performance indexes (SQL format)
└── 20260220-003-add-transaction-logs.sql  # New feature (SQL format)
```

### Conventions

- **Format**: Use SQL changesets (`.sql` files), not YAML
- **Naming**: Follow `YYYYMMDD-NNN-description.sql` convention
- **Master changelog**: Explicitly `include` each file (never use `includeAll`)
- **Rollbacks**: Every changeset must have rollback instructions
- **DB columns**: Use UPPER_CASE (e.g., `CUSTOMER_ID`, `CREATED_AT`)
- **Entity fields**: Use camelCase in Java entity classes (e.g., `customerId`, `createdAt`)

- **Startup**: Run migrations automatically on startup via `spring.liquibase.enabled=true`
- **Changes**: All schema changes must be Liquibase changesets.
