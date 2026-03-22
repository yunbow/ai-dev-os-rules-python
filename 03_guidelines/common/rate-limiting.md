# Rate Limiting Guidelines

This document defines the implementation patterns for API and endpoint rate limiting.

---

## 1. Core Policy

- **ASGI-compatible**: Use a memory-based store by default. Migrate to Redis when any of these conditions are met:
  - Running 2+ server instances (memory stores are per-instance, so limits are not shared)
  - Sustained traffic exceeds 100 requests/second (memory cleanup overhead becomes significant)
  - Rate limit accuracy is business-critical (e.g., billing-related quotas where approximate counts are unacceptable)
- **Preset configurations**: Provide default settings by use case
- **User identification**: Limit by IP address or user ID

---

## 2. Implementation Patterns

### 2.1 Memory-Based Store

```python
# lib/api/rate_limit.py
import time
import threading
from dataclasses import dataclass, field


@dataclass
class RateLimitEntry:
    count: int = 0
    reset_time: float = 0.0


# Memory store (each instance is independent in multi-process deployments)
_store: dict[str, RateLimitEntry] = {}
_lock = threading.Lock()


def _cleanup_store() -> None:
    """Remove expired entries from the store."""
    now = time.time()
    with _lock:
        expired_keys = [key for key, entry in _store.items() if entry.reset_time < now]
        for key in expired_keys:
            del _store[key]


# Periodic cleanup (run in background thread)
def _start_cleanup_timer(interval: float = 60.0) -> None:
    def cleanup_loop():
        while True:
            time.sleep(interval)
            _cleanup_store()

    thread = threading.Thread(target=cleanup_loop, daemon=True)
    thread.start()


_start_cleanup_timer()
```

### 2.2 Preset Configurations

```python
from dataclasses import dataclass
from enum import Enum


@dataclass(frozen=True)
class RateLimitConfig:
    limit: int
    window_seconds: float


class RateLimitPreset(str, Enum):
    AUTH = "auth"
    GENERATION = "generation"
    API = "api"
    WEBHOOK = "webhook"
    STRICT = "strict"


RATE_LIMIT_PRESETS: dict[RateLimitPreset, RateLimitConfig] = {
    # Auth: 10 requests/minute (brute force prevention)
    RateLimitPreset.AUTH: RateLimitConfig(limit=10, window_seconds=60),
    # AI generation: 30 requests/hour
    RateLimitPreset.GENERATION: RateLimitConfig(limit=30, window_seconds=3600),
    # General API: 100 requests/minute
    RateLimitPreset.API: RateLimitConfig(limit=100, window_seconds=60),
    # Webhook: 50 requests/minute
    RateLimitPreset.WEBHOOK: RateLimitConfig(limit=50, window_seconds=60),
    # Strict: 5 requests/minute (password reset, etc.)
    RateLimitPreset.STRICT: RateLimitConfig(limit=5, window_seconds=60),
}
```

### 2.3 Rate Limit Check

```python
@dataclass
class RateLimitResult:
    allowed: bool
    limit: int
    remaining: int
    reset_time: int  # Unix timestamp (seconds)
    retry_after: int | None = None


def check_rate_limit(
    identifier: str,
    preset: RateLimitPreset | RateLimitConfig,
) -> RateLimitResult:
    config = RATE_LIMIT_PRESETS[preset] if isinstance(preset, RateLimitPreset) else preset
    now = time.time()
    key = f"{identifier}:{config.limit}:{config.window_seconds}"

    with _lock:
        entry = _store.get(key)

        # Reset if no entry exists or window has elapsed
        if not entry or entry.reset_time < now:
            entry = RateLimitEntry(count=0, reset_time=now + config.window_seconds)
            _store[key] = entry

        allowed = entry.count < config.limit
        if allowed:
            entry.count += 1

    return RateLimitResult(
        allowed=allowed,
        limit=config.limit,
        remaining=max(0, config.limit - entry.count),
        reset_time=int(entry.reset_time),
        retry_after=None if allowed else int(entry.reset_time - now) + 1,
    )
```

---

## 3. Client IP Retrieval

### 3.1 Proxy Support

```python
# lib/api/client_ip.py
from starlette.requests import Request


def get_client_ip(request: Request) -> str:
    # Standard proxy header
    x_forwarded_for = request.headers.get("x-forwarded-for")
    if x_forwarded_for:
        return x_forwarded_for.split(",")[0].strip()

    # Cloudflare
    cf_connecting_ip = request.headers.get("cf-connecting-ip")
    if cf_connecting_ip:
        return cf_connecting_ip

    # AWS ALB / Nginx
    x_real_ip = request.headers.get("x-real-ip")
    if x_real_ip:
        return x_real_ip

    # Fallback to ASGI client
    if request.client:
        return request.client.host

    return "unknown"
```

### 3.2 Identifier Generation

```python
def get_rate_limit_identifier(
    request: Request,
    user_id: str | None = None,
) -> str:
    """
    Generate an identifier for rate limiting.
    Prioritize user ID if available, otherwise use IP address.
    """
    if user_id:
        return f"user:{user_id}"
    return f"ip:{get_client_ip(request)}"
```

---

## 4. HTTP Headers

### 4.1 Setting Standard Headers

```python
from starlette.responses import Response


def set_rate_limit_headers(response: Response, result: RateLimitResult) -> None:
    response.headers["X-RateLimit-Limit"] = str(result.limit)
    response.headers["X-RateLimit-Remaining"] = str(result.remaining)
    response.headers["X-RateLimit-Reset"] = str(result.reset_time)

    if result.retry_after is not None:
        response.headers["Retry-After"] = str(result.retry_after)
```

### 4.2 Usage in FastAPI Route Handlers

```python
# routes/generate.py
from fastapi import APIRouter, Request
from fastapi.responses import JSONResponse

from lib.api.rate_limit import (
    check_rate_limit,
    get_rate_limit_identifier,
    set_rate_limit_headers,
    RateLimitPreset,
)

router = APIRouter()


@router.post("/api/generate")
async def generate(request: Request):
    identifier = get_rate_limit_identifier(request)
    rate_limit_result = check_rate_limit(identifier, RateLimitPreset.GENERATION)

    if not rate_limit_result.allowed:
        response = JSONResponse(
            status_code=429,
            content={
                "error": {
                    "code": "RATE_LIMIT_EXCEEDED",
                    "message": "Too many requests. Please try again later.",
                    "retry_after": rate_limit_result.retry_after,
                },
            },
        )
        set_rate_limit_headers(response, rate_limit_result)
        return response

    # Normal processing...
    response = JSONResponse(content={"success": True})
    set_rate_limit_headers(response, rate_limit_result)
    return response
```

---

## 5. Usage as FastAPI Dependency

```python
# lib/api/rate_limit_dependency.py
from fastapi import Depends, Request, HTTPException

from lib.api.rate_limit import (
    check_rate_limit,
    get_rate_limit_identifier,
    RateLimitPreset,
)
from lib.dependencies.auth import get_current_user_optional


async def require_rate_limit(
    preset: RateLimitPreset,
):
    """Create a FastAPI dependency for rate limiting."""

    async def dependency(
        request: Request,
        current_user=Depends(get_current_user_optional),
    ) -> None:
        user_id = current_user.id if current_user else None
        identifier = get_rate_limit_identifier(request, user_id)
        result = check_rate_limit(identifier, preset)

        if not result.allowed:
            raise HTTPException(
                status_code=429,
                detail={
                    "code": "RATE_LIMIT_EXCEEDED",
                    "message": f"Rate limit reached. Please retry after {result.retry_after} seconds.",
                },
            )

    return dependency


# Usage example
@router.post(
    "/api/generate/image",
    dependencies=[Depends(require_rate_limit(RateLimitPreset.GENERATION))],
)
async def generate_image(input_data: GenerateInput):
    # Generation processing...
    ...
```

---

## 6. Protecting Authentication Endpoints

```python
# routes/auth.py
from fastapi import APIRouter, Request
from lib.api.rate_limit import check_rate_limit, RateLimitPreset
from lib.api.client_ip import get_client_ip

import structlog

logger = structlog.get_logger()
router = APIRouter()


@router.post("/api/auth/login")
async def login(request: Request):
    ip = get_client_ip(request)

    # Rate limit login attempts
    rate_limit_result = check_rate_limit(f"login:{ip}", RateLimitPreset.AUTH)

    if not rate_limit_result.allowed:
        logger.warning("login_rate_limit_exceeded", ip=ip)

        return JSONResponse(
            status_code=429,
            content={
                "error": {
                    "code": "TOO_MANY_ATTEMPTS",
                    "message": "Too many login attempts",
                },
            },
        )

    # Login processing...
```

---

## 7. Scaling: Migration to Redis

### 7.1 Migration Criteria

| Metric | Continue with memory store | Consider Redis |
|------|-----------------|-----------|
| Number of instances | 1 | 2+ |
| Requests/second | < 100 | > 100 |
| Accuracy requirements | Approximate is acceptable | Strict accuracy needed |

### 7.2 Redis Implementation Example

```python
# lib/api/rate_limit_redis.py
import redis.asyncio as redis
import os
import time


_redis_client: redis.Redis | None = None


def get_redis_client() -> redis.Redis:
    global _redis_client
    if _redis_client is None:
        _redis_client = redis.from_url(os.environ["REDIS_URL"])
    return _redis_client


async def check_rate_limit_redis(
    identifier: str,
    config: RateLimitConfig,
) -> RateLimitResult:
    """Rate limit check using Redis sliding window."""
    client = get_redis_client()
    key = f"ratelimit:{identifier}:{config.limit}:{config.window_seconds}"
    now = time.time()
    window_start = now - config.window_seconds

    pipe = client.pipeline()
    # Remove old entries
    pipe.zremrangebyscore(key, 0, window_start)
    # Count current entries
    pipe.zcard(key)
    # Add current request
    pipe.zadd(key, {str(now): now})
    # Set expiry
    pipe.expire(key, int(config.window_seconds) + 1)

    results = await pipe.execute()
    current_count = results[1]

    allowed = current_count < config.limit

    return RateLimitResult(
        allowed=allowed,
        limit=config.limit,
        remaining=max(0, config.limit - current_count - (1 if allowed else 0)),
        reset_time=int(now + config.window_seconds),
        retry_after=None if allowed else int(config.window_seconds),
    )
```

### 7.3 Environment Variables

```env
# Redis (optional)
REDIS_URL=redis://localhost:6379/0
```

---

## 8. Testing

```python
# tests/test_rate_limit.py
import pytest
from lib.api.rate_limit import check_rate_limit, RateLimitConfig, _store


@pytest.fixture(autouse=True)
def clear_store():
    _store.clear()
    yield
    _store.clear()


def test_allows_requests_within_limit():
    result = check_rate_limit("test-user", RateLimitConfig(limit=5, window_seconds=60))
    assert result.allowed is True
    assert result.remaining == 4


def test_rejects_requests_exceeding_limit():
    config = RateLimitConfig(limit=5, window_seconds=60)
    for _ in range(5):
        check_rate_limit("test-user", config)

    result = check_rate_limit("test-user", config)
    assert result.allowed is False
    assert result.remaining == 0
    assert result.retry_after is not None
    assert result.retry_after > 0


def test_counts_independently_for_different_users():
    config = RateLimitConfig(limit=5, window_seconds=60)
    for _ in range(5):
        check_rate_limit("user-a", config)

    result = check_rate_limit("user-b", config)
    assert result.allowed is True
```

---

## 9. Monitoring and Alerting

```python
# Log rate limit exceeded events
import structlog

logger = structlog.get_logger()

if not rate_limit_result.allowed:
    logger.warning(
        "rate_limit_exceeded",
        identifier=identifier,
        preset=preset,
        remaining=rate_limit_result.remaining,
        reset_time=rate_limit_result.reset_time,
    )

# Metrics (optional)
# Prometheus counters or custom aggregation
```

---

## 10. Alternative: slowapi

For projects preferring a third-party solution, **slowapi** provides a FastAPI-native rate limiting integration:

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@router.post("/api/generate")
@limiter.limit("30/hour")
async def generate(request: Request):
    ...
```

Use slowapi when you want convention-over-configuration and are comfortable with decorator-based limits.

---

## 11. Summary

| Pattern | Usage |
|---------|------|
| Memory store | Development environment, single instance |
| Redis | Production environment, multiple instances |
| slowapi | Convention-over-configuration alternative |
| Presets | Default settings by use case |
| HTTP headers | Information provision to clients |

| Preset | Limit | Use Case |
|-----------|------|-------------|
| auth | 10/min | Login, password reset |
| generation | 30/hour | AI generation |
| api | 100/min | General API |
| webhook | 50/min | Webhook reception |
| strict | 5/min | Critical operations |
