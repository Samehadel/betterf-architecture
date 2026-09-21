# Backend Guides

Start with [Backend Architecture](../backend-architecture.md) for reusable design guidance. These guides are reference patterns; framework-specific content applies only after BetterF adopts the relevant technology. See [documentation status](../README.md#documentation-status).

| Topic | Scope |
|---|---|
| [Module structure](module-structure/module-structure.md) | Public/private boundaries and layering |
| [Persistence & JPA](database/persistence-jpa.md) | Optional entity, repository, mapping, and transaction patterns |
| [Database schema](database/database-schema.md) | Outline for future project schema documentation |
| [Database migrations](database/database-migrations.md) | Versioning discipline and optional Liquibase examples |
| [Security](security/security.md) | Authentication decisions and authorization boundaries |
| [Exception handling](exception-handling/exception-handling.md) | Typed failures, safe messages, and error translation |
| [Response handling](response-handling/response-handling.md) | Response consistency and optional wrapping |
| [Testing](testing/testing.md) | Unit, integration, and architecture checks |

Product behavior and accepted decisions belong in the strategic documents and `decisions/`. Keep implementation examples aligned with those decisions without duplicating the authoritative rule.
