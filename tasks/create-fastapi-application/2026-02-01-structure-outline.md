---
task: create-fastapi-application
type: structure-outline
repo: analytics
branch: main
sha: no-commits-yet
---

# FastAPI Skeleton Application Implementation Plan

Create a skeleton FastAPI application with factory pattern, async lifespan, and versioned health/readiness endpoints. The implementation follows patterns documented in ARCHITECTURE.md while creating only the minimal structure needed.

## Current State

- `src/commerce_intelligence/__init__.py` contains only a placeholder `main()` function printing "Hello from commerce-intelligence!"
- No FastAPI application, routers, or configuration exists
- Dependencies installed: `fastapi[standard]>=0.128.0`, `pydantic>=2.12.5`, `pydantic-settings>=2.12.0`, `uvicorn>=0.40.0`
- Entry point `commerce-intelligence` configured in `pyproject.toml` pointing to `commerce_intelligence:main`
- Architecture documentation exists in ARCHITECTURE.md but nothing is implemented

## Desired End State

- Working FastAPI application starting via `commerce-intelligence` CLI command
- Application factory pattern with `create_app()` function in `main.py`
- Async lifespan context manager for startup/shutdown lifecycle
- `/api/v1/health` endpoint returning `{"status": "healthy"}`
- `/api/v1/readiness` endpoint returning `{"status": "ready", "checks": {}}`
- Configuration using pydantic-settings with `Settings` class
- Uvicorn serving the app with reload capability for development

## What we're not doing

- Database connections or BigQuery client setup
- Analytics service layer, query building, or business logic
- Authentication or authorization
- Full router structure for orders/customers/revenue
- Caching layer
- Production deployment configuration (Docker, etc.)
- Creating empty placeholder directories for future features

### Patterns to follow

#### Application Factory Pattern

Use `create_app()` factory function returning configured FastAPI instance. Enables testing with different configurations.

```python
# src/commerce_intelligence/main.py
def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.api_title,
        version=settings.api_version,
        lifespan=lifespan,
    )
    app.include_router(health_router, prefix="/api/v1")
    return app
```

#### Async Lifespan Context Manager

FastAPI's modern lifecycle management using `@asynccontextmanager` (replaces deprecated `on_event` decorators).

```python
from contextlib import asynccontextmanager
from typing import AsyncIterator

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    # Startup
    yield
    # Shutdown
```

#### Configuration with pydantic-settings (from ARCHITECTURE.md:759-774)

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    api_title: str = "Commerce Intelligence API"
    api_version: str = "0.1.0"
    host: str = "0.0.0.0"
    port: int = 8000
    debug: bool = False

    class Config:
        env_file = ".env"
```

#### Dependency Injection with @lru_cache

```python
from functools import lru_cache

@lru_cache()
def get_settings() -> Settings:
    return Settings()
```

### Design Summary

Full discussion doc: [rpi/tasks/create-fastapi-application/2026-02-01-design-discussion.md](2026-02-01-design-discussion.md)

#### create_app() in separate main.py module

`create_app()` lives in `src/commerce_intelligence/main.py` with `__init__.py` containing the `main()` CLI entry point. Follows architecture document structure and separates concerns.

#### Readiness structured for future dependency checks

Readiness endpoint returns `{"status": "ready", "checks": {}}` to allow adding database/service checks later without changing the response schema.

#### Single health.py router

Both `/health` and `/readiness` in one file since they serve similar operational purposes. Split later if complexity grows.

#### Minimal directory structure

Create only what's needed: `main.py`, `core/config.py`, and `api/routers/health.py`. No empty placeholders.

#### Uvicorn with factory and reload

Use string import with `factory=True` to enable hot reload during development, controlled by `settings.debug`.

---

## Phase 1: Configuration Module

Create the core configuration using pydantic-settings. This establishes the foundation for all other components.

### File Changes

- **`src/commerce_intelligence/core/__init__.py`**: Empty `__init__.py` for core package
- **`src/commerce_intelligence/core/config.py`**: Settings class with API title, version, host, port, and debug flag. Includes `get_settings()` factory function with `@lru_cache()`.

### Validation

Run `python -c "from commerce_intelligence.core.config import get_settings; print(get_settings())"` to verify settings load correctly with defaults.

---

## Phase 2: Health Router

Create the health and readiness endpoints as a versioned router.

### File Changes

- **`src/commerce_intelligence/api/__init__.py`**: Empty `__init__.py` for api package
- **`src/commerce_intelligence/api/routers/__init__.py`**: Empty `__init__.py` for routers package
- **`src/commerce_intelligence/api/routers/health.py`**: Router with `/health` and `/readiness` endpoints. Health returns simple status, readiness returns status with empty checks dict.

### Validation

Run `python -c "from commerce_intelligence.api.routers.health import router; print(router.routes)"` to verify router is created with expected routes.

---

## Phase 3: Application Factory and Entry Point

Create the FastAPI application factory with lifespan and wire up the entry point.

### File Changes

- **`src/commerce_intelligence/main.py`**:
  - Async lifespan context manager (empty for now, placeholder for future startup/shutdown)
  - `create_app()` factory function creating FastAPI with title/version from settings, lifespan, and health router mounted at `/api/v1`
- **`src/commerce_intelligence/__init__.py`**: Update `main()` to import `create_app` and run with uvicorn using string import + `factory=True` for reload support

### Validation

1. Run `commerce-intelligence` command
2. Verify server starts on http://0.0.0.0:8000
3. Test `curl http://localhost:8000/api/v1/health` returns `{"status": "healthy"}`
4. Test `curl http://localhost:8000/api/v1/readiness` returns `{"status": "ready", "checks": {}}`
5. Verify OpenAPI docs available at http://localhost:8000/docs

---

## Open Questions

None - all design decisions resolved in design discussion document.
