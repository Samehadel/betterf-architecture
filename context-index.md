# Context Index

## What This Is

This file is the navigation entry point for the `architecture/` directory. Before writing any code, use the task-based index below to find the exact documents that govern what you are about to do. Do not guess — look it up here first.

Every architecture document in this project is a binding constraint. Code must conform to the architecture, not the other way around. When the implementation diverges from a document, either the code is wrong or the document needs to be updated — one of them must change.

## Who Uses This and How

| Consumer | Entry point | Purpose |
|---|---|---|
| AI agent (Claude, etc.) | `CLAUDE.md` → `architecture/context-index.md` | Reads rules before generating any code; produces a session summary after |
| Human developer | `architecture/development-guide.md` | Setup, workflow, conventions |

## Document System Overview

The architecture directory is organized in two layers:

**Strategic layer** — `architecture/*.md`

The *what and why*: rules, boundaries, decisions. Every contributor reads these before touching code.

| Document | Owns |
|---|---|
| [backend-architecture.md](./backend-architecture.md) | Module structure, layering rules, inter-module communication, persistence, security, migrations, testing strategy |
| [frontend-architecture.md](./frontend-architecture.md) | Frontend structure, component model, state management, reactivity patterns, routing, HTTP, styling, testing strategy |
| [api-specification.md](./api-specification.md) | Endpoint conventions, pagination, filtering, error-code conventions, live API docs |
| [development-guide.md](./development-guide.md) | Local setup, build commands, code review checklist, IDE setup, troubleshooting |

**Tactical layer** — `architecture/backend-guides/`, `architecture/frontend-guides/`

The *how*: implementation patterns and code examples. Used while writing code, after reading the strategic layer.

- [Backend Guides](backend/README.md) — module structure, endpoint implementation, persistence, migrations, security, error handling, testing
- [Frontend Guides](frontend/README.md) — signals, NgRx Signal Store, routing, HTTP interceptors, styling, testing

Rules live in the strategic layer. Patterns live in the tactical layer. Never duplicate content across them — link instead.

---

## Task-Based Document Index

Use this table to find the exact documents for the task at hand. Read the strategic layer first — it defines the rules. Consult the tactical guide while writing code.

#### Backend Tasks

| Task | Read first (strategic) | Then consult (tactical) |
|---|---|---|
| Create a new domain module | [backend-architecture.md](./backend-architecture.md) | [backend-guides/module-structure/module-structure.md](backend/module-structure/module-structure.md) |
| Add or change an API endpoint | [api-specification.md](./api-specification.md), [backend-architecture.md](./backend-architecture.md) | [backend-guides/module-structure/module-structure.md](backend/module-structure/module-structure.md) |
| Add or change database tables/columns | [backend-architecture.md](./backend-architecture.md) | [backend-guides/database/database-schema.md](backend/database/database-schema.md), [backend-guides/database/database-migrations.md](backend/database/database-migrations.md) |
| Write JPA entities or repositories | [backend-architecture.md](./backend-architecture.md) | [backend-guides/database/persistence-jpa.md](backend/database/persistence-jpa.md) |
| Add or modify security rules | [backend-architecture.md](./backend-architecture.md) | [backend-guides/security/security.md](backend/security/security.md) |
| Add or change error codes / exception handling | [backend-architecture.md](./backend-architecture.md), [api-specification.md](./api-specification.md) | [backend-guides/exception-handling/exception-handling.md](backend/exception-handling/exception-handling.md) |
| Write backend tests | [backend-architecture.md](./backend-architecture.md) | [backend-guides/testing/testing.md](backend/testing/testing.md) |

#### Frontend Tasks

| Task | Read first (strategic) | Then consult (tactical) |
|---|---|---|
| Create a new component | [frontend-architecture.md](./frontend-architecture.md) | [frontend-guides/components/components.md](frontend/components/components.md) |
| Add or change feature state | [frontend-architecture.md](./frontend-architecture.md) | [frontend-guides/state-management/state-management.md](frontend/state-management/state-management.md) |
| Add or change routing | [frontend-architecture.md](./frontend-architecture.md) | [frontend-guides/routing/routing.md](frontend/routing/routing.md) |
| Consume a backend API | [api-specification.md](./api-specification.md), [frontend-architecture.md](./frontend-architecture.md) | [frontend-guides/http-communication/http-communication.md](frontend/http-communication/http-communication.md) |
| Apply or change UI styling | [frontend-architecture.md](./frontend-architecture.md) | [frontend-guides/styling/styling.md](frontend/styling/styling.md) |
| Use reactivity primitives | [frontend-architecture.md](./frontend-architecture.md) | [frontend-guides/reactivity/reactivity.md](frontend/reactivity/reactivity.md) |
| Write frontend tests | [frontend-architecture.md](./frontend-architecture.md) | [frontend-guides/testing/testing.md](frontend/testing/testing.md) |
| Add or translate UI strings (i18n) | [frontend-architecture.md](./frontend-architecture.md) | [frontend-guides/i18n/i18n.md](frontend/i18n/i18n.md) |

#### Cross-cutting Tasks

| Task | Read first (strategic) | Then consult (tactical) |
|---|---|---|
| Review a pull request | [development-guide.md](./development-guide.md) | — |
| Onboard to the project / local setup | [development-guide.md](./development-guide.md) | — |
| Design a new API contract | [api-specification.md](./api-specification.md), [backend-architecture.md](./backend-architecture.md) | [backend-guides/module-structure/module-structure.md](backend/module-structure/module-structure.md) |
| Debug unexpected behavior / tracing | [backend-architecture.md](./backend-architecture.md), [api-specification.md](./api-specification.md) | [backend-guides/exception-handling/exception-handling.md](backend/exception-handling/exception-handling.md) |

---

## How to Maintain the Docs

- Architecture rules change in the strategic layer (`architecture/*.md`) — never in the guides
- Implementation patterns change in the guides — never in the strategic layer
- If a rule is removed or changed, update it in one place only; do not duplicate rules across files
- The API spec (YAML) is always updated before any code change — docs follow the same discipline as code
- When a guide contradicts an architecture document, the architecture document wins; fix the guide
