# Individual identity and social login

Implementation design for BTF-26. Consulted architecture `origin/main` at
`cdd59b639c3136a6311640b7f403a5aca60af5ba`. Search is explicitly deferred to BTF-17
by the product owner; this implementation covers capture, browsing, refinement,
classification, and viewing in a personal workspace.

## Account and resource ownership

Identity owns one global ACCOUNT table. Canonical emails use trimmed input and
`Locale.ROOT` lowercase; dots and plus suffixes are not removed. Database uniqueness
is global, with lowercase/trim checks rejecting noncanonical persisted values.
Existing accounts migrate to ORGANIZATION. INDIVIDUAL accounts have no organization,
use the INDIVIDUAL access role when active, and cannot change classification or
receive organization membership. A database constraint enforces null organization
ownership for individuals; an update trigger prevents conversion. No transfer API
exists. Invitation send/resend, preview, and acceptance recheck individual
ineligibility, including invitations issued before individual registration.

Knowledge owns tickets. Null ORGANIZATION_ID means a personal ticket owned by
AUTHOR_ID. Every personal detail/browse query requires both null organization and
the authenticated account ID. Organization reads continue to require the current
organization ID. Create/refine operations require an active account through Identity's
public service. Personal ownership cannot change through updates, including transfer
into an organization. Personal browse indexes cover author, time, ID, and Details type.
No client-provided owner or organization is trusted.

## Password registration and verification

`POST /api/registration/individual` takes fullName, email, and password, following the
existing 10–128-character password rule with uppercase and special characters. No
professional-role questionnaire is required; the internal role placeholder is OTHER.
A pending individual and queued verification delivery commit together, without any
organization row. Existing normalized email returns EMAIL_IN_USE without exposing
membership details or overwriting profile/credentials. Retries cannot create duplicates.

The existing verification worker supplies 24-hour links, a 60-second resend cooldown,
replacement-token invalidation on successful SMTP acceptance, and retryable delivery.
Failed replacement delivery preserves the previous valid link. Verification enables
access and establishes the usual application session; repeated verification cannot
be reused as a login credential. Pending individuals follow the existing 30-day
abandonment cleanup. Cleanup ignores in-flight email sends. No deleted-active-account
email reuse policy is introduced.

## Provider authentication

Foundation integrates Spring Security's authorization-code flow, session-bound state,
and LinkedIn OIDC signature/issuer/audience/expiry/nonce and UserInfo-subject validation.
LinkedIn requests openid/profile/email. Facebook requests public_profile/email and
resolves the subject through authenticated `/me`; Facebook's general `verified`
profile flag is not treated as email verification.

Only an authenticated provider claim set reaches Identity's public social service.
Explicit false email verification is rejected, including false in the signed ID token
when UserInfo differs. Missing verification metadata is accepted. A network/5xx
UserInfo outage may use a previously validated LinkedIn ID token containing email;
401/403, invalid signatures, bad nonce/state, subject mismatch, missing usable email,
and token exchange failures never use that fallback. Profile failure without an
independently authenticated email fails. Provider access tokens are not retained.

OAUTH_IDENTITY stores provider, stable subject, and account separately. Its unique
(provider, subject) constraint and email constraint protect concurrent requests.
Transactions acquire provider then email advisory locks. New user and link commit
atomically. A subsequent linked provider authenticates its original account ID, even
if the provider later changes its email; it cannot reassign the link. Inactive accounts
cannot authenticate. Social-only accounts have an unmatchable password marker,
which also participates in existing refresh credential revocation.

The OAuth filter does not save its raw provider principal in the application session.
Only successful domain validation creates the application principal, rotates the
session/CSRF credential, and issues the existing refresh cookie. Errors redirect with
a bounded public error code, never tokens, email, provider response bodies, or
membership details. Provider callbacks use `/api/auth/oauth/callback/{provider}`;
all browser API traffic stays same-origin.

## Explicit secure linking

A new provider presenting an existing email creates no account or link and grants
no application access. A five-minute server-session challenge records the validated
provider identity and clears any prior reauthentication proof. The user must then
log in using an existing password or already-linked provider. Password verification
or trusted provider login records a fresh proof; session refresh and email verification
do not. The browser explicitly confirms `POST /api/auth/social/link` with CSRF.

Linking requires the same session, a live challenge, a fresh proof after the challenge,
and an active account whose canonical email matches the provider email. Persistence
rechecks provider ownership under the same locks and unique constraints. Already
linked-to-self is idempotent; linked-to-another-user fails. Successful linking clears
the challenge. A lost response can be recovered by starting provider login again;
repeated callbacks do not recreate links or memberships.

## Configuration and validation

Configure LINKEDIN_CLIENT_ID/SECRET and FACEBOOK_CLIENT_ID/SECRET only in the server
secret environment. Register exact PUBLIC_ORIGIN callback URLs with providers.
Facebook API version is configurable (default v23.0). Missing configuration disables
the corresponding UI option. Production requires HTTPS and Secure cookies. Apply
both migrations before exposing the new frontend; old application nodes cannot handle
individual rows, so stop old nodes during rollout. Rollback requires disabling individual
traffic and a compatible data migration or tested backup; do not make applied migrations
silently destructive.

PostgreSQL integration checks exercise verification, concurrent uniqueness, linking,
invitation rejection, and personal/organization isolation. A local provider simulator
exercises real OAuth filters, token exchange, signed JWTs, invalid callbacks, optional
lookup failures, and callback replay. Browser tests cover personal signup, Mailpit
verification, private workspace use, and existing organization journeys. Live provider
configuration and consent approval remain deployment prerequisites.

References: [LinkedIn OIDC](https://learn.microsoft.com/en-us/linkedin/consumer/integrations/self-serve/sign-in-with-linkedin-v2),
[Spring Security OAuth login](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/advanced.html),
[Facebook login flow](https://developers.facebook.com/docs/facebook-login/guides/advanced/manual-flow/).
