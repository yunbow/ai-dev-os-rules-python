# Performance Guidelines

This document covers only performance optimization items that **do not clearly overlap** with content covered in other documents.

**Related documents:**

- DB query optimization (N+1, indexes, pagination) → frameworks/fastapi/database.md
- API caching → frameworks/fastapi/api.md
- State management and dependency injection → frameworks/fastapi/state.md

---

## 1. Async / Concurrency Optimization
### Use async Endpoints by Default
* Define FastAPI endpoints as `async def` whenever they perform I/O (HTTP calls, DB queries via async drivers, file I/O).
* Use synchronous `def` only for CPU-bound endpoints — FastAPI runs them in a thread pool automatically.

### Avoid Blocking the Event Loop
Decision rule for when to use async vs sync:
* **Use `async def`** when the endpoint awaits I/O (database, HTTP, file reads)
* **Use `def`** when the endpoint does CPU-bound work (FastAPI offloads to a thread pool)
* Never call blocking I/O (e.g., `requests.get()`, synchronous DB drivers) inside an `async def` endpoint
* If a library only offers synchronous I/O, use `asyncio.to_thread()` or `run_in_executor()`

```python
import asyncio

# ❌ Blocks the event loop
async def bad_endpoint():
    result = requests.get("https://api.example.com/data")  # Blocking!
    return result.json()

# ✅ Offload blocking call to thread pool
async def good_endpoint():
    result = await asyncio.to_thread(requests.get, "https://api.example.com/data")
    return result.json()

# ✅ Best: use an async HTTP client
import httpx

async def best_endpoint():
    async with httpx.AsyncClient() as client:
        result = await client.get("https://api.example.com/data")
        return result.json()
```

---

## 2. Connection Pooling
### Database Connection Pooling
* Use SQLAlchemy's built-in connection pool (`pool_size`, `max_overflow`, `pool_recycle`).
* For async, use `create_async_engine` with `AsyncSession`.

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=10,
    max_overflow=20,
    pool_recycle=3600,
)

AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

### HTTP Client Connection Pooling
* Reuse `httpx.AsyncClient` instances instead of creating new ones per request.
* Create a shared client at startup and close it on shutdown.

```python
from contextlib import asynccontextmanager
import httpx

@asynccontextmanager
async def lifespan(app):
    app.state.http_client = httpx.AsyncClient(timeout=30.0)
    yield
    await app.state.http_client.aclose()
```

---

## 3. Caching Strategy
### In-Memory Caching with `functools.lru_cache`

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def get_config(key: str) -> str:
    """Cache configuration values that rarely change."""
    return expensive_config_lookup(key)
```

### Async-Compatible Caching

```python
from functools import lru_cache
from cachetools import TTLCache

# TTL-based cache for async functions
_cache = TTLCache(maxsize=100, ttl=300)  # 5 minutes

async def get_user_profile(user_id: str) -> dict:
    if user_id in _cache:
        return _cache[user_id]

    profile = await fetch_profile_from_db(user_id)
    _cache[user_id] = profile
    return profile
```

### Redis Caching for Distributed Systems

```python
import redis.asyncio as redis

redis_client = redis.from_url("redis://localhost:6379")

async def get_cached_data(key: str) -> str | None:
    return await redis_client.get(key)

async def set_cached_data(key: str, value: str, ttl: int = 60) -> None:
    await redis_client.set(key, value, ex=ttl)
```

### Cache-Control Headers for API Responses

```python
from fastapi import Response

@app.get("/api/public-data")
async def get_public_data(response: Response):
    response.headers["Cache-Control"] = "public, max-age=60, s-maxage=300"
    return {"data": "..."}
```

---

## 4. Database Query Optimization
### Avoid N+1 Queries
* Use `selectinload` / `joinedload` in SQLAlchemy to eagerly load relationships.

```python
from sqlalchemy.orm import selectinload

# ❌ N+1: loads each user's posts separately
users = session.execute(select(User)).scalars().all()
for user in users:
    print(user.posts)  # Triggers a query per user

# ✅ Eager load: single query for all posts
users = session.execute(
    select(User).options(selectinload(User.posts))
).scalars().all()
```

### Use Pagination Consistently
* Always paginate large result sets with `limit` and `offset` (or cursor-based pagination).

### Index Critical Columns
* Add indexes for columns used in `WHERE`, `ORDER BY`, and `JOIN` clauses.

---

## 5. Serialization Optimization
### Use Pydantic Model Serialization Efficiently

```python
# ❌ Slow: converting to dict then re-serializing
data = [item.model_dump() for item in items]
return JSONResponse(content=data)

# ✅ Fast: let FastAPI handle serialization
@app.get("/items", response_model=list[ItemResponse])
async def get_items():
    return items  # FastAPI serializes via Pydantic
```

### Avoid Excessive JSON Parsing
* Use `orjson` for high-performance JSON serialization when handling large payloads.

```python
import orjson
from fastapi.responses import ORJSONResponse

app = FastAPI(default_response_class=ORJSONResponse)
```

---

## 6. Lazy Loading and Deferred Computation
### Lazy Module Imports for CLI Tools

```python
# For CLI apps (Typer), defer heavy imports to improve startup time
import typer

app = typer.Typer()

@app.command()
def process(file: str):
    import pandas as pd  # Import only when needed
    df = pd.read_csv(file)
    # ...
```

### Background Tasks for Non-Critical Work

```python
from fastapi import BackgroundTasks

@app.post("/api/orders")
async def create_order(order: OrderCreate, background_tasks: BackgroundTasks):
    result = await save_order(order)
    background_tasks.add_task(send_confirmation_email, order.email)
    return result
```

---

## 7. Concurrency and Parallelism
### Use `asyncio.gather` for Concurrent I/O

```python
import asyncio

async def get_dashboard_data(user_id: str):
    # Run independent queries concurrently
    profile, orders, notifications = await asyncio.gather(
        get_user_profile(user_id),
        get_recent_orders(user_id),
        get_notifications(user_id),
    )
    return {"profile": profile, "orders": orders, "notifications": notifications}
```

### Use `ProcessPoolExecutor` for CPU-Bound Work

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

executor = ProcessPoolExecutor(max_workers=4)

async def process_heavy_computation(data: list[float]) -> float:
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, cpu_intensive_function, data)
    return result
```

---

## 8. Memory Optimization
### Use Generators for Large Datasets

```python
# ❌ Loads everything into memory
def get_all_records():
    return list(session.execute(select(Record)).scalars().all())

# ✅ Stream results with a generator
def stream_records():
    result = session.execute(select(Record))
    for row in result.scalars():
        yield row
```

### Use Streaming Responses for Large Payloads

```python
from fastapi.responses import StreamingResponse

@app.get("/api/export")
async def export_data():
    async def generate():
        async for chunk in fetch_large_dataset():
            yield chunk.to_csv()

    return StreamingResponse(generate(), media_type="text/csv")
```

---

## 9. Network Optimization
### Use HTTP/2 When Possible
### Reduce Excessive API Calls
* Batch requests where possible; aggregate data on the server side.

### Use Response Compression

```python
from fastapi.middleware.gzip import GZipMiddleware

app.add_middleware(GZipMiddleware, minimum_size=1000)
```

---

## 10. Startup Optimization
### Use FastAPI Lifespan for Initialization
* Preload database connections, caches, and ML models during startup.

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app):
    # Startup: initialize resources
    await init_db_pool()
    await warm_up_cache()
    yield
    # Shutdown: cleanup
    await close_db_pool()

app = FastAPI(lifespan=lifespan)
```

---

## 11. Profiling
### cProfile / py-spy

```bash
# Profile a script
python -m cProfile -o profile.out app.py

# Live profiling of a running process
py-spy top --pid <PID>
py-spy record --pid <PID> -o profile.svg
```

### line_profiler for Hot Functions

```python
# Decorate functions to profile
@profile
def slow_function():
    ...
```

```bash
kernprof -l -v script.py
```

### Memory Profiling with memray

```bash
memray run app.py
memray flamegraph output.bin
```

### Request-Level Timing

```python
import time
from fastapi import Request

@app.middleware("http")
async def timing_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start
    response.headers["X-Process-Time"] = f"{duration:.4f}"
    return response
```
