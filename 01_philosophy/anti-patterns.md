$NOTE

# Anti-Patterns

This defines repeatedly observed "do not do this" patterns.
Each anti-pattern includes why it is problematic and what should be done instead.

---

## 1. Abandoning Type Safety

### Careless Use of `Any`

```python
# BAD: Creates a hole in the type system
data: Any = await fetch_data()
data.non_existent.property  # Explodes at runtime

# GOOD: Use unknown-style patterns with runtime checks
data: object = await fetch_data()
if is_valid_response(data):
    assert isinstance(data, ResponseModel)
    data.property  # Type-safe

# BEST: Use Pydantic for parsing
data = ResponseModel.model_validate(await fetch_data())
data.property  # Type-safe
```

**Why it's problematic**: `Any` disables type checking. A single `Any` propagates through type inference — when `Any` is assigned to a variable, every expression derived from it also becomes `Any`, silently disabling checks across the entire call chain without static analysis warnings.
**Exception**: Inadequate type stubs in external libraries. Always attach a `# FIXME` comment.

---

## 2. Architectural Deviation

### Direct DB Access from Route Handlers

```python
# BAD: Skipping the service layer
@router.get("/users/{user_id}")
async def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    return user

# GOOD: Go through the service layer
@router.get("/users/{user_id}")
async def get_user(user_id: int, service: UserService = Depends(get_user_service)):
    return await service.get_user(user_id)
```

**Why it's problematic**: Error handling, authentication, authorization, caching, and logging end up implemented inconsistently.

### Introducing Global Mutable State

```python
# BAD: Module-level mutable state
_cache: dict[str, Any] = {}
_current_user: User | None = None

# GOOD: Use dependency injection + proper scoping
class UserService:
    def __init__(self, repo: UserRepository, cache: CacheBackend) -> None:
        self._repo = repo
        self._cache = cache
```

**Why it's problematic**: Global mutable state obscures the origin of data, makes testing difficult, and causes subtle concurrency bugs in async applications.

---

## 3. Lack of Observability

### Missing Trace ID

```python
# BAD: Cannot identify which request caused the error
logger.error("Something went wrong")

# GOOD: Traceable via Request ID
logger.error(
    "Payment processing failed",
    extra={"request_id": request_id, "user_id": user_id, "action": "process_payment"},
)
```

**Why it's problematic**: Without it, requests cannot be traced end-to-end during incident investigation.

---

## 4. Over-Abstraction

### Generalizing Code Used in Only One Place

```python
# BAD: Abstraction with no reuse potential
def _single_purpose_helper() -> None: ...
def create_single_use_factory(**options: Any) -> Any: ...

# GOOD: Write it inline. Extract only after duplication appears in 3 places
```

**Why it's problematic**: Abstraction increases comprehension cost. An abstraction used in only one place leaves only the cost.

### Bloated Configuration Dicts

```python
# BAD: Too many options = the abstraction is heading in the wrong direction
action = create_action(
    schema=schema,
    handler=handler,
    middleware=middleware,
    error_handler=error_handler,
    retry_policy=retry_policy,
    cache_policy=cache_policy,
    rate_limit_policy=rate_limit_policy,
    audit_policy=audit_policy,
)

# GOOD: Extract only the common parts; implement special cases individually
base_action = create_base_action(common_options)

async def special_action(input_data: InputModel) -> OutputModel:
    # Special logic
    transformed = transform(input_data)
    return await base_action(transformed)
```

**Why it's problematic**: A massive configuration object is not "flexible" — it is "complex."

---

## 5. Misuse of Naming and Comments

### Vague Naming

```python
# BAD
data = await get_data()
result = process(data)
def handle_click() -> None: ...

# GOOD
user_profile = await fetch_user_profile()
validated_order = validate_order(raw_order)
def handle_submit_payment() -> None: ...
```

**Why it's problematic**: `data`, `result`, `handle` convey nothing. The reader must read the implementation to understand the intent.

### Comments That Explain Logic

```python
# BAD: Stating what the code already says
# Check if user's age is 18 or above
if user.age >= 18:
    ...

# GOOD: Document intent, exceptions, and side effects
# Legal requirement: minors cannot use the payment feature (see compliance guidelines)
if user.age >= 18:
    ...
```

**Why it's problematic**: "What it does" is told by the code. Comments should tell "why it does it that way."

---

## 6. Inadequate Testing

### Testing Only the Happy Path

```python
# BAD: Only the success case
def test_creates_user():
    user = create_user(valid_input)
    assert user is not None

# GOOD: Include error cases, boundary values, and security scenarios
def test_rejects_invalid_email(): ...
def test_prevents_idor_access(): ...
def test_handles_duplicate_webhook_events(): ...
def test_fails_gracefully_on_external_api_timeout(): ...
```

**Why it's problematic**: Bugs occur in error cases and at boundary values, not in the happy path.

---

## 7. Ignoring Performance

### Unnecessary Heavy Dependencies

```python
# BAD: Importing heavy libraries for trivial tasks
import pandas as pd  # Just to read a small CSV
import requests  # When httpx is already in the project

# GOOD: Use lightweight alternatives or stdlib
import csv  # For simple CSV operations
import httpx  # Consistent with the rest of the project, supports async
```

**Why it's problematic**: Excessive dependencies increase install time, attack surface, and deployment size. Each dependency is a maintenance liability — it can introduce breaking changes, security vulnerabilities, or license conflicts. Prefer stdlib or existing project dependencies when they can accomplish the task.
