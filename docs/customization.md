# Customization Guide

How to extend and customize the shared CI/CD pipelines for your service's specific needs.

---

## Overriding defaults

All reusable workflows accept optional inputs with sensible defaults. Override them in your service repo's `cd.yml`:

```yaml
jobs:
  pipeline:
    uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1
    with:
      # Required
      dev-url: ${{ vars.DEV_URL }}
      prod-url: ${{ vars.PROD_URL }}
      railway-service-id-dev: ${{ vars.RAILWAY_SERVICE_ID_DEV }}
      railway-service-id-prod: ${{ vars.RAILWAY_SERVICE_ID_PROD }}

      # Optional overrides
      dockerfile: "./docker/Dockerfile.production"
      health-check-path: "/api/health"
      integration-test-command: "npm run test:e2e"
      unit-test-command: "npm run test:unit -- --coverage"
      node-version: "20"
    secrets: inherit
```

---

## Common customizations

### Custom Dockerfile location

```yaml
with:
  dockerfile: "./docker/Dockerfile"
```

### Different health endpoint

```yaml
with:
  health-check-path: "/healthz"
  # or
  health-check-path: "/api/v1/health"
```

### Python service instead of Node.js

In `ci.yml`:
```yaml
jobs:
  ci:
    uses: bb-claw/ci-workflows/.github/workflows/python-ci.yml@v1
    secrets: inherit
```

In `cd.yml`:
```yaml
with:
  integration-test-command: "pytest tests/integration"
  unit-test-command: "pytest tests/unit"
```

### Longer smoke test timeout

For services with slow cold starts:

```yaml
with:
  # Default is 15 retries × 10 seconds = 150 seconds
  # This extends to 30 × 15 = 450 seconds (7.5 minutes)
  max-retries: 30
  retry-interval: 15
```

### Different Node.js version

```yaml
with:
  node-version: "20"
```

---

## Adding pre/post steps

You cannot modify the shared workflow's internal steps, but you can add jobs before or after the pipeline.

### Run something before the pipeline

```yaml
jobs:
  pre-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Custom validation
        run: ./scripts/validate-config.sh

  pipeline:
    needs: pre-checks  # Wait for pre-checks to pass
    uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1
    with:
      # ...
    secrets: inherit
```

### Run something after successful deployment

```yaml
jobs:
  pipeline:
    uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1
    with:
      # ...
    secrets: inherit

  post-deploy:
    needs: pipeline  # Only runs if pipeline succeeds
    runs-on: ubuntu-latest
    steps:
      - name: Notify external service
        run: curl -X POST https://api.example.com/deploy-hook

      - name: Update status page
        run: |
          curl -X POST https://status.example.com/api/deployments \
            -d '{"service": "my-service", "version": "${{ github.sha }}"}'
```

### Run something only on failure

```yaml
jobs:
  pipeline:
    uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1
    with:
      # ...
    secrets: inherit

  notify-failure:
    needs: pipeline
    if: failure()  # Only runs if pipeline fails
    runs-on: ubuntu-latest
    steps:
      - name: Send PagerDuty alert
        run: |
          curl -X POST https://events.pagerduty.com/v2/enqueue \
            -H "Content-Type: application/json" \
            -d '{"routing_key": "${{ secrets.PAGERDUTY_KEY }}", ...}'
```

---

## Adding custom CI checks

Extend the CI workflow with additional jobs:

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches-ignore: [main]
  pull_request:
    branches: [main]

jobs:
  # Standard CI from shared workflows
  ci:
    uses: bb-claw/ci-workflows/.github/workflows/node-ci.yml@v1
    secrets: inherit

  # Custom additional checks
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  license-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check licenses
        run: npx license-checker --onlyAllow "MIT;ISC;Apache-2.0;BSD-2-Clause;BSD-3-Clause"
```

---

## Service-specific secrets

Add secrets to your GitHub environment and they'll automatically flow through to integration tests:

1. Go to: Service repo → Settings → Environments → `dev`
2. Add secret: `STRIPE_TEST_KEY`, `DATABASE_URL`, etc.
3. Access in tests via `process.env.STRIPE_TEST_KEY`

No changes needed to the shared workflows — `secrets: inherit` handles it.

---

## Skipping CI on certain commits

Add `[skip ci]` or `[ci skip]` to your commit message:

```bash
git commit -m "docs: update README [skip ci]"
```

This is built into GitHub Actions and skips all workflows.

---

## Running only specific jobs

You can't skip jobs in the shared workflow, but you can:

1. **Skip the entire CD pipeline** by not merging to main
2. **Run CI without CD** by pushing to a feature branch (CI runs, CD doesn't)

---

## Custom Docker build arguments

Pass build-time arguments to the Docker build:

```yaml
# This requires a change to full-pipeline.yml inputs
# Contact @bb-claw/platform to add support
with:
  build-args: |
    NODE_ENV=production
    API_VERSION=v2
```

In your Dockerfile:
```dockerfile
ARG NODE_ENV=development
ARG API_VERSION=v1
ENV NODE_ENV=$NODE_ENV
```

---

## Multi-platform builds

By default, images are built for `linux/amd64` only. For ARM support:

```yaml
# This requires a change to full-pipeline.yml inputs
# Contact @bb-claw/platform to add support
with:
  platforms: "linux/amd64,linux/arm64"
```

Note: Multi-platform builds take longer (~2x build time).

---

## Monorepo support

For monorepos with multiple services, create separate workflow files:

```
my-monorepo/
├── services/
│   ├── api/
│   │   └── Dockerfile
│   └── worker/
│       └── Dockerfile
└── .github/workflows/
    ├── ci-api.yml
    ├── cd-api.yml
    ├── ci-worker.yml
    └── cd-worker.yml
```

Each workflow uses path filters:

```yaml
# .github/workflows/cd-api.yml
name: CD - API
on:
  push:
    branches: [main]
    paths:
      - 'services/api/**'
      - '.github/workflows/*-api.yml'

jobs:
  pipeline:
    uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1
    with:
      dockerfile: "./services/api/Dockerfile"
      # ... separate Railway service IDs for API
```

---

## Requesting new features

If you need customization not covered here:

1. Check if it's already possible via inputs (see CLAUDE.md workflow reference)
2. Open an issue in `bb-claw/ci-workflows` with your use case
3. Tag `@bb-claw/platform` for prioritization

**Important**: Changes to shared workflows affect all services. New inputs must be optional and backwards-compatible.

---

## Pinning to specific versions

For maximum stability, pin to an exact version instead of the floating `@v1`:

```yaml
# Floating tag (recommended) — gets automatic updates
uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1

# Exact version — no automatic updates
uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1.2.3

# Commit SHA — maximum stability
uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@a1b2c3d4e5f6
```

Trade-off: Exact pins don't get bug fixes automatically.
