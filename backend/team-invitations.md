# Team invitation sending (BTF-6)

Source: [BTF-6](https://linear.app/betterf/issue/BTF-6/invite-team-members-by-email).
Baseline consulted: `0765baaa675be6232b9df9c025d415dca1ab3f49` on `origin/main`.

## Ownership and access

The identity module owns organization accounts and team invitations. Its public
`InvitationService` exposes single-recipient send and delivery-status operations.
Spring Security requires ADMIN and CSRF for both endpoints; the service rechecks
active account, active organization, and ADMIN role on every call. Company identity
comes from the authenticated account, never from a request-supplied company ID.
Emails are validated and normalized to lowercase. Personal addresses are allowed.
Established membership (including inactive accounts retaining an access role) is
checked before pending invitations. Pending organization registration does not
establish membership.

## Persistence and delivery

An additive Liquibase migration creates TEAM_INVITATION and an optional positive
ORGANIZATION.MAX_ACTIVE_ACCOUNTS override. The configured fallback is
`INVITATION_MAX_ACTIVE_ACCOUNTS`, default 10. Capacity counts ACTIVE accounts,
including administrators. Invitations never create accounts or reserve capacity.

One invitation record exists per company and normalized email, protected by a
unique constraint and PostgreSQL transaction advisory locks. Sending acquires
`email:<normalized email>` followed by `organization:<UUID>`. This lock order must
also be used by acceptance to serialize company capacity and recipient membership.
Other companies may invite the same unregistered email.

The service commits SENDING, the recipient/company binding, administrator identity,
SHA-256 token hash, and seven-day expiry before SMTP. Raw 256-bit random tokens exist
only in memory and the email link fragment. The resource template is
`mail/team-invitation.html`; names are HTML escaped, with responsive inline styling
and a plain-text alternative. SMTP configuration reuses registration's MAIL_FROM,
PUBLIC_ORIGIN, and SMTP settings. Each request submits exactly one message.

A send succeeds only when SMTP accepts it and SMTP_ACCEPTED is persisted with the
provider message ID. This means relay acceptance, not guaranteed inbox delivery.
A known SMTP failure persists FAILED, a safe diagnostic code, and clears the token
hash. The administrator may correct the email or retry. Failed sends have no
background retries. Repeated requests during SENDING do not contact SMTP. A valid
SMTP_ACCEPTED record returns INVITATION_PENDING; expired or revoked records can be
reused for a fresh send, subject to capacity, with a fresh token.

A response timeout is reconciled through the company-scoped status endpoint.
A crash or completion-persistence failure leaves SENDING. The UI allows status
checks but blocks another send while uncertain. Operators must investigate provider
logs using recipient, attempt time, and message ID where available before changing
an abandoned record; stop relevant senders before recovery. SMTP and PostgreSQL
cannot commit atomically. Ambiguous transport failures may still result in duplicate
messages on a later manual retry; no exactly-once inbox-delivery claim is made.

## Contracts and dependencies

`POST /api/invitations` accepts `{email}` and returns the standard `{data,error}`
envelope containing `id`, `email`, `status`, `expiresAt`, and `message`.
`POST /api/invitations/status` accepts the same input and reads only the caller's
company record. Invalid input is 400; denied access/CSRF is 403; missing status is
404. Conflict codes are ALREADY_MEMBER, OTHER_COMPANY_MEMBER, INVITATION_PENDING,
and ACCOUNT_LIMIT. Delivery failures return a FAILED view, not a success status
in the UI. See the application `openapi.yaml` for the complete wire contract.

BTF-7 owns `/invitation/accept#<id>.<token>`, profile setup, and acceptance. It must
require SMTP_ACCEPTED, compare hashes securely, bind membership to the stored email
and company, reject expiry/revocation/replaced tokens, recheck membership and
capacity under the locks above, and atomically activate at most one membership.
Capacity denial must leave the invitation unconsumed. These acceptance operations
are not implemented by BTF-6.

BTF-8 owns resends, revoke, and history. It must enforce capacity, a 60-second
cooldown, old-token invalidation only for an allowed replacement, and a fresh
seven-day expiry. BTF-6 supplies an explicit pending-management navigation
placeholder; resend is unavailable until BTF-8 is delivered.

## Persisted invitation history

The project owner requested that earlier invitations remain visible after route
navigation and refresh. `GET /api/invitations?page=0` returns a read-only projection
of records for the verified authenticated administrator's own company, with 25
records per page and `hasMore`. It sorts by attempted time and ID descending,
exposes only InvitationView fields, and never sends email or returns token hashes.
Invalid page values return 400; denied access returns 403. No persistence migration
is needed. Acceptance/resend/revoke actions remain owned by BTF-7/BTF-8.
Architecture guidance consulted for this addition:
`16a2955250dd36a2f423423656ca919f0e87d539` on `origin/main`.
