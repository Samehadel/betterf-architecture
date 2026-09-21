# Database Migrations — Implementation Guide

> Detailed reference for managing schema changes with Liquibase SQL changesets.

---

## Structure

```
src/main/resources/db/
    ├── changelog-master.yaml                 ← root — lists all changesets explicitly
    ├── business/
    │   ├── 20260101-001-create-business-accounts-table.sql
    │   └── 20260319-002-add-country-aware-address-fields.sql
    ├── user/
    │   └── 20260316-001-create-users-table.sql
    └── auth/
        └── 20260318-001-create-refresh-tokens-table.sql
```

Migrations are grouped by module under `db/{module}/`. The master changelog is the only file Liquibase reads directly — it includes each SQL file by path. Never use `includeAll` — explicit inclusion makes the migration order visible and reviewable.

The `NNN` counter is **per-module** — it resets to `001` for each module and increments only within that module. It exists solely to keep changeset IDs unique within a module and to signal ordering intent within that module.

**Cross-module execution order is determined exclusively by the order of `include` entries in `changelog-master.yaml`.** The date and NNN in the filename have no effect on when Liquibase runs a migration — only its position in the master changelog does. Place entries in the order the schema changes must actually be applied (e.g. a table that has a FK dependency must appear after the table it references).

---

## Master Changelog

```yaml
# changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/business/20260101-001-create-business-accounts-table.sql
  - include:
      file: db/user/20260115-001-create-users-table.sql
  - include:
      file: db/auth/20260220-001-create-refresh-tokens-table.sql
  - include:
      file: db/business/20260301-002-add-address-fields.sql
```

Every new migration file must be added here. **The position in this file is the execution order** — Liquibase runs entries top-to-bottom. Place a new entry at the bottom unless an earlier position is required by a dependency.

---

## Writing a Changeset

Each `.sql` file is a single Liquibase changeset using the SQL format with comment-based metadata.

```sql
-- liquibase formatted sql

-- changeset dev:20260220-003
-- comment: Create queue_entries table for the queue module
-- labels: queue

CREATE TABLE QUEUE_ENTRIES (
    ID                      VARCHAR(36)     NOT NULL,
    BUSINESS_ID             VARCHAR(36)     NOT NULL,
    CUSTOMER_PHONE          VARCHAR(20)     NOT NULL,
    SERVICE_TYPE            VARCHAR(100)    NOT NULL,
    STATUS                  VARCHAR(20)     NOT NULL DEFAULT 'WAITING',
    POSITION                INTEGER         NOT NULL,
    ESTIMATED_WAIT_MINUTES  INTEGER         NOT NULL DEFAULT 0,
    TOKEN                   VARCHAR(36)     UNIQUE,
    TICKET_NUMBER           INTEGER         NOT NULL,
    JOINED_AT               TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CALLED_AT               TIMESTAMP,
    SERVED_AT               TIMESTAMP,
    CREATED_AT              TIMESTAMP       NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UPDATED_AT              TIMESTAMP,

    CONSTRAINT PK_QUEUE_ENTRIES PRIMARY KEY (ID),
    CONSTRAINT FK_QUEUE_ENTRIES_BUSINESS FOREIGN KEY (BUSINESS_ID)
        REFERENCES BUSINESS_PROFILES(ID)
);

-- rollback DROP TABLE QUEUE_ENTRIES;
```

**Rules:**
- Always start with `-- liquibase formatted sql`
- The changeset ID must be unique — use the date-number-description from the filename (e.g. `20260220-003`)
- Author is the developer creating the migration (`dev` is fine for team-shared migrations)
- Always include a `-- rollback` instruction — even if it's just `DROP TABLE`
- Column names use `UPPER_CASE`
- Include `CREATED_AT` and `UPDATED_AT` audit columns on every table
- Define foreign key constraints explicitly — don't rely on JPA to create them

---

## Adding Columns

```sql
-- liquibase formatted sql

-- changeset dev:20260310-005
-- comment: Add avg_wait_per_customer to business_profiles for estimated wait calculation

ALTER TABLE BUSINESS_PROFILES
    ADD COLUMN AVG_WAIT_PER_CUSTOMER INTEGER NOT NULL DEFAULT 5;

-- rollback ALTER TABLE BUSINESS_PROFILES DROP COLUMN AVG_WAIT_PER_CUSTOMER;
```

---

## Adding Indexes

```sql
-- liquibase formatted sql

-- changeset dev:20260115-002
-- comment: Add indexes for common queue queries

CREATE INDEX IDX_QUEUE_ENTRIES_BUSINESS_STATUS
    ON QUEUE_ENTRIES (BUSINESS_ID, STATUS);

CREATE INDEX IDX_QUEUE_ENTRIES_TOKEN
    ON QUEUE_ENTRIES (TOKEN);

CREATE INDEX IDX_BUSINESS_PROFILES_WHATSAPP_PHONE
    ON BUSINESS_PROFILES (WHATSAPP_PHONE);

-- rollback DROP INDEX IDX_QUEUE_ENTRIES_BUSINESS_STATUS;
-- rollback DROP INDEX IDX_QUEUE_ENTRIES_TOKEN;
-- rollback DROP INDEX IDX_BUSINESS_PROFILES_WHATSAPP_PHONE;
```

---

## Naming Conventions

| Object | Convention | Example |
|---|---|---|
| Table | `UPPER_CASE`, module-prefixed, plural | `QUEUE_ENTRIES`, `BUSINESS_PROFILES` |
| Column | `UPPER_CASE` | `CUSTOMER_PHONE`, `CREATED_AT` |
| Primary key constraint | `PK_{TABLE}` | `PK_QUEUE_ENTRIES` |
| Foreign key constraint | `FK_{TABLE}_{REFERENCED_TABLE}` | `FK_QUEUE_ENTRIES_BUSINESS` |
| Index | `IDX_{TABLE}_{COLUMNS}` | `IDX_QUEUE_ENTRIES_BUSINESS_STATUS` |
| Unique constraint | `UQ_{TABLE}_{COLUMN}` | `UQ_QUEUE_ENTRIES_TOKEN` |
| Migration file | `YYYYMMDD-NNN-description.sql` — `NNN` is per-module sequential, never resets per date | `20260220-003-queue-entries-table.sql` |

---

## Application Configuration

```properties
# application.properties
spring.liquibase.change-log=classpath:db/changelog/db.changelog-master.yaml
spring.liquibase.enabled=true

# Never use ddl-auto=update — Liquibase manages the schema
spring.jpa.hibernate.ddl-auto=validate
```

On startup, Liquibase:
1. Creates the `DATABASECHANGELOG` and `DATABASECHANGELOGLOCK` tables if they don't exist
2. Compares the master changelog against previously run changesets
3. Runs any new changesets in order
4. Validates all previously run changesets haven't been modified (fails startup if they have)

**Never modify a changeset after it has been run** — Liquibase checksums each changeset and will refuse to start if one has changed.

---

## Running Migrations Manually

```bash
# Apply pending migrations
./gradlew liquibaseUpdate

# Roll back the last N changesets
./gradlew liquibaseRollbackCount -PliquibaseCommandValue=1

# View pending changesets
./gradlew liquibaseStatus

# Validate all changesets
./gradlew liquibaseValidate
```

Migrations also run automatically on `./gradlew bootRun` and `./gradlew bootRun --args='--spring.profiles.active=local'`.

---

## Testing Migrations

Integration tests run against H2 in-memory — Liquibase applies all migrations to H2 at test startup. This validates that:
- All SQL is compatible with H2 (which catches most PostgreSQL syntax issues)
- Migration files are syntactically valid
- The order of migrations produces a valid schema

If a migration fails in tests, fix it before committing — do not disable the Liquibase test context.

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Editing a changeset after it has been run | Never edit — create a new changeset to amend |
| Using `includeAll` in master changelog | Use explicit `include` — order matters and must be visible |
| Missing `-- rollback` instruction | Every changeset needs one — even `DROP TABLE` |
| Lowercase column names | Use `UPPER_CASE` consistently |
| `ddl-auto=update` in properties | Use `validate` — Liquibase manages the schema |
| Module-less table names (e.g. `entries`) | Prefix with module (`QUEUE_ENTRIES`) |
| Forgetting to add the new file to master changelog | Migration won't run until it's listed in `db.changelog-master.yaml` |
| Reusing `NNN` for a new date (e.g. `20260319-001` when `001` already exists in that module) | `NNN` is per-module — check the highest number in that module's directory and increment from there |
