# Onboarding — adopting ci-workflows in a new service repo

This guide walks you through wiring a brand-new (or existing) service repo into the shared CI/CD framework. End-to-end setup takes about 15 minutes.

---

## Prerequisites

Before you start, confirm:

- Your service repo lives under the `bb-claw` GitHub organisation
- Your service has a `Dockerfile` at the repo root (or note the path)
- Your service exposes a health endpoint (default: `GET /health → 200 OK`)
- You have access to the Railway dashboard for your project
- You have maintainer access to the service repo (to configure secrets and environments)

---

## Step 1 — Configure Railway

Your Railway project needs two environments: **dev** and **prod**.

### 1a. Create environments

In the Railway dashboard for your project:

1. Open **Settings → Environments**
2. Create environment `dev` (if it doesn't exist)
3. Create environment `production` (if it doesn't exist)

### 1b. Set the image source

Both Railway services must be configured to pull from ghcr.io (not to build from source). For each service (dev and prod):

1. Open the service → **Settings → Source**
2. Set source type to **Docker Image**
3. Set the image to `ghcr.io/bb-claw/<your-repo-name>:latest`
4. Save

Railway will pull the image when `railway redeploy` is called from the pipeline.

### 1c. Collect service IDs

You need the service ID for each environment:

1. Open the service → **Settings**
2. Copy the **Service ID** (looks like `abc12345-...`)

Note these down — you'll add them as GitHub variables in step 3.

### 1d. Generate Railway tokens

Each environment needs its own project token scoped to that environment:

1. Railway dashboard → **Settings → Tokens**
2. Create token: name it `github-actions-dev`, scope to **dev** environment
3. Create token: name it `github-actions-prod`, scope to **production** environment
4. Copy both tokens — you'll add them as GitHub secrets in step 3

---

## Step 2 — Add workflow files to your service repo

Copy the template files from this repo into your service repo:

```bash
# From the ci-workflows repo root
cp templates/service-repo/.github/workflows/ci.yml  ../your-service/.github/workflows/ci.yml
cp templates/service-repo/.github/workflows/cd.yml  ../your-service/.github/workflows/cd.yml
```

Or create them manually — the contents are in [`templates/service-repo/`](../templates/service-repo/).

**For Python services**, edit `ci.yml` and change:
```yaml
uses: bb-claw/ci-workflows/.github/workflows/node-ci.yml@v1
```
to:
```yaml
uses: bb-claw/ci-workflows/.github/workflows/python-ci.yml@v1
```

Commit and push these files to a feature branch (do not push directly to `main`).

---

## Step 3 — Configure GitHub environments, variables, and secrets

In your service repo: **Settings → Environments**

### dev environment

Click **New environment** → name it `dev`.

Add **Variables** (not secrets — these are non-sensitive):

| Name | Value |
|---|---|
| `DEV_URL` | `https://your-service-dev.up.railway.app` |
| `RAILWAY_SERVICE_ID_DEV` | Service ID from Railway (dev) |

Add **Secrets**:

| Name | Value |
|---|---|
| `RAILWAY_TOKEN_DEV` | Railway project token for dev environment |

### production environment

Click **New environment** → name it `production`.

Add **Variables**:

| Name | Value |
|---|---|
| `PROD_URL` | `https://your-service.up.railway.app` |
| `RAILWAY_SERVICE_ID_PROD` | Service ID from Railway (prod) |

Add **Secrets**:

| Name | Value |
|---|---|
| `RAILWAY_TOKEN_PROD` | Railway project token for production environment |

### Optional — Slack notifications

If your repo has a Slack webhook, add it to both environments:

| Name | Value |
|---|---|
| `SLACK_WEBHOOK_URL` | Incoming webhook URL from Slack |

---

## Step 4 — Enable the shared workflow repo access

The `bb-claw/ci-workflows` repo must be accessible to your service repo. This is a one-time org-level setting.

If you get a `workflow not found` error, check:

1. Go to `bb-claw/ci-workflows` → **Settings → Actions → General**
2. Under **Access**, select **Accessible from repositories in the organization**
3. Save

---

## Step 5 — Set up branch protection on `main`

In your service repo: **Settings → Branches → Add rule** (or edit existing).

For the `main` branch, enable:

- [x] Require a pull request before merging
- [x] Require approvals (minimum 1)
- [x] Require status checks to pass before merging
  - Add: `Lint`, `Unit Tests`, `Build Validation`
- [x] Require branches to be up to date before merging
- [x] Do not allow bypassing the above settings

This ensures no code reaches `main` without CI passing.

---

## Step 6 — Open a test PR

With everything configured:

1. Push your feature branch (with the ci.yml and cd.yml files)
2. Open a PR to `main`
3. Watch the **CI** workflow run — you should see Lint, Unit Tests, and Build Validation jobs
4. All three should pass (if your service has tests and a valid Dockerfile)
5. Merge the PR
6. Watch the **CD** workflow run — you should see the full pipeline execute

If anything fails, check [`docs/troubleshooting.md`](troubleshooting.md).

---

## Step 7 — Add integration tests (if not yet present)

The pipeline expects a command `npm run test:integration` (or whatever you configure via `integration-test-command`). If your service doesn't have integration tests yet:

1. Add a `test:integration` script to `package.json` that at minimum hits your `/health` endpoint:

```json
{
  "scripts": {
    "test:integration": "node -e \"require('http').get(process.env.API_BASE_URL + '/health', r => { process.exit(r.statusCode === 200 ? 0 : 1) })\""
  }
}
```

2. Gradually expand with real integration tests over time. The pipeline just needs the command to exit 0 on success.

---

## Checklist

- [ ] Railway dev and prod services configured with Docker image source
- [ ] Railway service IDs collected
- [ ] Railway project tokens created (one per environment)
- [ ] `ci.yml` and `cd.yml` added to service repo
- [ ] GitHub `dev` environment configured (variables + secret)
- [ ] GitHub `production` environment configured (variables + secret)
- [ ] `bb-claw/ci-workflows` access enabled for the org
- [ ] Branch protection rules set on `main`
- [ ] Test PR opened and CI passes
- [ ] Merge to main and CD pipeline runs successfully
