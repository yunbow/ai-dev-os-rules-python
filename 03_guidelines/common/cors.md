# CORS Design Guidelines

This document defines the design policy and implementation patterns for **CORS (Cross-Origin Resource Sharing)** in FastAPI applications.
It leverages FastAPI's built-in `CORSMiddleware` for standard CORS handling, with custom utilities for advanced scenarios like webhook protection and referer validation.

---

## 1. Core Policy

- **Configurable via environment variables**: Permitted origins can vary by deployment environment
- **Wildcard subdomain support**: Use `*.example.com` format to allow multiple subdomains at once
- **Webhook protection**: Replay attack prevention via timestamp validation
- **Referer validation**: Referer header validation as a CSRF countermeasure

---

## 2. Directory Structure

```text
app/
  core/
    cors.py              # CORS configuration (scope of this guideline)
  api/
    routes/
      webhook.py         # Webhook endpoints (CORS usage example)
  main.py                # FastAPI app with CORSMiddleware
```

---

## 3. Allowed Origin Management

### Configuration via Environment Variables

```python
# app/core/cors.py

import os


def get_allowed_origins() -> list[str]:
    env_origins = os.environ.get("ALLOWED_ORIGINS")

    if env_origins:
        return [o.strip() for o in env_origins.split(",")]

    # Default allowlist: used when ALLOWED_ORIGINS env var is not set.
    # Includes localhost for development and auto-detected deployment URLs.
    default_origins = [
        "http://localhost:3000",
        "http://localhost:8000",
    ]

    app_url = os.environ.get("APP_URL")
    if app_url:
        default_origins.append(app_url)

    return default_origins
```

**Key points:**

- Specify via comma-separated `ALLOWED_ORIGINS` environment variable (e.g., `https://app.example.com,https://admin.example.com`)
- When `ALLOWED_ORIGINS` is not set, falls back to localhost development URLs and `APP_URL`

---

## 4. FastAPI CORSMiddleware Setup

### Basic Configuration

```python
# app/main.py

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.cors import get_allowed_origins

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=get_allowed_origins(),
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization", "X-Request-ID"],
    max_age=86400,  # Cache preflight results for 24 hours
)
```

**Key points:**

- `CORSMiddleware` automatically handles preflight `OPTIONS` requests
- `allow_credentials=True` means `Access-Control-Allow-Origin` cannot be `*` (the middleware returns the specific matching origin)
- `max_age` caches preflight results (default: 24 hours)

### Wildcard Subdomain Support

For wildcard subdomain matching (`*.example.com`), use `allow_origin_regex`:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origin_regex=r"https://.*\.example\.com",
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization", "X-Request-ID"],
    max_age=86400,
)
```

---

## 5. Custom Origin Validation

For advanced scenarios where `CORSMiddleware` is not sufficient (e.g., combining wildcard and explicit origins):

```python
# app/core/cors.py

from urllib.parse import urlparse


def validate_origin(
    origin: str | None,
    allowed_origins: list[str] | None = None,
) -> bool:
    """Validate an origin against the allowlist."""
    if not origin:
        return True  # Same-origin requests have no Origin header

    origins = allowed_origins or get_allowed_origins()

    return any(
        _match_origin(origin, allowed) for allowed in origins
    )


def _match_origin(origin: str, pattern: str) -> bool:
    """Match an origin against a pattern, supporting wildcard subdomains."""
    if pattern.startswith("*."):
        domain = pattern[2:]
        parsed = urlparse(origin)
        return parsed.hostname is not None and parsed.hostname.endswith(f".{domain}")
    return origin == pattern
```

### Usage in Endpoints

```python
# app/api/routes/webhook.py

from fastapi import APIRouter, Request, HTTPException
from app.core.cors import validate_origin

router = APIRouter()


@router.post("/api/webhook/{service}")
async def handle_webhook(service: str, request: Request):
    origin = request.headers.get("origin")
    if not validate_origin(origin):
        raise HTTPException(status_code=403, detail="Forbidden")

    body = await request.json()
    # ... business logic

    return {"status": "ok"}
```

---

## 6. Timestamp Validation (Replay Attack Prevention)

Prevent replay attacks from old request resends in Webhook and external API integrations.

```python
# app/core/cors.py

import time
from datetime import datetime
from dataclasses import dataclass


@dataclass
class ValidationResult:
    valid: bool
    error: str | None = None


def validate_timestamp(
    timestamp: str | int | float,
    max_age_seconds: float = 300,  # Default: 5 minutes
) -> ValidationResult:
    """Validate a timestamp to prevent replay attacks."""
    try:
        if isinstance(timestamp, (int, float)):
            event_time = float(timestamp)
        else:
            event_time = datetime.fromisoformat(timestamp).timestamp()
    except (ValueError, TypeError):
        return ValidationResult(valid=False, error="Invalid timestamp format")

    now = time.time()
    age = now - event_time

    # Reject requests that are too old
    if age > max_age_seconds:
        return ValidationResult(
            valid=False,
            error=f"Timestamp too old: {int(age)}s ago",
        )

    # Also reject future timestamps
    if age < -max_age_seconds:
        return ValidationResult(
            valid=False,
            error=f"Timestamp in future: {int(-age)}s ahead",
        )

    return ValidationResult(valid=True)
```

**Key points:**

- Reject not only past but also future timestamps (timestamp tampering prevention)
- Default tolerance is 5 minutes (accounting for network latency)
- Supports both ISO 8601 strings and Unix timestamps

---

## 7. Referer Validation

Validate Referer headers as a CSRF countermeasure.

```python
# app/core/cors.py

from urllib.parse import urlparse


def validate_referer(
    referer: str | None,
    allowed_domains: list[str] | None = None,
) -> ValidationResult:
    """Validate the Referer header against allowed domains."""
    if not referer:
        return ValidationResult(valid=True)

    domains = allowed_domains or [
        urlparse(o).hostname
        for o in get_allowed_origins()
        if urlparse(o).hostname
    ]

    try:
        referer_host = urlparse(referer).hostname
        if referer_host is None:
            return ValidationResult(valid=False, error="Invalid referer URL")

        is_allowed = any(
            referer_host == domain or referer_host.endswith(f".{domain}")
            for domain in domains
        )
        return ValidationResult(valid=is_allowed, error=None if is_allowed else "Referer not allowed")
    except Exception:
        return ValidationResult(valid=False, error="Invalid referer URL")
```

---

## 8. Webhook Signature Verification

Type definitions and patterns for implementing service-specific signature verification:

```python
# app/core/webhook.py

import hmac
import hashlib
from app.core.cors import validate_timestamp, ValidationResult


async def verify_webhook_signature(
    headers: dict[str, str],
    body: bytes,
    webhook_secret: str,
) -> ValidationResult:
    """Verify webhook signature with timestamp validation."""
    # 1. Timestamp validation
    timestamp = headers.get("x-webhook-timestamp")
    if timestamp:
        ts_check = validate_timestamp(timestamp)
        if not ts_check.valid:
            return ts_check

    # 2. Signature verification (HMAC-SHA256 example)
    signature = headers.get("x-webhook-signature")
    if not signature:
        return ValidationResult(valid=False, error="Missing signature")

    expected = hmac.new(
        webhook_secret.encode(),
        body,
        hashlib.sha256,
    ).hexdigest()

    if not hmac.compare_digest(signature, expected):
        return ValidationResult(valid=False, error="Invalid signature")

    return ValidationResult(valid=True)
```

**SSRF prevention**: When accessing URLs received via Webhook, always validate the domain.

---

## 9. Best Practices

### DO (Recommended)

```python
# ✅ Use CORSMiddleware for standard CORS handling
app.add_middleware(
    CORSMiddleware,
    allow_origins=get_allowed_origins(),
    allow_credentials=True,
    allow_methods=["POST"],  # Restrict per-app needs
    allow_headers=["Content-Type", "Authorization"],
)

# ✅ Manage allowed origins via environment variables
# ALLOWED_ORIGINS=https://app.example.com,https://admin.example.com

# ✅ Add timestamp validation for webhooks
ts_result = validate_timestamp(request.headers.get("x-webhook-timestamp"))

# ✅ Use origin validation + error handling together
origin = request.headers.get("origin")
if not validate_origin(origin):
    raise HTTPException(status_code=403, detail="Forbidden")
```

### DON'T (Not Recommended)

```python
# ❌ Allow all origins in production
app.add_middleware(CORSMiddleware, allow_origins=["*"])

# ❌ Access webhook certificate URL without validation (SSRF)
cert = httpx.get(headers.get("cert-url"))  # Dangerous
```

---

## 10. Summary

| Feature | Implementation | Purpose |
|------|---------|------|
| Standard CORS | `CORSMiddleware` | Preflight handling and response headers |
| Origin validation | `validate_origin()` | Custom origin checking for advanced cases |
| Timestamp validation | `validate_timestamp()` | Replay attack prevention |
| Referer validation | `validate_referer()` | CSRF countermeasure |
| Webhook verification | `verify_webhook_signature()` | Webhook integrity check |
| Wildcard | `allow_origin_regex` or `*.example.com` | Batch subdomain allowance |
