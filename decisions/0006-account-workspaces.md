# Account and workspace membership

Status: Accepted by product direction for BTF-26 (2026-10-21).

Accounts represent people with globally unique email addresses. Workspaces replace organizations as the knowledge and collaboration boundary. An account can belong to many workspaces through explicit memberships; OWNER and MEMBER are workspace roles.

Manual signup requires only name, email and password. Email verification activates the account and initializes its default workspace. Invitation acceptance for a new person also initializes an independent default workspace before adding the invited membership. Existing accounts retain their memberships and default workspace. This explicit product direction supersedes the invitation-led signup exception in the business stories.

Every knowledge operation requires an explicit workspace context and validates active membership. Workspace switching does not mutate the global identity or move tickets. Ticket authorship and workspace ownership remain immutable.

Organization registration, domain-based workspace discovery and professional-role onboarding are removed. New social signup is outside this implementation; existing linked social login is retained.

See [backend lifecycle and migration](../backend/workspace-identity.md) and [frontend journeys](../frontend/workspaces.md). The migration requires coordinated backend rollout because old account scope columns are removed.
