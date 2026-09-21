# Response Handling — Reference Pattern

> Reusable reference, not an accepted BetterF technology or product decision. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Consistent contracts

Use a documented response contract across endpoints and keep frontend models aligned with it. Choose direct resource responses or a shared envelope explicitly; neither is established for BetterF yet.

If an envelope is selected, define payload, error, pagination, and diagnostic fields in [API conventions](../../api-specification.md). Document null handling, timestamp format, serialization names, and examples once rather than maintaining competing definitions.

## Optional centralized wrapping

In Spring MVC, `ResponseBodyAdvice` is one possible place to apply a shared envelope. Controllers return transport models and delegate domain behavior to services. Services should not build HTTP envelopes.

If wrapping is adopted, specify and test:

- Which controllers and content types it applies to.
- How already-wrapped responses avoid double wrapping.
- How status codes and response headers are preserved.
- How downloads, streams, and other non-JSON responses bypass wrapping.
- How no-content responses remain body-free.
- How exception and security handlers use the same error contract.

An exclusion annotation and response builder can be implementation conveniences, but no such classes are assumed to exist.

## Validation

Test actual serialized HTTP responses for successful requests, validation failures, denied access, internal errors, and exclusions. Verify consistency with the client models and API documentation.

See [exception handling](../exception-handling/exception-handling.md) for typed failures, localization, and correlation references.
