# Database Guidelines

# 1. SQLAlchemy Configuration

## 1.1 Engine and Session Setup
* MUST use async engine and session for FastAPI
* MUST use dependency injection for session lifecycle
* MUST NOT create sessions manually in route handlers

```python
# core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession

engine = create_async_engine(settings.DATABASE_URL, echo=settings.DEBUG)
async_session = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

## 1.2 Model Definition
* MUST use `DeclarativeBase` (SQLAlchemy 2.0 style)
* MUST use `Mapped` and `mapped_column` for type-safe column definitions
* MUST use snake_case for table and column names
* MUST define `__tablename__` explicitly

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True, index=True)
    name: Mapped[str]
    created_at: Mapped[datetime] = mapped_column(default=func.now())
```

# 2. Repository Pattern

## 2.1 Repository Design
* MUST encapsulate all database queries in repository classes
* MUST NOT write SQLAlchemy queries in service or router layers
* MUST accept `AsyncSession` via constructor injection

```python
# features/users/repository.py
class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def get(self, user_id: int) -> User | None:
        return await self.session.get(User, user_id)

    async def get_by_email(self, email: str) -> User | None:
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def create(self, user: User) -> User:
        self.session.add(user)
        await self.session.flush()
        return user
```

## 2.2 Query Optimization
* MUST use `selectinload` or `joinedload` to prevent N+1 queries
* MUST use `.scalars()` for single-column results
* MUST add database indexes for frequently queried columns
* SHOULD use `select()` instead of legacy `session.query()`

# 3. Migrations (Alembic)

## 3.1 Migration Rules
* MUST generate migrations via `alembic revision --autogenerate`
* MUST review auto-generated migrations before applying
* MUST use descriptive revision messages: `alembic revision -m "add users table"`
* MUST NOT modify migrations that have been applied to production
* MUST test migrations with both upgrade and downgrade

## 3.2 Migration Naming
* Use verb + noun format: `add_users_table`, `add_email_index_to_users`, `remove_legacy_column`
