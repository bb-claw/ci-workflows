# Secrets & Variables Reference

Complete reference for every secret and variable used by the pipeline.
Organised by where they are configured and who consumes them.

---

## Mental model

```
GitHub Environment (dev / production)
  ├── Variables   — non-sensitive config (URLs, IDs)   → readable in logs
  └── Secrets     — sensitive credentials              → masked in logs

secrets: inherit  →  all env secrets flow into reusable workflows automatically
```

Secrets set at the **environment** level are only available to jobs that declare
`environment: dev` or `environment: production`. This scoping is intentional —
production credentials are never exposed to dev jobs.

---

## Required — every service repo

### `dev` environment

Configure at: **Service repo → Settings → Environments → dev**

| Type | Name | Example | Description |
|---|---|---|---|
| Variable | `DEV_URL` | `https://my-service-dev.up.railway.app` | Base URL of the Railway dev service. Used by smoke tests and integration tests. No trailing slash. |
| Variable | `RAILWAY_SERVICE_ID_DEV` | `abc12345-def6-7890-...` | Railway service ID for the dev environment. Found in Railway dashboard → service → Settings. |
| Secret | `RAILWAY_TOKEN_DEV` | `xxxxxxxxxxxxxxxx` | Railway project token scoped to the dev environment. Grants `railway redeploy` rights. |

### `production` environment

Configure at: **Service repo → Settings → Environments → production**

| Type | Name | Example | Description |
|---|---|---|---|
| Variable | `PROD_URL` | `https://my-service.up.railway.app` | Base URL of the Railway production service. |
| Variable | `RAILWAY_SERVICE_ID_PROD` | `xyz98765-abc1-2345-...` | Railway service ID for the production environment. |
| Secret | `RAILWAY_TOKEN_PROD` | `yyyyyyyyyyyyyyyy` | Railway project token scoped to the production environment. |

---

## Optional — Slack notifications

Add to either or both environments if you want deployment notifications.

| Type | Name | Description |
|---|---|---|
| Secret | `SLACK_WEBHOOK_URL` | Slack incoming webhook URL. Create at api.slack.com → Your Apps → Incoming Webhooks. |

If `SLACK_WEBHOOK_URL` is not set, the `notify-slack` action skips silently.

---

## Automatic — provided by GitHub

These are injected by GitHub automatically. You do not configure them.

| Name | Available to | Description |
|---|---|---|
| `GITHUB_TOKEN` | All jobs | Auto-generated token for the run. Used by `docker/login-action` to authenticate against ghcr.io. Scoped to the current repo. Requires `packages: write` permission to push images. |
| `github.repository` | All jobs | Org/repo name (e.g. `bb-claw/my-service`). Used to derive the ghcr.io image name. |
| `github.sha` | All jobs | Full commit SHA. Used as the Docker image tag. |
| `github.actor` | All jobs | Username of the user who triggered the run. Used as the ghcr.io login username. |

---

## Project-specific secrets

Service repos often have their own secrets for calling external services during
integration tests (e.g. Stripe, SendGrid, database connection strings).

**You do not need to declare these in `ci-workflows`.**

Because service repo CD workflows use `secrets: inherit`, any secret defined in
the GitHub environment is automatically forwarded to the `integration-test` job.
The integration test runner makes them available as environment variables.

Example: if you add `STRIPE_TEST_KEY` as a secret to the `dev` environment,
it will be available as `process.env.STRIPE_TEST_KEY` in your integration tests.

---

## How secrets flow through the pipeline

```
Service repo cd.yml
  └── calls full-pipeline.yml (secrets: inherit)
        ├── docker-build-push.yml
        │     └── uses GITHUB_TOKEN  (auto)
        │
        ├── unit-test-container.yml
        │     └── uses GITHUB_TOKEN  (auto)
        │
        ├── deploy-railway.yml  [dev]
        │     └── uses RAILWAY_TOKEN_DEV  (from dev environment)
        │
        ├── smoke-test.yml  [dev]
        │     └── no secrets needed
        │
        ├── integration-test.yml
        │     └── uses all inherited secrets  (including project-specific)
        │         API_BASE_URL injected as env var (not a secret)
        │
        ├── deploy-railway.yml  [production]
        │     └── uses RAILWAY_TOKEN_PROD  (from production environment)
        │
        └── smoke-test.yml  [production]
              └── uses RAILWAY_TOKEN_PROD  (for rollback if needed)
```

---

## Security notes

**Railway tokens are environment-scoped.** The dev token can only redeploy the
dev service. Even if compromised, it cannot affect production.

**`GITHUB_TOKEN` permissions are minimal by default.** The `packages: write`
permission in `cd.yml` only grants access to packages (container images) in
the calling repository's org. It cannot write packages in other orgs.

**Secrets are never echoed in logs.** GitHub automatically masks any value that
matches a registered secret. Do not construct secret values from parts of other
secrets (e.g. concatenation) — the masked value will not match the constructed
string.

**Environment secrets are scoped to jobs that declare that environment.** A job
without `environment: production` will never receive `RAILWAY_TOKEN_PROD`, even
if `secrets: inherit` is set. This is a GitHub safety mechanism.
