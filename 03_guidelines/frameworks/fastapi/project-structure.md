# Project Structure Guidelines

This document defines the directory policies and architecture guidelines for FastAPI applications.

## 1. Overall Principles

- Adopt **vertical slicing (feature-based) as the base structure**
  → Group router / service / repository / schema / model per feature
- **Horizontal slicing is limited to shared functionality only**
  → Shared layers such as core / lib / config / middleware

---

## 2. Directory Structure

```text
src/
├─ app/                     # Application entry point
│  ├─ main.py               # FastAPI app factory, router mounting
│  ├─ config.py             # Settings (pydantic-settings)
│  └─ dependencies.py       # Shared dependencies (DB session, auth, etc.)
│
├─ features/                # Core of vertical slicing
│  ├─ {domain}/             # Example: users, billing, content
│  │  ├─ router.py          # Route definitions (thin handlers)
│  │  ├─ service.py         # Business logic
│  │  ├─ repository.py      # Database access (SQLAlchemy queries)
│  │  ├─ schemas.py         # Pydantic request/response models
│  │  ├─ models.py          # SQLAlchemy ORM models
│  │  ├─ dependencies.py    # Feature-specific dependencies
│  │  ├─ exceptions.py      # Feature-specific exceptions
│  │  └─ constants.py       # Feature-specific constants
│  ├─ auth/
│  ├─ admin/
│  └─ ...
│
├─ core/                    # Shared infrastructure
│  ├─ database.py           # Engine, session factory, Base
│  ├─ security.py           # JWT, password hashing
│  ├─ exceptions.py         # Base exception classes, handlers
│  ├─ middleware.py          # CORS, logging, request ID
│  └─ pagination.py         # Shared pagination utilities
│
├─ lib/                     # External service clients
│  ├─ email.py              # Email service client
│  ├─ storage.py            # File storage client
│  └─ payment.py            # Payment service client
│
├─ migrations/              # Alembic migrations
│  ├─ versions/
│  └─ env.py
│
└─ tests/
   ├─ conftest.py           # Fixtures (test DB, client, auth)
   ├─ features/
   │  ├─ test_users.py
   │  └─ test_billing.py
   └─ core/
```

## 3. Layer Responsibilities

| Layer | Responsibility | Dependencies |
|-------|---------------|-------------|
| **router** | HTTP handling, request parsing, response formatting | service |
| **service** | Business logic, orchestration, validation | repository, lib |
| **repository** | Database queries, ORM operations | models |
| **schemas** | Request/response validation (Pydantic) | — |
| **models** | Database table definitions (SQLAlchemy) | — |

## 3.1 Dependency Direction

```text
router → service → repository → models
                 → lib (external)
schemas (used by router and service, but defines no dependencies)
```

- MUST NOT import router from service
- MUST NOT import service from repository
- MUST NOT import features from each other directly — use events or shared service

## 4. File Naming Conventions

| Pattern | Convention | Example |
|---------|-----------|---------|
| Feature directory | Singular noun, snake_case | `features/billing/` |
| Router file | `router.py` (one per feature) | `features/billing/router.py` |
| Service file | `service.py` (one per feature) | `features/billing/service.py` |
| Schema file | `schemas.py` (Pydantic models) | `features/billing/schemas.py` |
| Model file | `models.py` (SQLAlchemy models) | `features/billing/models.py` |
| Test file | `test_{feature}.py` | `tests/features/test_billing.py` |

## 5. Import Rules

- MUST use absolute imports from the project root
- MUST NOT use relative imports across features
- MUST NOT use wildcard imports (`from x import *`)

```python
## GOOD
from src.features.billing.service import BillingService
from src.core.database import get_session

## BAD
from ..billing.service import BillingService  # relative cross-feature
from src.features.billing.service import *     # wildcard
```
