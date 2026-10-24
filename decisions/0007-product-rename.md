# 0007 — Product and application namespace rename

- Status: Accepted naming decision; live infrastructure cutover requires verification
- Date: 2026-10-24
- Authority: The project owner requested replacing the former product name throughout
  the application, packages, deployment, and live resources with `theeverthread`.
- Baseline: `cdd59b639c3136a6311640b7f403a5aca60af5ba` (`origin/main`, fetched).

Use `com.theeverthread` as the Java root namespace, `theeverthread-backend` as
its Gradle project name, and `theeverthread-frontend` as the npm package name.
Spring properties and metric names use `theeverthread`; environment overrides
use `THEEVERTHREAD`. New refresh cookies use `THEEVERTHREAD_REFRESH`.
Deploy backend and frontend together; clients with the previous refresh cookie
must sign in again when their session expires.

Use `/opt/theeverthread` for the server deployment directory and
`theeverthread-dev` for the development Compose project. Publish packages as
`ghcr.io/<owner>/theeverthread-backend` and
`ghcr.io/<owner>/theeverthread-frontend`, independent of the repository slug.
Keep the stable executable artifact `app.jar`.

Preserve immutable applied Liquibase author/ID/path identities and SQL contents.
The application rename must not replay migrations, recreate existing databases,
replace credentials, or implicitly select an empty PostgreSQL volume. Existing
installations require explicit database and volume selection during cutover.
Existing external tracker addresses remain until their workspace is renamed.

Renaming GitHub changes the OIDC subject even when the repository ID is stable.
Gate deployments until IAM trust, verified role ARNs, registry pull access,
server paths, and Compose projects agree. Stop the old project before starting
its replacement; retain its deployment files and digests for rollback.
See the application repository's `docs/RENAMING.md` for the cutover sequence.
