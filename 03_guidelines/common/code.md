# Coding Standards & Static Analysis Guidelines

This guideline defines the **Python coding standards and static analysis tool policies** to be enforced across the entire project.

# 1. Key Principles
* **Prioritize readability and maintainability**
  Write code that communicates intent, rather than being concise or abbreviated.
* **Enforce type safety**
  Express intent through type annotations and enforce with mypy/pyright.
* **Ensure quality through static analysis**
  Enforce formatting and code quality through tooling, not manual effort.

# 2. Python Coding Standards

## 2.1 Type Annotation Placement
* Project type definitions **must be separated into dedicated modules**
* Use `dataclass` for data structures, `Protocol` for contracts expected to change

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass
class User:
    id: str
    name: str


class UserRepository(Protocol):
    async def find_by_id(self, id: str) -> User: ...
```

## 2.2 Handling Side Effects

* Functions with side effects must use naming conventions that signal mutation or I/O. Specifically:
  * **Prefix with a verb that implies external interaction**: `fetch`, `save`, `send`, `delete`, `upload`, `sync`, `notify`, `enqueue`
  * **Pure functions use computational verbs**: `calculate`, `format`, `parse`, `transform`, `validate`, `build`, `merge`
  * **Never use generic names** like `handle`, `process`, `do`, `run` for functions with side effects unless the name also includes the specific resource (e.g., `process_payment` is acceptable, `process_data` is not)
* Do not create ambiguous utility functions

```python
fetch_user()       # Has side effects - "fetch" signals network I/O
calculate_price()  # Pure function - "calculate" signals computation only
save_order()       # Has side effects - "save" signals persistence
build_query()      # Pure function - "build" signals construction
```

## 2.3 Use Enum Classes Properly

```python
from enum import Enum


class Theme(str, Enum):
    DARK = "dark"
    LIGHT = "light"
```

Use `str, Enum` (or `StrEnum` in Python 3.11+) so values are JSON-serializable. Use `Literal` types only when a simple union of strings suffices and no iteration or membership checks are needed.

# 3. Lint Standards

## 3.1 Rule Sets

Use **Ruff** as the unified linter and formatter:

```toml
# pyproject.toml
[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort (import ordering)
    "N",    # pep8-naming
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # flake8-simplify
    "TCH",  # flake8-type-checking
    "RUF",  # Ruff-specific rules
]
```

Rules to enforce:

| Rule | Reason |
| --- | --- |
| strict mypy (`--strict`) | Foundation of type safety |
| `disallow_any_explicit` (mypy) | Maintains type safety |
| `F841` (unused variables) | Reduces unnecessary code |
| `I` (isort) | Improves readability |
| `T201` (print statements) | Prevents leftover debug statements |

## 3.2 Unused Imports

Ruff's `F401` rule handles unused import detection and auto-removal:

```toml
[tool.ruff.lint]
select = ["F401"]
```

Auto-removal is required via `ruff check --fix`.

## 3.3 Import Order Rules

```
stdlib → third-party → first-party → local
```

Enforced by Ruff's `I` (isort) rule set.

# 4. Formatting

## 4.1 Configuration

```toml
# pyproject.toml
[tool.ruff.format]
quote-style = "double"
indent-style = "space"
line-ending = "auto"

[tool.ruff]
line-length = 100
```

Alternatively, use **Black** as the formatter:

```toml
# pyproject.toml
[tool.black]
line-length = 100
target-version = ["py312"]
```

| Setting | Value | Reason |
|------|-----|------|
| `line-length` | `100` | Ensures readability during code review |
| `quote-style` | `"double"` | Prioritizes consistency |
| `indent-style` | `"space"` (4 spaces) | PEP 8 standard indentation |

# 5. CI / Git Hooks Integration

Static analysis and formatting **must not rely on manual execution**.
Always automate the following:

* Lint & format checks on commit (via pre-commit hooks)
* CI as first line of defense on push / PR
* mypy type checking in CI

---

# 6. Comment Standards

* Do not use comments to explain logic
* Only document intent, side effects, and exceptional conditions

```python
def calculate_order_amount(items: list[OrderItem]) -> Decimal:
    """Calculate order amount.

    Raises:
        ValueError: If items list is empty.
    """
```

---

# 7. Code Reuse Patterns

## 7.1 Service Layer Factory Pattern

**Purpose**: Centralize authentication, validation, and error handling

### Base Pattern: service decorator

```python
# lib/actions/action_helpers.py
from functools import wraps
from pydantic import BaseModel
from typing import TypeVar, Callable, Awaitable

T = TypeVar("T", bound=BaseModel)
R = TypeVar("R")


def with_auth_and_validation(
    model: type[T],
) -> Callable[
    [Callable[[T, str], Awaitable[R]]],
    Callable[[dict], Awaitable[R]],
]:
    """Decorator that handles authentication, validation, and error handling."""
    def decorator(handler: Callable[[T, str], Awaitable[R]]) -> Callable[[dict], Awaitable[R]]:
        @wraps(handler)
        async def wrapper(raw_input: dict) -> R:
            session = await require_auth()
            validated = model.model_validate(raw_input)
            return await handler(validated, session.user.id)
        return wrapper
    return decorator
```

### Resource Save Factory

```python
# features/{domain}/services/resource_save_helper.py
from dataclasses import dataclass
from typing import Callable, Awaitable, TypeVar
from pydantic import BaseModel

T = TypeVar("T", bound=BaseModel)


@dataclass
class ResourceSaveOptions:
    model: type[BaseModel]
    quota_cost: int
    save: Callable[..., Awaitable]


def create_resource_save_action(options: ResourceSaveOptions):
    @with_auth_and_validation(options.model)
    async def action(validated: BaseModel, user_id: str):
        await require_project_ownership(validated.project_id, user_id)
        await check_quota(user_id, options.quota_cost)
        return await options.save(validated)
    return action


# Usage example
save_product = create_resource_save_action(
    ResourceSaveOptions(
        model=ProductSchema,
        quota_cost=10,
        save=lambda data: product_repository.upsert(data),
    )
)
```

### Image Generation Factory

```python
# features/{domain}/services/ai_generation_helper.py
@dataclass
class ImageGenerationOptions:
    model: type[BaseModel]
    quota_cost: int
    job_type: str


def create_image_generation_action(options: ImageGenerationOptions):
    @with_auth_and_validation(options.model)
    async def action(validated: BaseModel, user_id: str):
        remaining = await check_quota(user_id, options.quota_cost)
        job_id = await enqueue_job(validated, type=options.job_type)
        return {"job_id": job_id, "quota_remaining": remaining}
    return action
```

## 7.2 Authentication & Authorization Helpers

```python
# lib/actions/auth_helpers.py
from sqlalchemy.ext.asyncio import AsyncSession


async def require_auth() -> Session:
    """Simple authentication check."""
    session = await get_session()
    if not session:
        raise UnauthorizedError()
    return session


async def require_project_ownership(
    project_id: str, user_id: str, db: AsyncSession
) -> Project:
    """Project ownership check."""
    project = await db.get(Project, project_id)
    if not project or project.user_id != user_id:
        raise ForbiddenError()
    return project


async def require_relation_ownership(
    find_record: Callable[[], Awaitable[T | None]],
    user_id: str,
) -> T:
    """Ownership check via relation."""
    record = await find_record()
    if not record or record.user_id != user_id:
        raise ForbiddenError()
    return record
```

## 7.3 HTTP Client Centralization

```python
# lib/api/http_client.py
import httpx


async def fetch_with_auth(url: str, **kwargs) -> dict:
    """Centralized HTTP client with default headers."""
    headers = {
        "Content-Type": "application/json",
        **kwargs.pop("headers", {}),
    }

    async with httpx.AsyncClient() as client:
        response = await client.request(url=url, headers=headers, **kwargs)

    if response.status_code >= 400:
        raise ApiError(response.status_code, response.text)

    return response.json()


# Usage example: eliminating duplication
# Bad: Writing httpx calls directly everywhere
# response = await httpx.post("/api/generate/auto-config", ...)

# Good: Using the shared client
# result = await fetch_with_auth("/api/generate/auto-config", method="POST", json=data)
```

## 7.4 Generic CRUD Repository

```python
# lib/repository/base_crud.py
from typing import TypeVar, Generic, Type
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

T = TypeVar("T")


class BaseCRUDRepository(Generic[T]):
    def __init__(self, model: Type[T], db: AsyncSession):
        self.model = model
        self.db = db

    async def get_by_id(self, id: str) -> T | None:
        return await self.db.get(self.model, id)

    async def list_all(self, **filters) -> list[T]:
        stmt = select(self.model).filter_by(**filters)
        result = await self.db.execute(stmt)
        return list(result.scalars().all())

    async def create(self, **kwargs) -> T:
        instance = self.model(**kwargs)
        self.db.add(instance)
        await self.db.flush()
        return instance

    async def delete(self, id: str) -> None:
        instance = await self.get_by_id(id)
        if instance:
            await self.db.delete(instance)
```

## 7.5 Criteria for Code Reuse

| Condition | Action |
|------|------|
| Same logic in 2+ places | Extract into a helper function |
| Same pattern in 3+ places | Extract into a factory function or decorator |
| Similar service patterns in 5+ places | Consider creating a base class |
| Authentication/authorization patterns | Consolidate in auth_helpers.py |
| Error handling | Consolidate in error_handlers.py |

## 7.6 Anti-Patterns

```python
# Bad: Premature abstraction
# Abstracting when only used in one place
def _single_use_helper():
    ...

# Bad: Bloated configuration objects
create_action(
    option1=..., option2=..., option3=..., option4=..., option5=...,  # Too many
)

# Good: Extract only the common parts, implement special cases individually
base_action = create_base_action(common_options)

async def special_action(input_data):
    # Special logic
    transformed = transform(input_data)
    return await base_action(transformed)
```
