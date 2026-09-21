# API Conventions

## Status

BetterF's endpoints and wire contracts have not yet been defined. This document retains a checklist for designing consistent APIs; it does not establish an API base path, response envelope, pagination format, error catalog, authentication transport, or documentation URL.

## Endpoint design

For each endpoint, document:

- Purpose, resource ownership, HTTP method, path, and success status.
- Request parameters and body, validation constraints, and response shape.
- Authentication and authorization requirements, including resource access checks.
- Failure responses, retry behavior, and idempotency where relevant.
- Compatibility implications for existing clients.

Use HTTP methods and status codes consistently. A no-content response must not contain a response envelope.

## Collections

Define bounded pagination for growing collections. Choose page-based or cursor-based navigation explicitly, with defaults, limits, ordering, and response metadata. Document supported filters and sort fields; do not expose arbitrary persistence fields as an API contract.

## Errors and responses

Maintain one authoritative contract for success and error responses. Stable machine-readable error identifiers must be distinct from user-facing messages. Keep internal details out of client responses and provide a correlation reference when useful for diagnostics.

Reusable implementation considerations live in [response handling](backend/response-handling/response-handling.md) and [exception handling](backend/exception-handling/exception-handling.md). These guides do not define an existing BetterF response implementation.

## Contract workflow

Select contract-first generation or code-derived documentation in an architecture decision. Keep the chosen contract source, implementation, client models, and tests aligned in the same change. Add live documentation URLs only after they are configured and verified.
