# Testing — Implementation Guide

> Reference for TDD, unit tests with Mockito, integration tests with H2, and Spring Modulith boundary tests.

---

## TDD Cycle

1. Write a failing test describing the behavior
2. Write the minimum code to make it pass
3. Refactor without changing behavior
4. Repeat

A failing test is a broken build. Never write implementation code without a failing test first.

---

## Test Organization

Tests mirror the production package structure exactly. Unit tests live alongside the class they test; integration tests live at the module root.

```
src/
    main/java/com/{company}/queue/internal/QueueServiceImpl.java
    test/java/com/{company}/queue/internal/QueueServiceImplTest.java   ← unit test

    main/java/com/{company}/queue/QueueController.java
    test/java/com/{company}/queue/QueueIntegrationTest.java            ← integration test
```

---

## Unit Tests

Test a single service class in isolation. All dependencies mocked with Mockito. No Spring context, no database.

**Setup:**
```java
@ExtendWith(MockitoExtension.class)
class QueueServiceImplTest {
    @Mock QueueRepository queueRepository;
    @Mock BusinessService businessService;
    @InjectMocks QueueServiceImpl queueService;
}
```

**Rules:**
- Use BDD-style Mockito: `given(...).willReturn(...)`, `verify(...)`
- Assert with AssertJ: `assertThat(...)` — not JUnit `assertEquals`
- Test behavior, not implementation details
- Structure each test with Arrange / Act / Assert

---

## Integration Tests

Validate the full stack (controller → service → repository → H2) with a real Spring context.

**Setup:**
```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional          // rolls back after each test — keeps H2 clean
abstract class IntegrationTestSupport { ... }

class QueueControllerIntegrationTest extends IntegrationTestSupport { ... }
```

**Rules:**
- Prefer a shared `IntegrationTestSupport` base class for controller integration tests when fixture setup is repeated across test classes
- Name reusable fixtures with `given*` methods so setup reads like test intent rather than infrastructure
- Use `MockMvc` only for the endpoint under test
- Create authentication fixtures with the same token/cookie mechanisms the application uses instead of `@WithMockUser` when the endpoint depends on JWT-derived claims
- Seed state through the narrowest realistic boundary:
  - Use repositories for simple persistence setup that does not depend on domain rules
  - Use public module services when fixture creation must respect business rules or module orchestration
- Assert on the full `ApiResponse` shape using `jsonPath`
- Test happy path, auth failure (401/403), and validation failure (400) for each endpoint

**Shared support pattern:**
```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
abstract class IntegrationTestSupport {
    protected User givenBusinessOwner() { ... }
    protected QueueEntity givenOpenQueue(User owner) { ... }
    protected QueueCustomerEntity givenCustomerInQueue(QueueEntity queue) { ... }
    protected Cookie loginAs(User user) { ... }
}
```

This pattern is preferred when HTTP-based setup would otherwise be repeated across multiple controller integration tests.

---

## Repository Tests

Use `@DataJpaTest` for non-trivial queries — loads only the JPA slice, faster than `@SpringBootTest`.

```java
@DataJpaTest
class QueueRepositoryTest {
    @Autowired QueueRepository queueRepository;
}
```

---

## Spring Modulith Boundary Test

One test class for the whole application — verifies no module imports from another module's `internal/`.

```java
@SpringBootTest
class ArchitectureTest {
    @Test
    void verifiesModularStructure() {
        ApplicationModules.of(Application.class).verify();
    }
}
```

Never suppress this test. A boundary violation is a build failure.

---

## Coverage

JaCoCo enforces 85% line coverage — configured in `build.gradle` and runs as part of `check`.

```bash
./gradlew test jacocoTestReport
open build/reports/jacoco/test/html/index.html
```

---

## Gradle Commands

```bash
./gradlew test                                          # all tests
./gradlew test --tests "com.{company}.queue.internal.*"         # single package
./gradlew test --tests "com.{company}.queue.internal.QueueServiceImplTest.someMethod"
./gradlew test jacocoTestReport                         # with coverage
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Writing tests after implementation | TDD — test first |
| `@SpringBootTest` for unit tests | Use `@ExtendWith(MockitoExtension.class)` |
| Repeating the same controller-test setup helpers in multiple classes | Extract them into `IntegrationTestSupport` with `given*` fixtures |
| Building controller-test fixtures through chained HTTP calls | Seed state through repositories or public services, then use `MockMvc` only for the target endpoint |
| Using `@WithMockUser` when JWT claims or auth cookies affect behavior | Generate realistic auth fixtures through the same token/cookie path the app uses |
| Missing `@Transactional` on integration test class | Tests pollute H2 state across runs |
| `assertEquals` assertions | Use AssertJ `assertThat()` |
| No auth failure test for protected endpoints | Every protected endpoint needs a 401/403 test case |
| Suppressing the Modulith boundary test | Fix the violation — never disable the test |
