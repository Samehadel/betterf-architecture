# Backend Architecture

## Status

BetterF uses Java and Spring Boot. The inherited technical stack and reusable conventions below are retained; domain models, deployment topology, and product-specific authentication behavior remain to be designed. The [backend guides](backend/README.md) explain implementation patterns. See the accepted [technology baseline](decisions/0001-technology-baseline.md).

## Technology stack

| Concern | Selected technology |
|---|---|
| Runtime and framework | Java, Spring Boot, Spring MVC |
| Module boundaries | Spring Modulith |
| Persistence | Spring Data JPA / Hibernate, PostgreSQL |
| Schema migrations | Liquibase SQL changesets |
| Security | Spring Security |
| Build | Gradle wrapper |
| Mapping and boilerplate | MapStruct, Lombok |
| API documentation | OpenAPI documentation aligned with handwritten controllers |
| Testing | JUnit, Mockito, AssertJ, Spring test support, JaCoCo |

Pin compatible versions in the application build. These choices specify the intended implementation; they do not claim dependencies or CI checks are already configured.

## Java and Spring conventions

- Use the domain-first `api/`, `internal/`, and `controller/` structure from the [module guide](backend/module-structure/module-structure.md).
- Expose service interfaces through `api/service/` and implement them in `internal/service/` using Spring `@Service` classes and constructor injection.
- Keep request objects and response Views in `api/dto/`; do not expose JPA entities. Use immutable response models and the `View` suffix.
- Keep entities and Spring Data repositories in `internal/entity/` and `internal/repository/`. Entity names use the `Entity` suffix.
- Use MapStruct `@Mapper(componentModel = "spring")` interfaces for entity/View mapping. Shared base entity, View, and mapper abstractions belong in common infrastructure when implemented.
- Put transaction boundaries on service methods. Mark read-only operations appropriately.
- Keep controllers as handwritten HTTP adapters with explicit Spring MVC annotations and boundary validation.
- Implement centralized response wrapping and exception handling; controllers return payload models. Preserve status codes and exclude downloads, streams, and no-content responses from wrapping.
- Use Spring Security for authentication integration and shared transport configuration; services enforce resource-specific access rules.

## Module boundaries

- Organize business behavior by domain, with technical roles grouped inside each domain.
- Expose a small public interface for cross-module collaboration. Keep persistence entities, repositories, mappers, and implementation classes private to the owning module.
- Access another module's data through its public interface, not its repository or internal tables.
- Keep cross-cutting infrastructure separate from business rules; shared code must have a clear responsibility.
- Use Spring Modulith architecture tests to verify module boundaries. Do not claim checks are enforced until they exist in the build.

Domain names and package names will follow BetterF requirements. No inherited domain map is prescribed.

## Responsibilities within a module

| Layer | Responsibility |
|---|---|
| HTTP adapter / controller | Parse and validate transport input, delegate use cases, translate results into HTTP responses |
| Optional facade | Compose use-case calls or translate caller context into explicit arguments |
| Service | Enforce business rules, resource access, invariants, and state transitions |
| Repository | Encapsulate persistence access |
| Mapper | Convert internal data into public models without exposing persistence details |

Keep domain rules out of controllers and infrastructure. An optional facade must not become a second home for business rules.

## Communication and contracts

Use explicit contracts across module and system boundaries. Document dependencies, failure behavior, and transaction boundaries. Retain direct calls through public service interfaces for in-process module collaboration. Introducing application events or remote service boundaries requires a documented architecture decision.

Keep request/response models, endpoint behavior, and [API documentation](api-specification.md) aligned. Use handwritten Spring MVC controllers and module-local request/response models. Retain centralized response wrapping and error translation as the implementation convention; their shared classes still need to be implemented. Document the exact wire contract before exposing endpoints.

## Security

Centralize authentication integration and shared transport security configuration. Enforce resource permissions and ownership at the server-side use-case boundary, including calls that do not originate in HTTP controllers.

Public endpoints, roles, credentials, session lifetime, refresh behavior, and identity providers require BetterF-specific decisions. See the [security guide](backend/security/security.md) for the reusable review structure.

## Persistence and migrations

- Give each data model an explicit owner.
- Keep transaction boundaries aligned with use cases.
- Bound reads on collections that can grow without limit.
- Use versioned migrations for production schema changes; do not rely on implicit ORM schema mutation.
- Preserve applied migration history. Document recovery or rollback limitations and verify compatibility with the selected production database.

Use PostgreSQL, Spring Data JPA, and Liquibase SQL changesets. Keep database names in a consistent UPPER_CASE convention and Java fields in camelCase. The actual schema, identifier strategy, and migration deployment execution model remain open.

## Testing

Test business behavior in isolation, integration behavior across real component boundaries, and module boundaries where tooling permits. Cover invalid input, denied access, missing resources, failure paths, and relevant concurrency behavior.

The test-first cycle remains useful: express behavior in a failing test, implement it, then refactor. Use JUnit, Mockito, AssertJ, and Spring test support, with JaCoCo for coverage reporting. Exact commands, test infrastructure, and coverage gates must be verified against the application build.

## Decisions to establish

Before implementing the relevant area, record domain boundaries, data ownership, API contracts, security behavior, and runtime/deployment constraints in this repository. Significant choices belong in `decisions/`, with an explicit status and rationale.
