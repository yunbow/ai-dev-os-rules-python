# Validation Guidelines
This document outlines how to manage **Types** and **Validation** in a unified manner across Python applications, centered on Pydantic and SQLAlchemy.

---

# 1. Core Policy
* Build a structure where types naturally synchronize in the order **Pydantic → SQLAlchemy → Python type hints**.
* API schemas can be exported from **Pydantic → JSON Schema (for AI APIs, etc.)** for external sharing.
* MUST validate on BOTH client (frontend) and server using the SAME Pydantic schema exported as JSON Schema. Never rely on server-side validation alone.

---

# 2. Business Logic Rules in Pydantic Schemas

Business logic rules that belong in Pydantic schemas (not just format validation):

* **Date range constraints**: MUST validate that future-facing dates (e.g., `due_date`, `expires_at`) are in the future using `@field_validator`. MUST validate that `end_date` is after `start_date`
* **Domain value boundaries**: e.g., quantity must be 1-999, discount percentage 0-100
* **Conditional required fields**: e.g., if `payment_method` is "card", `card_number` is required
* **Enumerated state transitions**: e.g., order status can only move from "pending" → "confirmed" → "shipped"
* **Composite uniqueness hints**: e.g., `slug` must match `^[a-z0-9-]+$` pattern (DB enforces actual uniqueness)

Rules that do NOT belong in Pydantic (handle in service/domain layer):
* Checks requiring DB lookups (e.g., "email must not already exist")
* Rules depending on external service state (e.g., "inventory must be available")
* Multi-aggregate consistency checks

---

# 3. Synchronizing Pydantic Schemas and SQLAlchemy Models

Recommended process:
1. **Write Pydantic first**
   UI requirements and business requirements are consolidated in Pydantic.
2. **Define SQLAlchemy models based on Pydantic**
   Enums and similar constructs are also mapped to the SQLAlchemy model.
3. Generate/update the DB via Alembic migrations.
4. API endpoints and services all use Pydantic-derived types.

### Single Source of Truth for Types
* **Truth for input types: Pydantic**
* **Truth for DB: SQLAlchemy**
* **Truth for API Schema: Pydantic (→ JSON Schema generation via `.model_json_schema()`)**

---

# 4. Directory Structure (Schema / Models / Types)

```
src/
 ├─ features/
 │   ├─ user/
 │   │   ├─ schemas/         # Pydantic schemas (source of truth for input)
 │   │   ├─ models/          # SQLAlchemy models
 │   │   ├─ services/        # Pydantic → SQLAlchemy transformation
 │   │   └─ routes/          # FastAPI route handlers
 │
 ├─ lib/
 │   ├─ db/                  # SQLAlchemy engine, session factory
 │
 └─ types/                   # Global type definitions (TypedDict, Protocol, etc.)
```

---

# 5. API Schema (JSON Schema) Generation

* Pydantic schemas can be converted to JSON Schema via `.model_json_schema()`,
  serving as **the foundation for AI API schemas, external APIs, and specification generation**.

Benefits:
* Prevents type mismatches
* Unifies the API ecosystem
* Facilitates automatic documentation generation (FastAPI generates OpenAPI specs automatically from Pydantic models)

---

# 6. Guidelines for Pydantic and SQLAlchemy Definition Duplication Risk
Since Pydantic and SQLAlchemy define the same fields, there is a **risk of inconsistency when changes are made**.
Follow these principles to mitigate:

### Principles
1. **Input constraints (validation) → Pydantic is the sole source of truth**
2. **Structural constraints (DB integrity) → SQLAlchemy is the sole source of truth**
3. Shared domain rules (enum / min-max / regex) should be **defined in both Pydantic and SQLAlchemy** where possible

### Operational Rules

| Case | Where to Define | Reason |
| --- | --- | --- |
| UI/external input validation | Pydantic only | Changes frequently; DB migrations are costly |
| DB integrity constraints | SQLAlchemy only | Cannot be guaranteed by Pydantic |
| Application-wide business rules | Pydantic → sync to SQLAlchemy | Single source management |
| Enum / fixed values | Both | Required to prevent type mismatches |

### Recommended Support Tools

| Tool | Effect |
| ---------------------- | ------------------ |
| `pydantic-sqlalchemy` | Assists Pydantic → SQLAlchemy conversion |
| FastAPI (built-in) | Generates OpenAPI from Pydantic automatically |
| `sqlalchemy-stubs` / `sqlalchemy[mypy]` | Improves type safety for SQLAlchemy queries |

> **Duplication is acceptable, but responsibilities must be clearly separated.**
> Maintain the structure where validation and transformation belong to Pydantic, and DB protection belongs to SQLAlchemy.

---

# 7. Summary (Pydantic + SQLAlchemy)

* Types from API input → service → DB naturally synchronize via Pydantic → SQLAlchemy
* JSON Schema generation unifies external API schemas as well
* **Definition duplication is tolerated from a responsibility separation perspective, but intentional synchronization management must be enforced**
