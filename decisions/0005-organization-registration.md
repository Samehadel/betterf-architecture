# 0005 — Organization registration, account access, and verification delivery

- **Status:** Accepted for BTF-5 implementation.
- **Date:** 2026-10-04.
- **Authority:** BTF-5 requirements and the product owner's explicit implementation correction on 2026-10-04: keep account data in ACCOUNT, company details and activation status in ORGANIZATION, and email delivery information and statuses in a separate table. The follow-up direction requires automatic failed-email retry, exclusion of SENDING rows, and a lock preventing overlapping worker runs.
- **Baseline consulted:** `0765baaa675be6232b9df9c025d415dca1ab3f49` (`origin/main`). Untracked local ADRs 0002–0004 were not used as committed authority.
- **Business source:** [BTF-5](https://linear.app/betterf/issue/BTF-5/register-an-organization-and-establish-its-administrator).

## Superseded story wording

The owner's newer direction explicitly supersedes the issue's earlier statements that submission creates no company row and that a company has no activation/status lifecycle. Submission now persists a **PENDING organization and PENDING account**. Verification atomically changes both to **ACTIVE** and establishes ADMIN access. An existing row counts as a registered company only when its status is ACTIVE. This is administrator email verification, not paperwork or SuperAdmin approval.

Other confirmed behavior is retained: pending registrations do not reserve domains; different emails can submit the same domain; the first successful verification wins. Competing pending organizations remain PENDING and confer no access. Duplicate company/account errors retain the agreed text. Professional roles do not confer permissions.

## Ownership and schema

The `identity` domain owns the initial account/organization onboarding invariant. Its public service and DTO packages are named Modulith interfaces. Infrastructure in `foundation` integrates Spring Security and central HTTP wrapping/errors through these interfaces; it never reads identity repositories. Future team workflows must use public identity contracts rather than reaching into its tables.

- `ORGANIZATION`: UUID, name, normalized domain (derived from website), free-text specialization, PENDING/ACTIVE status. A partial unique PostgreSQL index reserves DOMAIN **only for ACTIVE rows**.
- `ACCOUNT`: UUID, normalized email (globally unique), name, password hash, professional-role ID, PENDING/ACTIVE status, organization FK, access role, update time. Pending account association is not active membership: ACCESS_ROLE is null until verification, then ADMIN. MEMBER is reserved for sibling onboarding stories. No company profile or email-delivery metadata lives here.
- `VERIFICATION_EMAIL`: one reusable row per account (ACCOUNT_ID PK/FK), recipient, PENDING/SENDING/SMTP_ACCEPTED/FAILED/CANCELLED delivery status, lifetime attempt count, consecutive failure count, attempt UUID, next-attempt time, last-attempt and last-successful-send timestamps, SMTP message ID, safe failure code, current token hash and expiry. This records the latest attempt, not an append-only mail history. Account email remains the account's login identity; this table owns delivery and verification data.

Status columns are VARCHAR without database enum or status-vocabulary CHECK constraints, as explicitly requested by the owner on 2026-10-04. Application enums and use-case rules own supported transitions; future values such as BLOCKED do not require altering database status constraints. The active-domain partial unique index remains an ownership invariant, not a status vocabulary restriction. A future blocked-organization workflow must decide whether blocked domains remain reserved.

PostgreSQL unquoted uppercase identifiers follow the existing convention. JPA validates the schema; Liquibase creates it. This is an additive migration with no destructive automatic rollback. Before rolling back a deployment after onboarding writes, preserve a database backup; restore only through the operator's tested recovery process. Do not drop identity tables to reverse a software release.

## State transitions and concurrency

Registration normalizes email and validates all fields, domain, role, and password. PostgreSQL transaction advisory locks serialize operations for an email across application instances. Different pending emails may have separate PENDING organization rows with identical domain. Repeated submissions return the original pending registration and do not overwrite password, company, or profile, send another email, or create another record; failed email delivery retries automatically; resend queues a replacement after SMTP acceptance.

Verification locks email then domain, checks the current credential and its expiry, then rechecks account and organization eligibility and active domain ownership. Updating organization status, account status, and ADMIN role commits together. PostgreSQL's partial unique index protects active domain identity even against a competing writer. No loser joins the winner's organization. Account normalization plus a unique email key prevents membership in two companies. Sibling invitation flows must follow this same global account invariant and lock protocol.

Authentication and the current-account service require both ACCOUNT.STATUS and ORGANIZATION.STATUS to be ACTIVE. Protected service access rechecks state, including for an already-authenticated HTTP session. Unknown API routes and invitation routes remain denied.

## Verification and mail delivery

Generate 32 cryptographically random bytes and store only SHA-256 of the URL-safe token. Email links use `/verify#<account-id>.<token>` so HTTP logs and Referer do not contain the token. The client removes the fragment from history and requires a user POST confirmation; link scanners cannot activate by GET. Never log request bodies, passwords, or verification credentials.

Registration/resend commits a queued email row without SMTP calls. Every five seconds a worker acquires a PostgreSQL session advisory lock on a dedicated connection for its entire run, including SMTP calls. A competing instance immediately skips the run. Each batch selects at most 25 due PENDING/FAILED rows ordered by next-attempt time and account ID; SENDING is never selected. Before each attempt, a short transaction takes the email lock, rechecks eligibility/due time, and commits SENDING, an attempt UUID, and attempt count/time. Ineligible accounts/organizations or a domain already activated by a competing registrant are CANCELLED.

SMTP runs outside the business transaction with bounded connection/read/write timeouts. Another short transaction takes the email lock and updates only the matching SENDING attempt. Success records SMTP_ACCEPTED and message ID, rotates the token hash, and sets a 24-hour expiry. SMTP_ACCEPTED is relay acceptance, not proof of mailbox delivery. Provider delivery/bounce/block events require provider logs or a future authenticated webhook integration; the UI must not claim confirmed delivery.

Failure records FAILED and a safe diagnostic code, preserving the previous accepted token and expiry. Automatic retry delays are 60, 120, 240, 480, then 900 seconds, capped at 900 for later failures. Successful acceptance resets consecutive failures. Resend is rejected before 60 seconds after the last accepted send. Once eligible, queued/sending/failed requests coalesce rather than replacing the in-flight attempt or bypassing backoff. Boundary times are inclusive: at 60 seconds resend is allowed; at 24 hours verification is expired. Automatic retries may rotate a previous token after the replacement is accepted.

SMTP and PostgreSQL cannot commit atomically. A crash or persistence failure after SMTP acceptance can leave SENDING and an unusable link. SENDING is deliberately excluded from retry and cleanup; operators must investigate provider logs and stop all workers before resetting a confirmed abandoned attempt to FAILED with a due timestamp. Do not reset live attempts. An ambiguous SMTP network error can cause duplicate messages on retry; no exactly-once guarantee is made and no raw token is stored for replay. A failed final persistence step is not handled as an SMTP failure.

An additive migration converts old SENT rows to SMTP_ACCEPTED and makes existing PENDING/FAILED rows due. Stop old application instances before migration; they cannot read the new status values.

JPA writes flush at transaction commit; request handling and delivery state transitions do not explicitly flush repositories. Retention issues ordered SQL deletes within its transaction to satisfy foreign keys without intermediate persistence-context flushes.

Repeated use of the current valid link after activation returns ALREADY_VERIFIED with a login next step. An expired or replaced link never grants access, even after activation. A failed-delivery record may still have a usable earlier credential; delivery status is not account status.

## Validation and authentication

- Website: HTTPS, root registrable domain under the ICANN/registry suffix from Guava's bundled public suffix data. Supports `company.co.uk`; rejects subdomains (including private-hosting tenants), unknown suffixes, IP literals, credentials, explicit ports, path components, query and fragment. A single root slash is accepted. ASCII/punycode DNS hosts are supported; raw Unicode input is rejected with root-website guidance. Lowercase domain comparison; no DNS fetch or website-domain ownership assertion. Keep Guava's suffix data current with dependency updates.
- Email: normalized with trim and Locale.ROOT lowercase, validated and limited to 254 characters. No website-domain match requirement.
- Password: 15–128 characters, no mandatory symbol/composition rules, spaces allowed; Spring's PBKDF2 HMAC-SHA256 (v5.8 defaults, per-password salt, 310,000 iterations) hashes the password. Never trim it or store it in browser persistence.
- Professional roles: the exact 25-item BTF-5 list, exposed by a read-only API for later member onboarding. `Other` has no follow-up questionnaire.
- Same-origin server-side HTTP sessions, 30-minute inactivity expiry. Session ID and CSRF rotate on login; POST logout invalidates the session. HttpOnly, SameSite=Lax, Secure cookie by default (explicit local HTTP override only). Sessions are node-local and expire on restart; horizontal scaling needs sticky sessions or shared storage before deployment.
- CSRF stays enabled for registration, resend, delivery-status lookup, verification, login, and logout. A public `/api/auth/csrf` supplies the masked session token; the client fetches it before each mutation. No permissive CORS. Generic login failure does not distinguish unknown, pending, or incorrect-password accounts.

## Retention and operational configuration

Hourly cleanup processes at most 100 pending accounts older than 30 days since their last successful registration/resend. It takes the same email lock and rechecks status and timestamp, then deletes the email record, account, and its pending organization. Active accounts/organizations and SENDING email attempts are never deleted. Larger backlogs may need operator scheduling adjustment. Expired active-account verification hashes contain no recoverable secret and remain in the reusable record for delivery diagnostics; normal data-erasure policy is separate.

Configure SMTP_HOST/PORT, SMTP_USER/PASSWORD, SMTP_AUTH, SMTP_STARTTLS, MAIL_FROM, PUBLIC_ORIGIN, and SESSION_COOKIE_SECURE through deployment secrets/settings. Public origin is trusted configuration, never request Host. Local development uses Mailpit on loopback and an explicit insecure-cookie override for HTTP. Production must use HTTPS and a verified sender; SMTP delivery credentials are not committed. Remote SMTP rejects the local default sender with SMTP_FROM_NOT_CONFIGURED; Brevo requires its SMTP login/key and a verified MAIL_FROM address. The application README contains configuration and diagnostic instructions. The session lock requires direct PostgreSQL or session pooling and at least two available connections per worker instance; transaction-mode pooling is unsupported. BETTERF_REGISTRATION_DELIVERY_ENABLED disables scheduling and BETTERF_REGISTRATION_DELIVERY_DELAY sets the polling interval in milliseconds.

## Client and contracts

Public `/register`, `/verify`, `/login`; `/company` loads current authenticated organization from the server and shows login guidance on denial. The landing page offers organization registration and login while preserving the product preview. Angular standalone pages use an identity API client and route-scoped Signal Store, translated text, labeled fields, visible validation/failure/loading/cooldown states, keyboard focus, and responsive controls. The client polls POST /api/registration/status every five seconds while queued/sending/retrying and shows the returned state without claiming mailbox delivery. The authoritative wire contract is application `app/backend/src/main/resources/openapi.yaml`.

## Verification and remaining release dependencies

Integration tests cover real PostgreSQL migrations/constraints, status transitions, competing verification and resend, failed delivery persistence/backoff, committed SENDING visibility, global worker exclusion, skip/retention of SENDING rows, cancellation after activation, credential invalidation/expiry, CSRF, session rotation/login/logout, and retention. Mail is captured in integration tests; local browser tests use SMTP delivery to Mailpit. Browser tests cover the landing entry, required fields, successful onboarding/login/logout, resend and superseded links, and narrow-screen layout.

Password recovery requires an explicit delivery owner before releasing account access, as stated in BTF-5; this implementation does not invent a recovery journey. Product owner must assign that owner. Invitation sending, accepted-invitation visibility, and active-member views belong to BTF-6/7/8, not this change. Deployments also need ingress rate limiting for registration/resend/login and appropriate email-provider abuse controls before public exposure. No deployment is authorized by this implementation task.
