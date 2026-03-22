# Logging Guidelines

This document outlines the **monitoring / logging / tracing (observability) strategy** for Python applications. Deployment target: **Cloud platform (e.g., AWS, GCP, or self-hosted)**

---

## Purpose of Observability

Observability in this project means the ability to answer "why is this request slow/failing?" within 5 minutes using logs, traces, and metrics. Specifically:

* Early detection and automated alerting for incidents
* Identify performance degradation trends
* Rapidly pinpoint the source of API, DB, and external API issues
* Safely visualize production environment behavior
* Monitor user impact in real-time

Build a system capable of observing performance issues that arise in
FastAPI (async endpoints / background tasks / middleware).

---

## 1. Monitoring

### Application Monitoring

**Use a monitoring platform appropriate for your deployment target.**

* **Prometheus + Grafana**
  * Request latency (p50 / p95 / p99)
  * Error rates by endpoint
  * Request throughput
* **Cloud-native monitoring** (e.g., CloudWatch, Cloud Monitoring)
  * CPU / memory utilization
  * Container health checks
  * Auto-scaling triggers

**Monitoring targets:**

* FastAPI route handlers
* Background tasks (Celery / arq / asyncio tasks)
* Database query performance
* External API call latency

---

## 2. Logging

### Python: Server Log Standards

```python
import structlog

logger = structlog.get_logger()

logger.info("creating_user", email=email)
logger.warning("slow_response", duration_ms=ms)
logger.error("external_api_error", provider=provider, message=message)
```

**Classification:**

* info: Normal flow (metrics support)
* warning: Latency, retries, slow external API responses
* error: Handled failures

### Recommended Log Management Tools

| Purpose | Tool |
| --- | --- |
| Error monitoring | Sentry |
| Request logs / JSON structured logs | Datadog / Grafana Loki / ELK Stack |

---

## 3. Tracing

### Purpose of Distributed Tracing

* Identify the source of latency from API → DB → external API
* Track by specific user request
* Flexible alert configuration

### Trace ID / Request ID Propagation Strategy

Trace ID propagation is what enables correlating a user-visible error with the exact server-side log entry, DB query, and external API call that caused it. Without it, debugging production issues requires manually searching logs by timestamp — which is slow and error-prone at scale.

* Attach Trace IDs to all API endpoints / middleware / SQLAlchemy queries / external API calls
* Insufficient propagation makes it difficult to correlate logs and traces, hindering rapid problem identification

### 3.1 Trace ID Generation and Propagation (Implementation Examples)

#### Logger Factory

```python
# src/lib/logger.py
import os
import structlog


def configure_logging() -> None:
    """Configure structlog for JSON structured logging."""
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            int(os.environ.get("LOG_LEVEL", "20"))  # 20 = INFO
        ),
    )


def get_logger(**initial_context) -> structlog.BoundLogger:
    """Create a logger with initial context."""
    return structlog.get_logger(**initial_context)


# Logger for service layer
def create_service_logger(service_name: str) -> structlog.BoundLogger:
    return get_logger(service="service", name=service_name)


# Logger for API routes
def create_route_logger(route_name: str) -> structlog.BoundLogger:
    return get_logger(service="api-route", route=route_name)


# Logger with request context
def create_request_logger(
    base: structlog.BoundLogger,
    request_id: str,
    user_id: str | None = None,
) -> structlog.BoundLogger:
    return base.bind(request_id=request_id, user_id=user_id)
```

#### Request ID Middleware

```python
# src/middleware/request_id.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response
import structlog


class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        request_id = request.headers.get("x-request-id") or str(uuid.uuid4())

        # Bind to structlog context vars for automatic propagation
        structlog.contextvars.clear_contextvars()
        structlog.contextvars.bind_contextvars(request_id=request_id)

        response = await call_next(request)
        response.headers["x-request-id"] = request_id

        return response
```

#### Log Usage in Service Layer

```python
# src/features/project/services/project_service.py
from lib.logger import create_service_logger, create_request_logger

logger = create_service_logger("project-service")


async def create_project(
    title: str | None, user_id: str, request_id: str, db: AsyncSession
) -> dict:
    req_logger = create_request_logger(logger, request_id)

    req_logger.info("creating_project", title=title)

    try:
        session = await require_auth()

        project = Project(user_id=user_id, title=title)
        db.add(project)
        await db.flush()

        req_logger.info("project_created", project_id=project.id)

        return {"project_id": project.id}
    except Exception:
        req_logger.exception("failed_to_create_project")
        raise
```

#### Logging External API Calls

```python
# src/lib/api/external_api_client.py
import time
import httpx
import structlog


async def call_external_api(
    url: str,
    method: str = "GET",
    request_id: str = "",
    logger: structlog.BoundLogger | None = None,
    **kwargs,
) -> dict:
    logger = logger or structlog.get_logger()
    start_time = time.monotonic()

    logger.info("external_api_call_started", url=url, request_id=request_id)

    try:
        async with httpx.AsyncClient() as client:
            response = await client.request(
                method,
                url,
                headers={"x-request-id": request_id, **kwargs.pop("headers", {})},
                **kwargs,
            )

        duration_ms = (time.monotonic() - start_time) * 1000

        if response.status_code >= 400:
            logger.error(
                "external_api_call_failed",
                url=url,
                status=response.status_code,
                duration_ms=duration_ms,
                request_id=request_id,
            )
            raise ExternalApiError(response.status_code)

        logger.info(
            "external_api_call_completed",
            url=url,
            status=response.status_code,
            duration_ms=duration_ms,
            request_id=request_id,
        )

        return response.json()

    except httpx.HTTPError as exc:
        duration_ms = (time.monotonic() - start_time) * 1000
        logger.error(
            "external_api_call_error",
            url=url,
            error=str(exc),
            duration_ms=duration_ms,
            request_id=request_id,
        )
        raise
```

### 3.2 Structured Log Format Standard

#### Recommended Fields

| Field | Description | Example |
|-----------|------|-----|
| `user_id` | User ID | "user_xyz" |
| `action` | Action name | "create_project" |
| `duration_ms` | Processing time | 150 |
| `error` | Error information | { code, message, stack } |

#### Log Output Examples

```json
{
  "level": "info",
  "service": "service",
  "name": "project-service",
  "request_id": "req_abc123",
  "user_id": "user_xyz",
  "timestamp": "2024-01-15T10:30:00.000000Z",
  "event": "project_created",
  "project_id": "proj_456",
  "duration_ms": 150
}
```

```json
{
  "level": "error",
  "service": "api-route",
  "route": "/api/resource",
  "request_id": "req_def456",
  "user_id": "user_xyz",
  "timestamp": "2024-01-15T10:31:00.000000Z",
  "event": "external_api_call_failed",
  "error": {
    "code": "RATE_LIMITED",
    "message": "API rate limit exceeded",
    "retry_after": 60
  }
}
```

### 3.3 External API Call Logs

Record calls to external services such as AI APIs and payment APIs in a dedicated log format.

```python
# lib/api/external_api_logger.py
from dataclasses import dataclass
import structlog


@dataclass
class ExternalApiLogParams:
    provider: str  # e.g., "ai-api", "payment", "vision"
    operation: str
    latency_ms: float
    status: str  # "success" or "error"
    # API-specific metrics
    prompt_tokens: int | None = None
    response_tokens: int | None = None
    model: str | None = None
    error_code: str | None = None


def log_external_api_call(
    logger: structlog.BoundLogger,
    params: ExternalApiLogParams,
    request_id: str | None = None,
    user_id: str | None = None,
) -> None:
    log_data = {
        "provider": params.provider,
        "operation": params.operation,
        "latency_ms": params.latency_ms,
        "status": params.status,
        "type": "external_api_call",
        "request_id": request_id,
        "user_id": user_id,
    }

    # Add optional fields
    if params.prompt_tokens is not None:
        log_data["prompt_tokens"] = params.prompt_tokens
    if params.response_tokens is not None:
        log_data["response_tokens"] = params.response_tokens
    if params.model:
        log_data["model"] = params.model
    if params.error_code:
        log_data["error_code"] = params.error_code

    event = f"{params.provider}:{params.operation}"
    if params.status == "success":
        logger.info(f"{event}_completed", **log_data)
    else:
        logger.error(f"{event}_failed", **log_data)
```

#### Usage Example

```python
start_time = time.monotonic()

try:
    result = await ai_client.generate_text(prompt)

    log_external_api_call(
        logger,
        ExternalApiLogParams(
            provider="ai-api",
            operation="generate_text",
            latency_ms=(time.monotonic() - start_time) * 1000,
            status="success",
            model="model-name",
            prompt_tokens=result.usage.prompt_tokens,
            response_tokens=result.usage.completion_tokens,
        ),
        request_id=request_id,
        user_id=user_id,
    )

    return result
except Exception as exc:
    log_external_api_call(
        logger,
        ExternalApiLogParams(
            provider="ai-api",
            operation="generate_text",
            latency_ms=(time.monotonic() - start_time) * 1000,
            status="error",
            error_code=getattr(exc, "code", None),
        ),
        request_id=request_id,
        user_id=user_id,
    )

    raise
```

### 3.4 Fire-and-Forget Log Pattern

For cases where you want to avoid blocking API calls, record logs asynchronously.

```python
# lib/api/api_call_file_logger.py
import asyncio


def log_api_call_async(params: dict, context: dict) -> None:
    """Fire-and-forget log recording. Does not block API responses."""
    try:
        loop = asyncio.get_running_loop()
        loop.create_task(_write_log(params, context))
    except RuntimeError:
        # No running loop, write synchronously
        import threading
        threading.Thread(
            target=_write_log_sync, args=(params, context), daemon=True
        ).start()


async def _write_log(params: dict, context: dict) -> None:
    try:
        await write_log_to_file(params, context)
    except Exception:
        logger.exception("failed_to_write_api_call_log")


# Usage example
response = await generate_text(prompt)

# Non-blocking log recording
log_api_call_async(
    {"prompt": prompt, "response": response.text, "tokens": response.usage},
    {"request_id": request_id, "user_id": user_id},
)

return response  # Return without waiting for log completion
```

#### JSONL File Logging (For Development)

```python
# Detailed logging for development environment debugging
import json
import os
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from pathlib import Path

LOG_DIR = Path(os.getcwd()) / "logs" / "api-calls"


@dataclass
class ApiCallLog:
    timestamp: str
    request_id: str
    provider: str
    operation: str
    prompt: str
    response: str
    latency_ms: float
    tokens: dict | None = None


async def write_log_to_file(log: ApiCallLog) -> None:
    if os.environ.get("ENVIRONMENT") != "development":
        return

    filename = datetime.now(timezone.utc).strftime("%Y-%m-%d") + ".jsonl"
    filepath = LOG_DIR / filename

    LOG_DIR.mkdir(parents=True, exist_ok=True)

    with open(filepath, "a") as f:
        f.write(json.dumps(asdict(log)) + "\n")
```

### 3.5 Performance Measurement Pattern

```python
# Performance measurement utility
import time
import structlog
from typing import TypeVar, Callable, Awaitable

T = TypeVar("T")


async def with_performance_log(
    logger: structlog.BoundLogger,
    operation: str,
    fn: Callable[[], Awaitable[T]],
) -> T:
    start_time = time.monotonic()

    try:
        result = await fn()
        duration_ms = (time.monotonic() - start_time) * 1000

        logger.info(f"{operation}_completed", operation=operation, duration_ms=duration_ms)

        # Warn when threshold is exceeded
        if duration_ms > 1000:
            logger.warning(
                f"{operation}_exceeded_threshold",
                operation=operation,
                duration_ms=duration_ms,
            )

        return result
    except Exception:
        duration_ms = (time.monotonic() - start_time) * 1000
        logger.error(f"{operation}_failed", operation=operation, duration_ms=duration_ms)
        raise


# Usage example
project = await with_performance_log(
    req_logger,
    "create_project",
    lambda: db.execute(insert(Project).values(**data)),
)
```

### Python × OpenTelemetry (OTEL)

**Tracing targets:**

* FastAPI route handlers / middleware / background tasks
* DB access (SQLAlchemy event hooks)
* Payment API / AI API / RSS / External APIs

**Tracing architecture example:**

```text
User Request
  → FastAPI Route Handler
    → Service Layer
    → SQLAlchemy (DB)
    → External API (Payment / AI, etc.)
    → Export to Sentry / Datadog / Jaeger
```

**Export destinations:**

| Purpose | Recommended Service |
| --- | --- |
| Error tracking & tracing | Sentry |
| Server-centric distributed tracing | Datadog / Jaeger |

---

## 4. Alert Configuration

### Application-Level Alerts

* Error rate increase (per endpoint)
* Response time threshold exceeded (p95 > 300ms)
* Background task failure rate
* Database connection pool exhaustion

### Sentry

* Exception occurrence
* Performance anomalies (Transactions)
* Version regression detection

---

## 5. SLO/SLA Definition and Metrics Mapping

* **Clarify what constitutes normal operation**
* Clarify the basis for incident detection and alerting
* Example: Define API latency 95th percentile < 300ms as SLO, error rate < 1% as SLA

---

## 6. Cost Optimization Strategy

* Log volume and trace volume for Datadog / Sentry / Grafana Loki can lead to cost escalation
* **Mitigations:**

  * Sampling configuration (e.g., trace 10% of requests)
  * Clear log retention periods
  * Suppress unnecessary detailed logs

---

## 7. Observability Standardization Guidelines

### Logging

* JSON structured logs (via structlog)
* request_id / trace_id required
* Error logs include traceback

### Tracing

* Attach Trace ID to all APIs
* Track DB queries via SQLAlchemy event hooks
* External APIs (payment services / AI APIs, etc.) also generate spans

### Monitoring

* Continuous health check endpoints
* Application metrics via Prometheus client
* Preserve performance comparisons for each deployment

---

## 8. PII Redaction

Use structlog processors to automatically redact sensitive information.

```python
# src/lib/logger.py
import re

REDACT_PATTERNS = {
    "password", "secret", "token", "access_token",
    "refresh_token", "credit_card", "authorization",
}


def redact_sensitive_fields(
    logger: structlog.BoundLogger, method_name: str, event_dict: dict
) -> dict:
    """Structlog processor that redacts sensitive fields."""
    for key in list(event_dict.keys()):
        if key.lower() in REDACT_PATTERNS:
            event_dict[key] = "[REDACTED]"
        elif isinstance(event_dict[key], dict):
            for nested_key in list(event_dict[key].keys()):
                if nested_key.lower() in REDACT_PATTERNS:
                    event_dict[key][nested_key] = "[REDACTED]"
    return event_dict
```

**Rules:**

* Use `logger.error()` instead of `print()` or bare `traceback.print_exc()`
* `print(os.environ["SECRET"])` is prohibited
* Error tracebacks are server logs only (never return to client)

---

## 9. Log Retention Policy

| Log Type | Retention Period | Storage |
|---------|---------|--------|
| Application logs | 30 days | Log aggregation service |
| API call logs (DB) | 90 days | `api_call_logs` table (auto-deleted via scheduled task) |
| Audit logs | 1 year | `audit_logs` table |
| Security logs (auth failures, etc.) | 90 days | Log service + DB |
| Webhook event logs | 90 days | `webhook_events` table |

---

## 10. Scheduled Task Failure Alerts

Send alerts on scheduled task failures.

```python
# src/lib/alerts.py
async def send_task_failure_alert(
    task_name: str,
    error: Exception,
) -> None:
    """Send failure notification to admin email."""
    ...
```

**Usage pattern:**

```python
# src/tasks/cleanup.py
try:
    # ... task processing ...
    pass
except Exception as error:
    await send_task_failure_alert("cleanup", error)
    raise
```

All scheduled tasks (cleanup, daily-report, process-jobs, etc.) should uniformly send alerts in the except block.

---

## Summary

* **structlog**: JSON structured logging with context propagation
* **Sentry / OTEL**: Error & tracing foundation
* **Logs are unified in JSON format**
* **Trace from API → Service → DB → External API** to pinpoint issues
* Trace ID / Request ID propagation is the key to observability
* SLO/SLA-based alert configuration and cost optimization should also be considered
