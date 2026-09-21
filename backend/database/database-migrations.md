# Database Migrations — Reference Pattern

> Reusable reference, not an accepted BetterF technology or product decision. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## General discipline

Version schema changes, preserve applied migration history, and give migrations an owning module. Document execution order, compatibility, and recovery. Where reversal would lose data, record the limitation and a recovery plan rather than claiming a destructive rollback restores everything.

## Optional Liquibase layout

```text
src/main/resources/db/
    changelog-master.yaml
    {module}/
        YYYYMMDD-001-create-resource.sql
        YYYYMMDD-002-add-description.sql
```

Explicit includes make ordering reviewable. With this layout, the master changelog is the authoritative execution order; filenames alone do not determine it.

```yaml
databaseChangeLog:
  - include:
      file: db/example/20260101-001-create-resource.sql
```

Illustrative SQL changeset:

```sql
-- liquibase formatted sql
-- changeset example:20260101-001-create-resource
CREATE TABLE EXAMPLE_RESOURCE (
    ID BIGINT NOT NULL,
    NAME VARCHAR(200) NOT NULL,
    CONSTRAINT PK_EXAMPLE_RESOURCE PRIMARY KEY (ID)
);
-- rollback DROP TABLE EXAMPLE_RESOURCE;
```

The example is not a BetterF table definition. Identifier generation, column casing, and timestamps must follow the selected schema conventions. The rollback drops stored data; assess that consequence before applying it.

## Configuration and execution

If the example layout is adopted in Spring Boot, the matching location is:

```properties
spring.liquibase.change-log=classpath:db/changelog-master.yaml
spring.jpa.hibernate.ddl-auto=validate
```

Choose whether migrations run during deployment or application startup. Document commands only after the build plugin or migration runner is configured.

## Validation

Test migration ordering, constraints, upgrades from existing data, and recovery procedures against the production database engine. An in-memory substitute alone does not establish database-specific SQL compatibility. Never edit an already-applied migration to fix a later schema problem; add a new migration.
