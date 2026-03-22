# Authentication & Authorization Guidelines

## 1. Authentication

## 1.1 JWT-Based Authentication

* MUST use JWT tokens for stateless authentication
* MUST store secrets in environment variables (never hardcode)
* MUST set token expiration (access: 15-60 min, refresh: 7-30 days)
* MUST use `Depends()` for all protected endpoints

```python
## core/security.py
from fastapi import Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_session),
) -> User:
    token = credentials.credentials
    payload = verify_token(token)
    user = await db.get(User, payload["sub"])
    if not user:
        raise NotFoundError("User not found")
    return user
```

## 1.2 Password Handling

* MUST hash passwords with bcrypt (or argon2)
* MUST NOT store plain-text passwords
* MUST NOT log passwords or tokens

## 2. Authorization

## 2.1 Role-Based Access Control

* MUST implement authorization as a dependency, not inline in handlers
* MUST check permissions before executing business logic

```python
## core/security.py
def require_role(*roles: str):
    async def dependency(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in roles:
            raise ForbiddenError("Insufficient permissions")
        return current_user
    return dependency

## Usage in router
@router.delete("/{user_id}", status_code=204)
async def delete_user(
    user_id: int,
    current_user: User = Depends(require_role("admin")),
    service: UserService = Depends(get_user_service),
): ...
```

## 2.2 Resource-Level Authorization

* MUST verify that the authenticated user has access to the specific resource
* MUST NOT rely solely on role checks — check resource ownership where applicable

```python
## features/orders/service.py
async def get_order(self, order_id: int, current_user: User) -> Order:
    order = await self.repository.get(order_id)
    if not order:
        raise NotFoundError("Order not found")
    if order.user_id != current_user.id and current_user.role != "admin":
        raise ForbiddenError("Access denied")
    return order
```
