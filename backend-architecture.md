# Backend Architecture & Design

**Template Version** — Customize all `[PLACEHOLDER]` sections below for your project.

> This document defines how the backend is structured, how modules relate to each other, and conventions every contributor — human or AI — must follow. It does not cover every detail; it covers the rules.

---

## Overview

The backend is a **[BACKEND_FRAMEWORK]** application using **[MODULE_ENFORCEMENT_TOOL]** to enforce domain boundaries within a single deployable (or services, depending on architecture). It follows a domain-driven layered architecture: code is organized by business domain first, technical role second.

### Stack

| Concern | Technology |
|---|---|
| Runtime | [BACKEND_RUNTIME] |
| Web | [WEB_FRAMEWORK] |
| Module enforcement | [MODULE_ENFORCEMENT_TOOL] |
| Persistence | [PERSISTENCE_FRAMEWORK] |
| Migrations | [MIGRATION_TOOL] |
| Security | [SECURITY_FRAMEWORK] |
| API contracts | [API_STYLE] |
| Build | [BUILD_TOOL] |
| Utilities | [UTILITIES] |
| Testing | [TESTING_FRAMEWORKS] |

---

## Module Structure

The codebase has two categories of modules: **domain modules** and **technical modules**. They follow different conventions intentionally.

### Domain Modules

Domain modules represent business concepts — `[DOMAIN_1]`, `[DOMAIN_2]`, etc. Each lives under `com.[COMPANY_DOMAIN].[domain]` and is split into two subpackages with further grouping by technical role:

```
com.[COMPANY_DOMAIN].[DOMAIN_1]/
    api/                            ← public surface of the module
        dto/
            [DOMAIN_1]View.java           ← response View (implements BaseView)
            Create[DOMAIN_1]Request.java  ← request object
        service/
            [DOMAIN_1]Service.java        ← public service interface
        exception/
            [DOMAIN_1]NotFoundException.java
        enum/
            [DOMAIN_1]Status.java         ← enums referenced in public Views
    internal/                       ← private implementation details
        entity/
            [DOMAIN_1]Entity.java         ← JPA entity (extends BaseEntity)
        repository/
            [DOMAIN_1]Repository.java     ← Spring Data repository
        service/
            [DOMAIN_1]ServiceImpl.java    ← implements [DOMAIN_1]Service
        mapper/
            [DOMAIN_1]Mapper.java         ← MapStruct mapper (extends GlobalMapper)
        util/
            [DOMAIN_1]Utils.java          ← internal helper classes
        enum/
            [DOMAIN_1]InternalStatus.java ← enums used only inside this module
    controller/                     ← HTTP adapter layer
        [DOMAIN_1]Controller.java         ← HTTP adapter; returns data objects; ResponseAdvice wraps
```

**Rules:**
- Other modules may only import from `com.[COMPANY_DOMAIN].{domain}.api.*` — never from `internal`
- Controllers live in `controller/` — a sibling package to `api/` and `internal/`, never inside either
- `controller/` is intentionally separate from `api/`: `api/` holds Java interfaces for other modules; `controller/` holds HTTP adapters that no other module should import
- Entities and repositories are always `internal` — they are never part of the public API
- Enums referenced in public Views or service interfaces belong in `api/enum/` — enums used only inside the module belong in `internal/enum/`
- **Entities extend from `BaseEntity`** — provides common fields (id, createdAt, updatedAt) and inheritance structure [CUSTOMIZE for your base class]
- **Views implement `BaseView`** — marker interface for public response models. Response classes use the `View` suffix [CUSTOMIZE naming convention]
- [MODULE_ENFORCEMENT_TOOL]'s architecture tests enforce these boundaries at build time — violations fail the build

### Technical Modules

Technical modules handle cross-cutting concerns: `security`, `config`, `storage`, etc. They do not follow the `api/internal` split — everything in them is available to the app by nature.

```
com.[COMPANY_DOMAIN].security/
    [SECURITY_CONFIG_CLASS].java
    [SECURITY_FILTER_CLASS].java
    [AUTHENTICATION_CLASS].java

com.[COMPANY_DOMAIN].config/
    [CONFIG_CLASS_1].java
    [CONFIG_CLASS_2].java
```

**Rules:**
- No business logic lives in technical modules
- Technical modules may depend on domain module `api` packages, never on `internal`
- New cross-cutting concerns get their own technical module — do not add them to an existing domain module

### Access Rules

- Other modules may only import from `com.[COMPANY_DOMAIN].{domain}.api.*` — never from `internal`
- Controllers live in the **module root** — not in a global `controllers/` folder
- Entities and repositories are always `internal` — never part of the public API
- Spring Modulith's architecture tests enforce these boundaries at build time — violations fail the build

---

## Layered Architecture (within a module)

```
HTTP Request
    → Controller      handles HTTP, delegates immediately — no logic
    → Facade          optional caller-scoped orchestration (auth context, use-case composition)
    → Service         all business logic, validation, orchestration
    → Repository      data access only — no logic
    → Database
```

**Where logic belongs:**

- **Controllers** — HTTP concerns only: parsing input, returning responses, setting status codes. No business conditions.
- **Facades** — Optional module-internal orchestration for caller-scoped use cases. A facade may translate authenticated context into explicit service arguments or compose multiple service calls, but it must not contain domain rules. One facade per domain concept — mixed-domain facades are a SRP violation regardless of how cleanly they delegate internally. Facade names reflect the domain concept, not the caller role.
- **Services** — All business logic, validation rules, and orchestration. The only place decisions are made.
- **Repositories** — Query the database. Return entities or projections. No logic.
- **Mappers** — Convert between entities (extending `BaseEntity`) and Views (implementing `BaseView`). Always a MapStruct `@Mapper(componentModel = "spring")` interface extending `GlobalMapper<E, V>`. Methods are `mapToView()` and `mapToEntity()`. Never a plain `@Component` class with handwritten field assignment. Always `internal`. Never called from outside the module.

---

## Inter-Module Communication

Modules communicate via **direct calls only**. There are no application events between modules.

- A module exposes a service interface in its `api` package
- Other modules inject and call that interface directly
- The implementation lives in `internal` and is invisible to the caller

```java
// ✅ Correct — calling another module's public API
@Service
@RequiredArgsConstructor
public class [DOMAIN_1]Service {
    private final [DOMAIN_2]Service [domain2Service]; // from com.[COMPANY_DOMAIN].[domain2].api
}

// ❌ Wrong — reaching into another module's internals
import com.[COMPANY_DOMAIN].[domain2].internal.[DOMAIN_2]Repository; // never do this
```

---

## API Endpoint Workflow

API endpoints are implemented directly in handwritten Spring MVC controllers and module-local request/response models.

The workflow:
1. Define or update the endpoint shape in code with the module's request objects and response Views
2. Implement or update the controller using explicit Spring MVC annotations
3. Keep [API Specification](./api-specification.md) aligned when endpoint conventions or shared API behavior change

**Response wrapping is handled by `ResponseAdvice`** (`com.[COMPANY_DOMAIN].common.service.ResponseAdvice`), a `@RestControllerAdvice` that automatically wraps every controller return value in `ApiResponse<T>`. Controllers return the payload object directly — they do not call `ResponseBuilderService` or construct `ResponseEntity` manually. [CUSTOMIZE response wrapping as needed]

Exceptions:
- Use `@ResponseStatus(HttpStatus.CREATED)` for 201 responses instead of returning `ResponseEntity`
- Use `@IgnoreResponseWrapper` on any method that must bypass the automatic wrapping (e.g. raw list endpoints, file downloads, `204 No Content` responses)
- `204 No Content` methods return `void` and carry both `@ResponseStatus(HttpStatus.NO_CONTENT)` and `@IgnoreResponseWrapper`

**Rules:**
- Keep controller annotations, request objects, response Views, and the API conventions docs aligned
- Never return `ResponseEntity` from a controller — use `@ResponseStatus` for non-200 codes
- Never call response builder methods in a controller — response wrapping middleware handles this automatically
- Business logic goes in the service implementation, never in a controller
- View the live API documentation at `http://localhost:[PORT]/[API_DOCS_PATH]`

---

## API Response Format

Define your API response envelope here. Below is an example structure:

```json
{
  "status":         "[SUCCESS_VALUE] | [FAILURE_VALUE]",
  "data":           "<payload or null>",
  "message":        "<localized human-readable message or null>",
  "code":           "<machine-readable error code, e.g. [ERROR_CODE_PREFIX]_001>",
  "[ERROR_TRACKING_FIELD]": "<auto-generated reference for correlation>",
  "[VALIDATION_FIELD]":     ["<field: message>", "..."],
  "timestamp":      "<ISO 8601>"
}
```

**Response fields (customize as needed):**
- Error code — present on all error responses; identifies the error type
- Tracking reference — present only on internal errors; use for log correlation and support tracing
- Validation errors — present only on validation failures; lists each field violation
- Payload — present only on success responses; null on errors

HTTP status codes follow standard REST conventions. See [API Specification](./api-specification.md) for the full list.

---

## Security

Security configuration is **centralized** in `com.[COMPANY_DOMAIN].security`. No domain module defines its own security rules. [CUSTOMIZE to match your security framework]

- Authentication validation happens in a centralized filter or middleware
- The filter chain, CORS, and CSRF are all configured in a central location
- Individual endpoints are secured via authorization annotations on controllers or service methods
- Domain modules do not import from `com.[COMPANY_DOMAIN].security` — security is transparent to them

---

## Persistence

**Database strategy:** [DESCRIBE: Single shared database / Multiple databases / NoSQL / etc.]

- Entities always extend your base class and are always `internal` — never exposed outside their module
- Repositories are always `internal` — other modules access data through service interfaces
- Follow your ORM/query conventions — avoid raw SQL when possible
- Apply consistent naming conventions across your database and code (e.g., snake_case in DB, camelCase in code)
- Module ownership: a module owns its data structure — only that module's repository queries it directly

---

## Database Migrations

All schema changes are managed with **[MIGRATION_TOOL]** using your preferred format. Changelogs are organized per module:

```
src/main/resources/db/
    ├── changelog-master.[FORMAT]      ← root — explicitly includes all module changelogs
    ├── [DOMAIN_1]/
    │   ├── YYYYMMDD-001-create-[domain1].sql
    │   └── YYYYMMDD-002-add-[domain1]-fields.sql
    ├── [DOMAIN_2]/
    │   └── YYYYMMDD-001-create-[domain2].sql
    └── config/
        └── YYYYMMDD-001-shared-config.sql
```

**Rules:**
- Every schema change requires a versioned migration — never rely on ORM auto-schema generation in production
- File naming: Follow your migration tool's naming convention (e.g., `YYYYMMDD-NNN-description`)
- The master changelog explicitly lists each migration file
- Every migration must include rollback instructions
- A module owns its migrations — changelogs are organized under the module's directory

---

## Testing

This project follows **strict TDD** — tests are written before implementation code, not after. This is not optional; it is the development process.

### Test Types

**Unit tests** — default for all service logic. Test a single service class in isolation with mocked dependencies. No framework context, no database.

```
src/test/[LANGUAGE]/com/[COMPANY_DOMAIN]/[domain]/internal/
    [DOMAIN_1]ServiceImplTest.[TEST_EXTENSION]   ← tests [DOMAIN_1]ServiceImpl, mocks [DOMAIN_1]Repository
```

**Integration tests** — validate full module wiring, persistence behavior, and cross-layer correctness using a real framework context and test database. No external services or Docker required.

```
src/test/[LANGUAGE]/com/[COMPANY_DOMAIN]/[domain]/
    [DOMAIN_1]IntegrationTest.[TEST_EXTENSION]   ← full context, test database, real repositories
```

**Architecture tests** — verify that no module violates encapsulation boundaries. Run as part of the standard test suite. [CUSTOMIZE based on your module enforcement tool]

Tests mirror the production package structure exactly:

```
src/
    main/[LANGUAGE]/com/[COMPANY_DOMAIN]/[domain]/internal/[DOMAIN_1]ServiceImpl.[CODE_EXT]
    test/[LANGUAGE]/com/[COMPANY_DOMAIN]/[domain]/internal/[DOMAIN_1]ServiceImplTest.[TEST_EXT]  ← mirrors exactly
```

### TDD Cycle

1. Write a failing test describing the desired behavior
2. Write the minimum implementation to make it pass
3. Refactor without changing behavior
4. Repeat

**Rules:**
- A failing test is a broken build — tests cannot be skipped or suppressed to unblock a merge
- Unit tests mock at the service boundary — repositories are mocked, service logic is what's under test
- Integration tests use a test database — no external services needed to run the full suite
- Maintain [BACKEND_COVERAGE_TARGET]%+ line coverage (enforced by your coverage tool)

---

## What Goes Where — Quick Reference

| Thing | Where it lives |
|---|---|
| Business logic | `internal/service/` implementation class |
| Public service interface | `api/service/` |
| HTTP endpoint (controller) | `controller/` — sibling to `api/` and `internal/` |
| Caller-scoped orchestration facade | `internal/facade/` |
| Entity class (your base class) | `internal/entity/` |
| Repository (data access) | `internal/repository/` |
| Request objects and response Views | `api/dto/` |
| Enums used in public Views or interfaces | `api/enum/` |
| Enums used only inside the module | `internal/enum/` |
| Entity ↔ View mapping (mapper, extends GlobalMapper) | `internal/mapper/` |
| Internal helper / utility classes | `internal/util/` |
| Public exceptions | `api/exception/` |
| Cross-module service calls | Via `api/service/` interface of the owning module |
| Security configuration | `com.[COMPANY_DOMAIN].security` |
| App-wide config (CORS, persistence, etc.) | `com.[COMPANY_DOMAIN].config` |
| DB migrations | `src/main/resources/db/{module}/` |
| Unit tests | Mirror of production package under `src/test/` |
| Integration tests | Module root package under `src/test/` |

---

## Things to Avoid

- **Never import from another module's `internal` package.** Your module enforcement tool will catch this and fail the build.
- **Never put business logic in a controller.** Controllers are HTTP adapters only — no business conditions.
- **Never create a global `controllers/`, `services/`, or `repositories/` folder.** Code is organized by domain, not by technical role.
- **Never return raw response entities from a controller.** Use status annotations for non-200 codes; use middleware/interceptors to handle wrapping automatically.
- **Never call response wrapping functions from a controller.** Let response middleware handle wrapping automatically; manual wrapping produces double-wrapped responses.
- **Never change the database schema outside your migration tool.** ORM auto-schema generation is not permitted in any environment.
- **Never add security rules inside a domain module.** All security configuration belongs in a centralized security module.
- **Never call another module's repository directly.** Cross-module data access goes through the owning module's `api/` service interface.
- **Never write implementation code without a failing test first.**
- **Never name an entity class without a type suffix.** Every entity class name should follow your naming convention (e.g., `[DOMAIN_1]Entity`). Use your ORM's table name annotation to control the database table name independently.
- **Never write a mapper by hand.** All entity ↔ View mapping must use your mapping framework (e.g., MapStruct). No manual field-by-field assignment in services.
