# Module Structure — Implementation Guide

> Detailed implementation reference for Spring Modulith domain modules and the `api`/`internal` boundary. Read [Backend Architecture](../../backend-architecture.md) for the high-level decisions.

---

## Package Layout

Each domain module follows the same internal layout. Within both `api/` and `internal/`, files are grouped into subdirectories by technical role.

```
com.{company}.{domain}/
    api/
        dto/
            QueueEntryView.java           ← response View (implements BaseView)
            CreateQueueEntryRequest.java  ← request object
        service/
            QueueService.java             ← public service interface
        exception/
            QueueNotFoundException.java
        enum/
            QueueEntryStatus.java         ← enum used in public Views
    internal/
        entity/
            QueueEntity.java              ← JPA entity (extends BaseEntity)
        repository/
            QueueRepository.java          ← Spring Data repository
        facade/
            QueueOwnerFacade.java         ← module-internal caller-scoped orchestration
        service/
            QueueServiceImpl.java         ← implements QueueService
        mapper/
            QueueMapper.java              ← MapStruct entity ↔ View mapper
        util/
            QueuePositionUtils.java       ← internal helper classes
        enum/
            QueueSortOrder.java           ← enums used only inside this module
    controller/
        QueueController.java              ← HTTP adapter — handwritten Spring MVC controller
```

The three top-level subpackages each have a distinct responsibility:
- `api/` — the **programmatic public surface** for other modules (interfaces, request objects, response Views, exceptions, enums)
- `internal/` — **private implementation details** invisible to the outside world
- `controller/` — the **HTTP adapter layer**, separate from `api/` because no other module should import a controller; it is accessed exclusively over the wire, not via Java

**Enum placement rule:** If an enum appears in any type under `api/` (a View field, a request field, or a service method parameter or return type), it belongs in `api/enum/`. If it is only used inside the module's own service, entity, or repository logic, it belongs in `internal/enum/`.

---

## Domain Module Checklist

When creating a new domain module (e.g. `notification`):

1. Create the package `com.{company}.notification`
2. Create the `api/` subpackages: `dto/`, `service/`, `exception/`, `enum/`
3. Create the `internal/` subpackages: `entity/`, `repository/`, `service/`, `mapper/`, `util/`, `enum/`
4. Create the `controller/` subpackage
5. Create `internal/facade/` only when a module needs caller-scoped orchestration that should not live in controllers or core domain services
6. Create handwritten request objects and response Views in `api/dto/`
7. Create `NotificationController.java` in `controller/` with explicit Spring MVC annotations
8. Add the service interface in `api/service/` and implementation in `internal/service/`
9. Add Liquibase changeset if new tables are needed
10. Write unit tests for the service, plus facade tests if `internal/facade/` is introduced, and an integration test for the controller

---

## Defining a Module's Public API

The `api/` package exposes exactly what other modules need — no more.

```java
// com/{company}/queue/api/service/QueueService.java
public interface QueueService {
    QueueEntryView joinQueue(CreateQueueEntryRequest request);
    QueueEntryView callNext(String businessId);
    QueueEntryView getEntry(String entryId);
    void markServed(String entryId);
    CustomerQueueStatusView getCustomerStatus(String token);
}
```

```java
// com/{company}/queue/api/dto/QueueEntryView.java
@Builder
public record QueueEntryView(
    Long id,
    int ticketNumber,
    String customerName,
    String serviceType,
    QueueEntryStatus status,      // ← from api/enum/
    int position,
    int estimatedWaitMinutes,
    Instant joinedAt
) implements BaseView {}
```

```java
// com/{company}/queue/api/enum/QueueEntryStatus.java
public enum QueueEntryStatus {
    WAITING,
    CALLED,
    SERVED,
    CANCELLED
}
```

```java
// com/{company}/queue/api/exception/QueueNotFoundException.java
public class QueueNotFoundException extends RuntimeException {
    public QueueNotFoundException(String entryId) {
        super("Queue entry not found: " + entryId);
    }
}
```

**Rules:**
- Expose only what callers need — keep the `api/` surface minimal
- Use `record` types for Views — immutable and concise
- Use `@Builder` (Lombok) on Views for readable construction in tests
- Exceptions that callers need to handle live in `api/exception/` — internal-only exceptions stay in `internal/`
- Enums in `api/enum/` must remain stable — they are part of the public contract of the module

---

## Implementing the Service

The service implementation lives in `internal/service/` and is annotated with `@Service`.

```java
// com/{company}/queue/internal/service/QueueServiceImpl.java
@Service
@RequiredArgsConstructor
@Transactional
public class QueueServiceImpl implements QueueService {

    private final QueueRepository queueRepository;   // from internal/repository/
    private final QueueMapper queueMapper;            // from internal/mapper/
    private final BusinessService businessService;   // from com.{company}.business.api/service/

    @Override
    public QueueEntryView joinQueue(CreateQueueEntryRequest request) {
        BusinessView business = businessService.getBusiness(request.businessId());
        if (!business.isQueueOpen()) {
            throw new QueueClosedException(request.businessId());
        }

        int position = queueRepository.countByBusinessIdAndStatus(
            request.businessId(), QueueEntryStatus.WAITING
        ) + 1;

        QueueEntity entry = QueueEntity.builder()
            .businessId(request.businessId())
            .customerPhone(request.customerPhone())
            .serviceType(request.serviceType())
            .status(QueueEntryStatus.WAITING)
            .position(position)
            .estimatedWaitMinutes(position * business.avgWaitPerCustomer())
            .token(UUID.randomUUID().toString())
            .build();

        return queueMapper.mapToView(queueRepository.save(entry));
    }
}
```

**Rules:**
- Service methods are `@Transactional` by default — mark read-only methods with `@Transactional(readOnly = true)`
- Cross-module calls go through the `api/service/` interface, injected via constructor (`@RequiredArgsConstructor`)
- Throw typed exceptions from `api/exception/` — never throw raw `RuntimeException` with a string message
- Never return entities from a service — always map to a View before returning

## Optional Facade Layer

Use `internal/facade/` when a module needs a thin, module-private orchestration layer between controllers and services.

Typical reasons to introduce a facade:
- Resolve authenticated caller context into explicit service arguments
- Compose multiple service calls for one HTTP use case
- Keep controllers as pure HTTP adapters without pushing transport concerns into domain services

Rules:
- Facades remain **internal** to the module — they are never imported across modules
- Facades may depend on technical helpers like `SecurityContextService` and on the module's own `api/service/` interfaces
- Facades must not contain domain rules, persistence logic, or mapping that belongs in core services
- Core service interfaces should stay explicit and reusable; prefer `openForDate(businessId, date)` over hidden security-context lookups inside a service

---

## The Entity

Entity class names **must** end with the `Entity` suffix (e.g., `QueueEntryEntity`, `CustomerAccountEntity`). The database table name is controlled independently by `@Table(name = "...")`.

```java
// com/{company}/queue/internal/entity/QueueEntryEntity.java
@Entity
@Table(name = "queue_entries")
@Getter
@Setter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class QueueEntryEntity extends BaseEntity {

    private String businessId;
    private String customerPhone;
    private String serviceType;
    private QueueEntryStatus status;    // from api/enum/ — part of public contract
    private int position;
    private int estimatedWaitMinutes;
    private String token;
}
```

---

## The Mapper

Mappers must always be MapStruct `@Mapper` interfaces extending `GlobalMapper<E, V>` — never plain `@Component` classes with handwritten field assignment. Naming convention: `{Domain}Mapper.java`. Methods are `mapToView()` and `mapToEntity()`.

```java
// com/{company}/queue/internal/mapper/QueueMapper.java
@Mapper(componentModel = "spring")
public interface QueueMapper extends GlobalMapper<QueueEntryEntity, QueueEntryView> {

    @Override
    QueueEntryView mapToView(QueueEntryEntity entity);
}
```

---

## Cross-Module Communication

A module calls another module only via its public service interface.

```java
// ✅ Correct — calling com.{company}.business.api.service.BusinessService
@Service
@RequiredArgsConstructor
public class QueueServiceImpl implements QueueService {
    private final BusinessService businessService;   // injected, from api/service/
}

// ❌ Wrong — reaching into another module's internal
import com.{company}.business.internal.repository.BusinessRepository;  // never
import com.{company}.business.internal.entity.BusinessEntity;          // never
```

---

## The Controller

The controller lives in the **module root** (not inside any subdirectory) and uses explicit Spring MVC annotations. It does nothing except delegate to the service.

```java
// com/{company}/queue/controller/QueueController.java
@RestController
@RequiredArgsConstructor
public class QueueController {

    private final QueueService queueService;   // from api/service/

    @PostMapping("/{businessId}/join")
    @ResponseStatus(HttpStatus.CREATED)
    public QueueEntryView joinQueue(
            @PathVariable String businessId,
            @Valid @RequestBody CreateQueueEntryRequest request) {
        return queueService.joinQueue(request);
    }

    @PostMapping("/{businessId}/call-next")
    public QueueEntryView callNext(@PathVariable String businessId) {
        return queueService.callNext(businessId);
    }
}
```

**Rules:**
- Controllers use explicit `@RequestMapping` / `@GetMapping` / `@PostMapping` annotations
- Controller methods contain at most 2–3 lines: delegate to service and return the payload
- No business logic, no conditionals based on business rules
- Never call a repository directly from a controller
- Controllers live in `controller/` — never inside `api/` or `internal/`
- No other module should ever import a class from `controller/` — it is accessed exclusively over HTTP

---

## Validation Layers: What Validation Happens Where

Validation occurs at **two distinct layers**, each with a specific responsibility:

### 1. HTTP Boundary Validation (Controller)

The controller is the HTTP adapter layer. Validation here is about **request format and completeness** — concerns specific to the HTTP protocol, not business logic.

**Where:** Controller methods

**What to validate:**
- Required fields are present and non-null (e.g., `@NotNull`, `@NotBlank`)
- Request body format is valid (e.g., date format in `@RequestBody`)
- Path parameters can be parsed (e.g., numeric IDs)
- HTTP-specific concerns (e.g., missing Authorization header, invalid Content-Type)

**Example:**
```java
@PostMapping
public ResponseView create(
        @RequestBody @Valid CreateRequest request) {  // ← framework validates format
    // At this point, request is guaranteed valid by Spring's @Valid
    return service.create(request.name(), request.email());
}
```

### 2. Business Logic Validation (Service)

The service layer implements the **business rules** — these are domain concerns, not HTTP concerns.

**Where:** Service methods (`internal/service/`)

**What to validate:**
- Resource existence (e.g., "queue not found" → 404)
- Resource ownership / authorization (e.g., "queue belongs to different business" → 404)
- Business state transitions (e.g., "queue cannot accept registrations in CLOSED status" → 422)
- Cross-entity invariants (e.g., "phone already registered for this queue" → 422)
- Derived state constraints (e.g., "position cannot exceed queue capacity")

**Example:**
```java
@Transactional
public QueueCustomerView registerPublic(Long queueId, Long businessId, String name, String phone) {
    // Business validation
    QueueEntity queue = queueRepository.findById(queueId)
            .orElseThrow(() -> /* 404 */);

    if (!queue.getBusinessId().equals(businessId)) {
        throw /* 404 - prevent cross-business access */;
    }

    if (!QueueStatus.REGISTERABLE_STATUSES.contains(queue.getStatus())) {
        throw /* 422 - invalid state */;
    }

    // Business logic...
}
```

**Rules:**
- Never perform business validation in a controller
- All resource existence and ownership checks belong in the service layer
- All state transitions and business rules belong in the service layer
- Exceptions from the service layer communicate business errors to the controller, which are then translated to HTTP status codes by `ResponseAdvice`

---

## Technical Modules

Technical modules (`security`, `config`) do not use the `api/`/`internal/` split — everything in them is available to the app. They contain no business logic and do not need the subdirectory structure.

```
com.{company}.security/
    SecurityConfig.java       ← filter chain, CORS, CSRF
    JwtFilter.java            ← token validation
    JwtService.java           ← token parsing and generation
    SecurityUserDetails.java  ← UserDetails adapter

com.{company}.config/
    WebConfig.java            ← MVC configuration, view resolvers
    JpaConfig.java            ← JPA / Hibernate settings
    OpenApiConfig.java        ← Swagger UI / springdoc configuration
```

**Rules:**
- Technical modules may import from a domain module's `api/` package — never from `internal/`
- New cross-cutting concerns get their own technical module — do not add them to a domain module
- Domain modules must not import from `com.{company}.security` or `com.{company}.config`

---

## Spring Modulith Architecture Tests

Add one test class per module to verify boundary compliance:

```java
// src/test/java/com/{company}/ArchitectureTest.java
@SpringBootTest
class ArchitectureTest {

    @Test
    void verifiesModularStructure() {
        ApplicationModules modules = ApplicationModules.of(Application.class);
        modules.verify();   // fails if any module imports from another's internal/
    }
}
```

This test runs with every build. A boundary violation is a build failure — not a warning.

---

## Adding a New Domain Module — Step by Step

1. **Create the package** `com.{company}.{domain}` with the following subpackages:
   - `api/dto/`, `api/service/`, `api/exception/`, `api/enum/`
   - `internal/entity/`, `internal/repository/`, `internal/service/`, `internal/mapper/`, `internal/util/`, `internal/enum/`
   - `controller/`
2. **Define the request objects and response Views** in `api/dto/`
3. **Implement the controller** in `controller/` with explicit Spring MVC annotations
4. **Define the service interface** in `api/service/`
5. **Implement the service** in `internal/service/`, annotated `@Service @Transactional`
6. **Define the JPA entity** in `internal/entity/`
7. **Define the repository** in `internal/repository/` extending `JpaRepository`
8. **Write the mapper** in `internal/mapper/` (MapStruct interface)
9. **Add enums** — public-facing enums in `api/enum/`, internal-only enums in `internal/enum/`
10. **Add utilities** in `internal/util/` if needed
11. **Add a Liquibase changeset** if new tables are needed
12. **Write unit tests** for the service implementation
13. **Write an integration test** covering the controller → service → repository flow

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Importing from `internal/` of another module | Only import from `api/` — Modulith test will catch it |
| Putting business logic in a controller | Move all logic to the service |
| Creating a global `controllers/` folder | Controllers live in their domain module root |
| Returning a JPA entity from a service | Always map to a View before returning |
| Putting a public-facing enum in `internal/enum/` | If another module's View, request, or service references the enum, move it to `api/enum/` |
| Placing the controller inside `api/` or `internal/` | Controllers belong in `controller/` — a sibling to both, never nested inside either |
| Entity class name missing `Entity` suffix | Rename to end with `Entity` (e.g., `Queue` → `QueueEntryEntity`); update `@Table` separately |
| Handwritten mapper `@Component` class | Replace with a MapStruct `@Mapper(componentModel = "spring")` interface |
