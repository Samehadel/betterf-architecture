# Session refresh

- Status: Proposed implementation decision
- Date: 2026-10-08
- Requirement: The project owner requested transparent recovery when session expiry otherwise interrupts actions and forces another login.
- Baseline consulted: `16a2955` (`origin/main`, fetched 2026-10-08). That branch contains reusable security guidance, not an accepted product authentication contract. The existing implementation uses server-side sessions. Uncommitted deployment ADRs in the primary architecture checkout were not used or modified.

## Design

Retain the 30-minute inactivity session and session-bound CSRF protection. Password login and first-time email verification issue an opaque refresh credential in an HttpOnly, Secure-by-default, SameSite=Strict cookie scoped to `/api/auth`. The proposed default absolute lifetime is 14 days, configurable with `REFRESH_TOKEN_LIFETIME`. Rotations do not extend this lifetime. Existing sessions need a new login to obtain a refresh credential.

Foundation owns `REFRESH_SESSION`. Persist only SHA-256 hashes of random 256-bit secrets and password fingerprints, plus a random identifier, email, and absolute expiry. A pessimistic database row lock serializes consumption and logout. Each successful refresh replaces the secret and keeps the immediately preceding hash for replay detection. Reuse of that preceding secret deletes the refresh record; older or unknown secrets cannot authenticate. Hourly cleanup removes expired records. No raw refresh credential is returned in JSON or stored in browser storage.

`POST /api/auth/refresh` requires a valid session-bound CSRF token, even when the associated session is anonymous. It rechecks account and organization activation, password fingerprint, and current role through Identity's public service; then rotates the session identifier and CSRF token. Invalid credentials return `401 SESSION_EXPIRED` and clear the cookie. Logout revokes the matching browser credential even if its short session has already expired. Revocation does not invalidate other already-established server sessions; those retain the existing inactivity and per-use business authorization rules.

Distinguish `401 AUTHENTICATION_REQUIRED`, `403 CSRF_INVALID`, and `403 ACCESS_DENIED`. The browser retries a CSRF rejection with a freshly fetched token, then can refresh once on a pre-controller authentication rejection. A failed refresh with 401 returns the user to login. Connectivity and server failures remain recoverable errors and do not clear the credential or trigger automatic mutation replay.

The interceptor covers current-account and invitation APIs, preserves request bodies, and shares an in-flight refresh within a tab. Web Locks serialize renewal across tabs where supported; after acquiring the lock the browser probes the shared session to avoid unnecessary rotation. Without Web Locks, only same-tab coordination is available and competing tabs can trigger replay rejection. After rotation, fetch fresh CSRF before retrying the original operation once. Do not refresh password-login failures or generic permission failures. Wait for a same-tab in-flight refresh before logout.

## Compatibility and verification

The Liquibase migration is additive. Deploy backend and frontend together: clients that interpret all anonymous failures as 403 must adopt the new 401 contract. Backend tests exercise real PostgreSQL migrations, Spring Security, session/CSRF rotation, revocation, expiry, password changes, and role/status changes. Frontend tests exercise concurrent requests, delayed failures, retry bounds, cross-tab locking, logout ordering, and terminal versus transient failures. SMTP is substituted in backend authentication tests. Browser verification used a disposable PostgreSQL account activated directly because the isolated Mailpit SMTP port timed out; real email delivery was not verified.
