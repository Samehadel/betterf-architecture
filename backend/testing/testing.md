# Backend Testing — Implementation Guide

> Implementation guidance for the selected BetterF technology stack; examples do not define product requirements. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Test scope

| Type | Purpose |
|---|---|
| Unit | Business behavior with dependencies isolated |
| Integration | Real component wiring, persistence, serialization, and transaction behavior |
| Architecture | Enforce accepted module/dependency boundaries |

Organize tests so the corresponding production behavior is easy to find. Express intent with arrange/act/assert and reusable fixtures where setup repeats.

## Spring/JUnit patterns

- Mockito can isolate service dependencies without starting Spring.
- `@SpringBootTest` with `@AutoConfigureMockMvc` can exercise controller, service, and persistence integration.
- `@DataJpaTest` can focus tests on non-trivial persistence queries.
- Spring Modulith can verify configured application-module boundaries.

These tools are selected but not yet configured. Use AssertJ consistently for assertions.

## Integration fixtures

Use the narrowest realistic setup boundary: repositories for simple persistence state and public services when fixture creation must enforce business rules. Exercise the endpoint under test through HTTP without repeating long HTTP setup chains in every test.

Authentication fixtures must represent the selected authentication model. Cover success, invalid input, denied access, missing resources, and relevant conflicts. Verify serialized responses against the agreed API contract.

Use the production database engine for database-specific behavior. Define cleanup/isolation explicitly; transaction rollback alone does not cover every asynchronous or separately committed operation.

## Workflow

A useful test-first cycle is: describe behavior in a failing test, implement it, and refactor. Test behavior rather than implementation details. Resolve failing checks instead of suppressing them.

Add actual test commands, CI gates, and coverage thresholds once the application build exists. No fixed percentage or preconfigured build task is inherited.
