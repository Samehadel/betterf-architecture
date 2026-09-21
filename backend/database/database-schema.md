# Database Schema — Documentation Outline

> Implementation guidance for the selected BetterF technology stack; examples do not define product requirements. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Status

PostgreSQL and Liquibase are selected. BetterF tables, relationships, and identifier strategy remain to be designed.

## Information to capture when the schema is designed

- Supported PostgreSQL version.
- Data ownership by module.
- Tables, fields, keys, constraints, indexes, and relationships.
- Identifier generation and naming conventions.
- Timestamp, retention, deletion, and audit requirements.
- Transaction and concurrency expectations.
- Migration location and deployment process.

Use [migration guidance](database-migrations.md) for versioning discipline and [persistence guidance](persistence-jpa.md) for Spring Data JPA. Keep credentials in the configured secret mechanism, outside documentation.
