# CLAUDE.md

Guidance for Claude Code when working in `bb-claw/ci-workflows`.

---

## What this repo is

Centralized CI/CD framework for all `bb-claw` GitHub repositories.  
Service repos call reusable workflows from here instead of owning pipeline logic.  
**Every change here affects every service repo in the org on their next run.**

---

## Validation

```bash
# Validate all workflow YAML syntax (install: brew install actionlint)
actionlint .github/workflows/*.yml .github/actions/*/action.yml

# Validate a single file
actionlint .github/workflows/full-pipeline.yml

# YAML lint (install: pip install yamllint)
yamllint -d "{extends: relaxed, rules: {line-length: disable, truthy: disable}}" .github/workflows/*.yml
```

**CI runs automatically** on every push to feature branches and PRs to main (see `.github/workflows/ci.yml`). It checks:
- actionlint (workflow syntax)
- shellcheck (bash in composite actions)
- yamllint (YAML validity)
- doc-references (files mentioned in docs exist)

Functional testing (actual workflow execution) requires a real service repo — see **Testing changes** below.

---

## Testing changes

Never test changes directly on `main`. The process:

1. Create a feature branch in this repo: `git checkout -b fix/my-change`
2. Make changes, push the branch
3. In a service repo (e.g. `bb-claw/openclaw-railway-template`), temporarily pin to your branch:
   ```yaml
   uses: bb-claw/ci-workflows/.github/workflows/node-ci.yml@fix/my-change
   ```
4. Push a commit to the service repo to trigger the workflow
5. Verify the run passes end-to-end in GitHub Actions
6. Revert the service repo pin, open a PR in this repo

**Never push directly to `main`.** CODEOWNERS enforces review, and branch protection blocks direct pushes.

---

## Release process

```bash
# Create an exact version tag after merging to main
git tag v1.x.x && git push --tags

# Move the floating major tag (what service repos pin to with @v1)
git tag -f v1 && git push --force origin v1

# Update CHANGELOG.md — move items from [Unreleased] to the new version
```

Floating tags (`@v1`) receive all non-breaking updates automatically.  
Breaking changes require a new major tag (`v2`) — service repos must manually update their `uses:` references.

---

## Architecture

### File layout

```
.github/
  workflows/          ← reusable workflows (workflow_call trigger, own runners)
    full-pipeline.yml
    node-ci.yml
    python-ci.yml
    docker-build-push.yml
    unit-test-container.yml
    deploy-railway.yml
    smoke-test.yml
    integration-test.yml
    ci.yml              ← CI for THIS repo (not reusable, runs on push/PR)
  actions/            ← composite actions (step-level, run on caller's runner)
    setup-railway/action.yml
    health-check/action.yml
    notify-slack/action.yml
  pull_request_template.md
templates/
  service-repo/       ← copy these into new service repos
    .github/workflows/ci.yml
    .github/workflows/cd.yml
    Dockerfile.node
    Dockerfile.python
    Dockerfile.generic
    .dockerignore
docs/
  onboarding.md
  architecture.md
  troubleshooting.md
  secrets-reference.md
  runbook.md              ← incident response and recovery procedures
  customization.md        ← how to extend and customize pipelines
  pipeline-overview.html
  cicd-overview.html
```

### CD pipeline job chain

```
build (docker-build-push)
  └─► unit-test (unit-test-container)
        └─► deploy-dev (deploy-railway, env: dev)
              └─► smoke-test-dev (smoke-test)
                    └─► integration-test
                          └─► deploy-prod (deploy-railway, env: production)
                                └─► smoke-test-prod (smoke-test, rollback on failure)
```

All jobs after `build` reference the same immutable `sha256:...` digest.  
If `smoke-test-prod` fails, `railway rollback` is called automatically.

### CI pipeline (feature branches + PRs)

```
On every push to feature/*:    lint  +  unit-test      (~2 min, no Docker)
On every PR to main:           lint  +  unit-test  +  build-validate (push: false)
```

---

## Workflow inputs/outputs reference

Use this when modifying or calling workflows to get inputs/outputs exactly right.

### `full-pipeline.yml`

| Input | Type | Default | Required | Description |
|---|---|---|---|---|
| `dev-url` | string | — | ✅ | Railway dev base URL (no trailing slash) |
| `prod-url` | string | — | ✅ | Railway prod base URL |
| `railway-service-id-dev` | string | — | ✅ | Railway service ID for dev |
| `railway-service-id-prod` | string | — | ✅ | Railway service ID for production |
| `dockerfile` | string | `./Dockerfile` | | Path to Dockerfile |
| `health-check-path` | string | `/health` | | Path for smoke tests |
| `integration-test-command` | string | (auto) | | Integration test command (defaults to `{pm} run test:integration`) |
| `unit-test-command` | string | `npm test` | | Unit test command run inside container |
| `node-version` | string | `22` | | Node.js version for integration test runner |
| `package-manager` | string | `npm` | | Package manager: `npm`, `pnpm`, or `yarn` |

Required secrets (forwarded from calling repo via `secrets: inherit`):  
`RAILWAY_TOKEN_DEV`, `RAILWAY_TOKEN_PROD`

### `node-ci.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `node-version` | string | `22` | Node.js version |
| `package-manager` | string | `npm` | Package manager: `npm`, `pnpm`, or `yarn` |
| `lint-command` | string | (auto) | Lint command (defaults to `{pm} run lint`) |
| `test-command` | string | (auto) | Unit test command (defaults to `{pm} test`) |
| `build-enabled` | boolean | `true` | Set `false` on feature pushes to skip Docker build |
| `dockerfile` | string | `./Dockerfile` | Path to Dockerfile |

### `python-ci.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `python-version` | string | `3.12` | Python version |
| `lint-command` | string | `ruff check . && ruff format --check .` | Lint command |
| `test-command` | string | `pytest` | Unit test command |
| `build-enabled` | boolean | `true` | Set `false` on feature pushes |
| `dockerfile` | string | `./Dockerfile` | Path to Dockerfile |
| `requirements-file` | string | `requirements.txt` | For pip cache key |

### `docker-build-push.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `dockerfile` | string | `./Dockerfile` | |
| `context` | string | `.` | Docker build context |
| `build-args` | string | `""` | Newline-separated `KEY=VALUE` build args |
| `platforms` | string | `linux/amd64` | Target platforms |

**Outputs:**

| Output | Description |
|---|---|
| `image` | Full image name: `ghcr.io/bb-claw/<repo>` (lowercase, no tag) |
| `digest` | Immutable digest: `sha256:abc123...` |
| `tags` | All applied tags (newline-separated) |

### `unit-test-container.yml`

| Input | Type | Required | Description |
|---|---|---|---|
| `image` | string | ✅ | Full image name from `docker-build-push` outputs |
| `digest` | string | ✅ | Digest from `docker-build-push` outputs |
| `test-command` | string | | Default: `npm test` |

### `deploy-railway.yml`

| Input | Type | Required | Description |
|---|---|---|---|
| `service-id` | string | ✅ | Railway service ID |
| `environment` | string | ✅ | GitHub environment name: `dev` or `production` |

Required secret: `RAILWAY_TOKEN` (not inherited — passed explicitly as named secret)

### `smoke-test.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `url` | string | — | ✅ Base URL (no trailing slash) |
| `health-check-path` | string | `/health` | |
| `max-retries` | number | `15` | |
| `retry-interval` | number | `10` | Seconds between retries |
| `expected-status` | number | `200` | Expected HTTP status code |
| `rollback-on-failure` | boolean | `false` | Set `true` for prod smoke test |
| `service-id` | string | `""` | Required when `rollback-on-failure: true` |

Optional secret: `RAILWAY_TOKEN` (needed only when `rollback-on-failure: true`)

### `integration-test.yml`

| Input | Type | Default | Description |
|---|---|---|---|
| `api-base-url` | string | — | ✅ Injected as `API_BASE_URL` env var |
| `test-command` | string | (auto) | Defaults to `{pm} run test:integration` |
| `node-version` | string | `22` | |
| `package-manager` | string | `npm` | Package manager: `npm`, `pnpm`, or `yarn` |
| `working-directory` | string | `.` | |
| `timeout-minutes` | number | `15` | |

---

## Naming and conventions

### Docker image naming
Images are named `ghcr.io/${GITHUB_REPOSITORY,,}` — the full org/repo path, lowercased.  
Example: `ghcr.io/bb-claw/openclaw-railway-template`

Tags applied per run:
- `:sha-<7-char-commit>` — e.g. `:sha-a1b2c3d` — immutable reference
- `:latest` — mutable, always points to the most recent main build

All jobs after `build` use the digest (`sha256:...`), never the `:latest` tag.

### Secrets naming
- Repo-level secrets: uppercase with underscores, e.g. `RAILWAY_TOKEN_DEV`
- Environment variables injected by the pipeline: uppercase, e.g. `API_BASE_URL`
- GitHub-provided: `GITHUB_TOKEN`, `github.sha`, `github.actor`, `github.repository`

### Job names
Job `name:` fields are what appear in GitHub's PR status checks and branch protection rules.  
Keep them human-readable: `Lint`, `Unit Tests`, `Build Validation` — not `lint-job` or `run-lint`.  
Do not rename existing jobs without updating branch protection rules in all service repos.

### Workflow file names
- Kebab-case: `docker-build-push.yml`, not `DockerBuildPush.yml`
- Action folder names match the action purpose: `setup-railway/`, `health-check/`
- Template files use language suffixes: `Dockerfile.node`, `Dockerfile.python`

---

## Key design rules — do not break these

**Build once.** `docker-build-push.yml` is called once per CD run. Never add a second `build` job or rebuild the image in a test/deploy job. Pass the digest output forward.

**No manual gates.** Do not add `environment:` with required reviewers to `deploy-prod`. The test stages are the gate. If a reviewer wants oversight, improve the tests.

**Shift-left.** `lint` and `unit-test` must trigger on every push to every branch. Do not gate them behind `pull_request` only.

**Secrets never in workflow files.** Secrets are injected via `secrets: inherit` from calling repos or via explicit `secrets:` mapping in reusable workflow calls. Never hardcode tokens, URLs with credentials, or API keys.

**`secrets: inherit` vs explicit passing.** Use `secrets: inherit` in caller files (service repos). Use explicit named secrets in `full-pipeline.yml` → `deploy-railway.yml` calls because `RAILWAY_TOKEN_DEV` and `RAILWAY_TOKEN_PROD` are different secrets that must be mapped to the generic `RAILWAY_TOKEN` input. Do not change this pattern.

**Backwards compatibility.** Never remove or rename an existing `input:`. Add new optional inputs with sensible defaults instead. Renaming an input is a breaking change requiring a major version bump.

**`push: false` on CI builds.** The `build-validate` job in `node-ci.yml` and `python-ci.yml` must never push to the registry. Only `docker-build-push.yml` pushes, and only when called from the CD pipeline.

---

## Common mistakes to avoid

- **Do not use `workflow_run` trigger** — it has complex ref-resolution behaviour that breaks in forks
- **Do not add `continue-on-error: true`** to any job in `full-pipeline.yml` — every job is a gate; silent failures defeat the purpose
- **Do not cache Docker layers between different service repos** — the GHA cache is scoped per repo already, but do not try to share it cross-repo
- **Do not use `latest` tag when referencing the built image in downstream jobs** — always use the digest. Tags are mutable; digests are not
- **Do not change the `environment:` field on `deploy-prod`** — it controls which GitHub secrets are injected. If you remove `environment: production`, the `RAILWAY_TOKEN_PROD` secret will not be available
- **Do not add steps after `railway redeploy`** in `deploy-railway.yml` that assume the service is live — Railway redeployment is asynchronous. The smoke test handles readiness checking

---

## Adding a new workflow

1. Create `.github/workflows/my-new-workflow.yml` with `on: workflow_call:`
2. Add an entry to the workflow reference table in `README.md`
3. Document all inputs in the **Workflow inputs/outputs reference** section of this file
4. Add a `CHANGELOG.md` entry under `[Unreleased]`
5. If the new workflow is called from `full-pipeline.yml`, update the pipeline flow diagram in both this file and `README.md`
6. Test via a feature branch pin in a service repo before merging

## Modifying an existing workflow

1. Check if the change is backwards compatible (no input renames/removals, no behaviour changes that break existing callers)
2. If breaking: plan major version bump, document migration in `docs/`, communicate to team
3. Update the inputs table in this file if inputs changed
4. Update `CHANGELOG.md`
5. Test in a service repo on a feature branch pin

---

## Repository governance

| Who | What they can approve |
|---|---|
| `@bb-claw/platform` | Everything |
| `@bb-claw/devops` | `.github/workflows/*` (co-review with platform) |
| Any org member | `docs/*` only (still needs platform approval) |

All PRs require the checklist in `.github/pull_request_template.md` to be completed.
