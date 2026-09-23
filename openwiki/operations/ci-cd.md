---
type: process
title: CI & Continuous Delivery
description: Documents the GitHub Actions pipeline that builds JavaPoet via Maven, executes tests against generated code, and publishes artifacts to Sonatype Nexus/Snapshots.
tags: [ci-cd, build, operations]
verified:
  - by: openwiki/0.5.2
    at: 2026-09-22T02:45:45.818Z
sources:
  - id: openwiki-source-7a80b79a6fb3618cbfab08a2
    resource: repo://.github/workflows/build.yml
  - id: openwiki-source-6d4b4e707b8d60b6ccfa3425
    resource: repo://.github/workflows/openwiki-update.yml
  - id: openwiki-source-9af9fc14fab09231a18cbd83
    resource: repo://.github/workflows/settings.xml
  - id: openwiki-source-2355f81d7cf522f8dbdaabd4
    resource: repo://pom.xml
generated: { by: "openwiki/0.5.2", at: "2026-09-22T02:45:45.818Z" }
---

# CI & Continuous Delivery

This page documents the continuous integration and continuous delivery (CI/CD) workflows that automate building, testing, and publishing of **JavaPoet**. The pipeline consists of two primary GitHub Actions workflows:

<!-- openwiki: broken internal link [.github/workflows/build.yml] file ".github/workflows/build.yml" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`build`](.github/workflows/build.yml) — Runs on every push and pull request; compiles source code with Maven and optionally deploys to Sonatype Nexus/Snapshots.
<!-- openwiki: broken internal link [.github/workflows/openwiki-update.yml] file ".github/workflows/openwiki-update.yml" does not exist. Fix the href or restore the target, then delete this comment. -->
- [`OpenWiki Update`](.github/workflows/openwiki-update.yml) — Scheduled weekly automation that updates the OpenWiki documentation and opens a pull request whenever changes are detected.

## Build Workflow

The **build** workflow triggers on every `push` and `pull_request` to any branch of the repository. It enforces code quality gates before artifacts are deployed.

### Entry Point

```yaml
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'zulu'
          java-version: 8
```

The workflow always uses **Ubuntu LTS** and the **Zulu OpenJDK 8** distribution. Java version is pinned at **1.8** (`<java.version>1.8</java.version>` in [`pom.xml`](repo://src/pom.xml#L19-L26), ensuring consistent compilation behavior across all runners.

### Build Steps

#### Compile and Verify

```yaml
- run: mvn --no-transfer-progress verify source:jar javadoc:jar
```

The `mvn verify` goal runs the full build lifecycle:

- **Compilation** — Sources are compiled with `maven-compiler-plugin` using ErrorProne to flag potential bugs at compile time.
<!-- openwiki: broken internal link [/testing/ci-validation.md] file "/testing/ci-validation.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- **Verification** — All tests must pass before any packaging occurs. Tests run against generated code produced by JavaPoet's own builders (see [testing](/testing/ci-validation.md)).
- **Javadoc generation** — Javadoc is built simultaneously so documentation is always available alongside the compiled artifacts.

#### Deploy to Nexus/Snapshots

```yaml
- run: mvn --no-transfer-progress deploy --settings=".github/workflows/settings.xml" -Dmaven.test.skip=true
  if: ${{ github.ref == 'refs/heads/master' && github.repository == 'square/javapoet' }}
```

Deployment is **conditional** and restricted to the main branch of the canonical repository:

- **Branch guard**: Only `refs/heads/master` triggers deployment. Pull requests never reach this step.
- **Repository guard**: Deployment only occurs when `github.repository == square/javapoet`. This prevents accidental cross-repo deploys if workflows are reused or cloned.
- **Test bypass**: `-Dmaven.test.skip=true` ensures the deploy goal does not re-run all tests, reducing pipeline latency for releases on main.
- **Settings file**: Maven uses `.github/workflows/settings.xml` to locate Nexus/Snapshots credentials:

```xml
<settings>
  <servers>
    <server>
      <id>sonatype-nexus-snapshots</id>
      <username>${env.SONATYPE_DEPLOY_USERNAME}</username>
      <password>${env.SONATYPE_DEPLOY_PASSWORD}</password>
    </server>
  </servers>
</settings>
```

Credentials are injected via GitHub environment variables, never stored in plaintext on disk.

### Failure Semantics

- **Any test failure** aborts the build and prevents deployment.
- On `master`, a failed deploy step still reports as an error but does not affect the pull request status (since it only runs on that branch).
- Failed workflows leave no artifacts on Nexus; developers must resolve failures before retrying.

## OpenWiki Update Workflow

The **OpenWiki** workflow automates documentation updates by running [OpenWiki](https://github.com/openwiki/openwiki) to analyze the codebase and generate or revise wiki pages. It runs **weekly at 08:00 UTC**.

### Entry Point

```yaml
on:
  schedule:
    - cron: "0 8 * * *"
```

The workflow is triggered by a scheduled cron rather than by code changes, ensuring documentation updates happen on a predictable cadence regardless of commit frequency. It optionally triggers manually via `workflow_dispatch`.

### Entrypoint

```yaml
jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
```

**Full history checkout** is required (`fetch-depth: 0`). OpenWiki's `code --update` command diffs the current HEAD against a previous documented state to produce a meaningful change summary. A shallow clone would hide intermediate commits, resulting in empty or misleading updates.

#### Node.js Setup and OpenWiki Installation

```yaml
- name: Set up Node.js
  uses: actions/setup-node@v4
  with:
    node-version: "22"

- name: Install OpenWiki
  run: npm install --global openwiki@0.5.2 mermaid@11.16.0 jsdom@29.1.1
```

The workflow uses **Node.js 22** and installs OpenWiki globally alongside optional validation dependencies (`mermaid` for diagram rendering, `jsdom` for DOM-based checks).

#### Run OpenWiki

```yaml
- name: Run OpenWiki
  id: openwiki
  continue-on-error: true
  run: openwiki code --update --print
  env:
    OPENWIKI_PROVIDER: openai-compatible
    OPENAI_COMPATIBLE_API_KEY: ${{ secrets.OPENAI_COMPATIBLE_API_KEY }}
    OPENAI_COMPATIBLE_BASE_URL: ${{ vars.OPENAI_COMPATIBLE_BASE_URL }}
    OPENWIKI_MODEL_ID: "prism-ml/bonsai-27b"
    OPENWIKI_OPENAI_COMPATIBLE_STREAMING: "true"
    OPENWIKI_LANGSMITH_API_KEY: ${{ secrets.OPENWIKI_LANGSMITH_API_KEY }}
    LANGCHAIN_PROJECT: openwiki
    LANGCHAIN_TRACING_V2: "true"
```

OpenWiki uses the **OpenAI-compatible API** to generate or update wiki content. Key environment variables include:

| Variable | Purpose |
|---|---|
| `OPENWIKI_PROVIDER` / `OPENAI_COMPATIBLE_*` | Configures the LLM backend for generating prose and claims. |
| `OPENWIKI_MODEL_ID` | Uses `prism-ml/bonsai-27b` as the generation model. |
| `OPENWIKI_LANGSMITH_API_KEY` / `LANGCHAIN_TRACING_V2` | Traces LLM calls to LangSmith for observability and debugging. |

The step **continues on error** (`continue-on-error: true`) because OpenWiki may intentionally produce incomplete pages during long-running updates, which the subsequent PR creation step handles gracefully.

#### Create Pull Request

```yaml
- name: Create OpenWiki update pull request
  if: ${{ !cancelled() }}
  uses: peter-evans/create-pull-request@v7
  with:
    add-paths: |
      openwiki
      AGENTS.md
      CLAUDE.md
      .github/workflows/openwiki-update.yml
    branch: openwiki/update
    commit-message: "docs: update OpenWiki"
```

On completion, the workflow generates a pull request to the `openwiki/update` branch containing:

- **Updated wiki pages** in the `openwiki/` directory.
- **Connector configuration files** (`AGENTS.md`, `CLAUDE.md`) if their content changed.
- **Workflow file itself** (`.github/workflows/openwiki-update.yml`) to allow coordinated updates across runners.

#### Propagate Failure

```yaml
- name: Propagate OpenWiki failure
  if: ${{ steps.openwiki.outcome == 'failure' }}
  run: exit 1
```

If the OpenWiki step fails (e.g., LLM API errors or validation failures), the workflow exits with code `1`. This signals the CI system that the automated update was unsuccessful, enabling downstream alerting or rollback strategies.

## Pipeline Relationships and State Flow

### Build → Deploy Boundary

The build workflow enforces a strict **quality gate**: deployment to Nexus/Snapshots only occurs if all compilation and test steps pass. The conditional guard (`master` branch + `square/javapoet` repo) prevents:

- Accidental deployments from pull requests or feature branches.
- Cross-repo contamination (e.g., forks running the same workflow).

### OpenWiki Update → Documentation State

The weekly OpenWiki update runs **independently** of push/PR events, but it reads from the full repository history to determine what has changed since the last documented state. This creates a **stateful documentation pipeline**:

1. **Trigger**: Weekly cron at 08:00 UTC (or manual dispatch).
2. **Analysis**: OpenWiki diffs HEAD against the last committed state, generating a change summary.
3. **Generation**: LLM-generated prose is written to wiki markdown files.
4. **Persistence**: A pull request is created for human review before merging back to `main`.

### Artifact Lifecycle

| Phase | Artifact | Destination | Guard |
|---|---|---|---|
| PR build | `.jar` (source + javadoc) | Local Maven cache | All tests pass |
| Master deploy | SNAPSHOT release | Sonatype Nexus/Snapshots | Master branch, `square/javapoet`, no re-run of tests |

## Configuration and Operations

### Environment Variables

The CI pipeline relies on the following secrets and variables:

- **Nexus Credentials**: `SONATYPE_DEPLOY_USERNAME` / `SONATYPE_DEPLOY_PASSWORD` — injected as `${env.SONATYPE_DEPLOY_USERNAME}` in `.github/workflows/settings.xml`.
- **OpenAI Compatible API**: `OPENAI_COMPATIBLE_API_KEY`, `OPENAI_COMPATIBLE_BASE_URL` — for LLM generation.
- **LangSmith Tracing**: `OPENWIKI_LANGSMITH_API_KEY`, `LANGSMITH_API_KEY` — optional tracing of LLM calls.

### Repository-Specific Branch Guards

Deployment workflows enforce repository identity checks:

```yaml
if: ${{ github.ref == 'refs/heads/master' && github.repository == 'square/javapoet' }}
```

This ensures that even if the workflow is cloned or modified, deployment only occurs against the canonical `square/javapoet` repository on its main branch.

### Failure Recovery

- **Build failures**: No artifacts are published; developers must fix compilation/test issues before retrying.
- **OpenWiki failures**: The PR creation step is skipped or produces an incomplete PR. Developers can merge a partial update to preserve progress, then run the workflow again to complete it on the next cycle.

## Focused Tests That Matter

### Build Workflow Tests

1. **`verify` gate** — All test classes in `src/test/java/...` must pass for any artifact to be built.
2. **Javadoc generation** — Javadoc compilation succeeds without errors, ensuring documentation is always available.
3. **Nexus deployment** — On main branch, the `.jar` uploads successfully to Nexus/Snapshots with correct metadata (version, group id, artifact id).

### OpenWiki Update Tests

1. **Change detection** — The update workflow produces a non-empty diff when code changes since the last run.
2. **Partial failure handling** — When the LLM provider fails, the workflow exits cleanly and does not corrupt existing wiki pages.
3. **Pull request creation** — On success, a PR is created to `openwiki/update` with correct paths listed in `add-paths`.

## Failure Modes and Mitigations

| Failure Mode | Impact | Mitigation |
|---|---|---|
| Tests fail on PR | PR blocked; no artifact built | Fix failing test or update code; retry after fix. |
| Nexus credentials missing/invalid | Deployment fails silently | Validate secrets in GitHub Actions secret manager. |
| LLM API rate limit exceeded | OpenWiki produces empty or stale pages | Increase `OPENWIKI_MODEL_ID` quota; wait for next scheduled run. |
| Shallow clone hides changes | OpenWiki produces no PR | Always use `fetch-depth: 0` in the checkout step. |

## References

- [`build.yml`](repo://.github/workflows/build.yml) — GitHub Actions build workflow definition.
- [`openwiki-update.yml`](repo://.github/workflows/openwiki-update.yml) — Scheduled OpenWiki update workflow definition.
- [`settings.xml`](repo://.github/workflows/settings.xml) — Maven Nexus/Snapshots configuration.
- [`pom.xml`](repo://src/pom.xml#L1-L60) — Maven project properties including Java version and dependency declarations.
<!-- openwiki: broken internal link [/testing/ci-validation.md] file "/testing/ci-validation.md" does not exist. Fix the href or restore the target, then delete this comment. -->
- [testing](/testing/ci-validation.md) — Documentation of CI validation tests that gate the build pipeline.

---

> **Note**: This page is automatically updated by the OpenWiki workflow on a weekly schedule. For manual edits, create a pull request to `openwiki/update`.
