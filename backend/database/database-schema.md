# Database Schema — Documentation Outline

> Reusable reference, not an accepted BetterF technology or product decision. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Status

No BetterF database, tables, relationships, or identifier strategy are defined here yet.

## Information to capture when the schema is designed

- Database engine and supported version.
- Data ownership by module.
- Tables, fields, keys, constraints, indexes, and relationships.
- Identifier generation and naming conventions.
- Timestamp, retention, deletion, and audit requirements.
- Transaction and concurrency expectations.
- Migration location and deployment process.

Use [migration guidance](database-migrations.md) for versioning discipline and [persistence guidance](persistence-jpa.md) if JPA is adopted. Keep credentials in the configured secret mechanism, outside documentation.
