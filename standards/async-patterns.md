# Async Patterns for First Viscount Platform

This document provides comprehensive guidance on implementing async patterns in Python services within the First Viscount platform. These patterns are essential for building scalable, maintainable, and thread-safe async applications.

## Table of Contents

1. [Context Variables for Request Context](#context-variables-for-request-context)
2. [Async-Safe Logging Patterns](#async-safe-logging-patterns)
3. [Database Transaction Management](#database-transaction-management)
4. [Converting Global State to Context-Safe Patterns](#converting-global-state-to-context-safe-patterns)
5. [Common Pitfalls and Solutions](#common-pitfalls-and-solutions)
6. [Testing Strategies for Async Code](#testing-strategies-for-async-code)
7. [Performance Considerations](#performance-considerations)

## Context Variables for Request Context

Context variables (`contextvars`) provide thread-safe and async-safe storage for request-scoped data. They automatically handle context isolation across concurrent requests.

### ✅ DO: Use ContextVar for Request-Scoped Data

```python
from contextvars import ContextVar
from typing import Optional

# Define context variables at module level
_request_id_context: ContextVar[Optional[str]] = ContextVar("request_id", default=None)
_correlation_id_context: ContextVar[Optional[str]] = ContextVar("correlation_id", default=None)
_user_id_context: ContextVar[Optional[str]] = ContextVar("user_id", default=None)

def set_request_id(request_id: str) -> None:
    """Set the request ID in the current context."""
    _request_id_context.set(request_id)

def get_request_id() -> Optional[str]:
    """Get the current request ID from context."""
    return _request_id_context.get()

def clear_context() -> None:
    """Clear all context variables."""
    _request_id_context.set(None)
    _correlation_id_context.set(None)
    _user_id_context.set(None)
```

### ✅ DO: Set Context in Middleware

```python
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import uuid

class RequestContextMiddleware(BaseHTTPMiddleware):
    """Middleware to set request context for all requests."""
    
    async def dispatch(self, request: Request, call_next) -> Response:
        # Generate unique identifiers
        correlation_id = str(uuid.uuid4())
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        
        # Set context
        set_correlation_id(correlation_id)
        set_request_id(request_id)
        
        try:
            response = await call_next(request)
            response.headers["X-Correlation-ID"] = correlation_id
            return response
        finally:
            # Context is automatically cleaned up when the coroutine ends
            pass
```

### ❌ DON'T: Use Global Variables for Request Data

```python
# BAD: This will cause race conditions
current_user_id = None
current_request_id = None

async def set_current_user(user_id: str):
    global current_user_id
    current_user_id = user_id  # Race condition!

async def get_current_user():
    return current_user_id  # May return wrong user!
```

## Async-Safe Logging Patterns

Logging in async applications requires careful handling to avoid race conditions and ensure proper context propagation.

### ✅ DO: Use Structured Logging with Context

```python
import structlog
from contextvars import ContextVar
from typing import Any, Dict

logger = structlog.get_logger(__name__)

def add_custom_context(
    logger: Any, method_name: str, event_dict: Dict[str, Any]
) -> Dict[str, Any]:
    """Add request context to all log entries."""
    # Automatically include context variables
    if request_id := get_request_id():
        event_dict["request_id"] = request_id
    if correlation_id := get_correlation_id():
        event_dict["correlation_id"] = correlation_id
    if user_id := get_user_id():
        event_dict["user_id"] = user_id
    
    return event_dict

# Configure structlog with context processor
structlog.configure(
    processors=[
        add_custom_context,
        structlog.processors.JSONRenderer(),
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)
```

### ✅ DO: Create Request-Bound Loggers

```python
def create_request_logger(
    correlation_id: str,
    user_id: Optional[str] = None,
    request_path: Optional[str] = None,
) -> structlog.stdlib.BoundLogger:
    """Create a logger with bound request context."""
    logger = structlog.get_logger("request")
    
    # Bind request-specific context
    logger = logger.bind(correlation_id=correlation_id)
    
    if user_id:
        logger = logger.bind(user_id=user_id)
    
    if request_path:
        logger = logger.bind(request_path=request_path)
    
    return logger

# Usage in endpoint
async def my_endpoint(request: Request):
    logger = create_request_logger(
        correlation_id=get_correlation_id(),
        user_id=get_user_id(),
        request_path=str(request.url.path)
    )
    
    logger.info("Processing request", extra_data="value")
    # All log entries will include the bound context
```

### ❌ DON'T: Share Logger Instances Across Requests

```python
# BAD: Shared logger can cause context bleeding
request_logger = None

async def setup_logger(correlation_id: str):
    global request_logger
    request_logger = logger.bind(correlation_id=correlation_id)  # Race condition!

async def log_event(message: str):
    request_logger.info(message)  # May log with wrong correlation_id!
```

## Database Transaction Management

Proper transaction management is crucial for data consistency in async applications.

### ✅ DO: Use Context Managers for Transactions

```python
from contextlib import asynccontextmanager
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker
from typing import AsyncGenerator

@asynccontextmanager
async def get_db_transaction() -> AsyncGenerator[AsyncSession, None]:
    """Context manager for database transactions."""
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

# Usage
async def create_user_with_profile(user_data: dict, profile_data: dict):
    async with get_db_transaction() as session:
        # Both operations in same transaction
        user = User(**user_data)
        session.add(user)
        await session.flush()  # Get user.id
        
        profile = UserProfile(user_id=user.id, **profile_data)
        session.add(profile)
        # Commit happens automatically if no exception
        return user
```

### ✅ DO: Use Repository Pattern with Dependency Injection

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar, Optional, List, Any

ModelType = TypeVar("ModelType")

class BaseRepository(ABC, Generic[ModelType]):
    """Abstract base repository with async methods."""
    
    def __init__(self, session: AsyncSession) -> None:
        self.session = session
    
    @abstractmethod
    async def create(self, **kwargs: Any) -> ModelType:
        """Create a new entity."""
        pass
    
    @abstractmethod
    async def get(self, id: Any) -> Optional[ModelType]:
        """Get entity by ID."""
        pass
    
    @abstractmethod
    async def update(self, id: Any, **kwargs: Any) -> Optional[ModelType]:
        """Update entity.""" 
        pass
    
    @abstractmethod
    async def delete(self, id: Any) -> bool:
        """Delete entity."""
        pass

class UserRepository(BaseRepository[User]):
    """User repository implementation."""
    
    async def create(self, **kwargs: Any) -> User:
        user = User(**kwargs)
        self.session.add(user)
        await self.session.flush()
        await self.session.refresh(user)
        return user
    
    async def get(self, id: int) -> Optional[User]:
        result = await self.session.execute(
            select(User).where(User.id == id)
        )
        return result.scalar_one_or_none()

# Dependency injection in FastAPI
async def get_user_repository(
    session: AsyncSession = Depends(get_db)
) -> UserRepository:
    return UserRepository(session)

# Usage in endpoint
async def create_user(
    user_data: UserCreate,
    user_repo: UserRepository = Depends(get_user_repository)
):
    return await user_repo.create(**user_data.dict())
```

### ❌ DON'T: Use Global Database Connections

```python
# BAD: Global connection causes concurrency issues
db_connection = None

async def get_connection():
    global db_connection
    if not db_connection:
        db_connection = await create_connection()  # Race condition!
    return db_connection

async def get_user(user_id: int):
    conn = await get_connection()
    # Multiple requests may share the same connection improperly
    return await conn.fetch("SELECT * FROM users WHERE id = $1", user_id)
```

## Converting Global State to Context-Safe Patterns

When refactoring existing code that uses global state, follow these patterns to make it async-safe.

### ✅ DO: Convert Global Cache to Context-Aware Cache

```python
# BEFORE: Global cache (problematic)
user_cache = {}

async def get_cached_user(user_id: int):
    if user_id not in user_cache:
        user_cache[user_id] = await fetch_user(user_id)
    return user_cache[user_id]

# AFTER: Context-aware cache
from contextvars import ContextVar
from typing import Dict, Any

_request_cache: ContextVar[Dict[str, Any]] = ContextVar("request_cache", default={})

def get_request_cache() -> Dict[str, Any]:
    """Get or create request-scoped cache."""
    cache = _request_cache.get()
    if not cache:
        cache = {}
        _request_cache.set(cache)
    return cache

async def get_cached_user(user_id: int):
    cache = get_request_cache()
    cache_key = f"user:{user_id}"
    
    if cache_key not in cache:
        cache[cache_key] = await fetch_user(user_id)
    
    return cache[cache_key]
```

### ✅ DO: Convert Global Configuration to Dependency Injection

```python
# BEFORE: Global config (problematic in testing)
app_config = None

def get_config():
    global app_config
    if not app_config:
        app_config = load_config()
    return app_config

# AFTER: Dependency injection
from functools import lru_cache
from pydantic import BaseSettings

class Settings(BaseSettings):
    database_url: str
    redis_url: str
    api_key: str
    
    class Config:
        env_file = ".env"

@lru_cache()
def get_settings() -> Settings:
    """Cached settings factory."""
    return Settings()

# Usage in FastAPI
async def my_endpoint(settings: Settings = Depends(get_settings)):
    # Settings injected per request, but cached globally
    database_url = settings.database_url
```

### ✅ DO: Use Async Context Managers for Resource Management

```python
# BEFORE: Global resources
redis_client = None

async def get_redis():
    global redis_client
    if not redis_client:
        redis_client = await aioredis.create_redis_pool()
    return redis_client

# AFTER: Proper resource management
import aioredis
from contextlib import asynccontextmanager

class RedisManager:
    def __init__(self):
        self._pool = None
    
    async def startup(self):
        self._pool = await aioredis.create_redis_pool()
    
    async def shutdown(self):
        if self._pool:
            self._pool.close()
            await self._pool.wait_closed()
    
    @asynccontextmanager
    async def get_redis(self):
        if not self._pool:
            raise RuntimeError("Redis pool not initialized")
        async with self._pool.get() as conn:
            yield conn

# FastAPI lifespan integration
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    redis_manager = RedisManager()
    await redis_manager.startup()
    app.state.redis = redis_manager
    
    yield
    
    # Shutdown
    await redis_manager.shutdown()

app = FastAPI(lifespan=lifespan)

# Usage
async def my_endpoint(request: Request):
    async with request.app.state.redis.get_redis() as redis:
        await redis.set("key", "value")
```

## Common Pitfalls and Solutions

### Pitfall 1: Blocking Operations in Async Code

❌ **Problem**: Using synchronous operations in async functions

```python
import time
import requests

async def fetch_data():
    # BAD: Blocks the event loop
    time.sleep(1)
    response = requests.get("https://api.example.com/data")
    return response.json()
```

✅ **Solution**: Use async alternatives

```python
import asyncio
import httpx

async def fetch_data():
    # GOOD: Non-blocking operations
    await asyncio.sleep(1)
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()
```

### Pitfall 2: Not Handling Connection Pooling

❌ **Problem**: Creating new connections for each request

```python
async def get_user(user_id: int):
    # BAD: Creates new connection each time
    async with asyncpg.connect(DATABASE_URL) as conn:
        return await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)
```

✅ **Solution**: Use connection pooling

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

# Create engine with connection pool
engine = create_async_engine(
    DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_pre_ping=True,
)

async_session = async_sessionmaker(engine, expire_on_commit=False)

async def get_user(user_id: int):
    # GOOD: Uses connection from pool
    async with async_session() as session:
        result = await session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()
```

### Pitfall 3: Race Conditions in Async Initialization

❌ **Problem**: Multiple coroutines initializing shared resources

```python
_initialized = False
_resource = None

async def get_resource():
    global _initialized, _resource
    
    if not _initialized:
        # BAD: Race condition between check and initialization
        _resource = await expensive_initialization()
        _initialized = True
    
    return _resource
```

✅ **Solution**: Use asyncio.Lock for thread-safe initialization

```python
import asyncio
from typing import Optional

_resource: Optional[Resource] = None
_init_lock = asyncio.Lock()

async def get_resource() -> Resource:
    global _resource
    
    if _resource is None:
        async with _init_lock:
            # Double-check pattern in async context
            if _resource is None:
                _resource = await expensive_initialization()
    
    return _resource
```

## Testing Strategies for Async Code

### ✅ DO: Use pytest-asyncio for Async Tests

```python
import pytest
import pytest_asyncio
from httpx import AsyncClient
from fastapi.testclient import TestClient

# Mark test as async
@pytest_asyncio.async_test
async def test_async_endpoint():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/users/1")
        assert response.status_code == 200

# Test with database transactions
@pytest_asyncio.async_test
async def test_user_creation():
    async with get_db_transaction() as session:
        user_repo = UserRepository(session)
        user = await user_repo.create(
            email="test@example.com",
            name="Test User"
        )
        assert user.email == "test@example.com"
        # Transaction rolls back automatically after test
```

### ✅ DO: Mock Async Dependencies

```python
from unittest.mock import AsyncMock, patch
import pytest

@pytest_asyncio.async_test
@patch('my_module.external_api_call')
async def test_with_mocked_async_call(mock_api_call):
    # Mock async function
    mock_api_call.return_value = AsyncMock(return_value={"data": "test"})
    
    result = await my_function_that_calls_api()
    assert result["data"] == "test"
    mock_api_call.assert_called_once()

# Mock context variables in tests
@pytest_asyncio.async_test
async def test_with_context():
    set_request_id("test-request-id")
    set_user_id("test-user-id")
    
    # Your test code here
    assert get_request_id() == "test-request-id"
    assert get_user_id() == "test-user-id"
```

### ✅ DO: Test Context Isolation

```python
@pytest_asyncio.async_test
async def test_context_isolation():
    """Test that context variables are isolated between coroutines."""
    results = []
    
    async def worker(worker_id: str, user_id: str):
        set_user_id(user_id)
        await asyncio.sleep(0.1)  # Simulate async work
        results.append(f"{worker_id}:{get_user_id()}")
    
    # Start multiple workers concurrently
    await asyncio.gather(
        worker("worker1", "user1"),
        worker("worker2", "user2"),
        worker("worker3", "user3"),
    )
    
    # Each worker should maintain its own context
    assert "worker1:user1" in results
    assert "worker2:user2" in results
    assert "worker3:user3" in results
```

### ✅ DO: Use Fixtures for Async Resources

```python
@pytest_asyncio.fixture
async def async_client():
    """Fixture providing async HTTP client."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest_asyncio.fixture
async def db_session():
    """Fixture providing database session."""
    async with get_db_transaction() as session:
        yield session
        # Session is automatically rolled back

@pytest_asyncio.async_test
async def test_with_fixtures(async_client, db_session):
    # Test using both fixtures
    user = User(email="test@example.com")
    db_session.add(user)
    await db_session.flush()
    
    response = await async_client.get(f"/users/{user.id}")
    assert response.status_code == 200
```

## Performance Considerations

### Connection Pool Configuration

```python
from sqlalchemy.ext.asyncio import create_async_engine

# Optimize connection pool for your workload
engine = create_async_engine(
    DATABASE_URL,
    # Core connections always maintained
    pool_size=10,
    
    # Additional connections when needed
    max_overflow=20,
    
    # Timeout waiting for connection from pool
    pool_timeout=30,
    
    # Test connections before use
    pool_pre_ping=True,
    
    # Connection lifetime
    pool_recycle=3600,
    
    # Enable logging for debugging
    echo=False,  # Set to True for SQL query logging
)
```

### Async Task Management

```python
import asyncio
from typing import List, Awaitable, TypeVar

T = TypeVar('T')

async def process_batch(items: List[T], batch_size: int = 10) -> List[T]:
    """Process items in batches to avoid overwhelming resources."""
    results = []
    
    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        batch_results = await asyncio.gather(
            *[process_item(item) for item in batch],
            return_exceptions=True
        )
        
        # Handle exceptions appropriately
        for result in batch_results:
            if isinstance(result, Exception):
                logger.error("Failed to process item", error=str(result))
            else:
                results.append(result)
    
    return results

async def with_timeout(coro: Awaitable[T], timeout: float) -> T:
    """Add timeout to any async operation."""
    try:
        return await asyncio.wait_for(coro, timeout=timeout)
    except asyncio.TimeoutError:
        logger.error("Operation timed out", timeout=timeout)
        raise
```

### Memory Management

```python
import weakref
from typing import Dict, Any

class RequestScopedCache:
    """Cache that automatically cleans up after request."""
    
    def __init__(self):
        self._cache: Dict[str, Any] = {}
        self._cleanup_callbacks: List[Callable] = []
    
    def get(self, key: str) -> Any:
        return self._cache.get(key)
    
    def set(self, key: str, value: Any) -> None:
        self._cache[key] = value
    
    def register_cleanup(self, callback: Callable) -> None:
        """Register cleanup callback."""
        self._cleanup_callbacks.append(callback)
    
    def cleanup(self) -> None:
        """Clean up resources."""
        for callback in self._cleanup_callbacks:
            try:
                callback()
            except Exception as e:
                logger.error("Cleanup callback failed", error=str(e))
        
        self._cache.clear()
        self._cleanup_callbacks.clear()

# Use weak references to avoid memory leaks
_request_caches: weakref.WeakValueDictionary = weakref.WeakValueDictionary()
```

## Best Practices Summary

1. **Always use contextvars** for request-scoped data instead of global variables
2. **Configure structured logging** with automatic context propagation  
3. **Use context managers** for database transactions and resource management
4. **Implement proper connection pooling** for database and external service connections  
5. **Test context isolation** to ensure no data bleeding between requests
6. **Handle exceptions properly** in async contexts with appropriate logging
7. **Use dependency injection** instead of global state for configuration and resources
8. **Monitor performance** and optimize connection pools based on actual usage patterns
9. **Clean up resources** properly using async context managers and lifecycle hooks
10. **Batch async operations** when processing multiple items to avoid resource exhaustion

Following these patterns will ensure your async Python services are scalable, maintainable, and free from the concurrency issues that can plague async applications.