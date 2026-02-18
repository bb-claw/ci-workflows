# Changelog

All notable changes to the `bb-claw/ci-workflows` framework are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

---

## [1.0.0] — 2026-02-18

### Added
- `full-pipeline.yml` — top-level CD orchestrator for all service repos
- `node-ci.yml` — CI template for Node.js services (lint + unit-test + build)
- `python-ci.yml` — CI template for Python services (flake8/ruff + pytest + build)
- `docker-build-push.yml` — reusable build & push to ghcr.io with GHA cache
- `deploy-railway.yml` — reusable Railway service redeployment via CLI
- `smoke-test.yml` — HTTP health check with 15-retry loop and configurable path
- `integration-test.yml` — runs caller repo's integration suite vs a live env URL
- `setup-railway/action.yml` — composite action: install Railway CLI + authenticate
- `health-check/action.yml` — composite action: curl retry loop with configurable params
- `notify-slack/action.yml` — composite action: Slack deployment notification
- `templates/service-repo/` — starter CI/CD workflow files for new service repos
- `docs/onboarding.md` — step-by-step adoption guide
- `docs/architecture.md` — design decisions and rationale
- `docs/troubleshooting.md` — common issues and fixes

### Pipeline behaviour (v1.0.0)
- Every push to feature branches triggers lint + unit tests immediately (shift-left)
- PRs to `main` additionally validate the Docker build (`push: false`)
- Merges to `main` trigger the full CD pipeline automatically
- Image is built and pushed **once** per pipeline run; all jobs share the same digest
- No manual approval gate — tests are the gate
- Automatic rollback if post-production smoke test fails

---

## How to read this file

Each release section lists:
- **Added** — new workflows, actions, or features
- **Changed** — backwards-compatible changes to existing behaviour
- **Fixed** — bug fixes
- **Breaking** — changes that require updates in consuming service repos

Breaking changes increment the major version (`v1` → `v2`) and require consuming
repos to update their `uses:` references.
