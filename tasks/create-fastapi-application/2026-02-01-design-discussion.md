---
task: create-fastapi-application
type: design-discussion
repo: analytics
branch: main
sha: no-commits-yet
---

### Summary of change request

Create a skeleton FastAPI application running with Uvicorn that uses a factory pattern (`create_app()`) to create the application. The app should have `/health` and `/readiness` endpoints versioned under `/api/v1`, and use an async context manager lifespan function for application lifecycle management.

### Current State

- `src/commerce_intelligence/__init__.py` contains only a placeholder `main()` function that prints "Hello from commerce-intelligence!"
- No FastAPI application exists
- No routers, configuration, or application factory pattern implemented
- Dependencies are installed: `fastapi[standard]>=0.128.0`, `pydantic>=2.12.5`, `pydantic-settings>=2.12.0`, `uvicorn>=0.40.0`
- Entry point `commerce-intelligence` is configured in `pyproject.toml` pointing to `commerce_intelligence:main`
- Two architecture documents exist (ARCHITECTURE.md and ARCHITECTURE_REVISED.md) with intended patterns but no implementation

### Desired End State

- A working FastAPI application that starts via `commerce-intelligence` CLI command
- Application factory pattern with `create_app()` function
- Async lifespan context manager for startup/shutdown lifecycle
- `/api/v1/health` endpoint returning basic health status
- `/api/v1/readiness` endpoint returning readiness status
- Application served via Uvicorn
- Configuration using pydantic-settings following architecture patterns

### What we're not doing

- Database connections or BigQuery client setup (no readiness checks against database yet)
- Analytics service layer, query building, or business logic
- Authentication or authorization
- Full router structure for orders/customers/revenue (only health routes)
- Caching layer
- Production deployment configuration (Docker, etc.)

### Patterns to follow

#### Application Factory Pattern

Use a `create_app()` factory function that constructs and returns the FastAPI application. This allows for testing with different configurations and follows the pattern implied by the architecture docs.

```python
# src/commerce_intelligence/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    yield
    # Shutdown

def create_app() -> FastAPI:
    app = FastAPI(
        title="Commerce Intelligence API",
        version="0.1.0",
        lifespan=lifespan,
    )
    # Register routers
    return app
```

#### Lifespan Context Manager Pattern

FastAPI's modern approach uses an async context manager for lifecycle events (replacing deprecated `on_event` decorators).

```python
from contextlib import asynccontextmanager
from typing import AsyncIterator

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    # Startup: initialize resources
    print("Starting up...")
    yield
    # Shutdown: cleanup resources
    print("Shutting down...")
```

#### Configuration with pydantic-settings (from ARCHITECTURE.md:759-774)

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    api_title: str = "Commerce Intelligence API"
    api_version: str = "0.1.0"
    host: str = "0.0.0.0"
    port: int = 8000

    class Config:
        env_file = ".env"

settings = Settings()
```

#### Dependency Injection with @lru_cache (from ARCHITECTURE.md:783-799)

```python
from functools import lru_cache

@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

#### Router Organization

Version routers under `/api/v1` prefix:

```python
from fastapi import APIRouter

router = APIRouter(prefix="/api/v1")

@router.get("/health")
async def health():
    return {"status": "healthy"}
```

### Design Questions

#### 1. Where should `create_app()` live?

The architecture docs show `main.py` as "FastAPI app initialization" but the current entry point is in `__init__.py`.

- **Option A**: `src/commerce_intelligence/main.py` - Separate module for app creation
  ```python
  # __init__.py
  from .main import create_app

  def main() -> None:
      import uvicorn
      from .core.config import settings
      app = create_app()
      uvicorn.run(app, host=settings.host, port=settings.port)
  ```

- **Option B**: `src/commerce_intelligence/__init__.py` - Keep everything in the package root
  ```python
  # __init__.py
  def create_app() -> FastAPI:
      ...

  def main() -> None:
      ...
  ```

**Recommendation**: Option A - Separating `create_app()` into `main.py` follows the architecture document structure and keeps concerns separated. The `__init__.py` `main()` function serves as the CLI entry point that imports and runs the app.

#### 2. What should health vs readiness endpoints check?

Health and readiness have different purposes in Kubernetes/container orchestration:
- **Health (liveness)**: Is the application process alive and not deadlocked?
- **Readiness**: Is the application ready to receive traffic?

- **Option A**: Both return simple static responses (no dependencies to check yet)
  ```python
  @router.get("/health")
  async def health():
      return {"status": "healthy"}

  @router.get("/readiness")
  async def readiness():
      return {"status": "ready"}
  ```

- **Option B**: Readiness checks future dependencies (returns ready for now, but structured for extension)
  ```python
  @router.get("/readiness")
  async def readiness():
      checks = {}
      # Future: checks["database"] = await check_database()
      return {"status": "ready", "checks": checks}
  ```

**Recommendation**: Option B - Structure readiness for future database/dependency checks, but return simple success for now since no dependencies exist.

#### 3. Router file organization

- **Option A**: Single `health.py` router with both endpoints
  ```
  api/routers/health.py  # Contains /health and /readiness
  ```

- **Option B**: Separate routers following different concerns
  ```
  api/routers/health.py     # /health
  api/routers/readiness.py  # /readiness
  ```

**Recommendation**: Option A - Both endpoints serve similar operational purposes and having them in one file is simpler. Split later if they grow complex.

#### 4. Initial directory structure

Following the architecture docs but only creating what's needed:

- **Option A**: Minimal structure (only what's immediately needed)
  ```
  src/commerce_intelligence/
  ├── __init__.py
  ├── main.py              # create_app() and lifespan
  ├── core/
  │   └── config.py        # Settings
  └── api/
      └── routers/
          └── health.py    # /health and /readiness
  ```

- **Option B**: Fuller structure matching architecture (with empty placeholders)
  ```
  src/commerce_intelligence/
  ├── __init__.py
  ├── main.py
  ├── core/
  │   ├── __init__.py
  │   ├── config.py
  │   └── exceptions.py
  ├── api/
  │   ├── __init__.py
  │   ├── dependencies.py
  │   └── routers/
  │       ├── __init__.py
  │       └── health.py
  └── services/           # Empty, for future
      └── __init__.py
  ```

**Recommendation**: Option A - Create only what's needed. Empty placeholder directories/files add noise. The architecture docs serve as the blueprint for future expansion.

#### 5. How should uvicorn be invoked?

- **Option A**: Programmatic invocation in `main()`
  ```python
  def main() -> None:
      import uvicorn
      app = create_app()
      uvicorn.run(app, host="0.0.0.0", port=8000)
  ```

- **Option B**: Programmatic with reload option for development
  ```python
  def main() -> None:
      import uvicorn
      uvicorn.run(
          "commerce_intelligence.main:create_app",
          factory=True,
          host="0.0.0.0",
          port=8000,
          reload=settings.debug,
      )
  ```

**Recommendation**: Option B - Using string import with `factory=True` enables hot reload for development. The `reload` option controlled by settings allows development convenience while defaulting to off for production.

### Resolved Design Questions

(To be filled in after discussion)
