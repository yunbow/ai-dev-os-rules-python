# Routing Guidelines

## 1. URL Conventions

## 1.1 General Rules

* MUST use kebab-case for multi-word URL segments: `/user-profiles`, not `/userProfiles`
* MUST use plural nouns for resource collections: `/users`, `/orders`
* MUST use path parameters for resource identification: `/users/{user_id}`
* MUST NOT nest resources more than 2 levels: `/users/{user_id}/orders` is OK, `/users/{user_id}/orders/{order_id}/items` should be `/orders/{order_id}/items`

## 1.2 API Versioning

* MUST prefix all API routes with version: `/api/v1/users`
* Use `APIRouter(prefix="/api/v1")` at the top-level router

## 1.3 Route Organization

```python
## features/users/router.py

router = APIRouter(prefix="/users", tags=["users"])

## Collection routes first, then item routes
@router.get("/", response_model=PaginatedResponse[UserResponse])
async def list_users(...): ...

@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(...): ...

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(...): ...

@router.patch("/{user_id}", response_model=UserResponse)
async def update_user(...): ...

@router.delete("/{user_id}", status_code=204)
async def delete_user(...): ...
```

## 2. Route Handler Design

## 2.1 Thin Handlers

* MUST keep route handlers thin — delegate business logic to service layer
* Route handlers SHOULD only: parse request → call service → format response
* MUST NOT contain database queries, complex logic, or external API calls directly

```python
## GOOD: Thin handler
@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(
    body: CreateUserRequest,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    user = await service.create_user(body)
    return UserResponse.model_validate(user)

## BAD: Fat handler with business logic
@router.post("/")
async def create_user(body: CreateUserRequest, db: Session = Depends(get_session)):
    existing = await db.execute(select(User).where(User.email == body.email))
    if existing.scalar():
        raise HTTPException(400, "Email exists")
    user = User(**body.model_dump())
    db.add(user)
    await db.commit()
    # ... more logic
```

## 2.2 Response Status Codes

* MUST use appropriate HTTP status codes:

| Operation | Success Code | Error Codes |
|-----------|-------------|-------------|
| GET (single) | 200 | 404 |
| GET (list) | 200 | — |
| POST (create) | 201 | 400, 409 |
| PATCH/PUT (update) | 200 | 400, 404 |
| DELETE | 204 | 404 |

## 2.3 Dependency Injection

* MUST use `Depends()` for all cross-cutting concerns (auth, DB session, services)
* MUST NOT instantiate services or DB sessions directly in handlers

```python
## GOOD
@router.get("/{user_id}")
async def get_user(
    user_id: int,
    current_user: User = Depends(get_current_user),
    service: UserService = Depends(get_user_service),
) -> UserResponse: ...

## BAD
@router.get("/{user_id}")
async def get_user(user_id: int):
    db = SessionLocal()  # manual instantiation
    service = UserService(db)  # manual wiring
```
