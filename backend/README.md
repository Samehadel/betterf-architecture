# Backend Implementation Guides

> These guides contain the detailed implementation patterns, code examples, and conventions for each technical area of the backend. They are the **"how"** — the [Backend Architecture](../backend-architecture.md) is the **"what and why"**.

---

## Guides

| Topic | Description |
|---|---|
| [Module Structure](./module-structure/module-structure.md) | Domain module layout, `api`/`internal` split, controllers, cross-module calls, Spring Modulith boundary tests |
| [OpenAPI Code Generation](./openapi-code-generation/openapi-code-generation.md) | Legacy note for the retired generator workflow; do not use for new work |
| [Persistence & JPA](database/persistence-jpa.md) | Entity conventions, repositories, projections, mappers, pagination, transaction rules |
| [Database Migrations](database/database-migrations.md) | Liquibase SQL format, changeset structure, rollbacks, naming conventions, master changelog |
| [Security](./security/security.md) | `SecurityConfig`, `JwtFilter`, JWT generation/validation, `@PreAuthorize`, role-based access |
| [Error Handling](exception-handling/exception-handling.md) | `ApiResponse` wrapper, domain exceptions, global exception handler, error codes, correlation IDs |
| [Testing](./testing/testing.md) | TDD cycle, unit tests with Mockito, integration tests with H2, `@DataJpaTest`, Modulith boundary tests |

---

## How to Use These Guides

- **Adding a new domain module?** Start with [Module Structure](./module-structure/module-structure.md).
- **Defining a new API endpoint?** Start with [Module Structure](./module-structure/module-structure.md) and [Backend Architecture](../backend-architecture.md).
- **Adding a new database table or column?** Read [Database Migrations](database/database-migrations.md).
- **Writing service logic and queries?** Read [Persistence & JPA](database/persistence-jpa.md).
- **Handling exceptions and returning consistent responses?** Read [Exception Handling](exception-handling/exception-handling.md).
- **Securing an endpoint or adding a role check?** Read [Security](./security/security.md).
- **Writing tests for new code?** Read [Testing](./testing/testing.md).
