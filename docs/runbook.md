# Runbook — Incident Response for CI/CD Failures

Quick reference for when pipelines fail or production deployments go wrong.

---

## Production deployment failed — service is down

**Severity: Critical** | **Response time: Immediate**

### 1. Check if automatic rollback triggered

The pipeline automatically rolls back if the production smoke test fails. Check the GitHub Actions run:

1. Go to the service repo → **Actions** → find the failed run
2. Look for the `smoke-test-prod` job
3. If it shows "Rollback triggered", the previous version should already be running

### 2. Verify rollback succeeded

```bash
# Check Railway deployment status
railway status --service <service-id>

# Or check the Railway dashboard
# https://railway.app/project/<project-id>
```

### 3. If automatic rollback did NOT trigger

Manual rollback via Railway CLI:

```bash
# Set the token for the production environment
export RAILWAY_TOKEN=<production-token>

# Rollback to previous deployment
railway rollback --service <service-id> --yes
```

Or via Railway dashboard:
1. Open the service → **Deployments**
2. Find the last successful deployment
3. Click **⋮** → **Rollback to this deployment**

### 4. If rollback fails or no previous deployment exists

This is the first deployment or Railway history is corrupted:

1. **Revert the merge commit** in the service repo:
   ```bash
   git revert <merge-commit-sha>
   git push origin main
   ```
2. This triggers a new pipeline with the previous code
3. Wait for the full pipeline to complete

### 5. Post-incident

1. Check Railway logs for the failed deployment: Dashboard → Service → **Logs**
2. Document what went wrong in the PR or a GitHub issue
3. Fix the root cause on a feature branch before re-merging

---

## Pipeline stuck or hanging

**Severity: Medium** | **Response time: 15 minutes**

### Symptoms
- Job running for longer than expected (>10 min for most jobs, >20 min for integration tests)
- No log output for several minutes

### Resolution

1. **Cancel the run**: Actions → Select run → **Cancel workflow**

2. **Re-run failed jobs**: Actions → Select run → **Re-run failed jobs**

3. If it hangs again, check for:
   - Railway service not starting (check Railway logs)
   - External service timeout (database, third-party API)
   - Resource exhaustion (GitHub Actions runner out of disk/memory)

4. **GitHub Actions outage**: Check [githubstatus.com](https://githubstatus.com)

---

## Smoke test failing but service works manually

**Severity: Low** | **Response time: 30 minutes**

### Symptoms
- Smoke test times out or gets wrong status code
- But `curl https://your-service.up.railway.app/health` works from your machine

### Common causes

1. **Wrong URL in GitHub environment variables**
   ```
   Check: Settings → Environments → dev/production → Variables
   Verify: DEV_URL / PROD_URL are correct (no trailing slash, correct protocol)
   ```

2. **Health endpoint path mismatch**
   ```yaml
   # In cd.yml, check health-check-path matches your actual endpoint
   with:
     health-check-path: "/health"  # or "/healthz", "/api/health", etc.
   ```

3. **Railway cold start taking too long**
   ```yaml
   # Increase retries in cd.yml
   with:
     max-retries: 30        # default is 15
     retry-interval: 15     # default is 10 seconds
   ```

4. **Health endpoint returns non-200**
   ```yaml
   # If your health endpoint returns 204 or redirects
   with:
     expected-status: 204
   ```

---

## Docker build failing

**Severity: Medium** | **Response time: 30 minutes**

### "denied: permission_denied" pushing to ghcr.io

```yaml
# Add to service repo cd.yml
permissions:
  contents: read
  packages: write
```

### Build succeeds locally but fails in CI

1. **Check Dockerfile uses explicit versions**, not `latest`:
   ```dockerfile
   # Bad
   FROM node:latest

   # Good
   FROM node:22-alpine
   ```

2. **Check for missing build context files** — `.dockerignore` might exclude needed files

3. **Check for ARM vs x86 issues** — CI runs on `linux/amd64`, local Mac M1/M2 is ARM

### Out of disk space

GitHub Actions runners have ~14GB. Large builds can exhaust this:

1. Add cleanup step before build:
   ```yaml
   - name: Free disk space
     run: |
       sudo rm -rf /usr/share/dotnet
       sudo rm -rf /opt/ghc
       docker system prune -af
   ```

2. Or use multi-stage builds to reduce image size

---

## Integration tests failing

**Severity: Medium** | **Response time: 30 minutes**

### Tests pass locally but fail in CI

1. **Missing secrets**: Add required secrets to GitHub `dev` environment
   ```
   Settings → Environments → dev → Secrets
   ```

2. **Hardcoded localhost**: Tests must use `API_BASE_URL` env var
   ```javascript
   const baseUrl = process.env.API_BASE_URL || 'http://localhost:3000';
   ```

3. **Timing issues**: Dev environment might be slower than local
   - Add retries to flaky tests
   - Increase timeouts

### Tests timing out

```yaml
# Increase timeout in cd.yml
with:
  timeout-minutes: 20  # default is 15
```

---

## Railway authentication errors

**Severity: Medium** | **Response time: 15 minutes**

### "Authentication required" error

1. **Token expired or revoked**: Generate new token in Railway dashboard
   ```
   Railway → Project → Settings → Tokens → Create new token
   ```

2. **Token in wrong GitHub environment**:
   - `RAILWAY_TOKEN_DEV` must be in the `dev` environment
   - `RAILWAY_TOKEN_PROD` must be in the `production` environment

3. **Token scope wrong**: Token must be scoped to the correct Railway environment

### Railway CLI not found

The `setup-railway` action failed silently. Check:
1. Network access to npm registry
2. The action reference tag exists: `@v1`

---

## Complete pipeline re-run

If you need to re-run the entire pipeline from scratch:

1. **From GitHub UI**: Actions → Select run → **Re-run all jobs**

2. **Force new run with empty commit**:
   ```bash
   git commit --allow-empty -m "ci: trigger pipeline re-run"
   git push origin main
   ```

3. **Re-run with fresh Docker cache** (if caching issues suspected):
   - Go to Actions → Select run → Re-run all jobs
   - Or push a whitespace change to Dockerfile to bust cache

---

## Escalation contacts

| Issue | Contact | Method |
|-------|---------|--------|
| Pipeline failures | @bb-claw/platform | GitHub issue in ci-workflows |
| Railway issues | @bb-claw/devops | Slack #devops |
| GitHub Actions outage | — | Wait, check githubstatus.com |
| Security incident | @bb-claw/security | Slack #security (urgent) |

---

## Useful commands cheat sheet

```bash
# Check Railway service status
railway status --service <service-id>

# View Railway logs
railway logs --service <service-id>

# Manual rollback
railway rollback --service <service-id> --yes

# Trigger redeploy without code change
railway redeploy --service <service-id> --yes

# Check ghcr.io image exists
docker manifest inspect ghcr.io/bb-claw/<repo>:sha-<commit>

# Validate workflow syntax locally
actionlint .github/workflows/*.yml
```
