# Troubleshooting

Common problems and how to fix them. If your issue isn't listed here, check the
GitHub Actions run log first — most failures have a clear error message.

---

## CI / workflow trigger issues

### Workflow doesn't trigger on push to feature branch

**Symptom:** You push to `feature/my-change` and no workflow appears under Actions.

**Causes and fixes:**

1. **The workflow file has a syntax error.** GitHub silently skips invalid YAML.
   Validate your workflow file at [https://rhysd.github.io/actionlint/](https://rhysd.github.io/actionlint/)
   or run `actionlint` locally.

2. **The `branches-ignore` pattern doesn't match.** Check that your branch name
   matches the pattern. `branches-ignore: [main]` ignores only exactly `main`,
   not `main/*`.

3. **The workflow file isn't on the default branch yet.** Workflow trigger rules
   are read from the default branch (`main`). A brand-new workflow file on a
   feature branch will only trigger `pull_request` events, not `push` events,
   until it's merged.
   > First-time setup workaround: merge a minimal version of `ci.yml` to `main`
   > first, then your feature branch pushes will trigger it.

---

### `workflow not found` or `reusable workflow not found`

**Symptom:**
```
Error: bb-claw/ci-workflows/.github/workflows/node-ci.yml@v1 not found
```

**Causes and fixes:**

1. **Access not enabled.** Go to `bb-claw/ci-workflows` → Settings → Actions →
   General → Access. Set to **"Accessible from repositories in the organization"**.

2. **Tag `v1` doesn't exist.** The floating tag must be created after the first
   release. Create it:
   ```bash
   git tag v1
   git push origin v1
   ```

3. **Typo in the `uses:` path.** Copy the path exactly — it's case-sensitive.
   Correct: `bb-claw/ci-workflows/.github/workflows/node-ci.yml@v1`

---

## Docker / ghcr.io issues

### `denied: permission_denied` when pushing to ghcr.io

**Symptom:**
```
ERROR: denied: permission_denied when accessing resource
```

**Cause:** The workflow's `GITHUB_TOKEN` doesn't have `packages: write` permission.

**Fix:** Add to your service repo's `cd.yml`:
```yaml
permissions:
  contents: read
  packages: write
```

---

### Image builds successfully but digest output is empty

**Symptom:** `needs.build.outputs.digest` is empty in downstream jobs.

**Cause:** The `docker/build-push-action` only populates the digest output when
`push: true`. If the build ran with `push: false`, the digest is not available.

**Fix:** Confirm `docker-build-push.yml` is being called for the CD pipeline
(not the CI `build-validate` job which uses `push: false`). The full pipeline's
build job always uses `push: true`.

---

### `unauthorized` when pulling image in unit-test-container job

**Symptom:**
```
Error response from daemon: pull access denied
```

**Cause:** New packages on ghcr.io default to **private** visibility. The
`unit-test-container` job needs `packages: read` permission.

**Fix 1** (preferred): Make the package public:
Go to `ghcr.io` → your org → the package → **Package settings → Change visibility → Public**.

**Fix 2**: Ensure `packages: read` permission is set and the job authenticates
using `docker/login-action` with `GITHUB_TOKEN` before pulling.

---

## Railway deployment issues

### `railway: command not found`

**Symptom:** The deploy step fails immediately with a command not found error.

**Cause:** The `setup-railway` composite action didn't run, or failed silently.

**Fix:** Check that `uses: bb-claw/ci-workflows/.github/actions/setup-railway@v1`
appears as a step before the `railway redeploy` command. Also verify the action
reference tag exists.

---

### `Authentication required` during railway redeploy

**Symptom:**
```
Error: Authentication required. Please run `railway login`
```

**Cause:** `RAILWAY_TOKEN` environment variable is not set or is empty.

**Fix:** Confirm:
1. The secret `RAILWAY_TOKEN_DEV` / `RAILWAY_TOKEN_PROD` exists in the correct
   GitHub **environment** (not just repo-level secrets).
2. The calling job specifies `environment: dev` or `environment: production`
   so GitHub injects the environment-scoped secrets.
3. The deploy workflow receives the secret: `secrets: { RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN_DEV }} }`.

Check the secret name matches exactly — GitHub secrets are case-sensitive.

---

### Railway redeploys but runs the old image

**Symptom:** Deployment succeeds, but the running service is still on the
previous version.

**Cause:** The Railway service is configured to build from source (Git-based
deploy) rather than pull from ghcr.io.

**Fix:**
1. In the Railway dashboard → your service → **Settings → Source**
2. Change source type to **Docker Image**
3. Set image to `ghcr.io/bb-claw/<your-repo-name>:latest`
4. Save and redeploy

---

### `railway rollback` fails after smoke test failure

**Symptom:**
```
Error: No previous deployment found to roll back to
```

**Cause:** This is the very first deployment to this Railway service. There is
no previous deployment to roll back to.

**Fix:** This only happens once (first-ever deploy). After the first successful
deployment, rollback will always have a target. Accept the first deployment
failure and fix the underlying issue manually.

---

## Smoke test failures

### Smoke test times out even though the service is running

**Symptom:** All 15 retry attempts get `000` (no response) or `502`, but the
service is healthy in the Railway dashboard.

**Causes and fixes:**

1. **Wrong URL.** Double-check `DEV_URL` / `PROD_URL` in your GitHub environment
   variables. Common mistake: missing `https://` or wrong subdomain.

2. **Wrong health check path.** The default path is `/health`. If your service
   uses `/healthz`, `/status`, or `/api/health`, override the input:
   ```yaml
   with:
     health-check-path: "/healthz"
   ```

3. **Railway cold-start takes longer than 150 seconds.** The default budget is
   15 retries × 10 seconds = 150 seconds. Increase if needed:
   ```yaml
   with:
     max-retries: 30
     retry-interval: 15
   ```

4. **Health endpoint returns non-200.** If `/health` returns `204` or redirects
   to `301`, the check fails. Override the expected status:
   ```yaml
   with:
     expected-status: 204
   ```

---

### Smoke test passes but integration tests fail immediately

**Symptom:** `/health` returns 200 but API endpoints return 500 during integration tests.

**Cause:** The service is alive but has a startup error (usually a missing
environment variable or failed database connection).

**Fix:** Check Railway logs for your dev service immediately after deployment.
Look for error messages during startup. Most common causes:
- Missing environment variable in Railway dev environment
- Database not reachable from Railway (wrong connection string or network)
- Third-party API key not set in Railway dev environment

---

## Branch protection issues

### Status checks don't appear in branch protection settings

**Symptom:** When configuring required status checks, the job names (`Lint`,
`Unit Tests`, `Build Validation`) don't appear in the autocomplete.

**Cause:** GitHub only knows about status check names after they have been
reported at least once. The checks must run before you can require them.

**Fix:** Run the CI workflow at least once (open a draft PR) before configuring
branch protection. After the first run, the job names will appear in the
dropdown.

---

### PRs can be merged despite CI failures

**Symptom:** The merge button is active even when CI jobs are failing.

**Cause:** The required status checks in branch protection don't match the
actual job names. Job names in branch protection must match the `name:` field
in the workflow YAML, not the workflow file name.

**Fix:** In branch protection settings, check the required status check names:
- `Lint` (not `node-ci / lint`)
- `Unit Tests` (not `node-ci / unit-test`)
- `Build Validation` (not `node-ci / build-validate`)

If the reusable workflow defines job names, those are the names GitHub reports.
Check the exact names in the Actions run UI and copy them precisely.

---

## Integration test issues

### `API_BASE_URL` not available in integration tests

**Symptom:** Tests fail with `undefined` URL or connection refused on localhost.

**Cause:** The test suite isn't reading `process.env.API_BASE_URL`.

**Fix:** Update your integration test configuration to use the env var:
```js
// jest.config.js or test setup
const baseUrl = process.env.API_BASE_URL || 'http://localhost:3000';
```

The `integration-test.yml` workflow always injects `API_BASE_URL` pointing at
the dev Railway environment. Tests should never hardcode `localhost`.

---

### Integration tests pass locally but fail in CI

**Symptom:** Tests work on your machine but fail with auth errors or 401s in CI.

**Cause:** CI doesn't have access to secrets that your local environment has
(e.g., API keys, auth tokens needed by the integration tests).

**Fix:** Add the missing secrets to your GitHub `dev` environment
(Settings → Environments → dev → Secrets). They will be forwarded to the
integration test job via `secrets: inherit`.

---

## Getting further help

1. Check the GitHub Actions run log — expand each step for full output
2. Review `docs/architecture.md` for context on why the pipeline is structured as it is
3. Open an issue in `bb-claw/ci-workflows` with the full Actions run URL
4. Tag `@bb-claw/platform` in the issue for priority support
