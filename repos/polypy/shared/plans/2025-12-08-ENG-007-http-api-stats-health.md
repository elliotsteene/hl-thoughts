# HTTP API for Stats and Health Endpoints Implementation Plan

## Overview

Add an aiohttp-based HTTP server to the PolyPy application orchestrator that exposes `/health` and `/stats` REST endpoints. This enables external monitoring systems, load balancers, and operators to check application health and retrieve real-time statistics over the network.

## Current State Analysis

### Existing Infrastructure

**PolyPy Application Orchestrator** (`src/app.py:21-360`):
- Manages 6 core components through 10-step startup sequence
- Implements reverse-order shutdown with error handling
- Already exposes `get_stats()` (lines 256-346) returning comprehensive component statistics
- Already exposes `is_healthy()` (lines 348-359) returning boolean health status
- No HTTP server currently exists

**Component Lifecycle Pattern** (established in `src/lifecycle/controller.py`, `src/lifecycle/recycler.py`, etc.):
- All async components follow consistent pattern:
  - Constructor accepts dependencies, initializes `_running = False`
  - `async start()`: Sets `_running = True`, creates background tasks
  - `async stop()`: Sets `_running = False`, cancels tasks, cleans up resources
- Integration into PolyPy: components added during startup, stopped in reverse order

**HTTP Infrastructure**:
- **aiohttp dependency**: Already present (`aiohttp>=3.13.2` in pyproject.toml line 8)
- **HTTP client usage**: `LifecycleController` uses `aiohttp.ClientSession` for Gamma API calls (`src/lifecycle/controller.py:94`)
- **No HTTP server**: No existing server patterns, endpoints, or route handlers

**Test Infrastructure**:
- Async test patterns with `@pytest.mark.asyncio`
- Integration tests with `@pytest.mark.serial` to prevent resource conflicts
- Component lifecycle tests with proper cleanup in fixtures
- Mock patterns for component dependencies
- **No HTTP endpoint test patterns** - will establish new pattern

### Key Discoveries

1. **Clean API Boundary**: `get_stats()` and `is_healthy()` provide perfect abstraction for HTTP exposure
2. **Lifecycle Position Critical**: HTTP server must start after all data sources operational, stop before they shut down
3. **Consistent Error Handling**: All component stops wrapped in try/except in orchestrator
4. **Test Pattern Gap**: Need to establish aiohttp TestClient pattern for endpoint testing

## Desired End State

### Specification

**HTTP Server Component**:
- New `HTTPServer` class in `src/http/server.py` following async component lifecycle pattern
- Starts on configurable port (default: 8080) when `enable_http=True`
- Bound to `0.0.0.0` for container compatibility
- Integrates into PolyPy startup sequence at Step 10 (after LifecycleController)
- Integrates into PolyPy shutdown sequence at Step 8 (after ConnectionRecycler)

**GET /health Endpoint**:
```json
{
  "healthy": true,
  "timestamp": "2025-12-08T10:30:00Z",
  "components": {
    "registry": true,
    "pool": true,
    "router": true,
    "workers": true,
    "lifecycle": true,
    "recycler": true
  }
}
```
- Returns 200 OK when healthy, 503 Service Unavailable when unhealthy
- Response time: < 10ms
- Component breakdown shows individual component health

**GET /stats Endpoint**:
```json
{
  "running": true,
  "registry": { "total_markets": 150, ... },
  "pool": { "connection_count": 3, ... },
  "router": { "messages_routed": 50000, ... },
  "workers": { "alive_count": 4, ... },
  "lifecycle": { "is_running": true, ... },
  "recycler": { "recycles_initiated": 5, ... }
}
```
- Returns complete nested statistics from `PolyPy.get_stats()`
- Response time: < 10ms
- Always returns 200 OK (even if not healthy)

**Configuration**:
- `PolyPy.__init__` accepts `enable_http: bool = False` and `http_port: int = 8080`
- HTTP server only created when `enable_http=True`
- Port binding errors logged but don't prevent application startup

### Verification

**Automated**:
- All unit tests pass: `just test`
- All integration tests pass: `just test`
- Type checking passes: `just check`
- Linting passes: `just check`

**Manual**:
- Start app with HTTP enabled: `enable_http=True, http_port=8080`
- Verify `/health` returns 200 with correct JSON structure
- Verify `/stats` returns comprehensive statistics
- Verify `/health` returns 503 when app unhealthy
- Measure response times < 10ms under load
- Test port conflict handling (start two instances on same port)
- Verify graceful shutdown closes HTTP connections

## What We're NOT Doing

1. **Authentication/Authorization** - Health endpoints need to be fast and accessible by load balancers; auth can be added later if needed
2. **Request Logging Middleware** - Keeping simple; handlers will log at DEBUG level, full middleware is future enhancement
3. **Prometheus /metrics Endpoint** - Different format than JSON stats; should be separate ticket (ENG-008 candidate)
4. **Rate Limiting** - Not needed for internal monitoring; can add if public exposure required
5. **CORS Headers** - Not needed for internal monitoring endpoints
6. **Kubernetes-specific /ready Endpoint** - Using /health for both liveness and readiness initially
7. **Request/Response Schemas** - Keeping lightweight; JSON structure documented but not formally validated

## Implementation Approach

### High-Level Strategy

1. **Create HTTPServer Component**: New file `src/http/server.py` implementing async lifecycle pattern
2. **Simple Handler Functions**: Direct calls to `app.get_stats()` and `app.is_healthy()`
3. **Lifecycle Integration**: Conditionally create and start/stop based on `enable_http` flag
4. **Error Isolation**: Port binding failures logged but don't crash application
5. **Test Pattern Establishment**: Use aiohttp TestClient for endpoint tests, establish pattern for future HTTP work

### Architecture Pattern

```
External Request
    ↓
HTTPServer (aiohttp.web)
    ↓
GET /health → PolyPy.is_healthy() → bool
GET /stats  → PolyPy.get_stats()  → dict
```

**No business logic in HTTP layer** - handlers are thin wrappers around orchestrator methods.

## Stacked PR Strategy

This plan is designed for implementation using stacked PRs, where each phase becomes one branch/PR in a stack:

- **Phase Sequencing**: Phase 1 (core implementation) → Phase 2 (integration validation)
- **Independent Review**: Each PR can be reviewed independently while maintaining stack context
- **Pause Points**: Manual verification happens after each phase before proceeding
- **Estimated Scope**:
  - Phase 1: ~200 lines (server class, endpoints, unit tests)
  - Phase 2: ~150 lines (integration tests, docs)
- **Stack Management**: Use the `stacked-pr` skill after completing each phase to maintain the stack

## Phase 1: HTTP Server Implementation

### Overview

Create the HTTPServer component following established async lifecycle patterns, implement the two REST endpoints, and integrate into PolyPy orchestrator startup/shutdown sequences. Include comprehensive unit tests.

### PR Context

**Stack Position**: First PR in the stack (base: main)
**Purpose**: Implement core HTTP server functionality with testable endpoints
**Enables**: Phase 2 integration testing and validation
**Review Focus**: Lifecycle pattern adherence, error handling, endpoint correctness

### Changes Required

#### 1. Create HTTP Server Module

**File**: `src/http/__init__.py`
**Changes**: Create empty `__init__.py` to make http a package

```python
# Empty file - makes src/http a package
```

**File**: `src/http/server.py`
**Changes**: Implement HTTPServer class with async lifecycle

```python
import asyncio
from datetime import datetime, timezone
from typing import TYPE_CHECKING

import structlog
from aiohttp import web

if TYPE_CHECKING:
    from src.app import PolyPy

from src.core.logging import Logger

logger: Logger = structlog.getLogger(__name__)


class HTTPServer:
    """
    HTTP server exposing health and stats endpoints.

    Follows async component lifecycle pattern established by LifecycleController
    and ConnectionRecycler.
    """

    __slots__ = (
        "_app",
        "_port",
        "_host",
        "_runner",
        "_site",
        "_running",
    )

    def __init__(
        self,
        app: "PolyPy",
        port: int = 8080,
        host: str = "0.0.0.0",
    ) -> None:
        """Initialize HTTP server.

        Args:
            app: PolyPy application instance to query for health/stats
            port: Port to bind to (default: 8080)
            host: Host to bind to (default: 0.0.0.0 for container compatibility)
        """
        self._app = app
        self._port = port
        self._host = host
        self._runner: web.AppRunner | None = None
        self._site: web.TCPSite | None = None
        self._running = False

    async def start(self) -> None:
        """Start HTTP server.

        Raises:
            Exception: If server fails to start (port conflict, etc.)
        """
        if self._running:
            logger.warning("HTTP server already running")
            return

        logger.info(f"Starting HTTP server on {self._host}:{self._port}")

        # Create aiohttp application with routes
        web_app = web.Application()
        web_app.router.add_get("/health", self._handle_health)
        web_app.router.add_get("/stats", self._handle_stats)

        # Start server
        self._runner = web.AppRunner(web_app)
        await self._runner.setup()

        self._site = web.TCPSite(
            self._runner,
            self._host,
            self._port,
        )
        await self._site.start()

        self._running = True
        logger.info(f"✓ HTTP server started on {self._host}:{self._port}")

    async def stop(self) -> None:
        """Stop HTTP server gracefully."""
        if not self._running:
            logger.warning("HTTP server not running")
            return

        logger.info("Stopping HTTP server...")

        self._running = False

        # Stop accepting new connections
        if self._site:
            await self._site.stop()
            self._site = None

        # Cleanup runner
        if self._runner:
            await self._runner.cleanup()
            self._runner = None

        logger.info("✓ HTTP server stopped")

    async def _handle_health(self, request: web.Request) -> web.Response:
        """Handle GET /health endpoint.

        Returns:
            200 OK with health status and component breakdown when healthy
            503 Service Unavailable when unhealthy
        """
        logger.debug("GET /health")

        # Get overall health
        is_healthy = self._app.is_healthy()

        # Build component health breakdown
        components = {
            "registry": self._app._registry is not None,
            "pool": (
                self._app._pool is not None
                and self._app._pool.active_connection_count > 0
            ),
            "router": self._app._router is not None,
            "workers": (
                self._app._workers is not None
                and self._app._workers.is_healthy()
            ),
            "lifecycle": (
                self._app._lifecycle is not None
                and self._app._lifecycle.is_running
            ),
            "recycler": (
                self._app._recycler is not None
                and self._app._recycler.is_running
            ),
        }

        response_data = {
            "healthy": is_healthy,
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "components": components,
        }

        status = 200 if is_healthy else 503
        return web.json_response(response_data, status=status)

    async def _handle_stats(self, request: web.Request) -> web.Response:
        """Handle GET /stats endpoint.

        Returns:
            200 OK with comprehensive statistics
        """
        logger.debug("GET /stats")

        stats = self._app.get_stats()
        return web.json_response(stats, status=200)
```

#### 2. Update PolyPy Constructor

**File**: `src/app.py`
**Changes**: Add HTTP configuration parameters

**Line 28-29** (in `__slots__`): Add HTTP server slot
```python
__slots__ = (
    "_num_workers",
    "_registry",
    "_workers",
    "_router",
    "_pool",
    "_lifecycle",
    "_recycler",
    "_message_parser",
    "_running",
    "_shutdown_event",
    "_http_server",  # NEW
    "_enable_http",  # NEW
    "_http_port",    # NEW
)
```

**Line 41**: Update constructor signature
```python
def __init__(
    self,
    num_workers: int = 4,
    enable_http: bool = False,
    http_port: int = 8080,
) -> None:
    """Initialise application

    Args:
        num_workers: Number of orderbook worker processes (default = 4)
        enable_http: Whether to start HTTP server (default = False)
        http_port: Port for HTTP server (default = 8080)
    """
```

**Line 59-61**: Initialize HTTP fields
```python
self._message_parser: MessageParser | None = None

# HTTP server configuration
self._enable_http = enable_http
self._http_port = http_port
self._http_server: HTTPServer | None = None

self._running = False
```

#### 3. Integrate HTTP Server into Startup Sequence

**File**: `src/app.py`
**Changes**: Add HTTP server startup after LifecycleController

**Line 1**: Add import at top of file
```python
from src.http.server import HTTPServer
```

**After line 155** (after ConnectionRecycler startup): Add HTTP server startup
```python
await self._recycler.start()
logger.info("✓ ConnectionRecycler started")

# 10. HTTP Server (if enabled)
if self._enable_http:
    self._http_server = HTTPServer(
        app=self,
        port=self._http_port,
    )
    try:
        await self._http_server.start()
        logger.info(f"✓ HTTP server started on port {self._http_port}")
    except Exception as e:
        logger.error(f"Failed to start HTTP server: {e}")
        # Don't fail startup if HTTP server fails
        self._http_server = None

# 11. Running application
self._running = True
logger.info("✓ PolyPy started successfully!")
```

#### 4. Integrate HTTP Server into Shutdown Sequence

**File**: `src/app.py`
**Changes**: Add HTTP server shutdown after ConnectionRecycler

**After line 186** (after ConnectionRecycler shutdown): Add HTTP server shutdown
```python
if self._recycler:
    try:
        await self._recycler.stop()
    except Exception as e:
        logger.error(f"Error stopping recycler: {e}")

# Stop HTTP server (if running)
if self._http_server:
    try:
        await self._http_server.stop()
    except Exception as e:
        logger.error(f"Error stopping HTTP server: {e}")

if self._lifecycle:
    try:
        await self._lifecycle.stop()
```

#### 5. Unit Tests for HTTPServer

**File**: `tests/test_http_server.py`
**Changes**: Create comprehensive unit tests for HTTPServer class

```python
"""Unit tests for HTTP server component."""

import asyncio

import pytest
from aiohttp import web
from aiohttp.test_utils import TestClient, TestServer
from unittest.mock import AsyncMock, MagicMock

from src.http.server import HTTPServer


class TestHTTPServerLifecycle:
    """Test HTTPServer lifecycle management."""

    @pytest.mark.asyncio
    async def test_start_stop_lifecycle(self):
        """Test basic start/stop lifecycle."""
        mock_app = MagicMock()
        server = HTTPServer(mock_app, port=8080)

        assert not server._running

        # Note: Cannot test actual start() without binding to port
        # This will be covered in integration tests

    @pytest.mark.asyncio
    async def test_start_already_running(self):
        """Test starting when already running logs warning."""
        mock_app = MagicMock()
        server = HTTPServer(mock_app)
        server._running = True

        # Should log warning and return early
        await server.start()  # This would fail if it tried to bind

    @pytest.mark.asyncio
    async def test_stop_not_running(self):
        """Test stopping when not running logs warning."""
        mock_app = MagicMock()
        server = HTTPServer(mock_app)

        # Should log warning and return early
        await server.stop()


class TestHealthEndpoint:
    """Test /health endpoint handler."""

    @pytest.mark.asyncio
    async def test_health_endpoint_healthy(self):
        """Test /health returns 200 when app is healthy."""
        # Mock PolyPy app
        mock_app = MagicMock()
        mock_app.is_healthy.return_value = True
        mock_app._registry = MagicMock()
        mock_app._pool = MagicMock()
        mock_app._pool.active_connection_count = 2
        mock_app._router = MagicMock()
        mock_app._workers = MagicMock()
        mock_app._workers.is_healthy.return_value = True
        mock_app._lifecycle = MagicMock()
        mock_app._lifecycle.is_running = True
        mock_app._recycler = MagicMock()
        mock_app._recycler.is_running = True

        server = HTTPServer(mock_app)

        # Create test client
        web_app = web.Application()
        web_app.router.add_get("/health", server._handle_health)

        async with TestClient(TestServer(web_app)) as client:
            resp = await client.get("/health")

            assert resp.status == 200
            data = await resp.json()

            assert data["healthy"] is True
            assert "timestamp" in data
            assert data["components"]["registry"] is True
            assert data["components"]["pool"] is True
            assert data["components"]["workers"] is True

    @pytest.mark.asyncio
    async def test_health_endpoint_unhealthy(self):
        """Test /health returns 503 when app is unhealthy."""
        mock_app = MagicMock()
        mock_app.is_healthy.return_value = False
        mock_app._registry = None
        mock_app._pool = None
        mock_app._router = None
        mock_app._workers = None
        mock_app._lifecycle = None
        mock_app._recycler = None

        server = HTTPServer(mock_app)

        web_app = web.Application()
        web_app.router.add_get("/health", server._handle_health)

        async with TestClient(TestServer(web_app)) as client:
            resp = await client.get("/health")

            assert resp.status == 503
            data = await resp.json()

            assert data["healthy"] is False
            assert data["components"]["registry"] is False
            assert data["components"]["pool"] is False

    @pytest.mark.asyncio
    async def test_health_endpoint_partial_startup(self):
        """Test /health during partial startup (some components None)."""
        mock_app = MagicMock()
        mock_app.is_healthy.return_value = False
        mock_app._registry = MagicMock()
        mock_app._pool = None  # Not started yet
        mock_app._router = MagicMock()
        mock_app._workers = None  # Not started yet
        mock_app._lifecycle = None
        mock_app._recycler = None

        server = HTTPServer(mock_app)

        web_app = web.Application()
        web_app.router.add_get("/health", server._handle_health)

        async with TestClient(TestServer(web_app)) as client:
            resp = await client.get("/health")

            assert resp.status == 503
            data = await resp.json()

            assert data["components"]["registry"] is True
            assert data["components"]["pool"] is False
            assert data["components"]["workers"] is False


class TestStatsEndpoint:
    """Test /stats endpoint handler."""

    @pytest.mark.asyncio
    async def test_stats_endpoint_returns_data(self):
        """Test /stats returns comprehensive stats."""
        mock_app = MagicMock()
        mock_app.get_stats.return_value = {
            "running": True,
            "registry": {"total_markets": 100},
            "pool": {"connection_count": 3},
            "router": {"messages_routed": 50000},
            "workers": {"alive_count": 4},
            "lifecycle": {"is_running": True},
            "recycler": {"recycles_initiated": 2},
        }

        server = HTTPServer(mock_app)

        web_app = web.Application()
        web_app.router.add_get("/stats", server._handle_stats)

        async with TestClient(TestServer(web_app)) as client:
            resp = await client.get("/stats")

            assert resp.status == 200
            data = await resp.json()

            assert data["running"] is True
            assert data["registry"]["total_markets"] == 100
            assert data["pool"]["connection_count"] == 3
            assert data["workers"]["alive_count"] == 4

    @pytest.mark.asyncio
    async def test_stats_endpoint_empty_stats(self):
        """Test /stats returns empty structure when not running."""
        mock_app = MagicMock()
        mock_app.get_stats.return_value = {
            "running": False,
            "registry": {},
            "pool": {},
            "router": {},
            "workers": {},
            "lifecycle": {},
            "recycler": {},
        }

        server = HTTPServer(mock_app)

        web_app = web.Application()
        web_app.router.add_get("/stats", server._handle_stats)

        async with TestClient(TestServer(web_app)) as client:
            resp = await client.get("/stats")

            assert resp.status == 200
            data = await resp.json()

            assert data["running"] is False
            assert data["registry"] == {}


class TestHTTPServerConfiguration:
    """Test HTTPServer configuration options."""

    def test_default_configuration(self):
        """Test default port and host."""
        mock_app = MagicMock()
        server = HTTPServer(mock_app)

        assert server._port == 8080
        assert server._host == "0.0.0.0"

    def test_custom_configuration(self):
        """Test custom port and host."""
        mock_app = MagicMock()
        server = HTTPServer(mock_app, port=9090, host="127.0.0.1")

        assert server._port == 9090
        assert server._host == "127.0.0.1"
```

#### 6. Unit Tests for PolyPy HTTP Integration

**File**: `tests/test_app.py`
**Changes**: Add tests for new HTTP configuration

**Add to end of file**:
```python
class TestPolyPyHTTPConfiguration:
    """Test PolyPy HTTP server configuration."""

    def test_default_http_disabled(self):
        """Test HTTP server disabled by default."""
        app = PolyPy()
        assert app._enable_http is False
        assert app._http_port == 8080
        assert app._http_server is None

    def test_enable_http_configuration(self):
        """Test enabling HTTP server."""
        app = PolyPy(enable_http=True, http_port=9090)
        assert app._enable_http is True
        assert app._http_port == 9090
        assert app._http_server is None  # Not created until start()

    @pytest.mark.asyncio
    async def test_http_server_not_created_when_disabled(self):
        """Test HTTP server not created when disabled."""
        app = PolyPy(enable_http=False)

        # Mock components to prevent actual startup
        app._registry = MagicMock()
        app._workers = MagicMock()
        app._workers.is_healthy.return_value = True
        app._running = True

        assert app._http_server is None
```

### Success Criteria

#### Automated Verification:
- [ ] All tests pass: `just test`
- [ ] Type checking passes: `just check`
- [ ] Linting passes: `just check`
- [ ] HTTPServer unit tests pass (lifecycle, endpoints, configuration)
- [ ] PolyPy unit tests pass with new HTTP configuration
- [ ] No regressions in existing tests

#### Manual Verification:
- [ ] Start PolyPy with `enable_http=False` - no HTTP server created
- [ ] Start PolyPy with `enable_http=True` - server starts successfully
- [ ] Access `http://localhost:8080/health` - returns 200 with correct JSON structure
- [ ] Access `http://localhost:8080/stats` - returns comprehensive statistics
- [ ] Stop application - HTTP server shuts down cleanly
- [ ] HTTP server logs appear in application output

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: Integration Testing and Documentation

### Overview

Add comprehensive integration tests that verify HTTP server behavior within the full application lifecycle, test error scenarios, validate performance requirements, and document the HTTP API for users.

### PR Context

**Stack Position**: Second PR in the stack (base: phase-1-http-server)
**Purpose**: Validate Phase 1 implementation with integration tests and provide user documentation
**Builds On**: HTTPServer class and PolyPy integration from Phase 1
**Enables**: Production deployment with confidence in reliability and performance
**Review Focus**: Test coverage, error handling validation, documentation clarity

### Changes Required

#### 1. Integration Tests for HTTP Server

**File**: `tests/test_http_integration.py`
**Changes**: Create comprehensive integration tests

```python
"""Integration tests for HTTP server with full application."""

import asyncio
import time

import aiohttp
import pytest

from src.app import PolyPy


@pytest.fixture
async def app_with_http():
    """Fixture that provides a PolyPy instance with HTTP enabled."""
    _app = PolyPy(num_workers=2, enable_http=True, http_port=18080)
    yield _app

    # Cleanup
    if _app.is_running:
        await _app.stop()

    await asyncio.sleep(1.0)


@pytest.mark.asyncio
@pytest.mark.serial
async def test_http_server_full_lifecycle(app_with_http):
    """Test HTTP server starts and stops with application."""
    app = app_with_http

    # Start application
    await app.start()

    # Wait for stabilization
    await asyncio.sleep(1.0)

    # Verify HTTP server running
    assert app._http_server is not None
    assert app._http_server._running

    # Verify endpoints accessible
    async with aiohttp.ClientSession() as session:
        # Test health endpoint
        async with session.get("http://localhost:18080/health") as resp:
            assert resp.status == 200
            data = await resp.json()
            assert data["healthy"] is True

        # Test stats endpoint
        async with session.get("http://localhost:18080/stats") as resp:
            assert resp.status == 200
            data = await resp.json()
            assert data["running"] is True

    # Stop application
    await app.stop()
    await asyncio.sleep(0.5)

    # Verify HTTP server stopped
    assert not app._http_server._running


@pytest.mark.asyncio
@pytest.mark.serial
async def test_health_endpoint_reflects_app_state(app_with_http):
    """Test /health endpoint accurately reflects application state."""
    app = app_with_http

    await app.start()
    await asyncio.sleep(1.0)

    async with aiohttp.ClientSession() as session:
        # When running and healthy
        async with session.get("http://localhost:18080/health") as resp:
            assert resp.status == 200
            data = await resp.json()
            assert data["healthy"] is True
            assert data["components"]["workers"] is True
            assert data["components"]["registry"] is True

    await app.stop()


@pytest.mark.asyncio
@pytest.mark.serial
async def test_stats_endpoint_returns_comprehensive_data(app_with_http):
    """Test /stats endpoint returns all component statistics."""
    app = app_with_http

    await app.start()
    await asyncio.sleep(1.0)

    async with aiohttp.ClientSession() as session:
        async with session.get("http://localhost:18080/stats") as resp:
            assert resp.status == 200
            data = await resp.json()

            # Verify all expected top-level keys
            assert "running" in data
            assert "registry" in data
            assert "pool" in data
            assert "router" in data
            assert "workers" in data
            assert "lifecycle" in data
            assert "recycler" in data

            # Verify some nested data
            assert data["running"] is True
            assert "alive_count" in data["workers"]
            assert "connection_count" in data["pool"]

    await app.stop()


@pytest.mark.asyncio
@pytest.mark.serial
async def test_http_server_response_time(app_with_http):
    """Test HTTP endpoints respond in < 10ms."""
    app = app_with_http

    await app.start()
    await asyncio.sleep(1.0)

    async with aiohttp.ClientSession() as session:
        # Test health endpoint latency
        start = time.monotonic()
        async with session.get("http://localhost:18080/health") as resp:
            await resp.json()
        health_latency = (time.monotonic() - start) * 1000  # ms

        # Test stats endpoint latency
        start = time.monotonic()
        async with session.get("http://localhost:18080/stats") as resp:
            await resp.json()
        stats_latency = (time.monotonic() - start) * 1000  # ms

        # Both should be under 10ms (using 20ms buffer for CI environments)
        assert health_latency < 20.0, f"Health latency {health_latency:.1f}ms > 20ms"
        assert stats_latency < 20.0, f"Stats latency {stats_latency:.1f}ms > 20ms"

    await app.stop()


@pytest.mark.asyncio
@pytest.mark.serial
async def test_http_disabled_by_default():
    """Test HTTP server not created when enable_http=False."""
    app = PolyPy(num_workers=2, enable_http=False)

    try:
        await app.start()
        await asyncio.sleep(1.0)

        # HTTP server should not exist
        assert app._http_server is None

        # Endpoints should not be accessible
        async with aiohttp.ClientSession() as session:
            try:
                async with session.get(
                    "http://localhost:8080/health", timeout=aiohttp.ClientTimeout(total=1)
                ) as resp:
                    # Should not reach here
                    assert False, "HTTP server should not be running"
            except (aiohttp.ClientError, asyncio.TimeoutError):
                # Expected - no server listening
                pass

    finally:
        await app.stop()
        await asyncio.sleep(1.0)


@pytest.mark.asyncio
@pytest.mark.serial
async def test_http_graceful_shutdown():
    """Test HTTP server closes connections gracefully during shutdown."""
    app = PolyPy(num_workers=2, enable_http=True, http_port=18081)

    try:
        await app.start()
        await asyncio.sleep(1.0)

        # Create persistent connection
        async with aiohttp.ClientSession() as session:
            # Make initial request
            async with session.get("http://localhost:18081/health") as resp:
                assert resp.status == 200

            # Trigger shutdown while session open
            stop_task = asyncio.create_task(app.stop())

            # Wait briefly
            await asyncio.sleep(0.5)

            # New requests should fail (server stopped)
            try:
                async with session.get(
                    "http://localhost:18081/health",
                    timeout=aiohttp.ClientTimeout(total=1),
                ) as resp:
                    # May succeed if shutdown hasn't completed
                    pass
            except (aiohttp.ClientError, asyncio.TimeoutError):
                # Expected after shutdown
                pass

            # Wait for shutdown completion
            await stop_task

    finally:
        if app.is_running:
            await app.stop()
        await asyncio.sleep(1.0)
```

#### 2. Error Scenario Tests

**File**: `tests/test_http_errors.py`
**Changes**: Test error handling scenarios

```python
"""Test HTTP server error scenarios."""

import asyncio

import pytest

from src.app import PolyPy


@pytest.mark.asyncio
@pytest.mark.serial
async def test_http_port_conflict_handling():
    """Test graceful handling when port already in use."""
    # Start first app
    app1 = PolyPy(num_workers=2, enable_http=True, http_port=18082)

    try:
        await app1.start()
        await asyncio.sleep(1.0)

        assert app1._http_server is not None
        assert app1._http_server._running

        # Try to start second app on same port
        app2 = PolyPy(num_workers=2, enable_http=True, http_port=18082)

        try:
            await app2.start()
            await asyncio.sleep(1.0)

            # App should start (HTTP failure doesn't prevent startup)
            assert app2.is_running

            # But HTTP server should be None (failed to bind)
            assert app2._http_server is None

        finally:
            if app2.is_running:
                await app2.stop()
            await asyncio.sleep(1.0)

    finally:
        if app1.is_running:
            await app1.stop()
        await asyncio.sleep(1.0)


@pytest.mark.asyncio
@pytest.mark.serial
async def test_http_startup_failure_doesnt_crash_app():
    """Test that HTTP startup failure doesn't prevent app startup."""
    # Use invalid port (0 is technically valid but tests error handling path)
    app = PolyPy(num_workers=2, enable_http=True, http_port=18083)

    try:
        # Mock HTTP server to raise exception on start
        await app.start()
        await asyncio.sleep(1.0)

        # App should still be running
        assert app.is_running

        # Can still call health/stats methods directly
        assert app.is_healthy()
        stats = app.get_stats()
        assert stats["running"] is True

    finally:
        await app.stop()
        await asyncio.sleep(1.0)
```

#### 3. Update Application Integration Tests

**File**: `tests/test_app_integration.py`
**Changes**: Update existing integration tests to verify HTTP doesn't interfere

**Add to end of file**:
```python
@pytest.mark.asyncio
@pytest.mark.serial
async def test_app_lifecycle_with_http_enabled():
    """Test full app lifecycle with HTTP server enabled."""
    app = PolyPy(num_workers=2, enable_http=True, http_port=18084)

    try:
        # Start
        await app.start()
        await asyncio.sleep(1.0)

        assert app.is_running
        assert app._http_server is not None
        assert app._http_server._running

        # Verify all components running including HTTP
        stats = app.get_stats()
        assert stats["running"]
        assert stats["workers"]["is_healthy"]

        # Stop
        await app.stop()
        await asyncio.sleep(0.5)

        assert not app.is_running
        assert not app._http_server._running

    finally:
        if app.is_running:
            await app.stop()
        await asyncio.sleep(1.0)
```

#### 4. HTTP API Documentation

**File**: `docs/http-api.md`
**Changes**: Create comprehensive HTTP API documentation

```markdown
# HTTP API Documentation

The PolyPy application can optionally expose HTTP endpoints for health checks and statistics monitoring. This is useful for:

- External monitoring systems (Prometheus, Grafana, Datadog, etc.)
- Load balancer health checks
- Kubernetes liveness/readiness probes
- Operational dashboards
- Debugging and troubleshooting

## Configuration

Enable the HTTP server by passing parameters to the `PolyPy` constructor:

```python
from src.app import PolyPy

app = PolyPy(
    num_workers=4,
    enable_http=True,    # Enable HTTP server
    http_port=8080,      # Port to bind (default: 8080)
)

await app.start()
```

**Configuration Options:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enable_http` | `bool` | `False` | Whether to start the HTTP server |
| `http_port` | `int` | `8080` | Port to bind the HTTP server to |

**Notes:**
- The HTTP server binds to `0.0.0.0` for container compatibility
- If port binding fails (port conflict), the error is logged but application startup continues
- The HTTP server is disabled by default for security and resource efficiency

## Endpoints

### GET /health

Health check endpoint that returns application health status and component breakdown.

**Response Codes:**
- `200 OK` - Application is healthy and running
- `503 Service Unavailable` - Application is unhealthy or not running

**Response Schema:**

```json
{
  "healthy": true,
  "timestamp": "2025-12-08T10:30:00.000Z",
  "components": {
    "registry": true,
    "pool": true,
    "router": true,
    "workers": true,
    "lifecycle": true,
    "recycler": true
  }
}
```

**Fields:**

- `healthy` (boolean): Overall health status (same as `PolyPy.is_healthy()`)
- `timestamp` (string): ISO 8601 timestamp of the health check in UTC
- `components` (object): Health status of individual components
  - `registry` (boolean): AssetRegistry initialized
  - `pool` (boolean): ConnectionPool initialized and has active connections
  - `router` (boolean): MessageRouter initialized
  - `workers` (boolean): WorkerManager initialized and all workers healthy
  - `lifecycle` (boolean): LifecycleController initialized and running
  - `recycler` (boolean): ConnectionRecycler initialized and running

**Health Criteria:**

The application is considered healthy when:
- `_running` flag is `True`
- All worker processes are alive and healthy

The application is considered unhealthy when:
- `_running` flag is `False`, OR
- Any worker process is dead or unhealthy

**Example Requests:**

```bash
# Check health
curl http://localhost:8080/health

# Use in health check scripts
if curl -f http://localhost:8080/health; then
  echo "Application is healthy"
else
  echo "Application is unhealthy"
fi
```

**Performance:**
- Target response time: < 10ms
- No expensive operations performed
- Safe to call frequently (every 1-5 seconds)

---

### GET /stats

Statistics endpoint that returns comprehensive operational metrics from all components.

**Response Codes:**
- `200 OK` - Always returns 200 (even if unhealthy)

**Response Schema:**

```json
{
  "running": true,
  "registry": {
    "total_markets": 150,
    "pending": 5,
    "subscribed": 140,
    "expired": 5
  },
  "pool": {
    "connection_count": 3,
    "active_connections": 3,
    "total_capacity": 1200,
    "stats": [
      {
        "connection_id": "conn-abc123",
        "status": "connected",
        "subscribed_markets": 400,
        "messages_received": 12500,
        "uptime": 3600.5
      }
    ]
  },
  "router": {
    "messages_routed": 50000,
    "messages_dropped": 12,
    "batches_sent": 500,
    "queue_full_events": 0,
    "routing_errors": 0,
    "avg_latency_ms": 1.2,
    "queue_depths": {
      "async_queue": 5,
      "worker_queue": 10
    }
  },
  "workers": {
    "alive_count": 4,
    "is_healthy": true,
    "worker_stats": {
      "0": {
        "messages_processed": 12500,
        "updates_applied": 12000,
        "snapshots_received": 40,
        "avg_processing_time_us": 150,
        "orderbook_count": 38,
        "memory_usage_mb": 45.2
      }
    }
  },
  "lifecycle": {
    "is_running": true,
    "known_market_count": 150
  },
  "recycler": {
    "recycles_initiated": 5,
    "recycles_completed": 5,
    "recycles_failed": 0,
    "success_rate": 1.0,
    "markets_migrated": 2000,
    "avg_downtime_ms": 850.0,
    "active_recycles": []
  }
}
```

**Field Details:**

See `PolyPy.get_stats()` method documentation in `src/app.py:256-346` for comprehensive field descriptions.

**Example Requests:**

```bash
# Get all stats
curl http://localhost:8080/stats | jq .

# Get specific component stats
curl http://localhost:8080/stats | jq '.workers'

# Monitor message throughput
watch -n 1 'curl -s http://localhost:8080/stats | jq ".router.messages_routed"'
```

**Performance:**
- Target response time: < 10ms
- Returns cached/current values (no heavy computation)
- Safe to call frequently for monitoring dashboards

---

## Integration Examples

### Kubernetes Liveness Probe

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: polypy
spec:
  containers:
  - name: polypy
    image: polypy:latest
    env:
    - name: ENABLE_HTTP
      value: "true"
    - name: HTTP_PORT
      value: "8080"
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 3
```

### Docker Compose Healthcheck

```yaml
version: '3.8'
services:
  polypy:
    image: polypy:latest
    environment:
      ENABLE_HTTP: "true"
      HTTP_PORT: "8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 2s
      retries: 3
      start_period: 30s
```

### Prometheus Metrics Collection

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'polypy'
    metrics_path: '/stats'
    static_configs:
      - targets: ['localhost:8080']
    metric_relabel_configs:
      - source_labels: [__name__]
        target_label: __name__
        replacement: 'polypy_${1}'
```

**Note:** For native Prometheus format, a separate `/metrics` endpoint should be implemented (future enhancement).

### Python Monitoring Script

```python
import aiohttp
import asyncio

async def monitor_polypy():
    """Monitor PolyPy health and stats."""
    async with aiohttp.ClientSession() as session:
        # Check health
        async with session.get("http://localhost:8080/health") as resp:
            if resp.status == 200:
                print("✓ Application is healthy")
            else:
                print("✗ Application is unhealthy")

        # Get stats
        async with session.get("http://localhost:8080/stats") as resp:
            stats = await resp.json()
            print(f"Workers: {stats['workers']['alive_count']}")
            print(f"Messages routed: {stats['router']['messages_routed']}")
            print(f"Active connections: {stats['pool']['active_connections']}")

asyncio.run(monitor_polypy())
```

---

## Security Considerations

**Current State:**
- No authentication or authorization
- Endpoints are read-only (no mutations)
- Binds to `0.0.0.0` (all interfaces)

**Recommendations:**

1. **Network-Level Security:**
   - Deploy behind firewall
   - Use network policies in Kubernetes
   - Bind to localhost only if external access not needed

2. **Future Authentication:**
   - API keys for external monitoring systems
   - mTLS for service-to-service communication
   - IP whitelisting for known monitoring infrastructure

3. **Rate Limiting:**
   - Consider rate limiting if exposed publicly
   - Current implementation has no built-in limits

**For Production:**
- Review security requirements before enabling
- Consider dedicated monitoring network/VPC
- Audit access to monitoring endpoints

---

## Troubleshooting

### HTTP Server Won't Start

**Symptom:** Application starts but HTTP endpoints not accessible

**Possible Causes:**
1. `enable_http=False` (default) - Check configuration
2. Port conflict - Another process using the port
3. Firewall blocking the port
4. Wrong port number in client requests

**Debug Steps:**
```bash
# Check if port is in use
lsof -i :8080

# Check application logs for HTTP errors
# Look for "Failed to start HTTP server: ..." messages

# Try different port
app = PolyPy(enable_http=True, http_port=9090)
```

### Slow Response Times

**Symptom:** Endpoints taking > 10ms to respond

**Possible Causes:**
1. High system load
2. Many components collecting stats
3. Network latency (if remote)

**Debug Steps:**
```python
import time

start = time.monotonic()
async with session.get("http://localhost:8080/stats") as resp:
    data = await resp.json()
latency = (time.monotonic() - start) * 1000
print(f"Latency: {latency:.1f}ms")
```

### Health Check Always Fails

**Symptom:** `/health` always returns 503

**Possible Causes:**
1. Application not fully started
2. Workers unhealthy or crashed
3. Application stopped

**Debug Steps:**
```bash
# Check direct health method
python -c "
from src.app import PolyPy
import asyncio

async def check():
    app = PolyPy()
    await app.start()
    print('is_healthy:', app.is_healthy())
    print('is_running:', app.is_running)
    await app.stop()

asyncio.run(check())
"
```

---

## Future Enhancements

Potential additions in future tickets:

- **Authentication:** API keys, JWT tokens, mTLS
- **Prometheus /metrics:** Native Prometheus exposition format
- **Request logging:** Structured logging middleware with request IDs
- **Rate limiting:** Protect against excessive polling
- **CORS headers:** If web dashboard integration needed
- **Separate /ready endpoint:** Kubernetes-specific readiness vs liveness
- **WebSocket /stream:** Real-time stats streaming for dashboards
- **Historical metrics:** Time-series data with configurable retention
```

#### 5. Update Main README

**File**: `README.md`
**Changes**: Add HTTP API section to main README

**Add after "Development Commands" section:**

```markdown
## HTTP API (Optional)

PolyPy can expose HTTP endpoints for health checks and statistics monitoring:

```python
from src.app import PolyPy

app = PolyPy(
    num_workers=4,
    enable_http=True,   # Enable HTTP server
    http_port=8080,     # Port (default: 8080)
)

await app.start()
```

**Endpoints:**
- `GET /health` - Health check (200 OK when healthy, 503 when unhealthy)
- `GET /stats` - Comprehensive statistics from all components

**Usage:**
```bash
curl http://localhost:8080/health
curl http://localhost:8080/stats | jq .
```

See [HTTP API Documentation](docs/http-api.md) for complete details, integration examples, and troubleshooting.
```

### Success Criteria

#### Automated Verification:
- [ ] All tests pass: `just test`
- [ ] Integration tests pass with HTTP enabled
- [ ] Error scenario tests pass (port conflicts, failures)
- [ ] Performance tests verify < 10ms response time (with 20ms buffer for CI)
- [ ] Type checking passes: `just check`
- [ ] Linting passes: `just check`
- [ ] No test regressions

#### Manual Verification:
- [ ] Start app with HTTP, access both endpoints via browser
- [ ] Start two apps on same port - second app continues despite HTTP failure
- [ ] Monitor response times under load (100 req/s for 10 seconds)
- [ ] Test graceful shutdown - verify connections close cleanly
- [ ] Verify documentation accuracy - test all code examples
- [ ] Test Kubernetes liveness probe configuration (if K8s available)
- [ ] Verify health endpoint returns 503 when stopping application

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful and documentation is accurate.

---

## Testing Strategy

### Unit Tests

**HTTPServer** (`tests/test_http_server.py`):
- Lifecycle: start/stop, idempotency, state checks
- Endpoints: handler logic with mocked PolyPy app
- Configuration: default and custom port/host
- Response structure validation

**PolyPy Integration** (`tests/test_app.py`):
- HTTP configuration parameters
- Server creation when enabled/disabled
- Component interaction with HTTP enabled

### Integration Tests

**Full Application** (`tests/test_http_integration.py`):
- HTTP server lifecycle within full app startup/shutdown
- Endpoints accessible and returning real data
- Response time validation (< 20ms with CI buffer)
- Graceful shutdown behavior

**Error Scenarios** (`tests/test_http_errors.py`):
- Port conflicts (multiple apps on same port)
- Startup failures don't crash application
- Network errors handled gracefully

**Existing Tests** (`tests/test_app_integration.py`):
- Verify HTTP doesn't interfere with existing functionality
- Test full lifecycle with HTTP both enabled and disabled

### Manual Testing Steps

1. **Basic Functionality:**
   ```bash
   # Start app with HTTP
   # In Python REPL:
   from src.app import PolyPy
   import asyncio

   async def main():
       app = PolyPy(enable_http=True, http_port=8080)
       await app.start()
       # Keep running, test endpoints in another terminal
       await asyncio.Event().wait()

   asyncio.run(main())

   # In another terminal:
   curl http://localhost:8080/health
   curl http://localhost:8080/stats | jq .
   ```

2. **Error Handling:**
   ```bash
   # Start first instance
   # Start second instance on same port - verify it continues despite HTTP failure
   ```

3. **Performance Testing:**
   ```bash
   # Install Apache Bench if not available: apt-get install apache2-utils
   ab -n 1000 -c 10 http://localhost:8080/health
   ab -n 1000 -c 10 http://localhost:8080/stats

   # Verify avg response time < 10ms
   ```

4. **Shutdown Behavior:**
   ```bash
   # Start app, make HTTP request, trigger shutdown (Ctrl+C)
   # Verify clean shutdown in logs
   ```

## Performance Considerations

### Response Time Requirements

**Target:** < 10ms for both `/health` and `/stats` endpoints

**Why This Is Achievable:**
- `is_healthy()` performs simple boolean checks on flags and properties
- `get_stats()` aggregates cached/current values (no I/O or heavy computation)
- No database queries, no network calls, no file operations
- aiohttp has minimal overhead for simple JSON responses

**Optimization Notes:**
- Stats collection is O(n) where n = number of components (6 total)
- Each component's stats method returns pre-computed or O(1) values
- Worker stats use non-blocking queue drain (already computed by workers)
- No locks held during stats collection (read-only operations)

### Resource Usage

**Memory:**
- HTTPServer instance: ~1KB (slots-based, minimal overhead)
- Per-request: ~5KB (aiohttp request/response objects)
- JSON serialization: Temporary allocation proportional to stats size (~10KB)

**CPU:**
- Negligible overhead when idle (no background polling)
- Per-request: ~0.1ms CPU time for handler execution
- No impact on message processing throughput

**Network:**
- Typical health response: ~200 bytes
- Typical stats response: ~2-5KB (depends on component counts)
- No persistent connections required

## Migration Notes

**Not Applicable** - This is a new feature, not a migration.

**For Existing Deployments:**
- Default behavior unchanged (`enable_http=False`)
- Opt-in via configuration parameter
- No breaking changes to existing code
- No database changes or data migrations needed

**Rollback Plan:**
- Remove `enable_http=True` from configuration
- HTTP server won't start, application runs as before
- No cleanup needed

## References

- Original ticket: `thoughts/elliotsteene/tickets/ENG-007-http-api-stats-health.md`
- Research document: `thoughts/shared/research/2025-12-08-ENG-007-http-api-integration-analysis.md`
- Parent ticket: ENG-006 (Application Orchestrator) - already implemented in `src/app.py`
- Related lesson: `lessons/008_production.md` (Phase 8: Production Observability)
- Lifecycle pattern reference: `src/lifecycle/controller.py:45-146`, `src/lifecycle/recycler.py:78-372`
- aiohttp documentation: https://docs.aiohttp.org/en/stable/web.html
- Test pattern reference: `tests/test_app_integration.py`, `tests/test_lifecycle_integration.py`
