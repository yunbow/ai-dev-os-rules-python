$NOTE

# Error Strategy Decision Criteria

Defines error handling policies, retry decisions, and how to communicate errors to users.

---

## Fundamental Principle

> *"Don't try to prevent all failures — detect, contain, and communicate them."*

---

## 1. Error Classification and Response Policies

| Error Type | Characteristics | Log Level | Display to User | Retry |
|----------|------|----------|-------------|---------|
| **System** | DB failure, uncaught exception, external API down | FATAL / ERROR | Generic error message | Automatic (backoff) |
| **Application** | Validation failure, domain rule violation, authentication error | WARN / ERROR | Specific error code + message | Don't retry |
| **User Operation** | Input format mistake, double click | — (no logging needed) | Prevent at API validation level | Don't retry |

### Decision Flow

The boundary between System and Application errors: "Could the developer have predicted it?" means — is this error a known, enumerable case in the domain logic? If yes (e.g., "user not found", "insufficient balance"), it's an Application error that should be handled with a specific code path. If no (e.g., database connection dropped, OOM), it's a System error that gets generic handling.

```text
Error Occurs
  |
  +-- Is this a known, enumerable domain case?
  |    +-- No -> System Error (FATAL/ERROR, generic message, alert)
  |    +-- Yes |
  |
  +-- Can the user fix it?
  |    +-- Yes -> User Operation Error (prevent at validation, no logging needed)
  |    +-- No -> Application Error (specific message, WARN)
```

---

## 2. Retry Decisions

| Scenario | Auto-retry | Strategy |
|---------|------------|------|
| Network failure / offline -> recovery | Yes | Exponential backoff (1s -> 2s -> 4s) |
| Temporary 500 / 503 | Yes | Exponential backoff, max N attempts |
| Validation error (400 / 422) | No | Return field errors to the user |
| Authentication error (401) | No | Redirect to login / re-authenticate |
| Authorization error (403) / Not Found (404) | No | Display error, no retry needed |
| Conflict (409) | No | Notify user of the conflict |
| Rate limit (429) | Conditional | Follow the Retry-After header |

### Retry Implementation Criteria

```text
Should we retry?
  |
  +-- Is the status code 5xx?
  |    +-- Yes -> Retry (max 3 times, exponential backoff)
  |    +-- No |
  |
  +-- Is it a network error (timeout, connection refused)?
  |    +-- Yes -> Retry (max 3 times, exponential backoff)
  |    +-- No -> Don't retry (return result to the caller)
```

**Why max 3 retries**: 3 attempts with exponential backoff (1s -> 2s -> 4s = 7s total) covers most transient failures (network blips, brief deployments) without making the user wait unreasonably. Beyond 3 retries, if the service is still failing, it's likely a sustained outage that retrying won't fix.

**Why exponential backoff starting at 1s**: Linear retry hammers a recovering service. Doubling the interval (1s -> 2s -> 4s) gives the failing service progressively more recovery time. Starting at 1s (not 100ms) avoids contributing to thundering herd problems while still being responsive for quick recoveries.

---

## 3. User-Facing Error Display Decisions

| Error Content | Display Method | Display Content |
|------------|---------|---------|
| Non-critical operation failure | Error response with details | Concise error message in response body |
| Error affecting the entire request | Exception handler | Structured error response |
| Authentication expired | 401 response | Prompt to re-authenticate |

### What Must NOT Be Displayed

- Stack traces
- DB query details
- Internal error codes (SQLAlchemy error codes, etc.)
- Environment variables / secrets

---

## 4. Non-Critical Feature Failure Decisions

| Feature Importance | Behavior on Failure |
|------------|--------------|
| Critical (authentication, payment) | Show error explicitly, abort processing |
| Important (data saving, generation) | Retry -> return error on failure |
| Auxiliary (analytics, suggestions) | Fail silently, log warning, continue |
| Decorative (non-essential enrichments) | Ignore, degrade gracefully |

**Principle**: Design so that auxiliary feature failures don't take down critical features.

---

## 5. External API Error Decisions

| External API Response | Action |
|---------------|------|
| Normal response (200) | Continue processing |
| Rate limit (429) | Wait according to Retry-After |
| Temporary failure (500, 503) | Retry with exponential backoff |
| Authentication error (401, 403) | Refresh key/token, alert |
| Timeout | Appropriate timeout value + retry |
| Specification change (unexpected response shape) | Detect with Contract Test, log + alert |

---

## 6. Exception Hierarchy Pattern

All application errors use a structured exception hierarchy:

```python
class AppError(Exception):
    """Base exception for all application errors."""
    def __init__(self, message: str, code: str, status_code: int = 500) -> None:
        self.message = message
        self.code = code
        self.status_code = status_code


class ValidationError(AppError):
    """Raised when input validation fails."""
    def __init__(self, message: str, field_errors: dict[str, list[str]] | None = None) -> None:
        super().__init__(message, code="VALIDATION_ERROR", status_code=422)
        self.field_errors = field_errors or {}


class NotFoundError(AppError):
    """Raised when a requested resource does not exist."""
    def __init__(self, resource: str, resource_id: str) -> None:
        super().__init__(f"{resource} not found: {resource_id}", code="NOT_FOUND", status_code=404)


class AuthorizationError(AppError):
    """Raised when the user lacks permission."""
    def __init__(self, message: str = "Insufficient permissions") -> None:
        super().__init__(message, code="FORBIDDEN", status_code=403)
```

### Why Use a Custom Exception Hierarchy Instead of Bare Exceptions

| Comparison | Bare `except Exception` | Custom Exception Hierarchy |
|---------|-----------------|------------------|
| Forced handling | Cannot distinguish error types | Catch specific exceptions at each layer |
| Type-safe errors | No structured data | Typed fields (code, status_code, field_errors) |
| Field error representation | Manual construction | Structured via `field_errors` dict |
| API integration | Manual conversion to HTTP responses | Exception handlers map directly to responses |

### Service Method Checklist

```text
- Use the service layer for all business logic
- Authentication check via dependency injection
- Resource ownership check (IDOR prevention)
- Input validation with Pydantic schema
- Raise specific AppError subclasses on failure
- Return typed response models on success
```
