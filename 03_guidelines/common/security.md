# Security Guidelines

This document summarizes the **security and privacy strategy** for Python web applications.
Deployment target: **Cloud platform (e.g., AWS, GCP, or self-hosted)**

---

## Purpose of the Security Strategy

* Handle sensitive data securely
* Prevent authentication and authorization vulnerabilities
* Minimize attack vectors for APIs, databases, and external integrations
* Implement privacy measures in compliance with applicable laws (e.g., data protection regulations, GDPR)
* Prevent misconfigurations and secret leaks during operations

---

## 1. Fundamental Principles (Zero Trust Architecture)

* Default deny: grant only the necessary permissions
* Separate responsibilities across each layer: Client -> Server -> DB
* Minimize scope of sessions, cookies, and tokens
* Require validation (Pydantic) for all input data
* Never include secrets in code
* Use minimal scopes for external API integrations (payment services, AI APIs, etc.)

---

## 2. Client-Side Security

## CSRF Prevention

* SameSite=Lax or stricter
* Enforce HTTPS
* Perform **Origin / Referer checks** in API routes
* Allow only POST/PUT/DELETE for state-changing APIs

## Clickjacking Prevention

HTTP Headers:

```yaml
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none';
```

---

## 3. API / Route Security

## Authorization

* Introduce RBAC/ABAC (role-based or attribute-based access control)
* Perform authorization checks at the route handler / dependency layer
* **Require IDOR prevention**: When accessing resources (e.g., `GET /api/users/{id}`), always verify that the requesting user is the owner of that resource
* Verify through unit tests and integration tests that horizontal and vertical privilege escalation does not occur in authorization logic

## Rate Limiting

* Apply IP-based rate limiting via FastAPI middleware
* When using an API Gateway, apply WAF + Throttling
* For external APIs (payment services, AI APIs, etc.), leverage each service's built-in rate limiting

---

## 3.1 IDOR Prevention Pattern (Implementation Examples)

> **Reference:** See common/error-handling.md for details on the exception hierarchy

## Basic Pattern: Ownership Verification Dependency

```python
## src/lib/dependencies/auth.py
from fastapi import Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

import structlog

logger = structlog.get_logger()


async def require_project_ownership(
    project_id: str,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> Project:
    project = await db.get(Project, project_id)

    if not project:
        raise HTTPException(status_code=404, detail="Project not found")

    # IDOR prevention: verify ownership
    if project.user_id != current_user.id:
        logger.warning(
            "idor_attempt_detected",
            user_id=current_user.id,
            project_id=project_id,
            owner_id=project.user_id,
        )
        raise HTTPException(status_code=403, detail="Forbidden")

    return project
```

## Usage Example in Route Handlers

```python
## Bad: No ownership verification
@router.put("/projects/{project_id}")
async def update_project(
    project_id: str, data: UpdateProjectSchema, db: AsyncSession = Depends(get_db)
):
    await db.execute(
        update(Project).where(Project.id == project_id).values(**data.model_dump())
    )  # Can update other users' projects


## Good: With ownership verification
@router.put("/projects/{project_id}")
async def update_project(
    data: UpdateProjectSchema,
    project: Project = Depends(require_project_ownership),
    db: AsyncSession = Depends(get_db),
):
    for key, value in data.model_dump().items():
        setattr(project, key, value)
    await db.flush()
```

## Query Filter Pattern

```python
## Query pattern to retrieve only the user's own resources
@router.get("/projects")
async def get_user_projects(
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
) -> list[ProjectSchema]:
    # Filter by user_id in the WHERE clause
    result = await db.execute(
        select(Project).where(Project.user_id == current_user.id)
    )
    return [ProjectSchema.model_validate(p) for p in result.scalars().all()]
```

---

## 3.2 Rate Limiting

> **Reference:** See common/rate-limiting.md for complete implementation patterns

## Presets

| Preset | Limit | Use Case |
|--------|-------|----------|
| auth | 10/min | Login, password reset |
| generation | 30/hour | AI generation |
| api | 100/min | General API |
| strict | 5/min | Critical operations |

## Implementation Approach

* **Development / single instance**: Memory-based store
* **Production / multiple instances**: Redis
* **Standard headers**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`

## MUST: Apply Rate Limiting to ALL Auth Endpoints

MUST apply rate limiting to ALL endpoints that handle authentication — including registration, login, password reset request, and password reset. Apply per-IP limits before any auth processing:

```python
@router.post("/auth/register")
async def register(request: Request, data: RegisterInput):
    # ✅ MUST: Rate limit before auth processing
    client_ip = request.client.host
    await check_rate_limit(f"auth:register:{client_ip}", max_requests=5, window_seconds=60)
    # ... rest of registration logic
```

---

## 3.3 Webhook Security

## Signature Verification Pattern

```python
## lib/webhooks/verification.py
import hmac
import hashlib
import os

import structlog

logger = structlog.get_logger()


async def verify_webhook_signature(
    body: bytes,
    signature: str,
) -> bool:
    """Verify webhook signature using HMAC-SHA256."""
    secret = os.environ["WEBHOOK_SECRET"]
    expected = hmac.new(
        secret.encode(),
        body,
        hashlib.sha256,
    ).hexdigest()

    is_valid = hmac.compare_digest(expected, signature)

    if not is_valid:
        logger.warning("invalid_webhook_signature")

    return is_valid
```

## Replay Attack Prevention

```python
async def process_webhook(event_id: str, db: AsyncSession) -> bool:
    """Check if webhook event was already processed (idempotency)."""
    existing = await db.execute(
        select(WebhookEvent).where(WebhookEvent.external_id == event_id)
    )

    if existing.scalar_one_or_none():
        logger.info("webhook_already_processed", event_id=event_id)
        return False  # Ensure idempotency

    # Record the start of processing
    db.add(WebhookEvent(external_id=event_id, status="processing"))
    await db.flush()

    return True
```

## Timestamp Validation

```python
## Reject webhooks that are too old (replay attack prevention)
from datetime import datetime, timezone

WEBHOOK_MAX_AGE_SECONDS = 5 * 60  # 5 minutes


def validate_timestamp(timestamp: str) -> bool:
    event_time = datetime.fromisoformat(timestamp)
    now = datetime.now(timezone.utc)
    age = (now - event_time).total_seconds()

    if age > WEBHOOK_MAX_AGE_SECONDS:
        logger.warning("webhook_too_old", timestamp=timestamp, age_seconds=age)
        return False

    return True
```

---

## 3.4 Security Headers Middleware

Add security headers to all responses via FastAPI middleware.

```python
## src/middleware/security_headers.py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        response = await call_next(request)
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["Content-Security-Policy"] = "frame-ancestors 'none'"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        return response
```

---

## 3.5 Webhook Certificate URL SSRF Prevention

When using certificate URLs for webhook signature verification, prevent SSRF attacks.

```python
## src/lib/external/webhook_utils.py
from urllib.parse import urlparse


def is_valid_webhook_cert_url(cert_url: str, allowed_hosts: list[str]) -> bool:
    parsed = urlparse(cert_url)
    if parsed.scheme != "https":
        return False
    # allowed_hosts example: ["api.example.com", "api.sandbox.example.com"]
    return any(
        parsed.hostname == host or (parsed.hostname and parsed.hostname.endswith(f".{host}"))
        for host in allowed_hosts
    )
```

* Use a domain allowlist approach (safer than IP blocklists)
* Call at the beginning of webhook signature verification

---

## 3.6 Email Template HTML Injection Prevention

Always escape user-derived data when embedding it in HTML emails.

```python
## src/lib/email.py
import html


def escape_html(value: str) -> str:
    return html.escape(value)
```

* Trusted values such as translation results, plan names, UUIDs, and numbers do not need escaping
* Free-text fields such as admin-entered notes must always be escaped

---

## 3.7 Error Response Information Leakage Prevention

* Never expose stack traces in API error responses
* Only safe error codes and messages may be returned
* Exclude `traceback` from all production error responses

---

## 3.8 Session Management

* Maximum concurrent sessions: 5 (per user)
* Session list endpoint: display device, IP, and location information
* Bulk logout functionality
* TOTP 2FA: lock for 15 minutes after 5 failed attempts, send lock notification email

### Suspicious Login Detection and Notification

```python
## src/lib/security/suspicious_login_detector.py
from dataclasses import dataclass


@dataclass
class SuspiciousLoginResult:
    is_suspicious: bool
    reasons: list[str]
    current_country: str | None
    known_countries: list[str]


async def detect_suspicious_login(
    user_id: str,
    current_country: str | None,
    db: AsyncSession,
) -> SuspiciousLoginResult:
    ...
```

* Compare the current country against successful login history from the past 30 days
* Return `is_suspicious=True` when a login is from a new country
* `LoginHistory` model has a `country` column (ISO 3166-1 alpha-2)
* Country information is obtained from `x-user-country` / `cf-ipcountry` or GeoIP headers
* On detection, send email notification via `send_suspicious_login_email()` (including IP, country, browser, and OS information)
* **Asynchronous sending**: do not block the login flow
* **Fail-safe**: on error, assume "not suspicious" (do not block legitimate users)
* First login (no history) is not treated as suspicious

---

## 3.9 OSS License Policy

> **Reference:** Refer to project-specific legal and compliance guidelines

* **Prohibited**: AGPL, GPL-2.0, GPL-3.0 (SaaS source code disclosure obligation)
* **Permitted**: MIT, Apache-2.0, ISC, BSD variants, CC0, Unlicense, MPL-2.0
* Run `pip-licenses` or `liccheck` when adding dependency packages

---

## 3.10 Admin Panel IP Restriction

```python
## src/lib/security/admin_ip_restriction.py
import ipaddress
import os


def is_admin_ip_allowed(client_ip: str | None) -> bool:
    allowed_ips = os.environ.get("ADMIN_ALLOWED_IPS")
    if not allowed_ips:
        return True  # No restriction if undefined

    if not client_ip:
        return False  # Block if IP cannot be determined

    try:
        client = ipaddress.ip_address(client_ip)
    except ValueError:
        return False

    for entry in allowed_ips.split(","):
        entry = entry.strip()
        try:
            if "/" in entry:
                if client in ipaddress.ip_network(entry, strict=False):
                    return True
            else:
                if client == ipaddress.ip_address(entry):
                    return True
        except ValueError:
            continue

    return False
```

* Set IP allowlist via the `ADMIN_ALLOWED_IPS` environment variable
* Multiple IPs can be specified with comma separation (e.g., `203.0.113.1,192.168.1.0/24`)
* **CIDR notation support**: subnet-level authorization
* If undefined, no restriction is applied (all IPs can access)
* Block access if IP cannot be determined
* Check in FastAPI dependency when accessing `/admin` paths

---

## 3.11 Maintenance Mode

```python
## src/middleware/maintenance.py
import os
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse

from lib.security.admin_ip_restriction import is_admin_ip_allowed


class MaintenanceMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if (
            os.environ.get("MAINTENANCE_MODE") == "true"
            and request.url.path != "/maintenance"
            and not is_admin_ip_allowed(request.client.host if request.client else None)
        ):
            return JSONResponse(
                status_code=503,
                content={"error": {"code": "MAINTENANCE", "message": "Service is under maintenance"}},
            )

        return await call_next(request)
```

* Enable with the environment variable `MAINTENANCE_MODE=true`
* Return 503 to all non-admin users
* **Admin IP bypass**: IPs listed in `ADMIN_ALLOWED_IPS` can access normally

---

## 4. Database Security

## SQLAlchemy + DB Protection

* Use minimum privileges for DB users (consider introducing read-only users). "Minimum privileges" means:
  * Application DB user: only `SELECT`, `INSERT`, `UPDATE`, `DELETE` on application tables — no `CREATE`, `DROP`, `ALTER`, or `GRANT`
  * Read-only DB user (for analytics/reporting): only `SELECT`
  * Migration DB user (CI only): full DDL permissions, never used at runtime
* Store DB credentials in `.env` or cloud secret management

## Data Encryption

* Set appropriate **file access permissions** for the database (SQLite)
* Consider application-layer encryption for personal data (e.g., AES-256 via `cryptography` library)
* **Encryption key management**: Manage keys in a dedicated secret management service (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault), retrieving them only at runtime
* **Key rotation**: Require periodic key rotation

---

## 5. Secrets / Environment Variable Management

## Cloud Platforms

* Use platform-specific secret management (AWS Secrets Manager, GCP Secret Manager)
* Manage separately for Development / Staging / Production
* Never include in Git
* Use `pydantic-settings` for typed environment variable management

```python
## src/config.py
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    database_url: str
    secret_key: str
    webhook_secret: str

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}
```

## GitHub

* Manage with GitHub Actions Secrets
* Rule: do not pass production environment variables to PR environments

---

## 6. HTTPS / Communication Security

* **Enforce HTTPS at the load balancer / reverse proxy level**
* Enforce wss:// when using WebSockets
* API calls must use HTTPS only
* Set the Secure attribute on cookies

---

## 7. External Service Integration Security

## Payment Services

* Protect webhooks with signature verification
* Always verify webhook signatures (see Section 3.3)
* Strictly separate Client ID and Secret

## AI API / OAuth

* Minimize OAuth authorization scopes
* Use short-lived Access Tokens (encrypt Refresh Tokens for storage)

## RSS / Markdown

* Validate external RSS URLs with Pydantic
* Fetch and render with sanitization to prevent XSS (use `bleach` or `nh3` for HTML sanitization)

---

## 8. Session Management

* Cookie-based sessions (Secure / HttpOnly / SameSite)
* Store encryption keys as Secrets even when using JWT mode
* Do not include sensitive data in session information
* Always enable OAuth Provider state parameter validation

---

## 9. Privacy Protection (Privacy by Design)

## Minimize Collected User Data

* Collect only data with a clear purpose and necessity
* Establish data retention period policies
* Recommend anonymization / pseudonymization

## Cookie & Tracking Management

* Implement Cookie Policy / Consent Banner
* Use anonymized analytics

## GDPR / Data Protection Law Compliance

* Support data deletion requests (Right to Erasure)
* Support user data export (Right to Access)
* Maintain a privacy policy

---

## 10. CI/CD Security

* Inject secrets securely in GitHub Actions
* All PRs must go through review and CI
* Detect vulnerabilities with Dependabot / `pip-audit` / `safety` / Snyk
* **Vulnerability response SLA**: Patch Critical (CVSS 9.0+) within 24 hours, High (CVSS 7.0+) within 7 days
* When patching is difficult, introduce mitigation measures at the WAF / API Gateway level
* Do not include sensitive data in containers or build artifacts

---

## 11. Security Audit / Automated Checks

* Static analysis with `bandit` (Python security linter)
* Vulnerability checks with `pip-audit` / `safety` / Snyk
* Dependency license checks with `pip-licenses` / `liccheck`

---

## 12. Security Monitoring and Response

* **Log collection**: Record authentication failures, authorization failures (IDOR, etc.), and Pydantic validation errors at ERROR/CRITICAL level in Sentry / CloudWatch
* **Alert operations**:

  * Abnormal login attempts (e.g., 100+ in 10 minutes)
  * API authorization failures (e.g., 50+ in 5 minutes)
    -> Immediately notify the responsible team
* **Operational rules**: Establish regular reviews and incident response procedures

---

## Summary

* Authentication and authorization based on **Zero Trust principles**
* **Require input validation for all inputs** (Pydantic)
* Manage secrets securely with **cloud secret managers / GitHub Secrets**
* Define explicit permission boundaries for API / DB / external API access; require IDOR prevention
* Manage encryption keys with a dedicated service and rotate them regularly
* Respond to CI/CD vulnerabilities promptly based on SLAs
* Ensure attack detection and response through logging, monitoring, and alerting
* Comply with privacy laws and minimize user data collection
