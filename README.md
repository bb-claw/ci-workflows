# bb-claw / ci-workflows 

Centralized CI/CD framework for all `bb-claw` GitHub repositories.

Every service repo references this repo's reusable workflows instead of maintaining its own pipeline logic. Changes to the pipeline are made **once here** and propagate to all services on their next run.

---

## Quick start — adopting in a new repo

**1. Create `.github/workflows/ci.yml`** in your service repo:

```yaml
name: CI
on:
  push:
    branches-ignore: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: bb-claw/ci-workflows/.github/workflows/node-ci.yml@v1
    secrets: inherit
```

> For Python services replace `node-ci.yml` with `python-ci.yml`.

**2. Create `.github/workflows/cd.yml`** in your service repo:

```yaml
name: CD
on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

jobs:
  pipeline:
    uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1
    with:
      dev-url: ${{ vars.DEV_URL }}
      prod-url: ${{ vars.PROD_URL }}
      railway-service-id-dev: ${{ vars.RAILWAY_SERVICE_ID_DEV }}
      railway-service-id-prod: ${{ vars.RAILWAY_SERVICE_ID_PROD }}
    secrets: inherit
```

**3. Add GitHub environment variables** (Settings → Environments):

| Environment | Variable | Value |
|---|---|---|
| `dev` | `DEV_URL` | `https://your-service-dev.up.railway.app` |
| `dev` | `RAILWAY_SERVICE_ID_DEV` | Railway service ID |
| `production` | `PROD_URL` | `https://your-service.up.railway.app` |
| `production` | `RAILWAY_SERVICE_ID_PROD` | Railway service ID |

**4. Add GitHub secrets** (Settings → Environments):

| Environment | Secret | Value |
|---|---|---|
| `dev` | `RAILWAY_TOKEN_DEV` | Railway project token for dev |
| `production` | `RAILWAY_TOKEN_PROD` | Railway project token for production |

**5. Enable package access** for the shared workflow repo:

In `bb-claw/ci-workflows` → Settings → Actions → General → Access:
set to **"Accessible from repositories in the organization"**.

That's it. See [`docs/onboarding.md`](docs/onboarding.md) for the full walkthrough.

---

## How the pipeline works

### Branch strategy

```
feature/my-change  ──push──►  CI: lint + unit tests  (every commit, ~2 min)
        │
        └── open PR ──────►  CI: lint + unit + build validation
                                     │
                              all checks pass?
                                     │
        ┌────────────────────────────┘
        │   merge PR
        ▼
      main  ──────────────►  CD: full pipeline (automatic)
```

### CD pipeline (triggered on every merge to `main`)

```
build & push image
      │
      ▼
  unit-test (against built container)
      │
      ▼
  deploy → Railway dev
      │
      ▼
  smoke-test (HTTP health check vs dev, 15 retries)
      │
      ▼
  integration-test (full test suite vs dev environment)
      │
      ▼
  deploy → Railway production  ◄── automatic, no manual gate
      │
      ▼
  smoke-test-prod (health check vs prod, rollback on failure)
```

### Key principles

- **Build once.** Docker image is built and pushed to `ghcr.io` exactly once per pipeline run. Every downstream job references the same immutable `sha256:...` digest.
- **Shift left.** Lint and unit tests run on every push to every branch — not just on PRs.
- **No manual gates.** The tests are the gate. If all test stages pass, the pipeline promotes automatically to production.
- **Automatic rollback.** If the post-deploy production smoke test fails, `railway rollback` is called immediately.
- **Code reuse.** All pipeline logic lives in this repo. Service repos contain only a 10-line caller file.

---

## Repository structure

```
ci-workflows/
├── .github/
│   ├── workflows/
│   │   ├── full-pipeline.yml        # Top-level CD orchestrator (call this from service repos)
│   │   ├── node-ci.yml              # CI template for Node.js services
│   │   ├── python-ci.yml            # CI template for Python services
│   │   ├── docker-build-push.yml    # Build & push to ghcr.io, output digest
│   │   ├── deploy-railway.yml       # Deploy pre-built image to Railway
│   │   ├── smoke-test.yml           # HTTP health check with retry loop
│   │   └── integration-test.yml     # Run integration test suite vs live env
│   └── actions/
│       ├── setup-railway/
│       │   └── action.yml           # Composite: install Railway CLI + auth
│       ├── health-check/
│       │   └── action.yml           # Composite: curl retry loop
│       └── notify-slack/
│           └── action.yml           # Composite: Slack deployment notification
├── templates/
│   └── service-repo/
│       └── .github/workflows/
│           ├── ci.yml               # Copy this into new service repos
│           └── cd.yml               # Copy this into new service repos
├── docs/
│   ├── onboarding.md                # Step-by-step adoption guide
│   ├── architecture.md              # Design decisions and rationale
│   └── troubleshooting.md           # Common issues and fixes
├── CHANGELOG.md
└── CODEOWNERS
```

---

## Workflow reference

| File | Trigger | Purpose |
|---|---|---|
| `full-pipeline.yml` | `workflow_call` | Orchestrates the complete CD pipeline |
| `node-ci.yml` | `workflow_call` | Lint + unit test for Node.js |
| `python-ci.yml` | `workflow_call` | Lint + unit test for Python |
| `docker-build-push.yml` | `workflow_call` | Build image, push to ghcr.io, output digest |
| `deploy-railway.yml` | `workflow_call` | Trigger Railway service redeployment |
| `smoke-test.yml` | `workflow_call` | HTTP health check with retries |
| `integration-test.yml` | `workflow_call` | Run caller repo's integration tests vs a URL |

---

## Versioning

Workflows are versioned with semantic tags on this repo. Service repos pin to a major version tag:

```yaml
uses: bb-claw/ci-workflows/.github/workflows/full-pipeline.yml@v1
```

The `@v1` tag is a **floating tag** kept up to date with the latest backwards-compatible release. Pin to an exact tag (`@v1.2.3`) or SHA for maximum stability.

### Release process

1. Merge changes to `main`
2. Create a release tag: `git tag v1.x.x && git push --tags`
3. Move the floating major tag: `git tag -f v1 && git push --force origin v1`
4. Update `CHANGELOG.md`

---

## Contributing

All changes require a PR reviewed by the platform team. `CODEOWNERS` enforces this automatically.

Since changes here affect every service repo, follow this checklist before merging:

- [ ] Tested in at least one service repo against a feature branch
- [ ] Backwards compatible (existing `inputs:` still work)
- [ ] `CHANGELOG.md` updated
- [ ] Version tag plan documented in PR description

---

## License

Internal use only — `bb-claw` organisation.
