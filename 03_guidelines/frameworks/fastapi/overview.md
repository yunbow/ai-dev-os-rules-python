# Overview

> **Note:** The technology stack below is a sample configuration. Replace libraries based on your project's requirements. Files marked with `[Replaceable]` in this directory contain library-specific patterns — update the corresponding files when switching libraries.

## Purpose

This is a guideline for designing scalable web APIs with FastAPI, balancing performance, type safety, observability, and developer productivity.

## Technology Stack

- Python 3.12+
  - Async I/O (asyncio)
  - Type Hints (strict mode)
- Web Framework: FastAPI
- Validation: Pydantic v2
- Database ORM: SQLAlchemy 2.0 (async)
- Migration: Alembic
- Authentication: JWT (python-jose) or OAuth2
- Testing: pytest + httpx (AsyncClient)
- Build / Packaging: uv (or pip + pyproject.toml)
- Type Checking: mypy or pyright
- Linting / Formatting: Ruff
- Logger: structlog

### Replaceable Libraries

| Category | Current | Alternatives | Related Files |
|----------|---------|-------------|---------------|
| Web Framework | FastAPI | Litestar, Starlette, Flask | All files |
| Database ORM | SQLAlchemy 2.0 (async) | Tortoise ORM, SQLModel, Prisma Client Python | `database.md` |
| Migration | Alembic | aerich (Tortoise) | `database.md` |
| Authentication | JWT (python-jose) | Authlib, FastAPI-Users | `auth.md` |
| Validation | Pydantic v2 | attrs, msgspec | `api.md` |
| Testing | pytest + httpx | pytest + TestClient (sync) | — |

## Basic Principles

- Thin route handlers — business logic lives in service layer
- Dependency injection via `Depends()` for all cross-cutting concerns
- Pydantic models as the single source of truth for request/response schemas
- Async-first for I/O-bound operations
