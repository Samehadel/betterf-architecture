# Development Guide

## Current status

The workspace separates architecture documentation from application code. Application setup commands, runtime versions, service dependencies, and CI checks are not yet documented. Do not run placeholder commands or assume inherited build plugins are installed.

Read the workspace `AGENTS.md` and `project-context.md` for working instructions and source locations. Start architecture navigation at [README.md](README.md) or the [context index](context-index.md).

## Setup documentation to add

When the application is initialized, document verified prerequisites, dependency installation, environment configuration, required services, startup commands, health checks, and troubleshooting. Keep secrets out of committed configuration; provide example variable names instead of credentials.

Record commands separately for `app/backend/` and `app/frontend/`, relative to the workspace root. Discover them from the actual build configuration and CI.

## Development workflow

1. Read the relevant requirements and accepted architecture decisions.
2. Identify the owning module or feature and any affected contracts.
3. Implement the change with appropriate behavior and integration checks.
4. Update affected contracts, migrations, and architecture documentation.
5. Run the checks configured for the changed application area and report results.

## Code review checklist

- Boundaries and responsibilities follow accepted architecture.
- Tests cover meaningful behavior and failure paths relevant to the change.
- Shared API behavior and client models remain consistent.
- Schema changes use versioned migrations with a documented recovery strategy.
- Server-side permissions protect affected operations.
- Errors use consistent handling and avoid exposing internal details.
- UI changes handle relevant loading, empty, error, permission, and accessibility states.
- Configuration contains no credentials or private runtime data.
- Validation results and any remaining limitations are clear.

Branch, commit, and publishing instructions belong in the workspace `AGENTS.md`; this guide does not introduce a separate Git policy.
