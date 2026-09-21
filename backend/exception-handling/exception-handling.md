# Exception Handling — Implementation Guide

> Implementation guidance for the selected BetterF technology stack; examples do not define product requirements. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Error responsibilities

Domain services raise typed failures for expected business conditions. A shared transport handler translates those failures into the agreed HTTP contract. Unexpected failures retain their cause in server-side diagnostics and return a safe client message.

Keep domain-specific errors in the owning module. Shared error infrastructure belongs in a technical module with a small public interface.

## Stable identifiers and messages

Separate machine-readable codes from human-readable messages. Clients should branch on stable codes, not localized text. The project's code catalog and HTTP mappings must be defined with the API contract; no inherited catalog is authoritative.

An optional Java interface for error metadata is:

```java
public interface ApplicationError {
    String getCode();
    String getMessageKey();
}
```

Each domain can implement that interface with its own error definitions. A shared base exception can carry an error identifier, safe message arguments, and an underlying cause. These classes are a possible design, not existing BetterF infrastructure.

## Central translation

Use Spring MVC; `@RestControllerAdvice` and `@ExceptionHandler` can centralize translation. Cover domain errors, request validation, access failures, persistence failures, and unexpected exceptions. Do not map every database failure to a client conflict: distinguish known constraint conflicts from internal faults.

Coordinate errors raised before controller invocation with the security layer so they follow the agreed contract too.

## Localization and tracing

Resolve user-facing messages from the selected locale mechanism, with a documented fallback. Keep internal diagnostics separate. Use a correlation reference to associate client reports with logs without exposing stack traces, query text, credentials, or sensitive data.

No locale set, message-file catalog, reference format, or custom exception factory is prescribed here.

## Adding an error

1. Identify whether a domain or shared infrastructure owns it.
2. Define its stable code and transport meaning in the API contract.
3. Provide safe user-facing text and translations for supported locales.
4. Add central translation without duplicating response construction.
5. Test the observable HTTP status and response shape, including validation and unexpected failures.

See [response handling](../response-handling/response-handling.md) for response responsibilities.
