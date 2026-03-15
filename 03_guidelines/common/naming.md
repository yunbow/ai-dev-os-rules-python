# Naming Convention Guidelines
This document defines **unified naming conventions** for Python applications.

Scope:
* SQLAlchemy models and field names
* Database physical names (table and column names)
* URL paths
* API routes
* Pydantic schemas
* Classes / Functions / Variables
* File and directory names

Prioritizes readability, searchability, and scalability.

---

# 1. General Principles
### Consistency
Unify naming within each domain. Do not mix different naming conventions.

### English Naming Thresholds
* **Variable/field names**: 1-3 words, max 30 characters (e.g., `created_at`, `order_item_count`)
* **Function names**: verb + noun, 2-4 words, max 40 characters (e.g., `fetch_user_profile`, `calculate_total_price`)
* **Class names**: 1-3 words describing the shape (e.g., `UserProfile`, `OrderSummary`)
* **Avoid abbreviations** unless universally understood (`id`, `url`, `api`, `db` are acceptable; `usr`, `msg`, `btn`, `cfg` are not)
* **When a name exceeds thresholds**, it signals the concept should be split or the scope narrowed

---

# 2. SQLAlchemy Relationship Names
* **Many-to-one / one-to-one → singular**
* **One-to-many / many-to-many → plural (to clearly indicate collections)**
* snake_case

Examples:

```python
class Order(Base):
    __tablename__ = "orders"

    # Many-to-one
    user_id: Mapped[str] = mapped_column(ForeignKey("users.id"))
    user: Mapped["User"] = relationship(back_populates="orders")

    # One-to-many
    order_items: Mapped[list["OrderItem"]] = relationship(back_populates="order")
```

---
# 3. Database Physical Names
## Table Names

* **snake_case**
* **Plural**

Examples:

```
users
blog_posts
order_items
```

## Column Names
* **snake_case**
* Singular

Examples:

```
created_at
user_id
is_active
```

## Constraint Names
This guideline adopts the convention of prefixing with the constraint type (pk, fk, idx) for readability and searchability.

Rationale:
* **Constraints are visually grouped by type when listed**
* **Easy to filter by prefix in tools and logs**
* Can be more readable than the common "orders_user_id_fk" format

Naming rules:
* Primary key: `pk_<table>`
* Foreign key: `fk_<table>_<ref_table>`
* Index: `idx_<table>_<column>`

Examples:

```
pk_users
fk_orders_users
idx_blog_posts_author_id
```

---

# 4. URL Path Naming (FastAPI Router)
* **kebab-case**
* Resources in **plural**
* No verbs (REST)

Examples:

```
/users
/users/{user_id}
/blog/posts
/blog/posts/{slug}
```

---

# 5. API Route Naming (/api)
* Follow REST principles
* Resource names in **plural**
* Operations expressed through HTTP methods
* No verbs in paths

Examples:

```
/api/users
/api/users/{user_id}
/api/orders
/api/orders/{order_id}/items
```

---

# 6. Service / Repository
* `{domain}_service.py`
* `{domain}_repository.py`

Examples:

```
user_service.py
order_repository.py
```

---

# 7. Validation Schema (Pydantic)
* Schema class names: **PascalCase**
* File names: **snake_case + _schema.py**

Example:

```python
from pydantic import BaseModel, EmailStr


class CreateUserSchema(BaseModel):
    email: EmailStr
    name: str
```

File name examples:

```
user_schema.py
order_schema.py
form_schema.py
```

---

# 8. CLI Commands (Typer)

* Command functions: `snake_case`
* Command group files: `{domain}_commands.py`

---

# 9. WebSocket / Realtime Names

* Event names: **snake_case**
* Channel names: **plural**

---

# Summary

| Target | Naming Convention |
| --- | --- |
| SQLAlchemy model class | PascalCase (singular) |
| SQLAlchemy column/field | snake_case |
| SQLAlchemy relationship (one-to-many / many-to-many) | **plural (emphasizing collections)** |
| DB table physical name | snake_case (plural) |
| DB column | snake_case (singular) |
| URL path | kebab-case (plural) |
| API path | REST / plural |
| Python module/package | snake_case |
| Class names | PascalCase |
| Functions / variables | snake_case |
| Constants | UPPER_SNAKE_CASE |
| Pydantic schema | PascalCase / file name is snake_case (_schema.py) |
| Enum | PascalCase class, UPPER_SNAKE_CASE members |
