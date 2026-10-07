# Team invitation entry (BTF-6)

The authenticated company home exposes `/company/invitations` for administrators.
The page loads the current account, handles loading and denied access, and uses a
route-scoped NgRx Signal Store. A typed API client obtains CSRF before each mutation
and uses same-origin session cookies. Backend authorization remains authoritative.

The page starts with one email input and one Send action. Sending locks the row;
SMTP_ACCEPTED produces green text and permanently locks the successful row. Only
the last successful row exposes the accessible add button. Adding focuses the new
email field and preserves earlier results. Failure shows red text and permits
correction/retry. Membership and pending outcomes do not become new-send successes.
Status text and live regions accompany colors; labels, keyboard controls, responsive
layout, and visible focus follow existing app conventions.

After an uncertain response, the address remains locked and the administrator can
check the send status without submitting another email. FAILED or a missing record
allows retry; persistent SENDING directs the user to support. Only email is collected.
Pending outcomes link to `/company/invitations/pending`, currently an explicit
coming-soon destination. BTF-8 owns its future management/resend implementation.
BTF-7 owns the email's `/invitation/accept` recipient registration route.

See [backend invitation design](../backend/team-invitations.md) for tokens, delivery,
capacity, API responses, and dependent story requirements. Source:
[BTF-6](https://linear.app/betterf/issue/BTF-6/invite-team-members-by-email).
