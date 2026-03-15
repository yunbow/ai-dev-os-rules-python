# CI/CD Guidelines

This document defines the CI/CD pipeline design policy for Python applications (FastAPI / CLI).

**Target Infrastructure:** Docker / GitHub Actions

---

## 1. Core Principles

- **Staged pipeline**: Safe rollout through CI → Staging → Production. "Safe" means: every stage must pass automated checks (lint, type-check, tests, build) before proceeding, with auto-rollback on post-deployment validation failure.
- **Post-Deployment Validation**: Execute health checks and Smoke Tests after deployment. Smoke Tests verify: the app starts, the health endpoint returns 200, authentication flow completes, and at least one API route returns a valid response.

---

## 2. GitHub Actions-Based CI

### Recommended Workflow

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main, develop]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      # Install uv for fast dependency management
      - uses: astral-sh/setup-uv@v4

      # Leverage caching
      - uses: actions/cache@v4
        with:
          path: ~/.cache/uv
          key: ${{ runner.os }}-uv-${{ hashFiles('uv.lock') }}

      - run: uv sync
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy .
      - run: uv run pytest
      - run: uv run docker build -t app:ci .

      # Upload build artifacts for reuse during deployment
      - uses: actions/upload-artifact@v4
        with:
          name: docker-image
          path: /tmp/app-ci.tar

  # Separate E2E tests into a dedicated job (to avoid blocking fast feedback from unit tests)
  e2e:
    runs-on: ubuntu-latest
    needs: verify
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - uses: astral-sh/setup-uv@v4
      - run: uv sync
      - run: uv run pytest tests/e2e/
```

---

## 3. Preview / Staging Deployment

### Docker-Based Staging

- On PR creation, **build and deploy a staging container** via CI
- Staging URL is posted in PR comments
- Used for API review, integration testing, and final review

### Reusing Build Artifacts

```yaml
- uses: actions/download-artifact@v4
  with:
    name: docker-image
```

Reuse Docker images created during CI to reduce duplicate build time.

---

## 4. Production Deployment

### Docker Production

- Auto-deploy after merging into the main branch
- Manual Approval required (Protected Branch settings or deployment environment protection rules)
- Container orchestration via Docker Compose, Kubernetes, or cloud container services

### Post-Deployment Validation

Execute external health checks / Smoke Tests after deployment completion. Auto-rollback on failure:

- **Docker**: Revert to the previous image tag
- **Kubernetes**: Automatic rollback via deployment strategy

---

## 5. Release Strategy

- **Semantic Versioning**: `MAJOR.MINOR.PATCH`
- **GitHub Release integration**
- **Automatic CHANGELOG generation**: `release-please` / `python-semantic-release`
- **Deployment gate**: Manual Approval required before main → Production deployment

---

## 6. Database Migration (Alembic)

- **CI**: `alembic check` (verify no pending migrations) + `alembic upgrade head --sql` (dry run)
- **Production migrate**: Manual or safe CD stage
  - Non-destructive migrations → Auto-execute within the CD pipeline
  - Changes requiring downtime → Execute manually during a maintenance window
- Manage execution order considering **version mismatches between application and DB schema**

---

## 7. Secrets / Environment Variable Management

| Platform | Management Location |
|----------------|---------|
| GitHub Actions | Actions secrets |
| Docker | Docker secrets / env files (not committed) |
| Cloud (AWS/GCP) | Secret Manager / SSM Parameter Store |

- Do not handle `.env` files directly in production; inject from Secrets
- Protect production secrets so they are not exposed in PRs

---

## 8. Rollback Strategy

| Platform | Method |
|----------------|------|
| Docker Compose | Revert to previous image tag |
| Kubernetes | `kubectl rollout undo` |
| GitHub | Release Revert → Auto-deploy PR to main |

**Always execute auto-rollback when Post-Deployment Validation fails.**

---

## 9. Performance Measurement / Monitoring

- Prometheus + Grafana (metrics)
- Sentry (incident monitoring)
- Structured logging with JSON (Datadog / ELK / CloudWatch)
- Automated API response time checks

---

## 10. Pipeline Optimization

- **Leverage dependency caching**: `actions/cache@v4` with `uv.lock`
- **Use `uv sync`** (fast install from lockfile)
- **Test parallelization**: `pytest-xdist` for parallel test execution
- **Docker layer caching**: Use multi-stage builds and BuildKit cache mounts

---

## 11. Monorepo Support (As Needed)

Leverage project organization tools:

- `uv` workspaces for multi-package projects
- Separate Docker builds per service
- Affected scope detection (only test/build changed packages)

---

## Summary

| Item | Policy |
|------|------|
| CI | Unified with GitHub Actions |
| Deployment | Docker containers |
| Quality Assurance | Staging deploy + Post-Deployment Validation |
| Release | Semantic Versioning + Manual Approval |
| Rollback | Automated + Previous image revert |
| Secrets | Strict management (via Secrets) |
