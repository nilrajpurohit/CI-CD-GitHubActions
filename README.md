# CI/CD Roadmap with GitHub Actions

A complete, phase-by-phase guide to building a production-grade CI/CD pipeline using GitHub Actions — from source control governance through to production monitoring and rollback.

---

## Table of Contents

1. [Phase 1 — Source Control](#phase-1--source-control)
2. [Phase 2 — Build](#phase-2--build)
3. [Phase 3 — Test](#phase-3--test)
4. [Phase 4 — Release](#phase-4--release)
5. [Phase 5 — Deploy](#phase-5--deploy)
6. [Phase 6 — Monitor & Feedback](#phase-6--monitor--feedback)
7. [Phase 7 — Expert Patterns](#phase-7--expert-patterns)
   - [7.1 Reusable Workflows](#71--reusable-workflows)
   - [7.2 Composite Actions](#72--composite-actions)
   - [7.3 Advanced Workflow Control](#73--advanced-workflow-control)
   - [7.4 Matrix Strategy (Advanced)](#74--matrix-strategy-advanced)
   - [7.5 Artifacts vs Caching (Deep Dive)](#75--artifacts-vs-caching-deep-dive)
   - [7.6 Permissions & Security Hardening](#76--permissions--security-hardening)
   - [7.7 Debugging & Observability](#77--debugging--observability)
   - [7.8 Workflow Optimization](#78--workflow-optimization)
   - [7.9 Workflow Outputs & Data Passing](#79--workflow-outputs--data-passing)
   - [7.10 Event System (Deep Dive)](#710--event-system-deep-dive)

---

## Phase 1 — Source Control

> Repository setup & governance

### Repository Structure

- Create a dedicated GitHub repository per service or a monorepo depending on your team size.
- Use a `.github/` directory to store all workflows, issue templates, and PR templates.
- Add a `.gitignore` appropriate for your stack to avoid committing build artifacts or env files.
- Include a `README.md` with setup instructions, architecture overview, and CI/CD flow description.

### Branching Strategy

- Adopt trunk-based development (short-lived branches) or GitFlow (feature/release/hotfix branches) consistently across the team.
- Name branches with a prefix convention: `feature/`, `fix/`, `chore/`, `hotfix/`.
- Protect `main` and `develop` branches from direct pushes — all changes must go through a PR.
- Require at least one approved review and all status checks to pass before a branch can be merged.
- Enable "Require branches to be up to date before merging" to prevent stale merges.

### Secrets & Environment Variables

- Store all sensitive values (API keys, tokens, passwords) in GitHub Secrets under **Settings → Secrets and variables → Actions**.
- Use repository-level secrets for shared values and environment-level secrets for staging vs production separation.
- Never hardcode secrets in workflow YAML files — always reference them as `${{ secrets.MY_SECRET }}`.
- Rotate secrets regularly and audit access logs to detect unauthorized usage.

---

## Phase 2 — Build

> Workflow triggers, dependencies & artifacts

### Workflow Triggers

- Define workflows in `.github/workflows/*.yml` — each file is an independent pipeline.
- Use `on: push` for automatic builds on every commit and `on: pull_request` to validate PRs before merge.
- Add `paths:` filters to only trigger builds when relevant files change, e.g. only run the backend pipeline when `src/backend/**` changes.
- Use `on: workflow_dispatch` to allow manual workflow runs directly from the GitHub Actions UI.
- Use `on: schedule` with a cron expression for nightly builds or dependency audit jobs.

### Runner & Environment Setup

- Choose a runner with `runs-on: ubuntu-latest` for most workloads — it is the fastest and cheapest GitHub-hosted option.
- Use `actions/setup-node`, `actions/setup-python`, or `actions/setup-java` to install the correct runtime version.
- Pin action versions with a full commit SHA (e.g. `actions/checkout@v4`) to prevent supply chain attacks from floating tags.
- Use self-hosted runners for workloads that need special hardware, private network access, or persistent caches.

### Dependency Caching

- Use `actions/cache` to cache package manager directories (`node_modules`, `.pip`, `.m2`) between workflow runs.
- Key the cache on the lockfile hash: `hashFiles('**/package-lock.json')` so the cache invalidates only when dependencies actually change.
- Always use `npm ci` instead of `npm install` in CI — it is deterministic, faster, and fails if the lockfile is out of sync.
- Add a `restore-keys:` fallback to use a partial cache on a cache miss rather than a cold install.

### Build & Artifact Upload

- Run your build command (e.g. `npm run build`, `mvn package`, `go build`) as a step after dependencies are installed.
- Upload compiled output using `actions/upload-artifact` so downstream jobs can download it without rebuilding.
- Set `retention-days: 7` on artifacts — keep them long enough for debugging but short enough to control storage costs.
- Use `needs:` in downstream jobs to declare dependency on the build job, ensuring correct execution order.

### Example Workflow Snippet

```yaml
name: Build

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Cache dependencies
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: ${{ runner.os }}-node-

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
          retention-days: 7
```

---

## Phase 3 — Test

> Unit, integration, lint & coverage

### Unit Tests

- Run unit tests as a separate job using frameworks like Jest (JS), pytest (Python), JUnit (Java), or RSpec (Ruby).
- Use a matrix strategy (`strategy.matrix`) to run tests across multiple Node/Python/Java versions simultaneously.
- Generate a code coverage report (e.g. `--coverage` in Jest) and upload it as an artifact or send it to Codecov/Coveralls.
- Set a minimum coverage threshold — fail the job if coverage drops below your defined floor (e.g. 80%).
- Use `continue-on-error: false` (the default) to ensure any test failure blocks the pipeline immediately.

### Linting & Static Analysis

- Run a linter (ESLint, Flake8, RuboCop, golangci-lint) as a parallel job to the unit tests to save total pipeline time.
- Enforce code formatting with Prettier, Black, or gofmt — fail the job if files are not properly formatted.
- Integrate a SAST (static application security testing) tool like CodeQL or Semgrep using `github/codeql-action`.
- Run dependency vulnerability scanning with `npm audit`, `pip-audit`, or Dependabot alerts to catch known CVEs.
- Use `actions/dependency-review-action` on PRs to block merges that introduce vulnerable packages.

### Integration Tests

- Spin up real service dependencies using the `services:` block in your workflow — e.g. Postgres, Redis, RabbitMQ run as Docker containers on the runner.
- Use `options: --health-cmd` to wait for services to be ready before test steps begin, preventing flaky race conditions.
- Run API and contract tests (e.g. Postman/Newman, Pact) against the locally built application to verify service boundaries.
- Run end-to-end tests with Playwright or Cypress against a locally started server, capturing screenshots on failure as artifacts.

### Example Test Workflow with Services

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    strategy:
      matrix:
        node-version: [18, 20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test -- --coverage
```

---

## Phase 4 — Release

> Versioning, containers & changelogs

### Semantic Versioning

- Follow Semantic Versioning: `MAJOR.MINOR.PATCH` — bump major for breaking changes, minor for new features, patch for bug fixes.
- Use `google-github-actions/release-please-action` to automate version bumping based on Conventional Commits in PR titles.
- Enforce Conventional Commits format (`feat:`, `fix:`, `chore:`) using commitlint in a PR check workflow.

### Container Images

- Build Docker images using `docker/build-push-action` and tag them with both the commit SHA and the version tag for traceability.
- Push images to GitHub Container Registry (GHCR) using `docker/login-action` with `secrets.GITHUB_TOKEN` — no separate credentials needed.
- Scan the built Docker image for vulnerabilities with `aquasecurity/trivy-action` before pushing to the registry.
- Use multi-stage Dockerfiles to keep final images small — compile in a builder stage, copy only the binary into a minimal runtime image.

### GitHub Release & Changelog

- Create a GitHub Release automatically using `softprops/action-gh-release` — attach binary artifacts, checksums, and a generated changelog.
- Auto-generate release notes from merged PR titles using GitHub's built-in release notes feature configured via `.github/release.yml`.
- Run the release job only on the `main` branch when a version tag is pushed, using `if: startsWith(github.ref, 'refs/tags/v')`.

### Example Release Workflow Snippet

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:latest

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.ref_name }}

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
```

---

## Phase 5 — Deploy

> Staging, approvals & production strategies

### GitHub Environments

- Create named environments in **GitHub Settings → Environments**: `staging`, `production`, each with its own secrets and rules.
- Set Required Reviewers on the production environment — this creates a manual approval gate that pauses the workflow until a named person approves.
- Set a Wait Timer on the production environment (e.g. 5 minutes) to allow time for smoke tests on staging before production deploy auto-proceeds.
- Restrict which branches can deploy to production using environment deployment branch rules — only `main` or release tags.

### Staging Deployment

- Deploy automatically to staging on every merge to `main` — staging should always reflect the latest state of the main branch.
- Use deployment actions specific to your platform: `aws-actions/amazon-ecs-deploy-task-definition`, `azure/webapps-deploy`, or `google-github-actions/deploy-cloudrun`.
- Run smoke tests against the staging URL after deployment using `curl` health check steps or a lightweight test suite.
- Post a Slack or GitHub comment with the staging URL and deployed commit SHA for quick access by reviewers and QA.

### Production Deployment Strategies

| Strategy | Description | Best For |
|---|---|---|
| **Blue/Green** | Maintain two identical environments, switch traffic after health checks pass | Zero-downtime, instant rollback |
| **Canary** | Gradually shift traffic (5% → 25% → 100%), monitor error rates at each step | Risk-averse, high-traffic services |
| **Rolling** | Replace instances one at a time, no full outage | Kubernetes, ECS, managed platforms |
| **Feature Flags** | Deploy code to production, enable features independently | Decoupling deploy from release |

- Always verify deployment success by polling the health endpoint (`/health` or `/ready`) in a post-deploy step before marking the job successful.

### Example Deploy Workflow Snippet

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    needs: [build, test]
    steps:
      - name: Deploy to staging
        run: echo "Deploy to staging here"

      - name: Smoke test
        run: |
          curl --fail https://staging.example.com/health

  deploy-production:
    runs-on: ubuntu-latest
    environment: production       # triggers required reviewer approval
    needs: [deploy-staging]
    steps:
      - name: Deploy to production
        run: echo "Deploy to production here"

      - name: Verify health
        run: |
          curl --fail https://example.com/health
```

---

## Phase 6 — Monitor & Feedback

> Alerts, rollback & observability

### Notifications & Alerting

- Send Slack notifications on pipeline success or failure using `slackapi/slack-github-action` with a formatted message including repo, branch, and job status.
- Use `if: failure()` conditionals to send alerts only on failures — avoid notification noise from routine successful builds.
- Post deployment annotations to your monitoring tool (Datadog, Grafana) so you can correlate metric changes with exact deploy events.
- Email or PagerDuty alerts on production deployment failures — these should wake someone up, not just fill a Slack channel.

### Automated Rollback

- Add a rollback job that triggers automatically if the post-deploy health check step fails — re-deploy the previous known-good image tag.
- Store the last stable image tag in a GitHub environment variable or SSM Parameter Store so the rollback job always knows what to revert to.
- Use `if: failure()` on the rollback job and `needs: [deploy]` to make it dependent on the deploy job completing first.
- Test your rollback procedure in staging regularly — a rollback that has never been exercised is one you cannot rely on in a real incident.

### Observability Integration

- Instrument your application with structured logs (JSON format), metrics (Prometheus/StatsD), and distributed traces (OpenTelemetry).
- Set up dashboards in Grafana or Datadog with SLIs: error rate, latency (p50/p95/p99), and request throughput — visible immediately after each deploy.
- Create deployment markers in your APM tool from within the GitHub Actions workflow using the provider's CLI or API call step.
- Track pipeline metrics over time — build duration, test pass rate, deploy frequency, and mean time to recovery (MTTR) — to identify bottlenecks and improve your DORA scores.
- Review failed workflow runs weekly using the GitHub Actions insights tab to catch recurring flaky tests or slow steps that should be optimized.

### Example Rollback & Notify Snippet

```yaml
  rollback:
    runs-on: ubuntu-latest
    needs: [deploy-production]
    if: failure()
    steps:
      - name: Rollback to previous version
        run: echo "Re-deploy last stable image"

      - name: Notify Slack on rollback
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": ":rotating_light: Production rollback triggered for ${{ github.repository }} on commit ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Phase 7 — Expert Patterns

> Advanced GitHub Actions design for real-world teams

---

### 7.1 🔁 Reusable Workflows

Reusable workflows allow you to define a pipeline once and call it from multiple repositories or workflows — eliminating YAML duplication across teams.

**Key concept:** A reusable workflow is triggered by `workflow_call` instead of `push` or `pull_request`. Any workflow file with `on: workflow_call` can be called by other workflows using the `uses:` keyword.

**Why it matters:** Real teams don't copy-paste YAML across repos. They build centralized CI/CD building blocks maintained in one place (a platform/infra repo) and call them from every service repo. This is the equivalent of writing a function instead of duplicating code.

**Defining a reusable workflow** (`.github/workflows/reusable-build.yml` in a central repo):

```yaml
name: Reusable Build Pipeline

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '20'
      working-directory:
        required: false
        type: string
        default: '.'
    secrets:
      NPM_TOKEN:
        required: false
    outputs:
      artifact-name:
        description: "Name of the uploaded build artifact"
        value: ${{ jobs.build.outputs.artifact-name }}

jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    outputs:
      artifact-name: ${{ steps.set-output.outputs.name }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci
      - run: npm run build
      - id: set-output
        run: echo "name=build-${{ github.sha }}" >> $GITHUB_OUTPUT
      - uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
```

**Calling a reusable workflow** from a service repo:

```yaml
name: CI

on:
  push:
    branches: [main]

jobs:
  build:
    uses: my-org/platform-workflows/.github/workflows/reusable-build.yml@main
    with:
      node-version: '20'
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

  test:
    needs: build
    uses: my-org/platform-workflows/.github/workflows/reusable-test.yml@main
```

**Cross-repo reuse best practices:**
- Version your reusable workflows using tags (`@v1`, `@v2`) so callers opt into breaking changes.
- Store all shared workflows in a dedicated `platform-workflows` or `ci-templates` repository.
- Use `inputs` for configuration and `secrets: inherit` when callers should pass all their secrets automatically.
- Document every `inputs` field with a description — it becomes the API contract for all callers.

---

### 7.2 🧩 Composite Actions

Composite actions are custom reusable steps packaged as a local or published action — different from reusable workflows in that they operate at the **step level**, not the job level. They are defined in an `action.yml` file and can be invoked using `uses:` just like any marketplace action.

**When to use composite vs reusable workflow:**

| | Composite Action | Reusable Workflow |
|---|---|---|
| Scope | Steps within a job | Entire jobs |
| Secrets access | Caller's context | Explicit `secrets:` passing |
| Runner | Runs on caller's runner | Can define its own runner |
| Best for | Repeated step bundles | Full pipeline templates |

**Defining a composite action** (`.github/actions/setup-and-build/action.yml`):

```yaml
name: 'Setup, Install & Build'
description: 'Sets up Node, installs deps with cache, and builds the project'

inputs:
  node-version:
    description: 'Node.js version to use'
    required: false
    default: '20'
  build-command:
    description: 'Build command to run'
    required: false
    default: 'npm run build'

outputs:
  cache-hit:
    description: 'Whether the npm cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: 'composite'
  steps:
    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Cache npm
      id: cache
      uses: actions/cache@v4
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: ${{ runner.os }}-node-

    - name: Install dependencies
      run: npm ci
      shell: bash

    - name: Build
      run: ${{ inputs.build-command }}
      shell: bash
```

**Using a composite action in a workflow:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup and build
        uses: ./.github/actions/setup-and-build
        with:
          node-version: '20'
          build-command: 'npm run build:prod'

      - name: Run tests
        run: npm test
```

**Common composite action patterns for teams:**
- A `setup-env` action that installs runtime, configures auth, and validates environment variables.
- A `notify-slack` action wrapping Slack notification logic with standardized message format.
- A `docker-build-push` action combining Buildx setup, login, build, scan, and push in one reusable block.
- An internal `deploy-service` action with your company's deployment conventions built in.

---

### 7.3 ⚡ Advanced Workflow Control

Beyond basic `needs:` and `if:` usage, GitHub Actions provides fine-grained control primitives that make pipelines efficient, safe, and production-ready.

#### Concurrency — Cancel Stale Runs

Prevent redundant runs by cancelling in-progress workflows when a new commit is pushed to the same branch:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Use a different group key for production deployments to prevent cancellation of in-flight deploys:

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false   # never cancel a deploy already in progress
```

#### Complex `if:` Conditions

```yaml
jobs:
  deploy:
    # Only deploy on push to main, not on PR events or forks
    if: |
      github.event_name == 'push' &&
      github.ref == 'refs/heads/main' &&
      github.repository == 'my-org/my-repo'

  notify:
    # Always notify, even if previous jobs failed
    if: always()

  integration-test:
    # Skip if commit message contains [skip-tests]
    if: "!contains(github.event.head_commit.message, '[skip-tests]')"
```

#### `continue-on-error` — Non-Blocking Steps

Mark certain steps as optional so pipeline continues even if they fail:

```yaml
steps:
  - name: Run experimental linter
    run: npx experimental-linter
    continue-on-error: true    # warning, not a blocker

  - name: Upload coverage to Codecov
    uses: codecov/codecov-action@v4
    continue-on-error: true    # flaky third-party, don't fail pipeline
```

#### `fail-fast` in Matrix

By default, GitHub cancels all in-progress matrix jobs when one fails. Control this behavior:

```yaml
strategy:
  fail-fast: false   # let all matrix variants finish even if one fails
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [18, 20, 22]
```

Disable `fail-fast` when you need full cross-platform test results before investigating failures.

#### Dynamic Job Generation

Generate matrix values at runtime instead of hardcoding them:

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.detect.outputs.services }}
    steps:
      - uses: actions/checkout@v4
      - id: detect
        run: |
          # Detect changed services dynamically
          SERVICES=$(git diff --name-only HEAD~1 | grep '^services/' | cut -d/ -f2 | sort -u | jq -R . | jq -sc .)
          echo "services=$SERVICES" >> $GITHUB_OUTPUT

  build-services:
    needs: setup
    strategy:
      matrix:
        service: ${{ fromJson(needs.setup.outputs.services) }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building ${{ matrix.service }}"
```

---

### 7.4 🔀 Matrix Strategy (Advanced)

The basic matrix (`node: [18, 20, 22]`) is just the beginning. Expert usage involves dynamic matrices, fine-grained inclusion/exclusion, and conditional matrix runs.

#### Dynamic Matrix from JSON Output

```yaml
jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
      - id: set-matrix
        run: |
          # Read matrix config from a file in the repo
          MATRIX=$(cat .github/test-matrix.json)
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT

  test:
    needs: generate-matrix
    strategy:
      matrix: ${{ fromJson(needs.generate-matrix.outputs.matrix) }}
    runs-on: ${{ matrix.os }}
    steps:
      - run: echo "Testing on ${{ matrix.os }} with Node ${{ matrix.node }}"
```

#### `include` — Add Extra Combinations

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]
    include:
      # Add an extra combination not in the base grid
      - os: macos-latest
        node: 20
        experimental: true
      # Augment an existing combination with extra variables
      - os: ubuntu-latest
        node: 20
        coverage: true
```

#### `exclude` — Remove Specific Combinations

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [16, 18, 20]
    exclude:
      # Node 16 on Windows is flaky — skip it
      - os: windows-latest
        node: 16
```

#### Conditional Steps Within Matrix

```yaml
steps:
  - name: Upload coverage
    if: matrix.coverage == true
    run: npm run coverage:upload

  - name: Run experimental tests
    if: matrix.experimental == true
    continue-on-error: true
    run: npm run test:experimental
```

---

### 7.5 📦 Artifacts vs Caching (Deep Dive)

Understanding when to use `actions/cache` versus `actions/upload-artifact` is critical for building correct, efficient pipelines. They solve different problems.

#### Decision Guide

| Concern | Use Cache | Use Artifact |
|---|---|---|
| **Purpose** | Speed up repeated steps | Share data between jobs |
| **Lifecycle** | Persists across workflow runs | Exists only within one workflow run |
| **Content** | Dependencies, build tools | Compiled output, test results, reports |
| **Scope** | Same repo, same branch | All jobs within the same run |
| **Example** | `node_modules`, Maven cache | `dist/`, coverage reports, Docker layers |

#### Cache Invalidation Strategies

The cache key is the heart of your caching strategy. A bad key means either stale cache hits (wrong data) or constant cache misses (no benefit):

```yaml
# Strategy 1: Invalidate only on lockfile change (most common)
key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}

# Strategy 2: Invalidate daily (for tool caches that drift)
key: ${{ runner.os }}-tools-${{ github.run_id }}-${{ hashFiles('**/Brewfile') }}

# Strategy 3: Invalidate on branch, fall back to main's cache
key: ${{ runner.os }}-npm-${{ github.ref }}-${{ hashFiles('**/package-lock.json') }}
restore-keys: |
  ${{ runner.os }}-npm-refs/heads/main-
  ${{ runner.os }}-npm-
```

**Cache pitfalls to avoid:**
- Never cache secrets, tokens, or any sensitive output — caches can be read by any workflow in the repo.
- Don't cache your `dist/` or build output — use artifacts for that (caches persist across runs, artifacts expire).
- Use `restore-keys` as a fallback chain from most-specific to least-specific.

#### Cross-Job Artifact Design Patterns

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output-${{ github.sha }}
          path: dist/
          retention-days: 3       # short retention for intermediate artifacts

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output-${{ github.sha }}
          path: dist/
      - run: npm test

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output-${{ github.sha }}
          path: dist/
      - run: ./deploy.sh
```

**Expert tip:** Include `github.sha` in artifact names to guarantee uniqueness and traceability — especially important in concurrent workflows.

---

### 7.6 🔐 Permissions & Security Hardening

Secrets management is just one layer of security. Production-grade repositories must also apply least-privilege permissions, scope tokens carefully, and prevent injection attacks.

#### `permissions:` Block — Least Privilege

By default, `GITHUB_TOKEN` has broad read/write access across all scopes. Restrict it explicitly at the workflow or job level:

```yaml
name: CI

# Set minimal defaults for the entire workflow
permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    # No additional permissions needed — inherits read-only

  release:
    runs-on: ubuntu-latest
    # Grant only what this job needs
    permissions:
      contents: write        # to create releases and push tags
      packages: write        # to push to GHCR
      id-token: write        # for OIDC-based cloud authentication
    steps:
      - uses: actions/checkout@v4
      - run: ./release.sh
```

**Recommended scope reference:**

| Permission | Value | Use case |
|---|---|---|
| `contents` | `read` | Checkout code |
| `contents` | `write` | Create releases, push tags |
| `packages` | `write` | Push to GHCR |
| `pull-requests` | `write` | Post PR comments |
| `issues` | `write` | Create or update issues |
| `id-token` | `write` | OIDC authentication with AWS/GCP |
| `security-events` | `write` | Upload SARIF results (CodeQL) |

#### Token Scoping — GITHUB_TOKEN vs PAT

- Use `GITHUB_TOKEN` for everything within the same repository — it's automatically scoped and rotated per run.
- Use a Personal Access Token (PAT) or a GitHub App token only when you need cross-repository access (e.g. calling a private action, pushing to another repo).
- Prefer GitHub App tokens over PATs — they have shorter lifetimes, are auditable, and can be scoped to specific repos.

#### Preventing Workflow Injection Attacks

Untrusted input (PR titles, issue bodies, branch names) can contain characters that break shell commands or inject YAML. Always sanitize:

```yaml
# DANGEROUS — injects PR title directly into shell
- run: echo "PR title is ${{ github.event.pull_request.title }}"

# SAFE — pass through environment variable, never inline in run:
- name: Print PR title
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "PR title is $PR_TITLE"
```

#### Fork PR Security Restrictions

Workflows triggered by `pull_request` from a fork run with read-only tokens and no access to secrets by design. This is intentional — a forked PR could modify the workflow file to exfiltrate secrets.

```yaml
# SAFE for PRs from forks — no secret access, read-only token
on:
  pull_request:

# DANGEROUS — grants secret access to any PR, including forks
on:
  pull_request_target:
    # Only use this event with explicit safety checks
```

When you need to combine `pull_request_target` with code checkout (e.g. to run checks on the PR's code), always check out the PR's SHA explicitly and treat that code as untrusted:

```yaml
on:
  pull_request_target:

jobs:
  safe-check:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      # Checkout the PR code — never the base — for analysis
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      # Run only safe, read-only analysis — never deploy or authenticate
      - run: npm audit
```

---

### 7.7 🧪 Debugging & Observability

When a workflow fails in CI, you need fast, accurate diagnostics. GitHub Actions provides several tools for deep inspection.

#### Enable Debug Logging

Set repository secrets to enable verbose runner and step logging:

| Secret | Value | Effect |
|---|---|---|
| `ACTIONS_STEP_DEBUG` | `true` | Logs every shell command and its exit code |
| `ACTIONS_RUNNER_DEBUG` | `true` | Logs runner infrastructure events (network, process, env) |

These secrets only affect log verbosity — they have no security impact. Enable them temporarily during incident investigation and remove them afterward to reduce log noise.

#### Step-Level Troubleshooting Techniques

```yaml
steps:
  - name: Debug environment
    run: |
      echo "Runner OS: ${{ runner.os }}"
      echo "GitHub ref: ${{ github.ref }}"
      echo "Event: ${{ github.event_name }}"
      env | sort              # print all environment variables
      pwd && ls -la           # verify working directory

  - name: Print context
    run: echo '${{ toJson(github) }}'

  - name: Inspect job matrix
    run: echo '${{ toJson(matrix) }}'
```

#### Re-Run Strategies

GitHub Actions provides granular re-run options — use them strategically:

- **Re-run all jobs** — full fresh run, equivalent to pushing a new commit.
- **Re-run failed jobs only** — restarts only the jobs that failed, using cached artifacts from the successful jobs. Use when a transient failure (network timeout, flaky test) is suspected.
- **Re-run with debug logging** — click "Re-run jobs" → enable "Enable debug logging" checkbox. This activates `ACTIONS_STEP_DEBUG` for that run only without setting a permanent secret.

#### Workflow Visualization

The GitHub Actions UI renders a dependency graph of your jobs. Use it to:
- Spot unexpected serial jobs (should they be parallel?).
- Identify fan-out/fan-in bottlenecks.
- See the critical path — the longest chain of dependent jobs determines your minimum pipeline duration.

---

### 7.8 🕒 Workflow Optimization

Slow pipelines are a tax on every developer on your team. Systematic optimization compounds across hundreds of daily runs.

#### Job Parallelization Strategy

By default, jobs with no `needs:` dependency run in parallel. Design your dependency graph intentionally:

```yaml
jobs:
  # These three run in parallel immediately
  lint:
    runs-on: ubuntu-latest
    steps: [...]

  unit-test:
    runs-on: ubuntu-latest
    steps: [...]

  type-check:
    runs-on: ubuntu-latest
    steps: [...]

  # This waits for all three, then runs in parallel with integration-test
  build:
    needs: [lint, unit-test, type-check]

  integration-test:
    needs: [unit-test]           # doesn't need lint or type-check

  # Deploy waits for both
  deploy:
    needs: [build, integration-test]
```

**Critical path analysis:** The total pipeline duration equals the longest chain, not the sum of all jobs. Draw the dependency DAG and identify the bottleneck chain.

#### Splitting Workflows vs Monolithic Pipelines

| Approach | Use When | Benefit |
|---|---|---|
| Single workflow | All jobs always run together | Simpler, easier to follow |
| Multiple workflows | Jobs have different triggers or audiences | Faster feedback on the critical path |
| Path-filtered workflows | Monorepo with independent services | Only run what changed |

```yaml
# Separate fast-feedback workflow — runs on every commit
name: Quick Checks
on: [push, pull_request]
jobs:
  lint-and-typecheck: [...]     # completes in <60s

# Slower workflow — only runs on main
name: Full Pipeline
on:
  push:
    branches: [main]
jobs:
  integration-test: [...]       # can take 5-10 min
  e2e-test: [...]
```

#### Cold Start Reduction

Slow job startups waste time before any actual work begins:

- **Cache aggressively:** Beyond `node_modules`, cache tool installations like `~/.cargo`, `~/.gradle`, `~/.sonar/cache`.
- **Use larger runners selectively:** A 4-core runner compiles 2–3× faster for CPU-bound builds. Only use them for jobs where compute is the bottleneck.
- **Avoid redundant checkouts:** If a downstream job only needs the artifact (not the source), skip `actions/checkout` — `actions/download-artifact` is faster.
- **Skip setup steps on cache hit:** Use `if: steps.cache.outputs.cache-hit != 'true'` to skip install steps when the cache is fresh.

```yaml
- id: cache
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

- name: Install dependencies
  if: steps.cache.outputs.cache-hit != 'true'
  run: npm ci
```

#### Runner Selection Impact

| Runner | vCPUs | RAM | Cost | Best for |
|---|---|---|---|---|
| `ubuntu-latest` | 2 | 7 GB | 1× | Scripts, linting, most tests |
| `ubuntu-latest` (4-core) | 4 | 16 GB | 2× | Compilation, heavy test suites |
| `ubuntu-latest` (16-core) | 16 | 64 GB | 8× | Docker builds, matrix-heavy jobs |
| Self-hosted | Custom | Custom | Hardware | Private network, persistent cache, GPU |

---

### 7.9 🧾 Workflow Outputs & Data Passing

Complex pipelines need to pass computed values between jobs — not just build artifacts, but dynamic strings like version numbers, deployment URLs, detected service names, and conditional flags.

#### Defining and Consuming Job Outputs

```yaml
jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.compute.outputs.tag }}
      sha: ${{ steps.compute.outputs.sha }}
    steps:
      - uses: actions/checkout@v4

      - id: compute
        run: |
          TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
          SHA=$(git rev-parse --short HEAD)
          echo "tag=$TAG" >> $GITHUB_OUTPUT
          echo "sha=$SHA" >> $GITHUB_OUTPUT

  build:
    needs: version
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Building version ${{ needs.version.outputs.tag }}"
          echo "Commit: ${{ needs.version.outputs.sha }}"
          docker build -t myapp:${{ needs.version.outputs.tag }} .

  deploy:
    needs: [version, build]
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Deploying ${{ needs.version.outputs.tag }} to production"
```

#### Multi-Level Output Chaining

```yaml
jobs:
  detect:
    outputs:
      environment: ${{ steps.env.outputs.value }}
    steps:
      - id: env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "value=production" >> $GITHUB_OUTPUT
          else
            echo "value=staging" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: detect
    environment: ${{ needs.detect.outputs.environment }}
    steps:
      - run: echo "Deploying to ${{ needs.detect.outputs.environment }}"
```

#### Outputs from Reusable Workflows

```yaml
# In the reusable workflow definition:
on:
  workflow_call:
    outputs:
      deployment-url:
        description: "URL of the deployed environment"
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    outputs:
      url: ${{ steps.get-url.outputs.url }}
    steps:
      - id: get-url
        run: echo "url=https://staging-${{ github.sha }}.example.com" >> $GITHUB_OUTPUT

# In the calling workflow:
jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml

  notify:
    needs: deploy
    steps:
      - run: echo "Deployed to ${{ needs.deploy.outputs.deployment-url }}"
```

**Limitations to know:**
- Output values are strings only — serialize complex data with `toJson()` and parse with `fromJson()`.
- Outputs are limited to 1 MB per step — write large data sets to files and pass them as artifacts instead.
- Step outputs only exist within the same job; job outputs are what cross the job boundary.

---

### 7.10 🧠 Event System (Deep Dive)

Most pipelines use `push` and `pull_request`. Expert-level designs use the full event system to build sophisticated, event-driven automation.

#### `workflow_run` — Chain Workflows Together

`workflow_run` triggers a workflow when another named workflow completes. Unlike `needs:`, it works **across separate workflow files** and supports fan-out patterns:

```yaml
name: Post-CI Checks

on:
  workflow_run:
    workflows: ["CI Pipeline"]   # exact name from the triggering workflow's `name:` field
    types: [completed]
    branches: [main]

jobs:
  notify:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "CI passed for ${{ github.event.workflow_run.head_sha }}"

  on-failure:
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    runs-on: ubuntu-latest
    steps:
      - run: ./notify-oncall.sh
```

**Important:** `workflow_run` runs with the permissions of the base branch, not the PR branch — use it for post-merge workflows and monitoring, not as a workaround for fork restrictions.

#### `repository_dispatch` — External API Triggers

`repository_dispatch` lets any authenticated HTTP client trigger a workflow. Ideal for integrating external systems (deployment tools, monitoring alerts, Slack bots) into your GitHub Actions pipeline:

```yaml
name: External Deploy Trigger

on:
  repository_dispatch:
    types: [deploy-staging, deploy-production]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: |
          echo "Triggered by: ${{ github.event.action }}"
          echo "Payload: ${{ toJson(github.event.client_payload) }}"
          echo "Service: ${{ github.event.client_payload.service }}"
          echo "Version: ${{ github.event.client_payload.version }}"
```

**Triggering via API:**

```bash
curl -X POST \
  -H "Authorization: token $GITHUB_PAT" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/my-org/my-repo/dispatches \
  -d '{
    "event_type": "deploy-staging",
    "client_payload": {
      "service": "payment-api",
      "version": "v2.3.1"
    }
  }'
```

#### `pull_request_target` — Security Implications

`pull_request_target` runs in the context of the **base branch**, not the fork's branch. This gives it access to secrets — which is both its power and its risk:

```yaml
# USE CASE: Post PR comments with secrets (e.g. test result summaries)
on:
  pull_request_target:
    types: [opened, synchronize]

jobs:
  comment:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      # CRITICAL: Never checkout the PR's code and run it with these permissions
      # If you must analyze PR code, use a separate unprivileged workflow
      - uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: 'Thanks for your contribution! CI has started.'
            })
```

**`pull_request_target` security checklist:**
- Never check out and execute code from the PR head in this event.
- Never use it as a general replacement for `pull_request` just to get secret access.
- Validate all inputs from `github.event.pull_request.*` before using them in shell commands.
- If you need to run code from the PR *and* post comments, split into two workflows: `pull_request` (runs code, no secrets) and `pull_request_target` (posts comments, has secrets, uses the `workflow_run` of the first as a signal).

#### Chained Workflows — Real-World Pattern

```
push to main
    └─► CI Pipeline (push event)
            └─► workflow_run: completed/success
                    └─► Deploy to Staging (workflow_run event)
                              └─► repository_dispatch: deploy-production
                                      └─► Deploy to Production (repository_dispatch event)
                                                └─► workflow_run: completed
                                                        └─► Post-Deploy Monitoring (workflow_run event)
```

This pattern creates a fully event-driven pipeline where each stage independently monitors the previous one — more resilient than a single monolithic workflow, and easier to debug in isolation.

---

## Key Tools & Actions Reference

| Category | Tool / Action | Purpose |
|---|---|---|
| Checkout | `actions/checkout@v4` | Clone the repository |
| Runtime setup | `actions/setup-node`, `setup-python`, `setup-java` | Install language runtime |
| Caching | `actions/cache@v4` | Cache dependencies between runs |
| Artifacts | `actions/upload-artifact`, `download-artifact` | Pass build output between jobs |
| Docker | `docker/build-push-action`, `docker/login-action` | Build & push container images |
| Security | `aquasecurity/trivy-action`, `github/codeql-action` | Vulnerability & SAST scanning |
| Release | `softprops/action-gh-release`, `release-please-action` | Automate versioning & releases |
| Notifications | `slackapi/slack-github-action` | Send Slack alerts |
| Reusable Workflows | `uses: org/repo/.github/workflows/file.yml@ref` | Call shared workflow templates |
| Composite Actions | `uses: ./.github/actions/my-action` | Use local composite action |
| Script | `actions/github-script@v7` | Run JS with GitHub API access |

---

## DORA Metrics to Track

- **Deployment Frequency** — How often you deploy to production (target: multiple times per day).
- **Lead Time for Changes** — Time from code commit to production (target: less than 1 hour).
- **Change Failure Rate** — Percentage of deployments that cause a production incident (target: below 5%).
- **Mean Time to Recovery (MTTR)** — Time to restore service after an incident (target: less than 1 hour).

---

## License

MIT
