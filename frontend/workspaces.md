# Workspace journeys

Signup at /signup requests only full name, email and password. Verification opens Knowledge with one owned default workspace. /register redirects to this flow. Social signup is outside this delivery.

The shared shell exposes the current workspace and membership picker. /workspace provides optional creation and owner-only rename; collaboration is available through workspace team/invitation screens. Invitation screens explain that joining gives access to the entire existing knowledge base. Invitation acceptance preserves a different signed-in session until an explicit switch; an authenticated recipient explicitly accepts without resubmitting profile or credentials. New users get a default workspace alongside the invited workspace, as clarified by the owner on 21 October 2026.

AuthStore owns the available membership list and selected workspace response. WorkspaceContext supplies X-Workspace-Id for resource calls. The picker navigates away from the current feature before changing context so route caches are destroyed and the capture unsaved-change guard can cancel navigation. Knowledge browsing ignores stale responses through sessionVersion. Refresh keeps selection only if membership still exists. APIs revalidate permissions; client role checks only control presentation.

See [backend identity and upgrade contract](../backend/workspace-identity.md).
