# 0001 — Retain the Spring Boot and Angular technology baseline

- **Status:** Accepted
- **Date:** 2026-09-22
- **Authority:** Project owner confirmed that the inherited technology specifications are to be retained for BetterF.

## Context

The architecture documents were imported from another product. Its business model and runtime-specific behavior do not define BetterF requirements. Its reusable technology choices and engineering conventions are intended for this project.

## Decision

Retain Java/Spring Boot and Angular, together with the supporting stack and conventions documented in [backend architecture](../backend-architecture.md#technology-stack) and [frontend architecture](../frontend-architecture.md#technology-stack). Those tables are the authoritative technology lists.

Keep implementation guidance for those technologies active. Package versions must be pinned when the applications are initialized; documentation does not imply an existing implementation or configured CI enforcement.

## Consequences

Implementation follows the selected stack. Domain models, database tables, endpoint and route inventories, roles, authentication transport and session behavior, supported locales, and deployment topology require BetterF-specific design. Neutral examples must not silently establish those requirements.

Future changes to the technology baseline require an explicit superseding decision.
