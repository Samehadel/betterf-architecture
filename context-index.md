# Context Index

## Start here

[README.md](README.md) is the repository entry point and defines [documentation status](README.md#documentation-status). Workspace instructions and business source locations live in the parent workspace's `AGENTS.md` and `project-context.md`.

The Spring Boot and Angular stack and its reusable technical conventions are selected for BetterF. See the accepted [technology baseline](decisions/0001-technology-baseline.md). Examples are not product requirements or evidence of installed tooling; product-specific choices remain open.

## Strategic documents

| Document | Responsibility |
|---|---|
| [Backend architecture](backend-architecture.md) | Boundaries, layering, data ownership, and backend decisions to establish |
| [Frontend architecture](frontend-architecture.md) | Feature organization, components, state, and frontend decisions to establish |
| [API conventions](api-specification.md) | Checklist for the future API contract |
| [Development guide](development-guide.md) | Setup status, workflow, and review checklist |

## Topic index

Read strategic guidance first, then the relevant implementation guide. Backend guides live in `backend/`; frontend guides live in `frontend/`.

| Topic | Read first | Reference |
|---|---|---|
| Module structure | [Backend architecture](backend-architecture.md) | [Guide](backend/module-structure/module-structure.md) |
| Persistence & JPA | [Backend architecture](backend-architecture.md) | [Guide](backend/database/persistence-jpa.md) |
| Database schema | [Backend architecture](backend-architecture.md) | [Guide](backend/database/database-schema.md) |
| Database migrations | [Backend architecture](backend-architecture.md) | [Guide](backend/database/database-migrations.md) |
| Security | [Backend architecture](backend-architecture.md) | [Guide](backend/security/security.md) |
| Exception handling | [Backend architecture](backend-architecture.md) | [Guide](backend/exception-handling/exception-handling.md) |
| Response handling | [Backend architecture](backend-architecture.md) | [Guide](backend/response-handling/response-handling.md) |
| Testing | [Backend architecture](backend-architecture.md) | [Guide](backend/testing/testing.md) |
| Angular signals | [Frontend architecture](frontend-architecture.md) | [Guide](frontend/angular-signals/angular-signals.md) |
| NgRx Signal Store | [Frontend architecture](frontend-architecture.md) | [Guide](frontend/ngrx-signal-store/ngrx-signal-store.md) |
| Standalone components | [Frontend architecture](frontend-architecture.md) | [Guide](frontend/standalone-components/standalone-components.md) |
| Routing | [Frontend architecture](frontend-architecture.md) | [Guide](frontend/routing/routing.md) |
| HTTP & interceptors | [Frontend architecture](frontend-architecture.md) | [Guide](frontend/http-interceptors/http-interceptors.md) |
| Tailwind styling | [Frontend architecture](frontend-architecture.md) | [Guide](frontend/tailwind-styling/tailwind-styling.md) |
| Internationalization | [Frontend architecture](frontend-architecture.md) | [Guide](frontend/i18n/i18n.md) |
| Testing | [Frontend architecture](frontend-architecture.md) | [Guide](frontend/testing/testing.md) |

## Maintenance

Keep rules and rationale in strategic documents, examples in guides, and significant decisions in `decisions/` with an explicit status. Add links when documents are created and remove links when documents are retired. Update implementation and contracts together according to the selected contract workflow.
