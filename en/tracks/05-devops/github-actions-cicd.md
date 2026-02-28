# GitHub Actions & CI/CD

## Overview

CI/CD (Continuous Integration / Continuous Delivery) is the practice of automatically building, testing, and deploying code on every change. It eliminates the "deploy day" ceremony, catches regressions immediately, and gives teams the confidence to ship dozens of times per day.

**Continuous Integration (CI):** Every push triggers automated tests. The main branch is always in a deployable state.

**Continuous Delivery (CD):** Every passing build is automatically deployed to staging. Production deployment is one click.

**Continuous Deployment:** Every passing build is automatically deployed to production — no human gate.

GitHub Actions is GitHub's built-in CI/CD platform. Workflows live as YAML files in `.github/workflows/`, run on GitHub's infrastructure, and integrate natively with pull requests, branches, and environments.

---

## Prerequisites

- A GitHub repository
- Basic understanding of Docker (for containerized builds)
- Node.js/TypeScript project with a test suite

---

## Core Concepts

### Anatomy of a Workflow

```yaml
name: CI                          # display name in GitHub UI

on:                               # triggers
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:                             # parallel units of work
  test:                           # job ID
    runs-on: ubuntu-latest        # runner environment
    steps:                        # sequential steps within a job
      - uses: actions/checkout@v4 # check out code
      - run: npm test             # shell command
```

### Key Building Blocks

| Concept | Description |
|---------|-------------|
| **Workflow** | YAML file in `.github/workflows/` |
| **Trigger** (`on`) | What starts the workflow: push, PR, schedule, manual |
| **Job** | A set of steps that run on the same runner |
| **Step** | A single task — either a `uses` action or a `run` command |
| **Action** | Reusable unit of work (`actions/checkout`, `docker/login-action`) |
| **Runner** | The VM or container that executes jobs |
| **Environment** | Named deployment target with protection rules and secrets |
| **Secret** | Encrypted value stored in repository/environment settings |
| **Matrix** | Run the same job with multiple parameter combinations |

### Expressions and Context

```yaml
# Access context objects
${{ github.sha }}          # commit SHA
${{ github.ref_name }}     # branch or tag name
${{ github.actor }}        # user who triggered the workflow
${{ secrets.MY_SECRET }}   # repository secret
${{ vars.MY_VAR }}         # repository variable (non-secret)
${{ env.MY_ENV }}          # environment variable set in workflow

# Conditional expressions
if: github.ref == 'refs/heads/main'
if: github.event_name == 'pull_request'
if: failure()              # run only if previous step failed
if: always()               # run regardless of previous step result
```

---

## Hands-On Examples

### Example 1: Node.js CI Pipeline

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Test (Node ${{ matrix.node-version }})
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [20, 22]

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: test
          POSTGRES_DB: app_test
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
        ports:
          - 5432:5432

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run type-check

      - name: Lint
        run: npm run lint

      - name: Unit tests
        run: npm run test:unit

      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgres://app:test@localhost:5432/app_test

      - name: Build
        run: npm run build
```

### Example 2: Docker Build and Push

```yaml
name: Build and Push

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          target: production
```

### Example 3: Full CI/CD with Deploy

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─── Job 1: Test ────────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: app
          POSTGRES_PASSWORD: test
          POSTGRES_DB: app_test
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 10
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - run: npm run test
        env:
          DATABASE_URL: postgres://app:test@localhost:5432/app_test

  # ─── Job 2: Build Docker Image ──────────────────────────────────
  build:
    name: Build Image
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ─── Job 3: Deploy to VPS ───────────────────────────────────────
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://myapp.com

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/myapp
            echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
            docker compose pull
            docker compose up -d --remove-orphans
            docker system prune -f
```

### Example 4: Scheduled Jobs and Cron

```yaml
name: Maintenance

on:
  schedule:
    - cron: '0 2 * * *'   # daily at 2 AM UTC
  workflow_dispatch:        # manual trigger button in GitHub UI

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Delete old packages
        uses: actions/delete-package-versions@v5
        with:
          package-name: myapp
          package-type: container
          min-versions-to-keep: 10
          delete-only-untagged-versions: true

  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Backup database
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            pg_dump $DATABASE_URL | gzip > /backups/$(date +%Y%m%d).sql.gz
            find /backups -mtime +30 -delete
```

### Example 5: Release Workflow with Semantic Versioning

```yaml
name: Release

on:
  push:
    tags: ['v*.*.*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # full history for changelog

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:latest

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true  # auto-generates from PRs/commits
          files: |
            dist/*.tar.gz
```

---

## Common Patterns & Best Practices

### Cache Dependencies

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 22
    cache: npm           # caches ~/.npm

# Or manually:
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: npm-
```

### Reusable Workflows

Define once, call from multiple workflows:

`.github/workflows/reusable-test.yml`:
```yaml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '22'
    secrets:
      DATABASE_URL:
        required: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - run: npm ci && npm test
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

Caller:
```yaml
jobs:
  test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '22'
    secrets:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

### Concurrency Control

Prevent multiple deploys from running simultaneously:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true   # cancel older run when new one arrives
```

### Path Filters — Run Only When Relevant Files Change

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package*.json'
      - 'Dockerfile'
      - '.github/workflows/**'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

### Environment Protection Rules

In GitHub Settings > Environments, configure:
- Required reviewers before deployment
- Deployment branches (only `main` can deploy to production)
- Wait timer (delay before auto-deploy)

```yaml
deploy:
  environment:
    name: production
    url: https://myapp.com
  # GitHub will require approval from configured reviewers
```

### Matrix Strategy for Cross-Platform Testing

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [20, 22]
  fail-fast: false   # continue other matrix jobs if one fails

runs-on: ${{ matrix.os }}
```

---

## Anti-Patterns to Avoid

### Hardcoding Secrets

```yaml
# BAD
run: docker login -u myuser -p mysecretpassword

# GOOD
run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u ${{ secrets.REGISTRY_USER }} --password-stdin
```

### Not Pinning Action Versions

```yaml
# BAD — can break if action author pushes breaking changes to @main
uses: actions/checkout@main

# BAD — v4 tag could move to a different commit
uses: actions/checkout@v4

# BEST — pin to exact commit SHA for security
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

### Giant Monolithic Jobs

```yaml
# BAD — if deploy fails, can't re-run just the deploy step
jobs:
  all:
    steps:
      - run: npm test
      - run: docker build
      - run: deploy

# GOOD — separate jobs with needs
jobs:
  test: ...
  build:
    needs: test
  deploy:
    needs: build
```

### Uploading Sensitive Data to Artifacts

```yaml
# BAD — .env files might contain secrets
- uses: actions/upload-artifact@v4
  with:
    path: .    # uploads everything including .env
```

### Running Long Jobs Without Timeouts

```yaml
jobs:
  test:
    timeout-minutes: 15   # fail if tests take more than 15 minutes
```

---

## Debugging & Troubleshooting

### Enable Debug Logging

In Repository Settings > Secrets, add:
```
ACTIONS_STEP_DEBUG = true
ACTIONS_RUNNER_DEBUG = true
```

### SSH Debug Session

Add a step to open a temporary SSH tunnel into a failing runner:

```yaml
- name: Debug with tmate
  if: failure()
  uses: mxschmitt/action-tmate@v3
  timeout-minutes: 15
```

### Inspect Workflow Run

```bash
# GitHub CLI
gh run list                         # list recent runs
gh run view 1234567890              # view run details
gh run view 1234567890 --log        # full logs
gh run view 1234567890 --log-failed # only failed step logs
gh run rerun 1234567890             # re-run all jobs
gh run rerun 1234567890 --failed    # re-run only failed jobs
```

### Common Failures

```
Error: Process completed with exit code 1.
```
Check the step's output. The exit code comes from the command you ran.

```
Error: Resource not accessible by integration
```
Missing `permissions` block. Add the required permissions to the job.

```
Error: No space left on device
```
Runner disk is full. Add `docker system prune -f` between heavy steps.

```
Error: Context access might be invalid: secrets.MY_SECRET
```
Secret not defined in repository settings, or wrong environment.

---

## Real-World Scenarios

### Scenario 1: PR Preview Deployments

```yaml
name: Preview Deploy

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy PR preview
        id: deploy
        run: |
          PREVIEW_URL=$(./scripts/deploy-preview.sh ${{ github.event.number }})
          echo "url=$PREVIEW_URL" >> $GITHUB_OUTPUT

      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `Preview deployed: ${{ steps.deploy.outputs.url }}`
            })
```

### Scenario 2: Automated Dependency Updates with Testing

Use `dependabot.yml` + this workflow to auto-merge passing Dependabot PRs:

```yaml
name: Auto-merge Dependabot

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  auto-merge:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: write
    steps:
      - name: Auto-merge minor/patch updates
        uses: actions/github-script@v7
        with:
          script: |
            const { data: pr } = await github.rest.pulls.get({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });
            const isMinor = pr.title.match(/bump .* from .* to .*/i);
            if (isMinor) {
              await github.rest.pulls.merge({
                owner: context.repo.owner,
                repo: context.repo.repo,
                pull_number: context.issue.number,
                merge_method: 'squash'
              });
            }
```

### Scenario 3: Build and Publish NPM Package

```yaml
name: Publish

on:
  push:
    tags: ['v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          registry-url: https://registry.npmjs.org

      - run: npm ci
      - run: npm test
      - run: npm run build

      - name: Publish to NPM
        run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## Further Reading

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow syntax reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [act — run Actions locally](https://github.com/nektos/act)

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Workflow triggers | `push`, `pull_request`, `schedule`, `workflow_dispatch` |
| Jobs | Run in parallel by default; use `needs` for ordering |
| Caching | Always cache `node_modules` or `~/.npm` — saves 1-3 min per run |
| Secrets | Store in GitHub Settings, reference as `${{ secrets.NAME }}` |
| Environments | Named deployment targets with protection rules |
| Matrix | Test against multiple Node versions or OSes in one config |
| Reusable workflows | DRY — define once, call from multiple workflows |
| Concurrency | Prevent simultaneous deploys with `concurrency` group |
| Action pinning | Pin to commit SHA for security, not just version tags |
| Debugging | `ACTIONS_STEP_DEBUG=true` + tmate for live SSH access |

CI/CD is not a luxury — it's the foundation of reliable software delivery. A passing CI badge means the code works. An automated CD pipeline means deployment is a non-event.
