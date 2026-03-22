# Testing Guidelines

This document defines testing strategy for large-scale Python applications (FastAPI / CLI), focusing on project-specific decisions and non-obvious patterns.

---

## 1. Testing Core Policy

## 1. Prioritize Testing Critical Logic

Critical logic = code where a bug has **high business cost or security impact**:

- **Financial**: Pricing calculations, subscription billing, payment flows ({Payment Service})
- **Security**: Authentication (OAuth / JWT), authorization decisions, IDOR prevention
- **Data integrity**: DB queries (SQLAlchemy) that modify state, cascade deletes
- **Core business rules**: Date calculations, scheduling logic, domain-specific validation
  → If a bug in this code would require immediate hotfix or cause revenue loss, it is critical.

## 2. Server-side logic uses **unit tests + API integration tests** as the baseline

- FastAPI endpoint logic is unit-testable via `TestClient`
- **Note DB environment differences**
  - Both production and test environments use SQLite (or PostgreSQL). Use a separate SQLite file (or in-memory DB) for testing.
  - Add a migration dry run (Alembic `--sql`) to CI/CD to verify schema consistency.

## 3. UI testing priority: **E2E > Integration > Unit**

- If the project has a frontend, UI tests are costly, so only guarantee "critical user experience areas" with E2E

## 4. Introduce Contract Tests for external service integrations

- For external APIs like payment APIs, AI APIs, and RSS,
  adopt Contract Tests that are resilient to specification changes in real services
- **Note**: Contract Tests serve a different purpose from response mocking (detecting external service specification changes)
  - Implementation method needs clarification (e.g., Pact + Webhook payload verification, SDK type definition comparison, OpenAPI specification comparison)

---

## 2. Layer-Specific Testing Strategy

## 1. Domain Logic (Unit Test)

- Pydantic models must always have standalone schema tests

### Example: Pricing logic / data transformation

```text
features/billing/domain/
features/calendar/domain/
```

---

## 2. API Endpoints (Integration Test)

### Key Points

- Switch SQLAlchemy to **a test-specific SQLite / in-memory DB**
- Mock external APIs (payment / AI API, etc.) with `unittest.mock` or `respx`/`responses`
- Add migration dry runs to CI/CD

### FastAPI Endpoint Test Pattern

```python
## tests/unit/api/test_project_endpoints.py

import pytest
from httpx import AsyncClient, ASGITransport
from unittest.mock import patch, AsyncMock

from app.main import app
from app.db import get_db
from tests.conftest import override_get_db, TestSessionLocal


@pytest.fixture(autouse=True)
def db_session():
    """Reset DB for each test."""
    db = TestSessionLocal()
    try:
        yield db
    finally:
        db.rollback()
        db.close()


class TestProjectEndpoints:
    """Project API endpoint tests."""

    @pytest.mark.anyio
    async def test_create_project_authenticated(self, auth_headers):
        """Allows authenticated users to create a project."""
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            response = await client.post(
                "/api/projects",
                json={"name": "Test Project"},
                headers=auth_headers,
            )

        assert response.status_code == 201
        data = response.json()
        assert "id" in data

    @pytest.mark.anyio
    async def test_create_project_unauthenticated(self):
        """Returns 401 for unauthenticated users."""
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            response = await client.post(
                "/api/projects",
                json={"name": "Test"},
            )

        assert response.status_code == 401

    @pytest.mark.anyio
    async def test_create_project_validation_error(self, auth_headers):
        """Returns 422 on validation error."""
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            response = await client.post(
                "/api/projects",
                json={"name": ""},  # Empty string
                headers=auth_headers,
            )

        assert response.status_code == 422
        data = response.json()
        assert "detail" in data


class TestProjectIDOR:
    """IDOR prevention tests."""

    @pytest.mark.anyio
    async def test_delete_other_users_project_forbidden(
        self, auth_headers, other_user_project_id
    ):
        """Returns 403 for another user's project."""
        transport = ASGITransport(app=app)
        async with AsyncClient(transport=transport, base_url="http://test") as client:
            response = await client.delete(
                f"/api/projects/{other_user_project_id}",
                headers=auth_headers,
            )

        assert response.status_code == 403
```

### Result Pattern Test Helper

```python
## tests/helpers/result_helpers.py

from typing import Any
from dataclasses import dataclass


@dataclass
class ActionResult:
    success: bool
    data: Any = None
    error: dict | None = None


def assert_success(result: ActionResult) -> None:
    """Assert that an ActionResult is successful."""
    assert result.success is True, f"Expected success but got error: {result.error}"
    assert result.data is not None


def assert_failure(result: ActionResult, code: str) -> None:
    """Assert that an ActionResult is a failure with a specific code."""
    assert result.success is False, "Expected failure but got success"
    assert result.error is not None
    assert result.error["code"] == code


## Usage example
def test_project_creation_succeeds(service, db_session):
    result = service.create_project(name="Test")
    assert_success(result)
    assert result.data["id"] is not None
```

---

## 3. UI / Pages (E2E Test)

### Critical Path Criteria

An E2E test is "critical path" if it covers a flow where **failure blocks the user from completing their primary goal** or **causes financial/security harm**:

- Authentication flow (login / sign up) — users cannot access the app without it
- Payment flow — revenue-impacting
- Dashboard main navigation — primary entry point after auth
- Critical CRUD operations (e.g., task create → update → delete) — core value proposition

### Security Test Playbook

- Attempt to access other users' data by ID replacement after login
- CSRF token exploitation
- XSS / Injection via form input

---

## 4. Contract Test (External Services)

### Payment API / AI API, etc

- Payment/subscription API contracts, Webhook payloads, SDK type definition consistency
- Implement Consumer-Driven Contracts with Pact, etc.
- Purpose: detect specification changes in external services
- `respx`/`responses` is for internal API mocking, not a substitute for Contract Tests

---

## 3. Performance & Quality Assurance

## 1. Performance Test

- API endpoint load testing with k6 or locust
- Response time measurement under realistic load

## 2. Automated API Performance Measurement

- Integrate into CI/CD, auto-measure on PR

## 3. Security Test

- Integrate static/dynamic analysis tools (ZAP, Bandit, SonarQube) into CI/CD
- Detect vulnerabilities in authentication, authorization, and payment flows

---

## 4. Test Target Priority

1. Authentication (OAuth / JWT)
2. Payments ({Payment Service})
3. DB (SQLAlchemy) + API Endpoints
4. Logic (including Pydantic models)
5. Critical API flows and integrations
6. Key external API integrations
7. Edge cases (e.g., invalid tokens / expired sessions)

---

## 5. Directory Structure

```text
tests/
  unit/
    calendar/
    billing/
  integration/
    api/
    services/
  e2e/
    auth/
    dashboard/
  conftest.py          # Shared fixtures
  helpers/
    result_helpers.py
app/
  features/
    billing/
      domain/
      routes/
      services/
    calendar/
      domain/
```

---

## 6. Summary

- Increase change resilience for external APIs (payment, AI API, etc.) with Contract Tests
- `respx`/`responses`-based external API mocking strategy
- E2E covers only flows where failure blocks users or causes financial/security harm
- Integrate Security Tests (Bandit, ZAP) into CI/CD for early vulnerability detection
- Run tests on test SQLite (file or in-memory) with production-identical schemas
