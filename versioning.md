# BetterF versioning and daily development

BetterF currently ships its backend and frontend together. The root [`VERSION`](../VERSION) file is the source of the application version, initially `0.1.0`. Frontend package metadata mirrors it; Gradle reads it directly.

**Every merge can deploy without receiving a new release number.** Ordinary branches do not reserve or increment a sequence number. Each development deployment is already identified by its Git commit and the two image digests. A named release groups a deliberately selected, verified set of changes.

## Three identifiers, three purposes

| Identifier | Meaning | Changes when |
|---|---|---|
| Application version, e.g. `0.1.0` | Shared human-facing release designation | Preparing an intentional release |
| Full Git commit SHA | Source revision | A new commit is created |
| Backend and frontend image digests | Exact images selected for deployment | Image contents change |

Several development builds can carry version `0.1.0` while having different commits and image digests. That does not mean several different official `v0.1.0` releases exist. Official release tags must identify one verified commit and should not be moved after publication. The version field alone must never be used to identify a development deployment.

Dependency versions such as Spring Boot, Angular, Java and PostgreSQL are separate from the BetterF release number. They are changed deliberately in their own configuration, with appropriate tests.

## Daily sequence: no version bump required

1. Branch from current `develop`, implement the change, and add appropriate checks. Leave `VERSION` alone for normal feature and fix PRs.
2. Run `node scripts/version.mjs --check` from the repository root, plus the application checks relevant to the change. The version check is also part of CI.
3. Open a PR and review it. PR workflows verify code; they do not publish or deploy application images.
4. Merge into `develop`. With repository variable `AWS_DEPLOY_ENABLED=true`, successful checks lead to image publication and the development deployment.
5. Identify that deployment by the Actions run, source commit and image digests. `/opt/betterf/current.env` records the last successful image references; inspect running containers when diagnosing partial failures.

Two branches can both keep version `0.1.0` without a version conflict. If another PR changes the release version while yours is open, update your branch and use the authoritative merged value rather than inventing a branch-specific version.

## Release sequence: deliberately choose the version

1. Select a batch of work ready for a named release. Choose the next version and prepare a release PR.
2. Edit the root `VERSION` file. For example, a new initial-development milestone might change `0.1.0` to `0.2.0`.
3. Synchronize the checked-in npm metadata from that file:

   ```bash
   node scripts/version.mjs --sync
   node scripts/version.mjs --check
   ```

4. Review and commit `VERSION`, `app/frontend/package.json`, and `app/frontend/package-lock.json`, plus release notes. The synchronization changes only the root package versions; it does not resolve or upgrade dependencies, create Git tags, or commit anything.
5. Let the release PR pass checks, merge, and verify the resulting development deployment. The backend and frontend now carry the same application release number even if only backend code changed.
6. Record the exact successful source commit and both image digests. After verification, create the release tag (for example `v0.2.0`) on that exact commit and publish release notes through your normal release process. Do not tag an unverified current branch tip by assumption.
7. When staging/production is introduced, promote the tested image digests. Avoid rebuilding different images merely to promote the release.

Steps 6–7 describe release procedure; automatic tagging, release publication and production promotion are **not implemented** by the current pipeline. Tag pushes do not pass its `refs/heads/develop` publishing condition. A manual run on `develop` deploys that branch, not an arbitrary release tag.

Do not overwrite the meaning of an already-published release. If it needs a fix, make a new release. You may keep the last version number during ordinary development or deliberately prepare the next prerelease version; commit/digest identity remains necessary either way.

## Choosing numbers

Use `MAJOR.MINOR.PATCH`, with optional prerelease identifiers such as `0.2.0-rc.1`.

- During `0.x`, the contract is still evolving. For this project, use minor increments for development milestones and patch increments for fixes to those milestones; document breaking changes explicitly.
- At `1.0.0`, define the public behavior/API contract you promise to support.
- Thereafter, use patches for compatible fixes, minor releases for compatible additions, and major releases for breaking supported-contract changes.

These post-1.0 compatibility rules follow [Semantic Versioning](https://semver.org/). A product version and an API URL such as `/api/v1` are different concerns; do not rename API routes merely because the application version changed.

## Files and enforcement

| File | Responsibility |
|---|---|
| [`VERSION`](../VERSION) | Authoritative SemVer value, no `v` prefix |
| [`scripts/version.mjs`](../scripts/version.mjs) | Validates SemVer and checks/synchronizes npm metadata |
| [`scripts/version.test.mjs`](../scripts/version.test.mjs) | Tests drift detection, validation and dependency preservation |
| [`app/backend/build.gradle`](../app/backend/build.gradle) | Reads `../../VERSION`, emits `build/libs/app.jar`, embeds `Implementation-Version` in the JAR manifest |
| [`app/frontend/package.json`](../app/frontend/package.json) | Checked-in mirror of the app version |
| [`app/frontend/package-lock.json`](../app/frontend/package-lock.json) | Mirrors the version in the lockfile and its root package entry |
| [Workflow](../.github/workflows/ci.yml) | Runs the check before application builds and consumes a stable JAR path |

CI uses `--check`, never `--sync`: inconsistencies fail visibly rather than being silently repaired during a build. Run the synchronization command locally and review its diff. The script resolves the repository relative to its own location, not the caller's working directory. It needs Node but no installed npm dependencies.

The frontend version in package metadata is not automatically displayed in the UI. The backend manifest version is not automatically a public HTTP endpoint. Neither feature is added by this change.

## What happens to deployment files when the version changes?

The executable artifact name is always `app/backend/build/libs/app.jar`. CI starts it for checks, uploads it, and downloads it into `package/backend/app.jar`. The backend Dockerfile already copies `app.jar`; no rename tied to the release number is needed.

**Do not edit CI or Dockerfiles for each version bump.** The frontend output directory also stays `app/frontend/dist/frontend/browser/`. Compose and `deploy.sh` continue receiving digest references. Database configuration, passwords, migrations and IAM policies are unrelated to changing the release number.

The workflow still tags images with the Git commit SHA. Human-readable release image tags, release manifests and version labels in image metadata are possible later improvements, not part of this implementation. Exact image selection continues to use digests.

## Verify locally

From the repository root:

```bash
node --test scripts/version.test.mjs
node scripts/version.mjs --check
cd app/backend
./gradlew bootJar --no-daemon
```

The output must be `app/backend/build/libs/app.jar` regardless of the number in `VERSION`. Its manifest includes `Implementation-Version` equal to the root value. Frontend dependency locks remain unchanged apart from the root version fields after synchronization.

If old versioned JARs remain from earlier local builds, they are stale outputs. CI selects `app.jar` explicitly; a normal Gradle `clean` can clear generated build outputs when desired. Never use a broad JAR wildcard that could accidentally select a plain or stale JAR.

## Future independent releases

Separate backend/frontend versions are useful if they eventually ship independently. That requires separate release records and a compatibility policy covering API consumers, cached browser clients and migrations. Until that is needed, a shared release version keeps this repository's delivery process straightforward.
