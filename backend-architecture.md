# Backend Architecture

## Status

This document retains reusable backend design guidance. BetterF's runtime, framework, deployment topology, domain model, database, authentication model, and API contract strategy are not yet established here. The technology-specific [backend guides](backend/README.md) are reference material until adopted through an accepted decision.

## Module boundaries

- Organize business behavior by domain, with technical roles grouped inside each domain.
- Expose a small public interface for cross-module collaboration. Keep persistence entities, repositories, mappers, and implementation classes private to the owning module.
- Access another module's data through its public interface, not its repository or internal tables.
- Keep cross-cutting infrastructure separate from business rules; shared code must have a clear responsibility.
- Add automated boundary checks when the selected tooling supports them. Do not claim checks are enforced until they exist in the build.

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

Use explicit contracts across module and system boundaries. Document dependencies, failure behavior, and transaction boundaries. Whether modules communicate synchronously, through events, or through external APIs remains a project decision.

Keep request/response models, endpoint behavior, and [API documentation](api-specification.md) aligned. Contract generation versus handwritten transport code remains undecided. Response wrapping is an optional implementation pattern, not an existing BetterF facility.

## Security

Centralize authentication integration and shared transport security configuration. Enforce resource permissions and ownership at the server-side use-case boundary, including calls that do not originate in HTTP controllers.

Public endpoints, roles, credentials, session lifetime, refresh behavior, and identity providers require BetterF-specific decisions. See the [security guide](backend/security/security.md) for the reusable review structure.

## Persistence and migrations

- Give each data model an explicit owner.
- Keep transaction boundaries aligned with use cases.
- Bound reads on collections that can grow without limit.
- Use versioned migrations for production schema changes; do not rely on implicit ORM schema mutation.
- Preserve applied migration history. Document recovery or rollback limitations and verify compatibility with the selected production database.

The schema, naming convention, identifier strategy, migration tool, and deployment execution model remain open.

## Testing

Test business behavior in isolation, integration behavior across real component boundaries, and module boundaries where tooling permits. Cover invalid input, denied access, missing resources, failure paths, and relevant concurrency behavior.

The test-first cycle remains useful: express behavior in a failing test, implement it, then refactor. Build commands, test databases, frameworks, and coverage thresholds must be defined from the actual application configuration rather than inherited assumptions.

## Decisions to establish

Before implementing the relevant area, record the selected stack, domain boundaries, communication strategy, data ownership, API contracts, security model, and runtime/deployment constraints in this repository. Significant choices belong in `decisions/`, with an explicit status and rationale.
