---
date: 2025-12-08 15:55:59 IST
researcher: Elliot Steene
git_commit: 951cea5984b8f5494f381d4d1a269e1fee39830e
branch: main
repository: polypy
topic: "HTTP API Integration with Application Orchestrator for ENG-007"
tags: [research, codebase, application-orchestrator, http-api, aiohttp, health-checks, stats, eng-007]
status: complete
last_updated: 2025-12-08
last_updated_by: Elliot Steene
---

# Research: HTTP API Integration with Application Orchestrator for ENG-007

**Date**: 2025-12-08 15:55:59 IST
**Researcher**: Elliot Steene
**Git Commit**: 951cea5984b8f5494f381d4d1a269e1fee39830e
**Branch**: main
**Repository**: polypy

## Research Question

How does the existing Application Orchestrator (PolyPy class in src/app.py) currently work, and how would an HTTP API server integrate with it to expose the `get_stats()` and `is_healthy()` methods as REST endpoints for ENG-007?

## Summary

The PolyPy class in `src/app.py` serves as the main application orchestrator, managing 6 core components through a carefully ordered 10-step startup sequence and reverse-order shutdown. It exposes two key monitoring methods:

1. **`get_stats()` (lines 256-346)**: Returns comprehensive statistics from all components in a nested dictionary structure
2. **`is_healthy()` (lines 348-359)**: Performs multi-condition health checks on the running state and worker processes

The codebase currently has **HTTP client patterns** (using aiohttp.ClientSession in LifecycleController for Gamma API calls) but **no HTTP server**. For ENG-007, an aiohttp.web-based HTTP server would integrate as:

- **Startup**: Step 10 (after LifecycleController, before ConnectionRecycler)
- **Shutdown**: Step 8 (after ConnectionRecycler, before LifecycleController)

This positioning ensures all data components are operational before the HTTP server accepts connections, and connections are closed before core systems begin shutdown.

## Detailed Findings

### Component 1: PolyPy Application Lifecycle

**Location**: `src/app.py:21-360`

#### Startup Sequence (10 Steps)

The `start()` method (lines 78-159) initializes components in strict dependency order:

1. **AssetRegistry** (line 96) - Central registry with no dependencies
2. **MessageParser** (line 100) - Shared parser instance
3. **WorkerManager init** (lines 104-108) - Creates worker pool and queues
4. **MessageRouter init** (lines 111-114) - Requires worker queues
5. **WorkerManager.start()** (line 118) - Spawns worker processes
6. **MessageRouter.start()** (line 122) - Starts routing loop task
7. **ConnectionPool init** (lines 126-131) - Requires registry, router, parser
8. **ConnectionPool.start()** (line 133) - Starts subscription loop
9. **LifecycleController init + start** (lines 137-145) - Market discovery/expiration
10. **ConnectionRecycler init + start** (lines 147-155) - Connection monitoring

**Running Flag**: Set to `True` at line 158 only after all components successfully started.

#### Shutdown Sequence (Reverse Order)

The `_stop()` method (lines 180-212) shuts down in exact reverse:

1. ConnectionRecycler.stop() (lines 182-186)
2. LifecycleController.stop() (lines 188-192)
3. ConnectionPool.stop() (lines 194-198)
4. MessageRouter.stop() (lines 200-204)
5. WorkerManager.stop() (lines 206-210) - **Synchronous**, 5s timeout per worker

Each component stop is wrapped in try/except to ensure shutdown continues despite errors.

#### Signal Handling

The `run()` method (lines 225-254) provides automated lifecycle:

- Registers SIGINT and SIGTERM handlers (lines 238-240)
- Signal handler sets `_shutdown_event` (asyncio.Event)
- Main loop blocks on `_shutdown_event.wait()` (line 245)
- Finally block ensures `stop()` is called (line 248)

#### HTTP Server Integration Point

**Recommended Startup Position**: Between LifecycleController and ConnectionRecycler (new Step 10)

```python
# After Step 9 (LifecycleController)
await self._lifecycle.start()
logger.info("✓ LifecycleController started")

# NEW Step 10: HTTP Server
if self._enable_http:
    self._http_server = HTTPServer(
        app=self,
        port=self._http_port,
    )
    await self._http_server.start()
    logger.info(f"✓ HTTP server started on port {self._http_port}")

# Step 11 (formerly 10): ConnectionRecycler
self._recycler = ConnectionRecycler(...)
```

**Recommended Shutdown Position**: After ConnectionRecycler, before LifecycleController (new Step 8)

```python
# After Step 9 (ConnectionRecycler)
if self._recycler:
    await self._recycler.stop()

# NEW Step 8: HTTP Server
if self._http_server:
    try:
        await self._http_server.stop()
    except Exception as e:
        logger.error(f"Error stopping HTTP server: {e}")

# Step 7 (formerly 8): LifecycleController
if self._lifecycle:
    await self._lifecycle.stop()
```

**Rationale**:
- HTTP server needs all data sources operational (registry, pool, router, workers)
- Stop accepting connections before core systems transition
- Clean API boundary: External API → Lifecycle → Data Processing → Workers

### Component 2: get_stats() Method

**Location**: `src/app.py:256-346`

#### Return Structure

Returns `dict[str, Any]` with 7 top-level keys (all initialized as empty dicts):

```python
{
    "running": bool,
    "registry": {...},
    "pool": {...},
    "router": {...},
    "workers": {...},
    "lifecycle": {...},
    "recycler": {...}
}
```

#### Data Sources by Component

**Registry Stats** (lines 273-287):
- `"total_markets"`: `len(self._registry._assets)` - Direct access to private dict
- `"pending"`: `self._registry.get_pending_count()` - Assets awaiting subscription
- `"subscribed"`: `len(self._registry.get_by_status(AssetStatus.SUBSCRIBED))`
- `"expired"`: `len(self._registry.get_by_status(AssetStatus.EXPIRED))`

**Pool Stats** (lines 289-295):
- `"connection_count"`: `self._pool.connection_count` property (line 88-91)
- `"active_connections"`: `self._pool.active_connection_count` property (line 93-101)
- `"total_capacity"`: `self._pool.get_total_capacity()` method (line 103-114)
- `"stats"`: `self._pool.get_connection_stats()` method (line 311-337) - Returns list of per-connection dicts

**Router Stats** (lines 297-307):
- Accesses `self._router.stats` property (RouterStats dataclass)
- Fields: `messages_routed`, `messages_dropped`, `batches_sent`, `queue_full_events`, `routing_errors`, `avg_latency_ms`
- `"queue_depths"`: `self._router.get_queue_depths()` - Returns dict with async_queue and worker_queue sizes

**Workers Stats** (lines 309-325):
- `"alive_count"`: `self._workers.get_alive_count()` - Count of alive worker processes
- `"is_healthy"`: `self._workers.is_healthy()` - True if all workers alive and running
- `"worker_stats"`: Dictionary comprehension transforming `WorkerStats` objects
  - Source: `self._workers.get_stats()` - Non-blocking drain of stats queue
  - Per-worker fields: `messages_processed`, `updates_applied`, `snapshots_received`, `avg_processing_time_us`, `orderbook_count`, `memory_usage_mb`

**Lifecycle Stats** (lines 327-331):
- `"is_running"`: `self._lifecycle.is_running` property
- `"known_market_count"`: `self._lifecycle.known_market_count` - Count of discovered markets

**Recycler Stats** (lines 333-344):
- Accesses `self._recycler.stats` property (RecycleStats dataclass)
- Fields: `recycles_initiated`, `recycles_completed`, `recycles_failed`, `success_rate`, `markets_migrated`, `avg_downtime_ms`
- `"active_recycles"`: `list(self._recycler.get_active_recycles())` - Connection IDs currently recycling

#### Conditional Population

All component stats use conditional checks:

```python
if self._component:
    stats["component"] = {...}
```

This allows the method to work during partial initialization (e.g., if startup fails midway).

### Component 3: is_healthy() Method

**Location**: `src/app.py:348-359`

#### Health Check Logic

Sequential checks with early return:

1. **Running Flag** (lines 349-350):
   ```python
   if not self._running:
       return False
   ```
   No logging on this path.

2. **Workers Health** (lines 352-354):
   ```python
   if self._workers and not self._workers.is_healthy():
       logger.warning("Health check failed: workers unhealthy")
       return False
   ```
   - Calls `WorkerManager.is_healthy()` (src/worker.py:414-424)
   - Returns True only if `_running` and all worker processes alive
   - Logs warning before returning False

3. **Connection Check** (lines 356-357):
   ```python
   if self._pool and self._pool.active_connection_count == 0:
       logger.debug("Health check: no active connections")
   ```
   - **Important**: Only logs debug message, does NOT return False
   - Execution continues to line 359

4. **Return True** (line 359):
   ```python
   return True
   ```
   Returns healthy if all checks pass.

#### Health Conditions

**Healthy when**:
- `self._running` is True
- `self._workers` is either None OR `self._workers.is_healthy()` is True
- (Pool connection count is informational only)

**Unhealthy when**:
- `self._running` is False, OR
- `self._workers` exists AND `self._workers.is_healthy()` is False

#### HTTP Endpoint Mapping

For ENG-007 `/health` endpoint:

```python
async def handle_health(request):
    is_healthy = app.is_healthy()

    # Build component health breakdown
    components = {
        "registry": app._registry is not None,
        "pool": app._pool is not None and app._pool.active_connection_count > 0,
        "router": app._router is not None,
        "workers": app._workers is not None and app._workers.is_healthy(),
        "lifecycle": app._lifecycle is not None and app._lifecycle.is_running,
        "recycler": app._recycler is not None and app._recycler.is_running,
    }

    response_data = {
        "healthy": is_healthy,
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "components": components,
    }

    status = 200 if is_healthy else 503
    return web.json_response(response_data, status=status)
```

### Component 4: Component Interfaces

All component files located and key methods identified:

#### AssetRegistry
- **File**: `src/registry/asset_registry.py`
- **Key Methods**: `get()`, `get_by_status()`, `get_pending_count()`, `connection_stats()`, `add_asset()`, `mark_subscribed()`, `mark_expired()`
- **No lifecycle methods** (no start/stop)

#### ConnectionPool
- **File**: `src/connection/pool.py`
- **Key Methods**: `start()`, `stop()`, `connection_count`, `active_connection_count`, `get_total_capacity()`, `get_connection_stats()`
- **Lifecycle**: async start/stop

#### MessageRouter
- **File**: `src/router.py`
- **Key Methods**: `start()`, `stop()`, `stats`, `get_queue_depths()`, `route_message()`
- **Lifecycle**: async start/stop

#### WorkerManager
- **File**: `src/worker.py`
- **Key Methods**: `start()`, `stop()`, `get_stats()`, `is_healthy()`, `get_alive_count()`
- **Lifecycle**: **synchronous** start/stop (multiprocessing)

#### LifecycleController
- **File**: `src/lifecycle/controller.py`
- **Key Methods**: `start()`, `stop()`, `is_running`, `known_market_count`
- **Lifecycle**: async start/stop
- **HTTP Client**: Uses `aiohttp.ClientSession` for Gamma API calls

#### ConnectionRecycler
- **File**: `src/lifecycle/recycler.py`
- **Key Methods**: `start()`, `stop()`, `stats`, `is_running`, `get_active_recycles()`
- **Lifecycle**: async start/stop

### Component 5: Existing HTTP/API Patterns

**Important Finding**: The codebase has **HTTP client patterns** but **no HTTP server**.

#### Existing HTTP Client Usage

**Location**: `src/lifecycle/controller.py:63-146`

The LifecycleController uses `aiohttp.ClientSession` for external API calls:

**Session Lifecycle Pattern**:
```python
# Startup (lines 87-89)
async def start(self) -> None:
    self._running = True
    self._session = aiohttp.ClientSession()

# Shutdown (lines 138-141)
async def stop(self) -> None:
    if self._session:
        await self._session.close()
        self._session = None
```

**API Request Pattern** (src/lifecycle/api.py:57-82):
- Retry logic with exponential backoff (3 retries, 2^n delay)
- Uses `async with session.get()` context manager
- Exception handling for `aiohttp.ClientError`
- `response.raise_for_status()` for HTTP errors
- `await response.json()` for async parsing

**Pagination Pattern** (src/lifecycle/api.py:21-54):
- Offset-based pagination with `limit` and `offset` params
- Loop continues until empty or partial page
- Collects all results and returns

**Configuration** (pyproject.toml:7-16):
```toml
dependencies = [
    "aiohttp>=3.13.2",
    ...
]
```

#### No HTTP Server Found

**Search Results**: No aiohttp server, no route handlers, no listening sockets

The application is:
- WebSocket client (connects to Polymarket CLOB API)
- HTTP client (fetches from Gamma API)
- **Not an HTTP server**

#### HTTP Server Implementation for ENG-007

Since no existing patterns exist, the HTTP server would be a new component. Recommended pattern based on existing aiohttp usage:

**Server Initialization**:
```python
from aiohttp import web

class HTTPServer:
    __slots__ = ("_app", "_port", "_runner", "_site")

    def __init__(self, app: PolyPy, port: int = 8080):
        self._app = app
        self._port = port
        self._runner = None
        self._site = None

    async def start(self) -> None:
        # Create aiohttp app with routes
        web_app = web.Application()
        web_app.router.add_get("/health", self._handle_health)
        web_app.router.add_get("/stats", self._handle_stats)

        # Start server
        self._runner = web.AppRunner(web_app)
        await self._runner.setup()

        self._site = web.TCPSite(self._runner, "0.0.0.0", self._port)
        await self._site.start()

    async def stop(self) -> None:
        if self._site:
            await self._site.stop()
        if self._runner:
            await self._runner.cleanup()
```

This follows the same async lifecycle pattern as existing components.

### Component 6: Asyncio Integration Considerations

The PolyPy class uses asyncio throughout:

**Event Loop Management**:
- `run()` method gets event loop: `loop = asyncio.get_event_loop()` (line 232)
- Signal handlers added to loop (lines 238-240)
- All async operations use `await`

**Task Creation Pattern**:
All background components create named tasks:
```python
self._task = asyncio.create_task(
    self._loop_function(),
    name="component-name",
)
```

**Task Cancellation Pattern**:
```python
if self._task:
    self._task.cancel()
    try:
        await self._task
    except asyncio.CancelledError:
        pass
```

**HTTP Server Compatibility**:
- aiohttp.web integrates natively with asyncio
- `web.AppRunner` and `web.TCPSite` are async context managers
- Server start/stop methods are async
- No thread conflicts (all single event loop)

**Important**: WorkerManager is the only component with synchronous lifecycle (lines 313-388), using multiprocessing for worker processes. HTTP server should follow async pattern like other components.

### Component 7: HTTP Server Implementation Requirements

Based on ENG-007 ticket and existing patterns:

#### Constructor Parameters
```python
class PolyPy:
    def __init__(
        self,
        num_workers: int = 4,
        enable_http: bool = False,  # NEW
        http_port: int = 8080,      # NEW
    ) -> None:
```

#### Lifecycle Integration

**Startup** (src/app.py:78-159):
- Add HTTP server after LifecycleController.start() (line 145)
- Check `if self._enable_http:` before creating server
- Create and start server
- Log success with port number

**Shutdown** (src/app.py:180-212):
- Add HTTP server stop after ConnectionRecycler.stop() (line 186)
- Follow existing error handling pattern with try/except
- Log errors but continue shutdown

#### Endpoint Requirements

**GET /health**:
- Returns JSON with `healthy`, `timestamp`, `components` fields
- 200 OK when healthy, 503 Service Unavailable when unhealthy
- Calls `self.is_healthy()` for overall status
- Component breakdown from checking each `self._component` reference
- Response time target: < 10ms

**GET /stats**:
- Returns JSON (same structure as `get_stats()`)
- Calls `self.get_stats()` directly
- Returns complete nested dictionary
- Response time target: < 10ms

#### Error Handling

**Port Binding Failures**:
- Log error if port already in use
- Allow application to continue if `enable_http=True` but server fails
- Set `self._http_server = None` on failure

**Graceful Shutdown**:
- Stop accepting new connections first
- Wait for in-flight requests with timeout
- Close server runner and site

## Code References

### Core Application Files
- `src/app.py:21-360` - PolyPy class (main orchestrator)
- `src/app.py:78-159` - Startup sequence (start method)
- `src/app.py:180-212` - Shutdown sequence (_stop method)
- `src/app.py:256-346` - Statistics collection (get_stats method)
- `src/app.py:348-359` - Health checking (is_healthy method)
- `src/main.py:55-112` - Application entry point

### Component Files
- `src/registry/asset_registry.py:22-38` - AssetRegistry initialization
- `src/connection/pool.py:66-149` - ConnectionPool lifecycle
- `src/router.py:102-332` - MessageRouter lifecycle
- `src/worker.py:263-388` - WorkerManager lifecycle
- `src/lifecycle/controller.py:45-146` - LifecycleController lifecycle
- `src/lifecycle/recycler.py:78-348` - ConnectionRecycler lifecycle

### HTTP Client Patterns (Reference)
- `src/lifecycle/controller.py:87-141` - aiohttp ClientSession lifecycle
- `src/lifecycle/api.py:21-82` - HTTP request patterns with retry logic
- `src/lifecycle/api.py:84-140` - JSON parsing and validation

### Configuration
- `pyproject.toml:7-16` - Dependencies (includes aiohttp>=3.13.2)

## Architecture Documentation

### Current Patterns

**Component Lifecycle Pattern**:
All managed components follow this pattern:
1. Constructor stores configuration and initializes state
2. `async def start()` - Sets `_running = True`, creates background tasks
3. `async def stop()` - Sets `_running = False`, cancels tasks, cleans up
4. Properties/methods for stats and health checks

**Startup Ordering Strategy**:
- Independent components first (Registry, Parser)
- Infrastructure components second (WorkerManager, Router)
- Data processing third (ConnectionPool)
- External integrations last (LifecycleController, Recycler)

**Shutdown Ordering Strategy**:
- External integrations first (stop accepting new work)
- Data processing second (drain queues)
- Infrastructure third (stop workers)
- Never explicitly clean up independent components (garbage collected)

**Error Handling Strategy**:
- Startup: Fast-fail on first error, log and raise
- Shutdown: Continue on errors, log each error but don't halt sequence

### HTTP Server Integration Pattern

**New Component: HTTPServer**

Following existing patterns:
- Async lifecycle (start/stop methods)
- Error handling in try/except blocks
- Conditional instantiation based on `enable_http` flag
- Named startup/shutdown position in sequence
- Minimal dependencies (only needs reference to PolyPy instance)

**Data Access**:
- HTTP handlers call `app.get_stats()` and `app.is_healthy()`
- No direct component access (encapsulation via orchestrator)
- Read-only operations (no state mutations)

**Performance Considerations**:
- Stats and health checks are fast operations (< 10ms target)
- No blocking I/O in endpoint handlers
- No heavy computation (just dict collection and boolean checks)
- Response directly from in-memory state

## Historical Context (from thoughts/)

### Ticket Documents
- `thoughts/elliotsteene/tickets/ENG-006-application-orchestrator.md` - Full Application Orchestrator implementation specification (parent ticket for ENG-007)
- `thoughts/elliotsteene/tickets/ENG-007-http-api-stats-health.md` - HTTP API for stats and health endpoints (current ticket)
- `thoughts/elliotsteene/tickets/ENG-004-market-lifecycle-controller.md` - Market lifecycle controller (integrated component)
- `thoughts/elliotsteene/tickets/ENG-005-connection-recycler.md` - Connection recycler (integrated component)

### Implementation Plans
- `thoughts/shared/plans/2025-12-05-ENG-006-application-orchestrator.md` - Detailed implementation plan for Application Orchestrator including current state analysis, startup/shutdown sequence, all 6 component dependencies
- `thoughts/shared/plans/2025-12-04-ENG-004-market-lifecycle-controller.md` - Implementation plan for lifecycle controller
- `thoughts/shared/plans/2025-12-05-ENG-005-connection-recycler.md` - Connection recycler implementation plan

### Research Documents
- `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md` - Research on all remaining system components (Components 5-10), including Component 10 (Full Application Orchestrator) specification, health checks, and production deployment requirements
- `thoughts/shared/research/2025-12-03-orderbook-state-management.md` - Research document on orderbook state management

### Phase 8 Context
- `lessons/008_production.md` - Phase 8 (Production Observability) lesson file, which aligns with ENG-006 and ENG-007 work focused on production-ready system deployment

## Related Research

This research builds on:
- Component 10 (Application Orchestrator) specification in remaining-system-spec-components research
- ENG-006 implementation (Application Orchestrator now complete in src/app.py)
- Phase 8 learning curriculum (Production Observability and Recovery)

## Key Insights

1. **Application is Fully Functional**: The PolyPy orchestrator is complete with all 6 components integrated. ENG-007 is an enhancement to add external monitoring, not a prerequisite for core functionality.

2. **Clean Separation**: The `get_stats()` and `is_healthy()` methods provide a clean API boundary. HTTP endpoints simply expose these existing methods over the network.

3. **No HTTP Server Precedent**: While the codebase uses aiohttp as an HTTP client, there's no existing HTTP server pattern. The implementation will establish this pattern.

4. **Lifecycle Position Matters**: HTTP server must start after all data sources are operational and stop before they begin shutting down. The recommended positions (Step 10 startup, Step 8 shutdown) satisfy these constraints.

5. **Stats Collection is Comprehensive**: The `get_stats()` method already collects all relevant metrics from all components. No additional instrumentation needed.

6. **Health Check is Conservative**: The `is_healthy()` method currently only fails on running=False or workers unhealthy. Pool connection count logs debug but doesn't fail health. This may need adjustment based on production requirements.

7. **Conditional Component Checks**: Both stats and health methods handle None components gracefully, allowing them to work during partial initialization or after partial shutdown.

8. **Asyncio Native**: The entire application is asyncio-based (except WorkerManager which uses multiprocessing). aiohttp.web integration will be seamless.

## Integration Checklist for ENG-007

Based on this research, the HTTP API implementation should:

- [ ] Add `enable_http` and `http_port` parameters to `PolyPy.__init__`
- [ ] Create `HTTPServer` class following existing component lifecycle pattern
- [ ] Implement `/health` endpoint calling `app.is_healthy()` with component breakdown
- [ ] Implement `/stats` endpoint calling `app.get_stats()`
- [ ] Integrate HTTP server startup at Step 10 (after LifecycleController)
- [ ] Integrate HTTP server shutdown at Step 8 (after ConnectionRecycler)
- [ ] Handle port binding errors gracefully (log and continue)
- [ ] Add aiohttp to dependencies (already present: aiohttp>=3.13.2)
- [ ] Write integration tests for both endpoints
- [ ] Verify response time < 10ms for both endpoints
- [ ] Document HTTP API in README or docs/http-api.md
- [ ] Test health endpoint returns 503 when unhealthy
- [ ] Test graceful shutdown closes HTTP server before components

## Open Questions

These questions from ENG-007 ticket should be resolved before implementation:

1. **Authentication**: Should we add authentication for these endpoints?
   - Current state: No authentication in codebase
   - Consideration: Production deployment may need auth or IP whitelisting

2. **Request Logging**: Do we need request logging middleware?
   - Current state: Component logging uses structlog
   - Consideration: Access logs for HTTP requests (timing, status codes)

3. **Metrics Endpoint**: Should Prometheus `/metrics` endpoint be in this ticket or separate?
   - Current state: Only JSON endpoints specified
   - Consideration: Prometheus format is different from JSON stats

4. **Granular Stats**: Do we want to expose more granular component stats (per-worker, per-connection)?
   - Current state: `get_stats()` includes per-worker and per-connection stats
   - Answer: Already available in stats endpoint response
