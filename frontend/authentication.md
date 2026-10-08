# Authentication state and session recovery

The application keeps its current account in a root-provided NgRx Signal Store
(`AuthStore`). The store owns account state across routes; registration and login
form state remains in the route-scoped `IdentityStore`. Pages derive the account
from the shared store rather than maintaining separate copies.

An application initializer loads `/api/auth/me` once at startup. Concurrent loads
share one request, and later navigation reuses the result, including an anonymous
or denied result. Temporary network or server failures remain retryable through
page error controls. Account data is kept only in memory; a full reload restores
it from the server again.

Successful login and email verification update the store from their account
response. Session refresh updates it from the `/api/auth/refresh` response. When
another browser tab has already restored the session, the refresh coordinator
uses the account returned by its `/me` check under the Web Lock. These recovery
checks remain necessary for coordinating cookie rotation across tabs.

Successful logout clears the shared account after server-side revocation.
Unrecoverable authentication failures also clear a cached account. An older
startup response cannot overwrite a later login or logout. Anonymous startup
on a public page does not redirect to login; a failed protected operation does.

The backend remains responsible for authentication and authorization. Client
account caching does not extend the server session or bypass protected API
checks. Session and refresh cookies and CSRF recovery retain the existing
backend contract and interceptor behavior.
