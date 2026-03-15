# Middleware Guidelines

# 1. Middleware Stack

## 1.1 Recommended Order
Middleware is executed in reverse order of registration (last registered = outermost). Register in this order:

```python
# app/main.py
app = FastAPI()

# 1. CORS (outermost — must handle preflight before anything else)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 2. Request ID (early — needed by all subsequent middleware and handlers)
app.add_middleware(RequestIDMiddleware)

# 3. Logging (after request ID is set)
app.add_middleware(LoggingMiddleware)

# 4. Rate limiting (before business logic)
app.add_middleware(RateLimitMiddleware)
```

## 1.2 Custom Middleware Pattern
* MUST use the `BaseHTTPMiddleware` pattern or pure ASGI middleware
* MUST NOT perform database queries in middleware (use dependencies instead)
* MUST handle exceptions gracefully — middleware errors should not crash the app

```python
from starlette.middleware.base import BaseHTTPMiddleware

class RequestIDMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid4()))
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
```

# 2. CORS

* MUST configure CORS origins from environment variables (never hardcode)
* MUST restrict `allow_origins` in production (never use `["*"]` in production)
* MUST allow credentials only when necessary
* See `common/cors.md` for general CORS principles

# 3. Logging Middleware

* MUST log request method, path, status code, and duration for every request
* MUST include request ID in all log entries
* MUST NOT log request/response bodies (may contain PII)
* See `common/logging.md` for general logging principles

```python
class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration = time.perf_counter() - start
        logger.info(
            "request_completed",
            method=request.method,
            path=request.url.path,
            status=response.status_code,
            duration_ms=round(duration * 1000, 2),
            request_id=getattr(request.state, "request_id", None),
        )
        return response
```
