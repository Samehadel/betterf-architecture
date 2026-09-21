# Module Structure — Reference Pattern

> Reusable reference, not an accepted BetterF technology or product decision. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Public and private boundaries

A domain-oriented Java module can separate its public contract, implementation, and HTTP adapter:

```text
{basePackage}.{domain}/
    api/
        dto/           request and response models
        service/       public service interfaces
        exception/     exceptions callers need to handle
        enums/         types exposed by the public contract
    internal/
        entity/
        repository/
        service/
        mapper/
        facade/        optional use-case orchestration
    controller/        HTTP adapters
```

Only the public contract is available to other modules. Keep controllers in `controller/`, a sibling of `api/` and `internal/`. Enums belong in the public contract only when public types reference them. Use a valid Java package name such as `enums`, not the reserved keyword `enum`.

## Responsibilities

Controllers parse and validate transport input, delegate to a service or facade, and return a public response model. Services enforce resource access, existence, state transitions, and business invariants. Repositories perform data access. Mappers keep persistence representations separate from public models.

An optional internal facade can resolve caller context or compose service calls. Keep domain rules in services and make required identity/context arguments explicit.

Technical modules contain shared infrastructure rather than product behavior. Their dependencies on domains should use public interfaces.

## Adding a module

1. Identify its responsibility and data ownership from approved requirements.
2. Define the smallest public interface needed by its callers.
3. Keep entities, repositories, services, and mappers behind that interface.
4. Add transport adapters only for required endpoints.
5. Add migrations and tests for the behavior being introduced.
6. Verify dependency boundaries with the selected architecture-test tooling.

Spring Modulith is one possible enforcement tool. Its module and named-interface configuration must match the chosen package structure; a directory named `api` alone is not proof of enforcement.

Base entity classes, response marker interfaces, mapper libraries, and package prefixes must be selected explicitly rather than assumed to exist.
