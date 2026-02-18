# Service repo pipeline templates

Copy the files from this directory into your service repo to adopt
the bb-claw shared CI/CD framework.

```
your-service-repo/
├── Dockerfile          ← copy the matching language template and rename
├── .dockerignore       ← copy as-is
└── .github/
    └── workflows/
        ├── ci.yml      ← copy from here
        └── cd.yml      ← copy from here
```

Then follow the full setup guide: [`docs/onboarding.md`](../../docs/onboarding.md).

---

## What each file does

### `ci.yml`

Triggers on every push to feature branches and every PR to `main`.

| Event | Jobs run |
|---|---|
| Push to `feature/*` | Lint, Unit Tests |
| PR opened/updated → `main` | Lint, Unit Tests, Build Validation |

Build validation runs `docker build` with `push: false` — confirms the
Dockerfile compiles without pushing anything to the registry.

### `cd.yml`

Triggers on every push to `main` (i.e. every merged PR).

Runs the full pipeline:

1. Build & push Docker image to `ghcr.io` → capture digest
2. Unit tests inside the built container (same digest)
3. Deploy to Railway dev
4. Smoke test dev (HTTP health check, 15 retries)
5. Integration tests against live dev environment
6. Deploy to Railway production (automatic — no manual gate)
7. Smoke test production (rollback on failure)

---

## Customising for your service

Both files accept inputs to override defaults. Common overrides:

**Non-standard health endpoint:**
```yaml
with:
  health-check-path: "/api/health"
```

**Python service:**
```yaml
# In ci.yml — change the uses: line to:
uses: bb-claw/ci-workflows/.github/workflows/python-ci.yml@v1
```

**Custom test commands:**
```yaml
with:
  unit-test-command: "npm run test:unit"
  integration-test-command: "npm run test:e2e"
```

**Non-standard Dockerfile location:**
```yaml
with:
  dockerfile: "./docker/Dockerfile.prod"
```

See the full input reference in
[`full-pipeline.yml`](../../.github/workflows/full-pipeline.yml)
and [`node-ci.yml`](../../.github/workflows/node-ci.yml).

---

## Dockerfile templates

Three templates are provided. Pick the one that matches your stack, copy it to your repo root as `Dockerfile`, and adjust the `ARG` values at the top.

| File | Use for |
|---|---|
| `Dockerfile.node` | Node.js / TypeScript services with a build step |
| `Dockerfile.python` | Python services (FastAPI/Flask/Django + uvicorn/gunicorn) |
| `Dockerfile.generic` | Go, Rust, or anything else — swap Stage 1 for your toolchain |

All three templates follow the same structure:

**Multi-stage build** — separate build and runtime stages so dev tools, compilers, and test files never end up in the final image. Smaller images deploy faster and have a smaller attack surface.

**Non-root user** — the app process runs as a dedicated `appuser` account, not root.

**`HEALTHCHECK` instruction** — the Docker daemon monitors the container and marks it unhealthy if `/health` stops responding. This is separate from the pipeline's smoke test (which checks Railway's public URL) and provides an additional layer of defence.

**PORT from environment** — Railway injects `PORT` at runtime. The templates read it via `ENV PORT=${PORT}`. Never hardcode the port.

### Copy `.dockerignore` too

The `.dockerignore` file prevents `node_modules`, `.git`, `.env.*`, test files, and other build artefacts from being sent to the Docker daemon. This is important for:
- **Speed** — smaller build context = faster uploads to GHA runners
- **Cache efficiency** — unrelated file changes don't bust layer cache
- **Security** — `.env` files with local secrets can never accidentally end up in an image
