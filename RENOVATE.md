# Renovate configuration

This document describes how Renovate dependency updates are configured and validated in this repository. The setup is a reference implementation for the adaptTo() 2026 session [Securing AEM Customer Code: Automated Fixes for Vulnerabilities in Dependencies](https://adapt.to/2026/schedule/securing-aem-customer-code-automated-fixes-for-vulnerabilities-in-dependencies).

The configuration has two main parts:

- [`renovate.json`](renovate.json) controls dependency discovery, update grouping, version constraints, labels, and automerge behavior.
- [`.github/workflows/renovate-validation.yml`](.github/workflows/renovate-validation.yml) validates Renovate pull requests with Maven and, when required, an Adobe Cloud Manager pipeline.

## General Renovate behavior

The repository extends Renovate's recommended configuration and adds these presets:

| Preset | Effect |
|---|---|
| `config:recommended` | Enables Renovate's recommended defaults. |
| `:separateMajorReleases` | Keeps major upgrades separate from minor and patch upgrades. |
| `:combinePatchMinorReleases` | Allows compatible minor and patch updates to share a branch/PR. |
| `:ignoreUnstable` | Avoids unstable versions unless the dependency already uses an unstable version. |
| `:prImmediately` | Creates a PR as soon as an update branch is available. |
| `:semanticPrefixFixDepsChoreOthers` | Uses semantic commit prefixes: `fix` for production dependency updates and `chore` for others. |
| `:updateNotScheduled` | Allows updates without a restricted Renovate schedule. |
| `:prHourlyLimitNone` | Removes the hourly PR creation limit. |

Additional repository-wide settings are:

- Every Renovate PR receives the `dependencies` label by default.
- At most **10 Renovate PRs** may be open concurrently.
- `rangeStrategy: bump` updates the declared version range rather than only refreshing lockfiles.
- Releases must be at least **three days old** before Renovate proposes them.
- OSV vulnerability alerts are enabled with `osvVulnerabilityAlerts: true`.

## Dependency grouping and policy

The package rules reduce PR noise while applying validation and automerge policies appropriate to each dependency class.

| Dependency class | Behavior |
|---|---|
| npm `devDependencies`, minor/patch | Grouped as **npm dev dependencies - fixes**, except Cypress packages; labeled `auto-merge` and configured for automerge. |
| Maven test dependencies, minor/patch | Grouped as **maven test dependencies - fixes**, except AEM Core Components; labeled `mvn-validation-only` and `auto-merge`, and configured for automerge. |
| Major updates | Labeled `major-update` so they stand out and configured not to automerge by the major-update rule. |
| Generic Maven plugins | `org.apache.maven.plugins:**` and `org.codehaus.mojo:**` are grouped as **Maven plugins** and labeled `mvn-validation-only`. |
| AEM Core Components | All `com.adobe.cq:core.wcm.components.**` artifacts are grouped into one PR. |
| Babel | `@babel/**` and `babel-**` packages are grouped as **Babel**. |
| webpack | webpack, webpack plugins, and loaders are grouped as **webpack**. |
| ESLint | ESLint and TypeScript-ESLint packages are grouped as **ESLint**, labeled `auto-merge`, and configured for automerge. |
| Cypress | Cypress packages are grouped as **Cypress** and excluded from the generic npm development-dependency group. |
| Maven test plugins | Surefire and Failsafe are grouped because they share a version; labeled `mvn-validation-only` and `auto-merge`, and configured for automerge. |

### Version compatibility constraints

Some versions are deliberately held below known compatibility boundaries:

- `org.apache.sling:org.apache.sling.models.impl` stays below `2.0.0` for AEM Cloud Service compatibility.
- `@babel/core` stays below `8.0.0` because some build tools do not yet support Babel 8.
- `typescript` stays below `6.0.0` because some build tools do not yet support TypeScript 6 or later.

### Rule-order note

Renovate combines every matching package rule, and later scalar settings can override earlier ones. In the current file, the ESLint and Maven test-plugin rules occur after the major-update rule and set `automerge: true`. Consequently, a matching major update may have automerge re-enabled by those later rules. If the requirement that major updates must never automerge is strict, move the major-update rule to the end of `packageRules` or restrict the automerge rules to `minor` and `patch` updates.

## Labels and validation levels

The labels communicate both update risk and the required validation path:

| Label | Meaning |
|---|---|
| `dependencies` | Standard label added to dependency update PRs. |
| `major-update` | Highlights a major version upgrade for additional review. |
| `auto-merge` | Identifies an update Renovate is configured to merge after repository requirements pass. |
| `mvn-validation-only` | The Maven build is sufficient; the Cloud Manager pipeline is skipped. |

The `auto-merge` label is descriptive; actual automatic merging is controlled by Renovate's `automerge` setting and remains subject to GitHub branch protection and required status checks.

## GitHub Actions validation

The **Renovate branch validation** workflow runs for pull request changes and manual dispatches. For pull requests, its Maven job is limited to branches whose head name starts with `renovate/`.

### 1. Maven verification

Every Renovate PR first runs the `verify` job:

1. Check out the update branch.
2. Install Temurin Java 21 and enable the Maven dependency cache.
3. Run `mvn --batch-mode --no-transfer-progress clean verify`.

Superseded workflow runs for the same PR are cancelled when Renovate updates the branch.

### 2. Cloud Manager verification

After Maven succeeds, the `cloud-manager` job runs unless the PR has the `mvn-validation-only` label. It:

1. Installs Adobe I/O CLI 11.1.2 and `@adobe/aio-cli-plugin-cloudmanager`.
2. Configures OAuth Server-to-Server authentication.
3. Changes the configured Cloud Manager pipeline to use the Renovate branch.
4. Starts a pipeline execution.
5. Polls every 30 seconds for up to 240 attempts (approximately two hours).
6. Passes on `FINISHED` and fails on an error, failure, cancellation, or timeout.

Cloud Manager permits only one pipeline session at a time. A fixed GitHub Actions concurrency group named `cloud-manager-pipeline` serializes pipeline jobs across all PRs without cancelling an execution already in progress.

## Required GitHub configuration

Configure these under **Settings → Secrets and variables → Actions**.

### Repository variables

| Variable | Required | Purpose |
|---|---:|---|
| `CM_PROGRAM_ID` | Yes | Cloud Manager program containing the pipeline. |
| `CM_PIPELINE_ID` | Yes | Pipeline to update and execute. |
| `CM_IMS_ENV` | No | IMS environment (`prod` or `stage`); defaults to `prod`. |
| `CM_BASE_URL` | No | Alternative Cloud Manager API base URL; when unset, the production endpoint is used. |

For a stage environment, set both `CM_IMS_ENV=stage` and `CM_BASE_URL` to the stage API URL.

### Repository secrets

| Secret | Purpose |
|---|---|
| `CM_CLIENT_ID` | OAuth Server-to-Server client ID. |
| `CM_CLIENT_SECRET` | OAuth client secret. |
| `CM_TECHNICAL_ACCOUNT_ID` | Adobe technical account ID. |
| `CM_TECHNICAL_ACCOUNT_EMAIL` | Adobe technical account email. |
| `CM_IMS_ORG_ID` | Adobe IMS organization ID. |
| `CM_SCOPES` | Comma-separated scopes from the Adobe Developer Console credential. |

## End-to-end update flow

1. Renovate scans Maven and npm dependency declarations.
2. It applies the age threshold, compatibility constraints, grouping rules, and labels from `renovate.json`.
3. Renovate creates a `renovate/*` branch and pull request.
4. GitHub Actions runs the full Maven verification build.
5. For updates without `mvn-validation-only`, GitHub Actions also executes the Cloud Manager pipeline against that branch.
6. Automerge-eligible updates can merge after all repository and branch-protection requirements pass; other updates remain for manual review and merge.
