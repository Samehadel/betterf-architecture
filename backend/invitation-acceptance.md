# Invitation acceptance (BTF-7)

Application branch: `BTF-7-member-registration`. Architecture baseline: `cdd59b6` on fetched `origin/main`. This implements BTF-7 and the owner's 2026-10-08 clarification: successful acceptance cancels an invited recipient's unfinished administrator signup. The inviting administrator must already be verified.

## Ownership and transaction

Identity owns invitations, account credentials, membership, pending organization records, and verification delivery. `InvitationAcceptanceService` exposes preview and explicit registration; Foundation orchestrates HTTP session changes through this public API. Professional roles use the existing registration vocabulary and never grant access roles. A successful acceptance assigns `MEMBER`.

Preview validates the token, expiry, delivery state, recipient membership, organization availability, and active-account capacity. It does not accept an invitation or mutate an account. Registration repeats these checks under transaction-scoped PostgreSQL advisory locks, in order: normalized `email:<email>`, then `organization:<id>`. This coordinates with invitation sends, administrator registration/verification, cleanup, and concurrent acceptances. Account email uniqueness remains the database backstop. Pending invitations do not reserve capacity. Active administrators count toward the configured organization limit (organization override, otherwise `betterf.invitations.max-active-accounts`, default 10).

Acceptance stores the submitted name, role and password hash, establishes sole membership, activates the account, and records acceptance atomically. The emailed credential verifies email; no new verification email is sent. Invalid input, capacity failure, changed membership, expiry and revocation leave acceptance and pending signup records unchanged.

For a recipient with a genuinely unfinished signup (PENDING account, no access role, PENDING organization), acceptance reuses the account ID, replaces its pending profile/credentials with the submitted member registration, removes its verification-delivery record, and deletes its abandoned organization after flushing the new organization reference. Email locking serializes this against verification and cleanup. A previously claimed SMTP attempt may still send its email, but cannot persist a new token after the delivery record has been removed. Old verification links cannot activate anything. An existing completed membership, even if the account subsequently becomes inactive, is never overwritten.

## Credential history and migration

Liquibase adds nullable `ACCEPTED_AT` and `ACCEPTED_TOKEN_HASH` to `TEAM_INVITATION`. On success, the live token hash moves into the history field and `TOKEN_HASH` is cleared. History is only used to return `INVITATION_USED`, before checking current expiry or account status. It never authorizes login, profile modification, reactivation, or another acceptance. Only hashes are stored. Existing send replacement preserves this history.

The migration is additive and leaves existing invitations usable. Do not roll back its columns after accepting invitations: dropping them loses historical used-link recognition. Roll back application traffic first and preserve the data; no automated destructive recovery is required.

## HTTP and browser sessions

- `POST /api/auth/invitation/preview`: `{id, token}` -> `{status, message, companyName, email}` in the standard envelope. Known link outcomes return 200; malformed input returns 400. Invalid/unrecognized credentials do not expose company or recipient data.
- `POST /api/auth/invitation/accept`: `{invitation: {id, token}, fullName, professionalRole, password}` -> account view after successful registration and session establishment. Input failures return 400; changed eligibility/used links return 409 with the preview outcome code.

Both are public, CSRF-protected, and use `Cache-Control: no-store`. The `/api/auth` prefix makes the existing path-scoped refresh cookie available for revocation. Preview of a recognized link invalidates a different authenticated browser session and revokes its refresh credential, without changing the other account. When no authenticated principal remains, preview also revokes any expired session's refresh credential. The same-account session may remain.

After acceptance commits, Foundation clears the old browser session, establishes a new MEMBER session, and issues a refresh credential. If session persistence or the HTTP response fails, the account remains accepted; replays cannot overwrite it or establish another session from the link. The UI offers status reconciliation and password login. Password login is the recovery path after a successful registration with a lost response. Existing session refresh behavior remains governed by the session refresh decision.

Outcome codes: `READY`, `INVALID_INVITATION`, `INVITATION_UNUSABLE`, `INVITATION_USED`, `ALREADY_MEMBER`, `OTHER_COMPANY_MEMBER`, `ACCOUNT_LIMIT`. The OpenAPI resource in the application is the exact wire contract.

## Verification

PostgreSQL integration coverage includes unchanged previews; validation and capacity preservation; simultaneous capacity, duplicate and cross-company attempts; acceptance versus administrator verification; invalid, expired, revoked and superseded credentials; used links after expiry/inactivation; cancellation with SMTP in flight; CSRF; member-only sessions; different-account session/refresh revocation; and session failure followed by password login.
