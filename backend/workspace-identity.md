# Global accounts and workspace memberships

Product requirements: BTF-26, BTF-7, BTF-27, BTF-28 and the owner's clarification on 21 October 2026. The clarification takes precedence over story text stating invitation-led signup creates no default workspace.

## Account and workspace model

ACCOUNT owns normalized email, credentials, verification state, name and a default workspace reference. Email normalization remains Java trim followed by Locale.ROOT lowercase, enforced by a global unique key and normalized-email check. There are no account classifications, company domains, company registration forms or required professional roles.

WORKSPACE owns its name, status and optional membership capacity. WORKSPACE_MEMBERSHIP associates accounts and workspaces many-to-many with a unique account/workspace pair, active flag, and OWNER or MEMBER role. Ownership grants membership administration, not authority to edit someone else's tickets. Each ticket has an immutable workspace and original creator. There is no personal-content fallback.

Independent password signup creates a pending account and verification delivery. Successful activation provisions exactly one default workspace and owner membership in the same transaction. New invitation-led accounts also receive a default workspace plus the invited membership. The response selects the invited workspace while ordinary login selects the default. Existing accounts keep all their workspaces, profile and credentials.

Social signup is outside this delivery. Its UI entry points are hidden and new provider-only account creation returns SIGNUP_REQUIRED. Existing provider identities and trusted linking/login code are retained; no credentials are discarded.

## Authorization and context

Knowledge and invitation requests require X-Workspace-Id. Identity validates active account, workspace and membership for every operation; invitation management and renaming additionally require the workspace OWNER role. AccountView includes available workspaces and the selected membership role. GET /api/workspaces/{id} validates selection without mutating a global active-workspace field, so separate tabs can work in different workspaces.

The UI passes explicit context, increments its session revision when selection changes, and leaves the current feature route before switching. This runs unsaved-capture guards and clears route-owned caches. Session refresh retains the selected workspace only while it remains in the returned active membership list. Missing memberships fail closed. Platform actuator data is denied to ordinary product identities; ownership does not grant platform operator access. Public health probes remain available.

## Invitation and verification transactions

Email advisory locks serialize signup, verification, acceptance and cleanup. Invitation operations then acquire the target workspace advisory lock, serializing membership capacity checks and acceptance. Successful acceptance marks the token consumed and creates membership atomically. Target-capacity failure creates neither account/default workspace nor membership. Seven-day invitation expiry, resend cooldown and accepted-token replay protection remain in force.

Existing active accounts must authenticate as the recipient and explicitly accept. Pending accounts prove their original password plus the valid invitation; the invitation verifies that email. Acceptance never replaces a pending account's profile or password. It deletes pending verification delivery, preventing an in-flight SMTP completion from publishing another activation token. Preview never logs out the current account. A mismatch offers an explicit account switch.

Verification validity remains 24 hours from successful SMTP acceptance with a 60-second resend cooldown. Successful resend replaces the token; failed delivery preserves the prior accepted token. Session creation follows the committed identity transaction; a session failure recovers through normal login, not replay of an accepted token.

## Upgrade and recovery

Apply historical migrations unchanged, then 20261021-004-workspaces.sql. It maps existing organizations to workspaces with preserved IDs, maps active legacy administrators/members to memberships, gives legacy individual accounts and non-owner members default workspaces, moves organization-free tickets into their creator's default workspace, preserves provider links and credential hashes, and removes the old account classifications/domain restrictions. New ticket ownership triggers prohibit changing workspace or creator.

This is a coordinated backend/frontend rollout. Stop old backend instances and invalidate old browser sessions before deploying; old contracts and binaries are not compatible with the new schema. Take and test a database backup before upgrade. The migration is transactional and should abort on unexpected constraints rather than silently merge accounts. Existing normalized-email uniqueness remains in force. No destructive downgrade is supplied: restore the pre-upgrade backup and matching binaries to recover.

Compatibility routes redirect /register to /signup and /company links to the corresponding workspace or Knowledge view. Old organization-registration payloads are no longer a supported contract. Independent workspace creation/rename are optional settings, never onboarding prerequisites.
