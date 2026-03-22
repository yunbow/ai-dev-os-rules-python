# API Design Guidelines

## 1. Request / Response Models

## 1.1 Schema Design

* MUST use Pydantic `BaseModel` for all request and response schemas
* MUST separate request and response models (never reuse the ORM model as a response)
* MUST use descriptive names: `CreateUserRequest`, `UserResponse`, `UserListResponse`

```python
## schemas.py
class CreateUserRequest(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)

class UserResponse(BaseModel):
    id: int
    email: str
    name: str
    created_at: datetime

    model_config = ConfigDict(from_attributes=True)

class UserListResponse(BaseModel):
    items: list[UserResponse]
    total: int
    page: int
    per_page: int
```

## 1.2 Validation

* MUST validate at the API boundary using Pydantic models — do not validate manually in handlers
* MUST use `Field()` constraints for string lengths, numeric ranges, etc.
* MUST use `field_validator` or `model_validator` for cross-field validation
* SHOULD use custom types for common patterns: `EmailStr`, `HttpUrl`, `constr`

## 1.3 Response Envelope

* MUST use a consistent error response format across all endpoints:

```python
## core/exceptions.py
class ErrorResponse(BaseModel):
    error: str
    detail: str | None = None
    code: str | None = None

## Register exception handlers in main.py
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(error=exc.message, code=exc.code).model_dump(),
    )
```

## 2. Pagination

* MUST support pagination for all list endpoints
* MUST use query parameters: `page` (1-based) and `per_page` (default: 20, max: 100)
* MUST return total count in the response

```python
## core/pagination.py
class PaginationParams(BaseModel):
    page: int = Field(1, ge=1)
    per_page: int = Field(20, ge=1, le=100)

    @property
    def offset(self) -> int:
        return (self.page - 1) * self.per_page

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    per_page: int
```

## 3. Error Handling

## 3.1 Exception Hierarchy

* MUST define a base exception class for the application
* MUST NOT raise `HTTPException` from the service layer — only from routers or exception handlers
* MUST map domain exceptions to HTTP status codes in exception handlers

```python
## core/exceptions.py
class AppException(Exception):
    status_code: int = 500
    message: str = "Internal server error"
    code: str = "INTERNAL_ERROR"

class NotFoundError(AppException):
    status_code = 404
    code = "NOT_FOUND"

class ConflictError(AppException):
    status_code = 409
    code = "CONFLICT"

class ValidationError(AppException):
    status_code = 400
    code = "VALIDATION_ERROR"
```

## 3.2 Error Responses

* MUST return consistent JSON error responses (never plain text or HTML)
* MUST NOT expose internal details (stack traces, SQL queries) in production
* MUST include a machine-readable `code` field for client-side error handling
