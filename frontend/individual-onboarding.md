# Individual onboarding

BTF-26 adds `/signup` alongside `/register`. Landing-page actions clearly distinguish
personal accounts from organization registration. Personal signup requests name,
email, and password, and explains permanent separation from organizations. LinkedIn
and Facebook buttons are shared with login; unavailable providers are disabled with
an email/password next step. Verification reuses `/verify` and the existing delivery,
resend, cooldown, failure, and expiry states.

Individuals land in `/knowledge`. The workspace shell labels the space private,
shows personal identity, and omits organization home/team navigation. Existing capture,
browse, filter, details, classification, and refinement components are reused. Private
visibility copy replaces organization visibility. Backend ownership checks remain
authoritative. Search remains deferred to BTF-17 by the product owner's instruction.

The shared AuthStore continues to own application identity. Its session-version change
on account ID invalidates prior personal or organization knowledge state. Nullable
organization IDs/names are reflected in the typed API model. Individuals visiting the
legacy `/company` home redirect to their knowledge workspace.

A provider collision redirects to `/login` with a safe error code and an explanation
that an existing method must authenticate first. `/link-account` presents explicit
confirmation, retry, expiry, and sign-in-again paths. Feature state belongs to the
LinkAccountStore; linking requests contain no client-supplied provider identity.
No OAuth access tokens or linking credentials are stored in browser storage.
