# Persistence & JPA — Reference Pattern

> Reusable reference, not an accepted BetterF technology or product decision. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Entities and repositories

If JPA is selected, keep entities and repositories inside the owning module. Expose public models through service interfaces rather than returning persistence entities across module boundaries.

- Use a consistent entity naming convention and explicit table mappings.
- Select identifiers and generation strategy to fit the database and workload.
- Persist enums deliberately; string values avoid coupling data to enum ordinal positions.
- Use an explicit timestamp/time-zone policy; `Instant` is suitable for points in time.
- Add audit fields where required and define who maintains them.
- Keep schema naming consistent with the migration definitions.

Spring Data repositories can extend `JpaRepository`. Use derived queries for simple access and explicit queries or projections for more complex reads. Keep database-specific queries isolated and tested.

## Projections and mapping

A read-only projection can fetch only the fields needed:

```java
public interface ResourceSummaryProjection {
    Long getId();
    String getName();
}
```

Keep entity-to-response mapping inside the module. MapStruct is an optional way to generate mapping code; shared mapper interfaces are not assumed to exist.

## Pagination and transactions

Bound reads on growing collections and define stable ordering. Keep transaction boundaries in the service/use-case layer. For Spring applications, use `@Transactional` for operations that must be atomic and consider `readOnly = true` for reads. Bulk modifying queries need appropriate transaction handling.

Avoid loading lazy associations during response serialization. A Spring/JPA configuration may disable open-in-view and validate the schema when migrations own schema changes:

```properties
spring.jpa.open-in-view=false
spring.jpa.hibernate.ddl-auto=validate
```

These are configuration examples, not existing project settings. Verify query behavior, constraints, and migrations against the selected database.
