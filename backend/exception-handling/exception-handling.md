# Exception Handling

This document is the single reference for how exceptions are structured, thrown, localized, and surfaced to API clients in this Spring Boot application.

The exception infrastructure lives in the `common` **technical module** (`com.{company}.common`). Domain modules define their own typed exceptions in their `api/` package — see [Module Placement](#module-placement).

---

## Overview

The exception system is built around a single abstract base class (`BaseException`) that ties together three concerns:

1. **Fixed error code** — a stable, machine-readable identifier (e.g., `AUTH_001`) sent to clients
2. **Localized message** — a human-readable string resolved at runtime from `messages.properties` using the request locale
3. **HTTP status** — the appropriate HTTP response code for the error category

A `GlobalExceptionHandler` catches every exception and converts it into a unified `ApiResponse`.

---

## Module Placement

### Common module — exception infrastructure

The exception infrastructure lives in the `common` technical module. Technical modules have no `api/`/`internal/` split — everything they contain is available app-wide.

```
com.{company}.common/
    exception/
        BaseException.java
        AuthorizationException.java
        InvalidCredentialsException.java
        TokenException.java
        ValidationException.java
        ResourceNotFoundException.java
        ResourceAlreadyExistsException.java
        InternalException.java
        UnavailableServiceException.java
        handler/
            GlobalExceptionHandler.java
    enums/
        ApplicationError.java        ← interface (the contract)
        CommonApplicationError.java  ← enum, implements ApplicationError
    service/
        ExceptionService.java
        ResponseBuilderService.java
        MessageSourceService.java
```

### Domain modules — module-specific exceptions

When a domain module needs its own typed exception that callers are expected to handle, define it in `api/` extending `BaseException`.

```
com.{company}.{domain}/
    api/
        {Domain}Exception.java          ← extends BaseException, public surface of the module
        {Domain}ApplicationError.java   ← enum, implements ApplicationError
    internal/
        ...
```

**Decision rule:**
| Artifact | Cross-cutting concern | Domain-specific |
|---|---|---|
| Exception class | `com.{company}.common.exception/` | `com.{company}.{domain}/api/` |
| Error enum | `CommonApplicationError` in `com.{company}.common.enums/` | `{Domain}ApplicationError` in `com.{company}.{domain}/api/` |

Domain modules import `BaseException` and the `ApplicationError` interface from `com.{company}.common` — neither is in an `internal/` package, so there is no Modulith boundary violation.

---

## Exception Hierarchy

All classes in the hierarchy below live in `com.{company}.common.exception`.

```
RuntimeException
└── BaseException (abstract)
    ├── AuthorizationException         → 403 Forbidden
    ├── InvalidCredentialsException    → 401 Unauthorized
    ├── TokenException                 → 401 Unauthorized
    ├── ValidationException            → 400 Bad Request
    ├── ResourceNotFoundException      → 404 Not Found
    ├── ResourceAlreadyExistsException → 409 Conflict
    ├── InternalException              → 500 Internal Server Error
    └── UnavailableServiceException    → 500 Internal Server Error
```

---

## BaseException

**Location**: `com.{company}.common.exception.BaseException`

All application exceptions extend this class. It stores everything the global handler needs to build an error response.

```java
public abstract class BaseException extends RuntimeException {
    private final ApplicationError applicationError; // interface — accepts any implementing enum
    private final String reference;                  // unique trackable error ID (nullable)
    private final HttpStatus status;                 // HTTP status to return
    private final Object[] args;                     // interpolation args for the localized message
    private final String message;                    // internal log message (not sent to client)
    private final Exception cause;                   // underlying exception (nullable)
}
```

The `message` field is built from the error enum's fallback English string and is used **only in server-side logging**. The client always receives the **localized message** resolved through `MessageSourceService` at response-build time.

```java
public String getCode() {
    return applicationError.getCode(); // e.g. "AUTH_001"
}
```

---

## ApplicationError Interface & Error Enums

### ApplicationError interface

**Location**: `com.{company}.common.enums.ApplicationError`

`ApplicationError` is an **interface**, not an enum. It defines the contract that all error enums — in `common` and in domain modules — must fulfill:

```java
public interface ApplicationError {
    String getCode();        // e.g. "AUTH_001" — sent to the client
    String getMessageKey();  // key in messages.properties for localization
    String getMessage();     // fallback English string for server-side logging only
}
```

`BaseException` accepts `ApplicationError` (the interface), so any implementing enum — from `common` or from any domain module — can be passed to an exception.

### CommonApplicationError enum

**Location**: `com.{company}.common.enums.CommonApplicationError`

The `common` module's implementation of `ApplicationError`. Defines all cross-cutting error codes:

```java
public enum CommonApplicationError implements ApplicationError {
    INVALID_CREDENTIALS("AUTH_001", "auth.login.failed", "Invalid credentials. [%s]"),
    //                   ^code       ^messageKey           ^internal fallback (log only)
    ;

    private final String code;
    private final String messageKey;
    private final String message;

    @Override public String getCode() { return code; }
    @Override public String getMessageKey() { return messageKey; }
    @Override public String getMessage() { return message; }
}
```

| Field | Purpose |
|---|---|
| `code` | Fixed string sent to the client in `ApiResponse.code` |
| `messageKey` | Key looked up in `messages.properties` for localization |
| `message` | Fallback English message for server-side logging only |

### Full Error Catalog (`CommonApplicationError`)

#### Authentication (`AUTH_*`)

| Code | Enum | HTTP Status |
|---|---|---|
| `AUTH_001` | `INVALID_CREDENTIALS` | 401 |
| `AUTH_002` | `ACCESS_DENIED` | 403 |
| `AUTH_003` | `TOKEN_EXPIRED` | 401 |
| `AUTH_004` | `INVALID_TOKEN` | 401 |
| `AUTH_005` | `TOKEN_MISMATCH` | 401 |
| `AUTH_006` | `TOKEN_IDLE` | 401 |
| `AUTH_007` | `UNAUTHORIZED_DATA_ACCESS` | 403 |
| `AUTH_008` | `UNAUTHORIZED` | 403 |
| `AUTH_009` | `ACTION_UNAUTHORIZED` | 403 |

#### Validation (`VAL_*`)

| Code | Enum | HTTP Status |
|---|---|---|
| `VAL_001` | `VALIDATION_ERROR` | 400 |
| `VAL_002` | `MISSING_REQUIRED_FIELD` | 400 |
| `VAL_003` | `VALIDATION_ERROR_WITH_MESSAGE` | 400 |

#### Resource (`RES_*`)

| Code | Enum | HTTP Status |
|---|---|---|
| `RES_001` | `RESOURCE_NOT_FOUND` | 404 |
| `RES_002` | `RESOURCE_ALREADY_EXISTS` | 409 |

#### System (`SYS_*`)

| Code | Enum | HTTP Status |
|---|---|---|
| `SYS_001` | `INTERNAL_ERROR` | 500 |
| `SYS_002` | `INVALID_ALGORITHM` | 500 |
| `SYS_003` | `SYS_REQUIRED_FIELD` | 500 |
| `SYS_004` | `CONFIGURATION_ERROR` | 500 |
| `SYS_005` | `EMAIL_SEND_FAILED` | 500 |
| `SYS_006` | `TEMPLATE_ERROR` | 500 |

---

## Localization

### How it works

```
request (Accept-Language: ar)
    → GlobalExceptionHandler
    → ResponseBuilderService.error(applicationError, reference, args)
    → MessageSourceServiceImpl.getMessage(messageKey, args)
    → Spring MessageSource → messages_ar.properties
    → localized string placed in ApiResponse.message
```

`MessageSourceServiceImpl` always uses `LocaleContextHolder.getLocale()`, which Spring populates from the request's `Accept-Language` header.

The `MessageSourceService` interface takes an explicit `Object[]` (not varargs) for its args parameter:

```java
public interface MessageSourceService {
    String getMessage(String key, Object[] args);

    default String getMessage(String key) {
        return getMessage(key, new Object[0]);
    }
}
```

Using `Object[]` rather than `Object...` avoids Java method resolution ambiguities when `null` is passed and makes Mockito stubbing straightforward in tests (`any(Object[].class)` matches all calls).

### Message files

| File | Locale |
|---|---|
| `src/main/resources/messages.properties` | Default (English) |
| `src/main/resources/messages_ar.properties` | Arabic (create when needed) |

### Current message definitions

```properties
# Authentication
auth.login.failed=Invalid credentials
auth.access.denied=Access denied to this resource
auth.token.expired=Token expired
auth.token.invalid=Invalid token
auth.token.mismatch=Token mismatch
auth.token.idle=Token is idle
auth.unauthorized.data.access=Unauthorized data access
auth.unauthorized=Unauthorized request
auth.action.unauthorized=UnauthorizedComponent action

# Validation
validation.error=Validation error
validation.error.with.message={0}
validation.missing.required.field=Missing required field {0}

# Resource
resource.not.found=Resource {0} not found with value {1}
resource.already.exists=Resource [{0}] already exists with value [{1}]

# System
system.error=An unexpected error occurred [{0}]
system.invalid.algorithm=Invalid algorithm
system.required.field={0} is required
system.configuration.error=An unexpected configuration error occurred for key {0}
system.email.send.failed=Email send failed
system.template.error=Error processing template: {0}
```

Placeholders `{0}`, `{1}`, ... are filled with the `args` array passed into the exception at throw time.

### Adding a new locale

Create `src/main/resources/messages_<locale>.properties` (e.g., `messages_ar.properties`) with the same keys and translated values. Spring resolves it automatically from the request locale — no code changes required.

---

## Exception Classes

### AuthorizationException

**HTTP**: `403 Forbidden` | **Reference**: none

Used when the user is authenticated but not permitted to perform an action.

```java
throw new AuthorizationException(CommonApplicationError.UNAUTHORIZED_DATA_ACCESS);
throw new AuthorizationException(CommonApplicationError.ACTION_UNAUTHORIZED);
```

---

### InvalidCredentialsException

**HTTP**: `401 Unauthorized` | **Reference**: none

No factory methods — instantiated directly.

```java
throw new InvalidCredentialsException();
// → code: AUTH_001
```

---

### TokenException

**HTTP**: `401 Unauthorized` | **Reference**: none

Carries an additional `tokenType` field (`ACCESS` or `REFRESH`) that is logged by the handler.

| Factory method | ApplicationError |
|---|---|
| `TokenException.invalid(tokenType)` | `INVALID_TOKEN` (AUTH_004) |
| `TokenException.expired(tokenType)` | `TOKEN_EXPIRED` (AUTH_003) |
| `TokenException.idle(tokenType)` | `TOKEN_IDLE` (AUTH_006) |
| `TokenException.mismatch(tokenType)` | `TOKEN_MISMATCH` (AUTH_005) |
| `TokenException.withMessage(tokenType, message)` | `INVALID_TOKEN` with a custom arg |

```java
throw TokenException.expired(TokenType.ACCESS);
throw TokenException.withMessage(TokenType.REFRESH, "Token family invalidated");
```

---

### ValidationException

**HTTP**: `400 Bad Request` | **Reference**: none

| Factory method | ApplicationError | Arg |
|---|---|---|
| `ValidationException.missingProperty(property)` | `MISSING_REQUIRED_FIELD` (VAL_002) | field name |
| `ValidationException.withMessage(message)` | `VALIDATION_ERROR_WITH_MESSAGE` (VAL_003) | free-form message |

```java
throw ValidationException.missingProperty("email");
// → message: "Missing required field email"

throw ValidationException.withMessage("Start date must be before end date");
// → message: "Start date must be before end date"
```

---

### ResourceNotFoundException

**HTTP**: `404 Not Found` | **Reference**: none

Always uses `RESOURCE_NOT_FOUND` (RES_001). Takes a resource name and its identifier as args.

```java
throw ResourceNotFoundException.withResource("User", userId);
// → message: "Resource User not found with value 42"
```

---

### ResourceAlreadyExistsException

**HTTP**: `409 Conflict` | **Reference**: none

Always uses `RESOURCE_ALREADY_EXISTS` (RES_002). Takes a field name and its value as args.

```java
throw ResourceAlreadyExistsException.withResource("email", "user@example.com");
// → message: "Resource [email] already exists with value [user@example.com]"
```

---

### InternalException

**HTTP**: `500 Internal Server Error` | **Reference**: auto-generated 10-digit numeric string

All factory methods automatically generate an error reference via `RandomStringGenerator.generateErrorReverence()`. This reference is included in the response and logged for traceability.

| Factory method | When to use |
|---|---|
| `InternalException.genericInternal()` | Generic 500, no extra context |
| `InternalException.internal(applicationError, args...)` | Specific system error with args |
| `InternalException.internal(applicationError, cause, args...)` | Wrapping a caught exception with context |
| `InternalException.internal(cause)` | Wrapping a caught exception generically |

```java
throw InternalException.genericInternal();
// → code: SYS_001, reference: "3847291056"

throw InternalException.internal(CommonApplicationError.EMAIL_SEND_FAILED, smtpException);
// → code: SYS_005, reference auto-generated, cause logged

throw InternalException.internal(CommonApplicationError.CONFIGURATION_ERROR, "jwt.secret");
// → code: SYS_004, message: "An unexpected configuration error occurred for key jwt.secret"
```

---

### UnavailableServiceException

**HTTP**: `500 Internal Server Error` | **Reference**: auto-generated

Used when an external or downstream service is unreachable.

```java
throw UnavailableServiceException.unavailableService("EmailService");
// → reference auto-generated, serviceName interpolated into message
```

---

## ExceptionService — Convenience Builders

**Location**: `com.{company}.common.service.ExceptionService`

`ExceptionService` provides static inner builder classes that wrap the exception factory methods. Prefer these in application code for consistency and discoverability.

### TokenExceptionBuilder

```java
ExceptionService.TokenExceptionBuilder.invalidRefreshTokenException()
ExceptionService.TokenExceptionBuilder.invalidTokenException(tokenType, message)
ExceptionService.TokenExceptionBuilder.expiredTokenException(tokenType)
ExceptionService.TokenExceptionBuilder.expiredRefreshTokenException()
ExceptionService.TokenExceptionBuilder.expiredAccessTokenException()
```

### ValidationExceptionBuilder

```java
ExceptionService.ValidationExceptionBuilder.validationException(property)         // → missingProperty
ExceptionService.ValidationExceptionBuilder.validationExceptionWithMessage(message)
```

### InternalExceptionBuilder

```java
ExceptionService.InternalExceptionBuilder.genericInternalExceptionWithReference()
ExceptionService.InternalExceptionBuilder.internalExceptionWithReference(applicationError, args...)
ExceptionService.InternalExceptionBuilder.internalExceptionWithReference(applicationError, cause, args...)
ExceptionService.InternalExceptionBuilder.internalExceptionWithReference(cause)
```

### ResourceExceptionBuilder

```java
ExceptionService.ResourceExceptionBuilder.resourceNotFoundException(resource, id)
ExceptionService.ResourceExceptionBuilder.resourceAlreadyExistsException(resource, value)
```

### AuthorizationExceptionBuilder

```java
ExceptionService.AuthorizationExceptionBuilder.authorizationException(applicationError)
```

### Usage examples

```java
// Token
throw ExceptionService.TokenExceptionBuilder.expiredAccessTokenException();
throw ExceptionService.TokenExceptionBuilder.invalidTokenException(TokenType.REFRESH, "Invalid format");

// Validation
throw ExceptionService.ValidationExceptionBuilder.validationException("email");
throw ExceptionService.ValidationExceptionBuilder.validationExceptionWithMessage("Custom rule violated");

// Internal
throw ExceptionService.InternalExceptionBuilder.internalExceptionWithReference(
    CommonApplicationError.EMAIL_SEND_FAILED, smtpException, "user@example.com"
);

// Resource
throw ExceptionService.ResourceExceptionBuilder.resourceNotFoundException("User", userId);
throw ExceptionService.ResourceExceptionBuilder.resourceAlreadyExistsException("email", email);

// Authorization
throw ExceptionService.AuthorizationExceptionBuilder.authorizationException(CommonApplicationError.ACCESS_DENIED);
```

---

## Error Reference

`InternalException` and `UnavailableServiceException` auto-generate a reference using `RandomStringGenerator.generateErrorReverence()`, producing a 10-digit random numeric string (e.g., `"3847291056"`).

This reference is:
- Included in `ApiResponse.errorReference` returned to the client
- Logged alongside the full exception stack trace on the server

Clients can supply this reference in bug reports for precise log lookup without exposing internal details.

---

## GlobalExceptionHandler

**Location**: `com.{company}.common.exception.handler.GlobalExceptionHandler`

A `@RestControllerAdvice` class with a dedicated `@ExceptionHandler` per exception type. Each handler:
1. Logs the error (message + reference + full stack trace)
2. Calls `ResponseBuilderService` to build the `ApiResponse` with the localized message
3. Returns `ResponseEntity` with the appropriate HTTP status

### Handler dispatch table

| Exception | HTTP Status | Notes |
|---|---|---|
| `AuthorizationException` | From exception | Uses `ex.getArgs()` for message interpolation |
| `TokenException` | `401` | Also logs `tokenType` |
| `InvalidCredentialsException` | `401` | |
| `ValidationException` | `400` | |
| `ResourceNotFoundException` | `404` | |
| `ResourceAlreadyExistsException` | `409` | |
| `InternalException` | `500` | |
| `BaseException` (catch-all) | From exception | Catches any unregistered subclass |
| `MethodArgumentNotValidException` | `400` | Bean Validation (`@Valid`) — collects all field errors into `errors[]` |
| `ConstraintViolationException` | `400` | Bean Validation on method params |
| `AccessDeniedException` | `403` | Spring Security — mapped to `AUTH_002` |
| `DataIntegrityViolationException` | `500` | DB constraint violation — auto-generates reference |
| `InvalidDataAccessResourceUsageException` | `500` | Bad query / schema mismatch — auto-generates reference |
| `Exception` (catch-all) | `500` | Last resort — logs full stack |

---

## ApiResponse Error Shape

All exception handlers return `ApiResponse<Void>`:

```java
@Data @Builder
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {
    private ResponseStatus status;       // "SUCCESS" or "FAILED"
    private String code;                 // e.g. "AUTH_001"
    private String errorReference;       // trackable reference (only for internal errors)
    private String message;              // localized human-readable message
    private T data;                      // null for errors
    private List<String> errors;         // populated for validation errors
    private final LocalDateTime timestamp;
}
```

**Standard error response**:

```json
{
  "status": "FAILED",
  "code": "RES_001",
  "message": "Resource User not found with value 42",
  "timestamp": "2024-02-14T10:30:00"
}
```

**Internal error response** (includes `errorReference`):

```json
{
  "status": "FAILED",
  "code": "SYS_001",
  "message": "An unexpected error occurred",
  "errorReference": "3847291056",
  "timestamp": "2024-02-14T10:30:00"
}
```

**Validation error response** (uses `errors[]` instead of `message`):

```json
{
  "status": "FAILED",
  "code": "VAL_001",
  "errors": [
    "email: must not be blank",
    "password: size must be between 8 and 64"
  ],
  "timestamp": "2024-02-14T10:30:00"
}
```

---

## When to Use Which Exception

| Scenario | Exception |
|---|---|
| Login or password check fails | `InvalidCredentialsException` |
| JWT is expired / invalid / idle / mismatched | `TokenException.*` factory |
| User lacks permission to access a resource | `AuthorizationException` |
| Required request field is missing | `ValidationException.missingProperty(field)` |
| Custom business rule violated | `ValidationException.withMessage(message)` |
| Database record not found | `ResourceNotFoundException.withResource(name, id)` |
| Duplicate creation attempt | `ResourceAlreadyExistsException.withResource(field, value)` |
| External service unreachable | `UnavailableServiceException.unavailableService(name)` |
| Unexpected system error | `InternalException.genericInternal()` |
| Caught exception wrapping | `InternalException.internal(cause)` |
| Specific system error with context | `InternalException.internal(applicationError, cause, args...)` |

---

## Adding a New Exception Type

Determine which path applies before writing any code:

- **Path A** — Cross-cutting concern (auth, validation, resource, system): exception + error code go in `com.{company}.common`
- **Path B** — Concern owned by a single domain module: exception + error code go in `com.{company}.{domain}/api/`

---

### Path A — Cross-cutting exception in `common`

**Step 1** — Add an entry to `CommonApplicationError`:

```java
PAYMENT_FAILED("PAY_001", "payment.failed", "Payment processing failed. [%s]"),
```

**Step 2** — Add the message key to `messages.properties` (and any locale files):

```properties
payment.failed=Payment processing failed: {0}
```

**Step 3** — Create the exception class in `com.{company}.common.exception/`:

```java
public class PaymentException extends BaseException {
    private PaymentException(String reason) {
        super(CommonApplicationError.PAYMENT_FAILED, HttpStatus.PAYMENT_REQUIRED, null, null, reason);
    }

    public static PaymentException declined(String reason) {
        return new PaymentException(reason);
    }
}
```

**Step 4** — Add a builder in `ExceptionService` (optional, for consistency):

```java
public static class PaymentExceptionBuilder {
    public static PaymentException declinedException(String reason) {
        return PaymentException.declined(reason);
    }
}
```

**Step 5** — Add a handler in `GlobalExceptionHandler`:

```java
@ExceptionHandler(PaymentException.class)
public ResponseEntity<ApiResponse<Void>> handlePaymentException(final PaymentException ex) {
    log.error("Payment exception: [{}] with reference: [{}]", ex.getMessage(), ex.getReference(), ex);
    return ResponseEntity
        .status(ex.getStatus())
        .body(buildErrorResponse(ex));
}
```

---

### Path B — Domain-specific exception in `{domain}/api/`

**Step 1** — Create `{Domain}ApplicationError` enum in the domain's `api/` package, implementing `ApplicationError`:

```java
// com.{company}.order/api/OrderApplicationError.java
public enum OrderApplicationError implements ApplicationError {
    ORDER_LIMIT_EXCEEDED("ORD_001", "order.limit.exceeded", "Order limit exceeded. [%s]"),
    ;

    private final String code;
    private final String messageKey;
    private final String message;

    @Override public String getCode() { return code; }
    @Override public String getMessageKey() { return messageKey; }
    @Override public String getMessage() { return message; }
}
```

**Step 2** — Add the message key to `messages.properties`:

```properties
order.limit.exceeded=Order limit exceeded: {0}
```

**Step 3** — Create the exception in the domain module's `api/` package:

```java
// com.{company}.order/api/OrderException.java
public class OrderException extends BaseException {
    private OrderException(OrderApplicationError error, String arg) {
        super(error, HttpStatus.UNPROCESSABLE_ENTITY, null, null, arg);
    }

    public static OrderException limitExceeded(String limit) {
        return new OrderException(OrderApplicationError.ORDER_LIMIT_EXCEEDED, limit);
    }
}
```

**Step 4** — Add a handler in `GlobalExceptionHandler`:

```java
@ExceptionHandler(OrderException.class)
public ResponseEntity<ApiResponse<Void>> handleOrderException(final OrderException ex) {
    log.error("Order exception: [{}]", ex.getMessage(), ex);
    return ResponseEntity
        .status(ex.getStatus())
        .body(buildErrorResponse(ex));
}
```

> No `ExceptionService` builder is needed for domain exceptions — they are thrown from within the owning module only.

---

## Testing

### Unit testing exception construction

```java
@Test
void tokenException_shouldHaveCorrectFields() {
    TokenException ex = TokenException.expired(TokenType.ACCESS);

    assertEquals(HttpStatus.UNAUTHORIZED, ex.getStatus());
    assertEquals(CommonApplicationError.TOKEN_EXPIRED, ex.getApplicationError());
    assertEquals("AUTH_003", ex.getCode());
    assertEquals(TokenType.ACCESS, ex.getTokenType());
}
```

### Unit testing the global handler

```java
@Test
void handleTokenException_shouldReturn401WithCorrectCode() {
    TokenException ex = TokenException.expired(TokenType.ACCESS);

    ResponseEntity<ApiResponse<Void>> response = globalExceptionHandler.handleTokenException(ex);

    assertEquals(HttpStatus.UNAUTHORIZED, response.getStatusCode());
    assertEquals(ResponseStatus.FAILED, response.getBody().getStatus());
    assertEquals("AUTH_003", response.getBody().getCode());
}
```

### Integration testing

Test exception scenarios through actual HTTP calls to verify the full path: controller → exception thrown → handler → `ApiResponse` JSON shape and HTTP status.
