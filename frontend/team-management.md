# Company team management (BTF-8)

`/company/team` replaces the pending management placeholder; the old
`/company/invitations/pending` route redirects there. The send page links to management
instead of showing a separate history section. The route-scoped NgRx Signal Store owns
list loading and resend state; the typed invitation API client handles HTTP and CSRF.

The page has keyboard-accessible Sent invitations and Active members tabs, with
loading, empty, failure/retry and permission feedback. Invitation columns are email,
status, last sent date and expiry. Resend is an action beside the email, with a live
60-second countdown; accepted, expired and revoked links have no action. Expired pending
records are removed locally as time passes. Failed entries show Not sent when no
submission succeeded and no active-link expiry. Active members shows full name,
email and professional role. Tables scroll within their container on narrow screens.
Result messages accompany success/failure colors and use an accessible live region.

API ordering and race/failure semantics are documented in
[backend team management](../backend/team-management.md) and the application OpenAPI
contract. No search, filters, capacity display or user-visible pagination are provided.
Backend authorization remains authoritative; client permission feedback is for usability.
