# Project architecture

This repository holds the version-controlled architecture for `betterf`. Backend and frontend documentation describe their respective responsibilities; shared decisions explain choices that affect the system as a whole.

## Documentation status

This repository currently contains a reusable architecture baseline. BetterF-specific domains, schema, endpoints, routes, authentication behavior, deployment topology, and technology selections remain to be designed.

- The strategic documents retain general engineering guidance.
- Technology-specific backend and frontend guides are optional reference patterns, not accepted project decisions or evidence of an implementation.
- Examples do not define BetterF product requirements. Validate framework-specific APIs against selected versions before implementation.
- Record significant project choices in `decisions/` with **Proposed**, **Accepted**, or **Superseded** status. Accepted decisions govern the applicable implementation.

Start with the [context index](context-index.md), [backend architecture](backend-architecture.md), [frontend architecture](frontend-architecture.md), [API conventions](api-specification.md), and [development guide](development-guide.md).

## Workspace

```text
betterf/
├── AGENTS.md
├── architecture/       # This repository
│   ├── README.md
│   ├── backend/
│   ├── frontend/
│   └── decisions/
└── app/                # Separate repository containing both applications
    ├── backend/
    └── frontend/
```

Paths in this README are relative to the architecture repository unless stated otherwise. The directories above describe the intended organization; create them when adding their first document.

## System overview

Complete this section as the project's architecture is established:

- Purpose and principal user journeys: `<describe>`
- System boundaries and external dependencies: `<describe>`
- Backend responsibilities: `<describe>`
- Frontend responsibilities: `<describe>`
- Communication between frontend and backend: `<describe>`
- Deployment context and significant constraints: `<describe>`

Use the workspace's `project-context.md` to locate business requirements when configured. Otherwise, consult the source settings in the workspace's `AGENTS.md`. Link to authoritative requirements rather than maintaining competing copies here.

## Backend architecture

Start backend documentation in `backend/README.md`. Cover the topics relevant to the application:

- Services, modules, domain boundaries, and responsibilities.
- Business rule enforcement and authorization boundaries.
- API contracts, errors, and compatibility expectations.
- Data models, ownership, migrations, and transactions.
- External integrations, background work, and failure handling.
- Runtime configuration, deployment, observability, and testing strategy.

Keep implementation in the sibling application's `backend/` directory. Link to code or contracts where they clarify the design.

## Frontend architecture

Start frontend documentation in `frontend/README.md`. Cover the topics relevant to the application:

- Application structure, routes, and principal user journeys.
- Component boundaries and shared design conventions.
- State ownership, data fetching, caching, and API integration.
- Authentication flows and how permissions appear in the UI.
- Loading, empty, and error states.
- Accessibility, responsive behavior, testing, and deployment.

Keep implementation in the sibling application's `frontend/` directory. Document how the UI consumes backend contracts; backend authorization remains responsible for protecting server-side operations.

## Shared decisions

Store significant architecture decisions in `decisions/` using descriptive names such as `0001-api-contract-strategy.md`. Each decision should include its status, context, choice, consequences, and relevant business or technical references.

Use explicit statuses such as **Proposed**, **Accepted**, and **Superseded**. Preserve historical decisions and link superseded decisions to their replacements. Document shared contracts once and reference them from both backend and frontend documentation.

## Maintaining this repository

- Keep documentation aligned with approved changes; clearly label proposals and known implementation gaps.
- Explain why a design exists, including relevant constraints and tradeoffs.
- Add document links to this index as documents are created.
- Reference related application changes when a change spans both repositories.
- Use the branch, tag, or commit configured for the task or workspace. Record the commit consulted when reporting an architecture review.
- Keep credentials, generated logs, and private runtime data out of the repository.

Agent working instructions live in the workspace's `AGENTS.md`. This README is the entry point to the architecture itself.
