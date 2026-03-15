# Error Handling Guidelines
This document covers **only content that does not overlap with other documents**, focusing on error design, notification, user-facing experience, log levels, and error classification.

It also **systematizes error classification and handling methods**, defining what messages to display and how to log for each error type: validation errors, authentication errors, system errors, etc.

---

# 1. Error Taxonomy
### 1.1 System Errors (Server Internal)
* DB connection failures
* Uncaught exceptions
* Runtime exceptions
* External API outages

**Characteristics:**
Unpredictable / unrecoverable → Notify administrators + monitor
**Logging:** Record at FATAL level, do not include sensitive information

---

### 1.2 Application Errors (Expected Errors)
* Validation errors
* Domain logic errors (e.g., 409 Conflict)
* Authorization errors (insufficient permissions)
* Rate limit reached

**Characteristics:**
Predictable / handling required → Provide clear feedback to users
**Logging:** Record at ERROR or WARN level

---

### 1.3 User Operation Errors (User Mistakes)
* Missing input
* Unexpected operations (duplicate submissions, etc.)
* **Input format errors preventable on the client side** — specifically:
  * Errors caught by Pydantic schema validation
  * Errors prevented by API input constraints (query parameter types, path parameter validation)
  * Criteria: if the client can validate before submission, it qualifies

**Characteristics:**
Return clear error responses and guide retry
Cases where the server returns error codes (e.g., 409 Conflict) are reclassified as 1.2

---

### 1.4 Error Propagation Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                        Error Source                               │
├──────────────────────────────────────────────────────────────────┤
│  External API  │  SQLAlchemy DB  │  Pydantic       │  Domain     │
│  (Integration) │  (Query)        │  Validation     │  Logic      │
│                │                 │  (Input)        │  (Business  │
│                │                 │                 │   Rules)    │
└─────┬──────────┴───────┬────────┴──────┬──────────┴──────┬───────┘
      │                  │               │                 │
      ▼                  ▼               ▼                 ▼
┌──────────────────────────────────────────────────────────────────┐
│              Exception Handler / Error Classifier                │
│              Error Classification & Custom Exception Mapping     │
│              → See frameworks/fastapi/error_handlers.py          │
└───────────────────────────────┬──────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│                    HTTP Error Response                            │
│         { "error": { "code": "...", "message": "..." } }        │
│              → frameworks/fastapi/error_handlers.py              │
└──────────────────────────────────────────────────────────────────┘
```

**Related Documents:**

- External API error classification → Refer to project-specific integration guidelines
- Exception hierarchy → common/exceptions.py
- FastAPI error handlers → frameworks/fastapi/error_handlers.py
- Logging → common/logging.md

---

# 2. Exception Hierarchy

### 2.1 Base Exception Classes

```python
# lib/exceptions.py

class AppError(Exception):
    """Base exception for all application errors."""
    def __init__(self, code: str, message: str, status_code: int = 500):
        self.code = code
        self.message = message
        self.status_code = status_code
        super().__init__(message)


class ValidationError(AppError):
    """Input validation failed."""
    def __init__(self, message: str = "Validation failed", details: list | None = None):
        self.details = details or []
        super().__init__(code="VALIDATION_ERROR", message=message, status_code=400)


class UnauthorizedError(AppError):
    """Authentication required."""
    def __init__(self, message: str = "Authentication required"):
        super().__init__(code="UNAUTHORIZED", message=message, status_code=401)


class ForbiddenError(AppError):
    """Insufficient permissions."""
    def __init__(self, message: str = "Insufficient permissions"):
        super().__init__(code="FORBIDDEN", message=message, status_code=403)


class NotFoundError(AppError):
    """Resource not found."""
    def __init__(self, resource: str = "Resource"):
        super().__init__(
            code="NOT_FOUND",
            message=f"{resource} not found",
            status_code=404,
        )


class ConflictError(AppError):
    """Resource conflict (e.g., duplicate)."""
    def __init__(self, message: str = "Resource conflict"):
        super().__init__(code="CONFLICT", message=message, status_code=409)


class RateLimitError(AppError):
    """Rate limit exceeded."""
    def __init__(self, retry_after: int | None = None):
        self.retry_after = retry_after
        super().__init__(
            code="RATE_LIMIT_EXCEEDED",
            message="Too many requests",
            status_code=429,
        )


class DomainError(AppError):
    """Business rule violation."""
    def __init__(self, code: str, message: str):
        super().__init__(code=code, message=message, status_code=400)
```

### 2.2 FastAPI Exception Handlers

```python
# lib/error_handlers.py
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import ValidationError as PydanticValidationError


def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
        return JSONResponse(
            status_code=exc.status_code,
            content={"error": {"code": exc.code, "message": exc.message}},
        )

    @app.exception_handler(PydanticValidationError)
    async def pydantic_error_handler(
        request: Request, exc: PydanticValidationError
    ) -> JSONResponse:
        return JSONResponse(
            status_code=422,
            content={
                "error": {
                    "code": "VALIDATION_ERROR",
                    "message": "Input validation failed",
                    "details": exc.errors(),
                }
            },
        )

    @app.exception_handler(Exception)
    async def unhandled_error_handler(request: Request, exc: Exception) -> JSONResponse:
        logger.exception("Unhandled exception", exc_info=exc)
        return JSONResponse(
            status_code=500,
            content={"error": {"code": "INTERNAL_ERROR", "message": "An internal error occurred"}},
        )
```

---

# 3. API Error Policy (FastAPI)
### 3.1 Unified Response Format
```json
{
  "error": {
    "code": "INVALID_INPUT",
    "message": "The input is invalid."
  }
}
```

### 3.2 HTTP Status Code Rules

| Condition | Status | Example |
| --- | --- | --- |
| Success | 200, 201 | Create, update succeeded |
| User operation error | 400 | Validation failed |
| Authentication | 401 | Login required |
| Authorization | 403 | Insufficient permissions |
| Resource not found | 404 | Data does not exist |
| Validation (Pydantic) | 422 | Pydantic validation error |
| Conflict | 409 | Duplicate registration / domain logic violation |
| Temporary load / external service outage | 503 | Client should retry |
| Server | 500 | Internal error |

### 3.3 Indicate Retry Eligibility

* Specify via response headers or JSON fields

---

# 4. Log Levels and Recording Criteria (Non-overlapping with Observability)

### 4.1 Log Levels

| Level | Usage |
| --- | --- |
| DEBUG | Debugging only (minimize in production) |
| INFO | Normal operations (startup, completion, etc.) |
| WARNING | Expected exceptions, retryable issues |
| ERROR | Application errors |
| CRITICAL | System-stopping events (notification target) |

### 4.2 Do Not Include Sensitive Information in Logs

* Passwords / tokens / raw email addresses

---

# 5. Tracing ID on Error Occurrence

> **Reference:** See common/logging.md for Trace ID implementation details

### 5.1 Assign `request_id` per Request

* Facilitates user support inquiries
* Example:

```
x-request-id: abc123
```

### 5.2 Return request_id to the client

* "Please provide this ID when contacting support"
* Enables easy cross-referencing with logs
* Reference: Can be used alongside traceparent or x-b3-traceid used by OpenTelemetry
  → Reason for adopting x-request-id: simple and easy to trace across microservices

---

# 6. User-Facing Message Strategy

### 6.1 Error Message Attributes

Each user-facing error message should address three dimensions:

* **Specificity** — State what failed in user terms, not technical terms. Bad: "Validation error". Good: "The email address format is invalid."
* **Action suggestion** — Tell the user what to do next. Include a concrete action: "Please check the email format and try again" or "Please try again in a few minutes."
* **Impact scope** — Clarify what was affected. "Your changes were not saved" or "The image was uploaded but processing failed — it will appear once processing completes."

---

# 7. Retry Strategy

### 7.1 Cases for Automatic Retry

* Network failures (connectivity recovery)
* Temporary 500 / 503

### 7.2 Cases Where Retry Must Not Occur

* Validation errors
* 403 / 404
* Conflicts (409)

---

# 8. Handling Business Logic Errors (Domain Exceptions)

### 8.1 Handle Domain Rule Violations with Exception Classes

```python
class DomainError(AppError):
    """Business rule violation."""
    def __init__(self, code: str, message: str):
        super().__init__(code=code, message=message, status_code=400)


# Example usage
class InsufficientBalanceError(DomainError):
    def __init__(self):
        super().__init__(
            code="INSUFFICIENT_BALANCE",
            message="Account balance is insufficient for this operation",
        )
```

### 8.2 Convert Domain Errors to HTTP in the API

* `DomainError` → 400 or 409
* Handle as a separate layer from SQLAlchemy errors

---

# 9. Fail-Safe / Graceful Degradation

### 9.1 Non-Essential Feature Failures Should Not Break Core Functionality

* API should still respond even if optional enrichment services are down
* Return partial results with degradation indicators

### 9.2 Cache Fallback on External API Failure

* Local cache
* Use the most recent successful response

---

# 10. Debug / Developer-Facing Errors During Development

### 10.1 Dev Mode Displays Detailed Errors

* Suppressed in production (never expose stack traces)

### 10.2 Error Testing

* Enables testing error scenarios at the service/endpoint level
* Use pytest fixtures to simulate error conditions

---

# 11. QA / Testing (Non-overlapping Perspectives)

### 11.1 Error Scenario Tests Should Comprise 30-50%

* Success cases alone are insufficient

### 11.2 Chaos Testing (Limited Scope)

* Simulate API failures / latency
