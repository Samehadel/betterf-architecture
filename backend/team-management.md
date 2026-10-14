# Company team management (BTF-8)

Implemented contract: application `app/backend/src/main/resources/openapi.yaml`.
Architecture baseline consulted: `cdd59b639c3136a6311640b7f403a5aca60af5ba`.

Verified active administrators may read only their company records through the identity
service. GET `/api/invitations/sent` returns all failed, accepted and unexpired pending
invitations, sorted by successful send date or initial attempt date when never sent,
then UUID, descending. Expired pending and revoked links are excluded. Accepted and
failed records remain visible. GET `/api/invitations/members` returns all ACTIVE MEMBER
accounts, sorted by activation timestamp (`ACCOUNT.UPDATED_AT`), then UUID, descending.
It returns full name, email and the professional-role display label. Neither endpoint
silently truncates records. Administrator accounts are excluded from this list but
still count toward capacity. The legacy paged GET remains compatible for older clients.

POST `/api/invitations/{id}/resend` rechecks ownership, recipient eligibility,
capacity (default 10 including administrators), current validity and a 60-second
cooldown under recipient then organization locks. Reserve a replacement hash and
expiry separately from the currently valid link. SMTP runs outside transactions.
Only confirmed submission promotes the replacement and updates `SENT_AT`; failure
preserves the old hash, expiry and last successful send. Cooldown uses the latest
initial attempt, confirmed send or replacement attempt. Failed initial sends have
null `SENT_AT`; retries reuse the same row, preventing duplicate live invitations.
The seven-day replacement expiry is established when its delivery attempt starts,
and that same expiry is printed in the email and persisted on successful promotion.

Acceptance can use the old link during delivery. Completion acquires the same locks
and never reopens ACCEPTED invitations or changes member credentials/membership.
If the recipient joins another company during delivery, the replacement is discarded.
A mail exception/timeout is unconfirmed: discard its replacement hash even if the
relay may have delivered a message, and preserve the old link. A delivered but
unconfirmed replacement link is unusable. A database failure after SMTP leaves the
replacement reserved, blocks duplicate submissions and preserves the old link;
operational reconciliation is needed before retry. No automatic takeover is allowed.
A lost HTTP response is reconciled by reloading the sent list before another action.

Migration `20261015-001-invitation-resend.sql` adds nullable replacement fields without
changing existing token or membership rows. Old application instances must be stopped
before enabling resend: they do not honor replacement reservations. Rollback requires
stopping new instances and resolving outstanding attempts before removing those fields.
