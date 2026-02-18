# Architecture — design decisions and rationale

This document explains the key decisions behind the `bb-claw/ci-workflows` framework and the reasoning behind each.

---

## Where shared workflows live

**Decision: dedicated `ci-workflows` repo, not the `.github` repo.**

GitHub's `.github` org repo serves a different purpose: org profile README, default community health files (CONTRIBUTING.md, CODE_OF_CONDUCT.md), and starter workflow templates. Putting CI logic there mixes concerns and creates the awkward double-path `uses: bb-claw/.github/.github/workflows/...`.

A dedicated `ci-workflows` repo is purpose-built for pipelines, has a clean reference path (`uses: bb-claw/ci-workflows/.github/workflows/X.yml@v1`), and its version history is entirely about pipeline changes — not org admin housekeeping.

---

## Reusable workflows vs composite actions

**Decision: reusable workflows for pipeline stages, composite actions for shared step sequences.**

| | Reusable workflows | Composite actions |
|---|---|---|
| Runs on | Its own runner | Caller's runner |
| Granularity | Job-level | Step-level |
| Secrets | Native `secrets:` support | Via env vars |
| Filesystem | Isolated | Shared with caller |
| Best for | Multi-job pipelines | Setup steps, notifications |

`docker-build-push.yml`, `deploy-railway.yml`, `smoke-test.yml`, and `integration-test.yml` are reusable workflows because they represent complete pipeline stages with their own runners and lifecycles.

`setup-railway`, `health-check`, and `notify-slack` are composite actions because they are small, step-level utilities that callers embed within their own jobs.

---

## Build once, deploy everywhere

**Decision: build and push the Docker image exactly once per pipeline run, then reference by digest.**

The key problem with rebuilding: two builds of the same commit can produce different images if a base image layer or dependency has changed between runs. The image you tested is not guaranteed to be the image you deploy.

The solution is to build once, capture the immutable `sha256:...` digest, and pass that digest to every downstream job as a job output. Every test, every deployment, every health check operates on the same bytes.

```
build job → sha256:a1b2c3d4... → unit-test
                               → deploy-dev (Railway pulls this digest)
                               → deploy-prod (Railway pulls this digest)
```

The `sha256:...` digest is content-addressable. Unlike tags (which are mutable pointers), a digest uniquely identifies a specific image manifest. It cannot be changed after the image is pushed.

---

## Shift-left testing strategy

**Decision: run lint and unit tests on every push to every branch, not just on PRs.**

The cost of a CI failure scales with how far the broken code has travelled:

- Caught in your editor: seconds to fix
- Caught on your branch (by CI): minutes to fix
- Caught on a PR: hours to fix (review cycle, back-and-forth)
- Caught in production: potentially days to fix

Lint and unit tests take ~2 minutes and cost nearly nothing. Running them on every push surfaces failures while the developer still has full context of what they changed. The GHA cache means repeat runs on the same branch are extremely fast.

The Docker build is deliberately excluded from the per-push run. It takes 3–5 minutes and rarely fails independently of code changes. It runs at PR time — early enough to gate merges, not so frequent that it slows the development loop.

---

## No manual approval gate

**Decision: production deployments are fully automatic when all test stages pass.**

Manual approval gates create a false sense of security. A human reviewer pressing "Approve" in the GitHub UI provides no technical validation — they are approving based on the same information the pipeline already has (test results, commit message, diff).

More importantly, manual gates introduce delay and inconsistency. Deployments happen when someone remembers to approve them, not when they are ready.

The correct response to "we don't trust the pipeline to self-promote to production" is to improve the tests until you do trust them. The test stages in this pipeline — unit tests inside the built container, smoke tests against a live Railway dev environment, and full integration tests against all external dependencies — cover the same concerns a human reviewer would, faster and more consistently.

The post-production smoke test with automatic rollback provides the safety net: if the production deployment fails its health check, the previous version is restored without manual intervention.

---

## Two-stage deployment: dev before prod

**Decision: deploy to dev first, run all tests there, then promote to production.**

Dev and prod are separate Railway environments with separate URLs, tokens, and service IDs. The pipeline deploys to dev, validates the service is working (smoke test + integration test), then promotes the exact same image to production.

This matters for two reasons:

**Integration tests need a live environment.** Unit tests run inside the container and test business logic in isolation. Integration tests need the full service running with real network access — hitting a live API, a real database, third-party services. Dev is that environment.

**Promotion, not redeployment.** Because the same immutable image digest is used for both environments, there is no rebuild, no recompilation, no risk of behaviour difference between what was tested on dev and what runs on prod. The container that passed integration tests on dev is literally the same container that serves production traffic.

---

## Automatic rollback

**Decision: run a smoke test after deploying to production and call `railway rollback` on failure.**

Railway maintains a deployment history. `railway rollback` reverts to the previous successful deployment without requiring a new image build or pipeline run.

The post-production smoke test is a last-resort safety net. In practice, if all five pre-production stages pass, the production smoke test should also pass. But Railway cold-starts can occasionally fail (OOM, misconfigured environment variables, infrastructure hiccups), and the rollback ensures these edge cases don't require manual intervention.

---

## `secrets: inherit` vs explicit secret passing

**Decision: use `secrets: inherit` in service repo caller files.**

`secrets: inherit` forwards all secrets from the calling workflow's environment to the reusable workflow without requiring enumeration. This allows service repos to define project-specific secrets (e.g., `STRIPE_API_KEY`, `DATABASE_URL`) without modifying the shared workflow.

The shared workflows only declare the secrets they explicitly require (e.g., `RAILWAY_TOKEN`). Project-specific secrets flow through transparently and are available via environment variables in the integration test job.

The trade-off is reduced visibility into which secrets a workflow consumes. This is acceptable for an internal org framework where the shared workflow source is fully auditable.

---

## Versioning with floating major tags

**Decision: service repos pin to `@v1`, not `@v1.2.3`.**

Semantic versioning determines how consuming repos are affected by changes:

- **Patch** (`v1.0.1`): bug fix, no input changes, automatically applied via floating `@v1` tag
- **Minor** (`v1.1.0`): new optional input added, backwards compatible, automatically applied
- **Major** (`v2.0.0`): breaking change (input removed, behaviour change), requires manual update in consuming repos

The floating `@v1` tag is force-pushed on every non-breaking release. Consuming repos automatically receive patches and minor improvements without any changes on their side. Breaking changes require intentional adoption.

Service repos can pin to an exact tag or SHA (`@v1.2.3`) for maximum stability, at the cost of not automatically receiving fixes.
