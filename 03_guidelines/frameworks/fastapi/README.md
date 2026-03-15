# FastAPI Framework Guidelines

Guidelines specific to FastAPI web applications.

## Files

| File | Description |
|------|-------------|
| [overview.md](./overview.md) | Technology stack definition |
| [project-structure.md](./project-structure.md) | Directory structure |
| [routing.md](./routing.md) | Route design and URL conventions |
| [api.md](./api.md) | API design (request/response, versioning) |
| [auth.md](./auth.md) | Authentication and authorization |
| [database.md](./database.md) | Database access (SQLAlchemy, Alembic) |
| [middleware.md](./middleware.md) | Middleware design |

## Relationship with common/

- `common/` defines language-independent rules ("what to do")
- This directory defines FastAPI-specific patterns ("how to implement")
- When both exist for the same topic, this directory takes priority (CSS Specificity)
