# HTTP & Interceptors — Reference Patterns

> Reusable reference, not an accepted BetterF technology or product decision. See [documentation status](../../README.md#documentation-status). Examples are illustrative; validate framework APIs against the versions selected for implementation.

## Typed API clients

Keep URL construction and HTTP serialization in dedicated API clients. Keep domain behavior and UI coordination outside those clients. Define separate request and response models aligned with the accepted backend contract.

A neutral Angular example, assuming `apiBaseUrl` is supplied through application configuration:

```typescript
interface ItemView { id: string; name: string; }

getItem(id: string): Observable<ItemView> {
  return this.http.get<ItemView>(
    `${this.apiBaseUrl}/items/${encodeURIComponent(id)}`
  );
}
```

The example path and direct response shape are illustrative, not BetterF endpoints or contracts.

## Shared interceptors

If Angular is selected, functional interceptors can handle common concerns such as configured credentials, tracing, loading indicators, and shared error translation. Register them deliberately and test request/response ordering.

Keep credential handling restricted to intended API destinations. Choose cookie or header transport only after the security contract is decided; no inherited authentication interceptor is required.

If refresh/retry is introduced, define its concurrency, loop-prevention, retry limits, and failure behavior. Do not silently retry operations without considering duplicate effects. The choice between redirecting, prompting, or showing an expired-session state belongs to the product's authentication flow.

## Loading and errors

Track concurrent requests with a count rather than a single boolean when using a global loading indicator. Ensure completion, error, and cancellation release loading state.

Feature state should represent actionable failures. Coordinate global notifications and local error messages to avoid duplicated or misleading feedback. Handle network failure separately from a structured server response.

## Configuration and testing

Read base URLs from environment/application configuration. No deployment hostname or port is established here.

With Angular's `HttpTestingController`, verify method, URL, parameters, body, headers/credentials where relevant, response mapping, and error propagation. Verify no unexpected requests remain after a test. Add integration coverage for the actual backend response format.
