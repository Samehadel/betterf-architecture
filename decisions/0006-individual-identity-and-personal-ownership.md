# Individual identity and personal ownership

Status: Proposed implementation design; product boundaries accepted in BTF-26.

## Context and decision

BTF-26 adds permanent individual accounts to an application whose authenticated
accounts and knowledge access previously required organization membership. Reusing
an implicit personal organization would contradict the accepted requirement.

Keep one globally unique normalized-email account model. Add explicit INDIVIDUAL
classification with null organization, separate unique provider/subject identities,
and personal tickets owned by their author without organization association. Retain
existing organization membership limits. Require trusted reauthentication before
linking a new provider to an existing email. Authentication integration stays in
Foundation; domain identity and ownership checks stay in Identity and Knowledge.

## Consequences

Authorization must explicitly distinguish organization and personal queries. Database
constraints prevent invalid account scope, conversion, and personal content transfer.
Existing organization records remain compatible with the new model, but old backend
instances cannot serve individual rows and must be stopped before rollout. Search is
deferred to BTF-17 by the product owner's implementation instruction.

The detailed contract, verification fallback, locking, migration, and linking lifecycle
are recorded once in [individual identity](../backend/individual-identity.md).
Architecture baseline: `cdd59b639c3136a6311640b7f403a5aca60af5ba`.
