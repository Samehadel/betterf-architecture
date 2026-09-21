# Persistence & JPA — Implementation Guide

> Reference for JPA entities, Spring Data repositories, and data access conventions.

---

## Entities

Always in `internal/entity/`. Never exposed outside the module.

**Rules:**
- **Entity class names must end with the `Entity` suffix** (e.g., `CustomerAccountEntity`, `QueueEntryEntity`). The database table name is set via `@Table(name = "...")` and is independent of the class name.
- Table and column names use `UPPER_CASE`; Java fields use `camelCase`
- Tables are prefixed by module name (`ORDERS`, `PRODUCTS`)
- Use `GenerationType.IDENTITY` (PostgreSQL `BIGSERIAL`) for primary keys — provides sequential `Long` IDs with zero fragmentation
- Use `EnumType.STRING` for enum columns — never `EnumType.ORDINAL`
- Use `Instant` for timestamps — not `LocalDateTime` or `Date`
- Use Lombok (`@Builder`, `@Getter`, `@Setter`, `@NoArgsConstructor`, `@AllArgsConstructor`) — no manual boilerplate
- Include `CREATED_AT` and `UPDATED_AT` audit columns on every table

---

## Repositories

Always in `internal/repository/`, extend `JpaRepository`.

**Rules:**
- Use Spring Data derived query methods for simple queries
- Use JPQL (`@Query`) for multi-condition queries — not native SQL
- Use projections for read-only queries that need only a subset of fields
- `@Modifying` is required for `UPDATE`/`DELETE` queries — the calling service method must be `@Transactional`
- Never call `findAll()` without `Pageable` on tables that grow unbounded
- Never call a repository from outside its module

---

## Projections

Use interface projections for lightweight reads — avoids loading the full entity.

```java
public interface OrderSummaryProjection {
    int getSequenceNumber();
    int getEstimatedWaitMinutes();
    String getStatus();
}
```

---

## Mappers

MapStruct `@Mapper(componentModel = "spring")` interfaces in `internal/mapper/`. Convert entities to Views or other API-facing models. Called by the service, never by the controller.

**Rules:**
- Mappers must always be MapStruct interfaces — never plain `@Component` classes with handwritten field assignment
- Mappers are always `internal` — never part of the module's public API
- Never return a JPA entity from a service method — always map to a View or other API-facing model first
- One mapper per module is usually sufficient
- Naming convention: `{Domain}Mapper.java` (e.g., `QueueMapper`, `BusinessProfileMapper`)

---

## Pagination

All list endpoints accept `Pageable` — Spring MVC auto-populates it from `?page=0&size=20&sort=position,asc`.

```java
Page<OrderEntity> findByOwnerIdAndStatus(String ownerId, OrderStatus status, Pageable pageable);
```

---

## Transactions

- `@Transactional` on service methods — not on controllers or repositories
- Mark read-only methods with `@Transactional(readOnly = true)`
- `@Modifying` queries require `@Transactional` on the calling service method

---

## Configuration

```properties
spring.jpa.hibernate.ddl-auto=validate   # Liquibase owns the schema — never use update
spring.jpa.open-in-view=false            # Prevents lazy loading outside a transaction
spring.jpa.show-sql=false                # Enable only in local profile
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Returning an entity from a service | Map to a View or API-facing model first |
| `EnumType.ORDINAL` | Use `EnumType.STRING` |
| `LocalDateTime` for timestamps | Use `Instant` |
| `findAll()` without pagination | Always use `Pageable` on unbounded tables |
| Native SQL queries | Use JPQL with `@Query` |
| `ddl-auto=update` | Use `validate` — Liquibase manages the schema |
| Calling another module's repository | Go through the owning module's service interface |
| Entity class name missing `Entity` suffix | Rename to end with `Entity`; set the DB table name via `@Table(name = "...")` |
| Handwritten mapper `@Component` class | Replace with a MapStruct `@Mapper(componentModel = "spring")` interface |
