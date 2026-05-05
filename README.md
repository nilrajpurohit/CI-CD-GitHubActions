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
- Tag every release commit with a Git tag (`v1.2.3`) — this is the anchor for tracing what is deployed in any environment.

### Docker Image Build & Push

- Use `docker/build-push-action` with `docker/setup-buildx-action` for fast, multi-platform Docker builds with layer caching.
- Push images to GitHub Container Registry (GHCR) at `ghcr.io/org/repo:tag` — free for public repos and tightly integrated with GitHub permissions.
- Tag images with both the Git SHA (`sha-abc1234`) and the semantic version (`v1.2.3`) so you can trace exactly which code is in a given image.
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

---

## DORA Metrics to Track

- **Deployment Frequency** — How often you deploy to production (target: multiple times per day).
- **Lead Time for Changes** — Time from code commit to production (target: less than 1 hour).
- **Change Failure Rate** — Percentage of deployments that cause a production incident (target: below 5%).
- **Mean Time to Recovery (MTTR)** — Time to restore service after an incident (target: less than 1 hour).

---

## License

MIT
