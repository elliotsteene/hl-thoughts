# Application Orchestrator Implementation Plan

## Overview

Implement the Application Orchestrator (Component 10) to tie all components together into a production-ready system. Provides unified startup/shutdown sequences, signal handling, stats aggregation, and health checks.

## Current State Analysis

**All 6 major components are implemented:**

| Component | File | Start/Stop Type | Dependencies |
|-----------|------|----------------|--------------|
| AssetRegistry | `src/registry/asset_registry.py:10` | None (data structure) | None |
| ConnectionPool | `src/connection/pool.py:48` | `async start/stop` | AssetRegistry, MessageCallback |
| MessageRouter | `src/router.py:76` | `async start/stop` | WorkerQueue list |
| WorkerManager | `src/worker.py:240` | **sync start/stop** | None |
| LifecycleController | `src/lifecycle/controller.py:23` | `async start/stop` | AssetRegistry, callbacks |
| ConnectionRecycler | `src/lifecycle/recycler.py:56` | `async start/stop` | AssetRegistry, ConnectionPool |

**Current entry point**: `src/main.py:20-113` is a Phase 1 baseline with:
- Single WebSocket connection
- 2 hardcoded asset IDs
- Synchronous orderbook updates in main event loop
- Basic signal handling

**Target**: Production orchestrator managing all 6 subsystems with proper lifecycle, stats, and health checks.

### Key Discoveries:

1. **WorkerManager has SYNC start/stop** (`src/worker.py:311-388`) - Not async! Uses `multiprocessing.Process.start()` synchronously
2. **Router requires queues before init** (`src/router.py:99-122`) - Must call `WorkerManager.get_input_queues()` before creating router
3. **ConnectionPool needs message_callback** (`src/connection/pool.py:69`) - Wire to `router.route_message`
4. **LifecycleCallback signature** (`src/lifecycle/types.py:22`) - `Callable[[str, str], Awaitable[None]]` takes (event_name, asset_id)
5. **Spec has syntax error** (`lessons/polymarket_websocket_system_spec.md:3284-3290`) - Inline import in dict comprehension, needs fixing

## Desired End State

A production-ready Application class that:
- Orchestrates all 6 components with correct startup/shutdown order
- Handles SIGINT/SIGTERM signals gracefully
- Exposes comprehensive stats via `get_stats()` method
- Provides health check via `is_healthy()` method
- Supports configuration via constructor parameters
- Completes shutdown within 30 seconds total
- Fast-fails on component startup failures

### Success Verification:

#### Automated Verification:
- [ ] All unit tests pass: `uv run pytest tests/test_app.py`
- [ ] Integration tests pass: `uv run pytest tests/test_app_integration.py`
- [ ] Linting passes: `uv run ruff check src/app.py`
- [ ] Type checking passes: `uv run mypy src/app.py`
- [ ] Can start and stop cleanly without errors

#### Manual Verification:
- [ ] Application starts all components in correct order with logging
- [ ] SIGINT (Ctrl+C) triggers graceful shutdown
- [ ] SIGTERM triggers graceful shutdown
- [ ] Stats endpoint returns accurate data for all components
- [ ] Health check reflects actual system state
- [ ] Can subscribe to 100+ markets successfully
- [ ] System runs for 5+ minutes without memory leaks
- [ ] Shutdown completes within 30 seconds

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding.

## What We're NOT Doing

- HTTP API endpoints (deferred to ENG-007)
- Prometheus metrics export (future work)
- Configuration via environment variables or config file (only constructor params)
- handler_factory for per-worker callbacks (not implemented)
- Graceful degradation (fast-fail on startup errors)
- Dry-run mode or partial startup
- Command-line argument parsing (use direct instantiation)

## Implementation Approach

1. **Create `src/app.py`** with Application class following the spec structure
2. **Critical ordering**: WorkerManager queues → Router init → Worker start → Router start
3. **Mixed sync/async**: Handle WorkerManager's synchronous methods in async context
4. **Fast-fail strategy**: If any component fails during startup, stop already-started components and raise exception
5. **Stats aggregation**: Safely collect stats from all components, handling None/uninitialized cases
6. **Signal handling**: Use asyncio event loop signal handlers with proper cleanup
7. **Logging**: Comprehensive logging at each startup/shutdown step

## Phase 1: Core Structure and Initialization

### Overview
Create the Application class skeleton with proper initialization, slots, and component storage.

### Changes Required:

#### 1. Create `src/app.py`
**File**: `src/app.py` (new file)

```python
"""
Main Application Orchestrator - Ties all components together.

Startup sequence:
1. Initialize AssetRegistry
2. Create WorkerManager and get queues
3. Create MessageRouter with worker queues
4. Start WorkerManager (sync)
5. Start MessageRouter (async)
6. Create ConnectionPool with router callback
7. Start ConnectionPool (async)
8. Create LifecycleController with callbacks
9. Start LifecycleController (async)
10. Create ConnectionRecycler
11. Start ConnectionRecycler (async)

Shutdown sequence (reverse order):
1. Stop ConnectionRecycler (async)
2. Stop LifecycleController (async)
3. Stop ConnectionPool (async)
4. Stop MessageRouter (async)
5. Stop WorkerManager (sync, 10s timeout)
"""

import asyncio
import signal
from typing import Any

import structlog

from src.connection.pool import ConnectionPool
from src.core.logging import Logger
from src.lifecycle.controller import LifecycleController
from src.lifecycle.recycler import ConnectionRecycler
from src.messages.parser import MessageParser
from src.registry.asset_registry import AssetRegistry
from src.router import MessageRouter
from src.worker import WorkerManager

logger: Logger = structlog.get_logger()


class Application:
    """
    Main application orchestrator.

    Manages lifecycle of all subsystems with proper startup/shutdown order.

    Usage:
        app = Application(num_workers=4)
        await app.start()
        # ... run until shutdown signal ...
        await app.stop()

    Or use run() for automatic signal handling:
        app = Application(num_workers=4)
        await app.run()
    """

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
    )

    def __init__(
        self,
        num_workers: int = 4,
    ) -> None:
        """
        Initialize application.

        Args:
            num_workers: Number of orderbook worker processes (default: 4)
        """
        if num_workers < 1:
            raise ValueError("num_workers must be at least 1")

        self._num_workers = num_workers

        # Components (initialized on start)
        self._registry: AssetRegistry | None = None
        self._workers: WorkerManager | None = None
        self._router: MessageRouter | None = None
        self._pool: ConnectionPool | None = None
        self._lifecycle: LifecycleController | None = None
        self._recycler: ConnectionRecycler | None = None
        self._message_parser: MessageParser | None = None

        self._running = False
        self._shutdown_event = asyncio.Event()

    @property
    def is_running(self) -> bool:
        """Whether the application is currently running."""
        return self._running
```

### Success Criteria:

#### Automated Verification:
- [ ] File created: `test -f src/app.py`
- [ ] Module imports successfully: `python -c "from src.app import Application"`
- [ ] Can instantiate: `python -c "from src.app import Application; app = Application()"`
- [ ] Validates num_workers: Unit test for ValueError on num_workers < 1
- [ ] Linting passes: `uv run ruff check src/app.py`

#### Manual Verification:
- [ ] Class structure matches spec requirements
- [ ] All slots defined correctly
- [ ] Constructor parameters have sensible defaults

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Phase 2: Startup Sequence Implementation

### Overview
Implement the complete startup sequence with proper component initialization order.

### Changes Required:

#### 1. Implement `Application.start()` method
**File**: `src/app.py`

```python
    async def start(self) -> None:
        """
        Start all components in correct order.

        Fast-fails on any component startup error, stopping already-started components.

        Raises:
            Exception: If any component fails to start
        """
        if self._running:
            logger.warning("Application already running")
            return

        logger.info("Starting Polymarket WebSocket application...")

        try:
            # 1. Initialize AssetRegistry (no start method)
            self._registry = AssetRegistry()
            logger.info("✓ AssetRegistry initialized")

            # 2. Initialize MessageParser (shared across pool)
            self._message_parser = MessageParser()
            logger.info("✓ MessageParser initialized")

            # 3. Create WorkerManager and get queues BEFORE starting
            self._workers = WorkerManager(num_workers=self._num_workers)
            worker_queues = self._workers.get_input_queues()
            logger.info(f"✓ WorkerManager created with {self._num_workers} workers")

            # 4. Create MessageRouter with worker queues
            self._router = MessageRouter(
                num_workers=self._num_workers,
                worker_queues=worker_queues,
            )
            logger.info("✓ MessageRouter created")

            # 5. Start WorkerManager (SYNC method!)
            self._workers.start()
            logger.info(f"✓ WorkerManager started ({self._num_workers} processes spawned)")

            # 6. Start MessageRouter (async)
            await self._router.start()
            logger.info("✓ MessageRouter started")

            # 7. Create ConnectionPool with router callback
            self._pool = ConnectionPool(
                registry=self._registry,
                message_callback=self._router.route_message,
                message_parser=self._message_parser,
            )
            logger.info("✓ ConnectionPool created")

            # 8. Start ConnectionPool (async)
            await self._pool.start()
            logger.info("✓ ConnectionPool started")

            # 9. Create LifecycleController with callbacks
            self._lifecycle = LifecycleController(
                registry=self._registry,
                on_new_market=self._on_new_market,
                on_market_expired=self._on_market_expired,
            )
            logger.info("✓ LifecycleController created")

            # 10. Start LifecycleController (async)
            await self._lifecycle.start()
            logger.info(
                f"✓ LifecycleController started "
                f"({self._lifecycle.known_market_count} known markets)"
            )

            # 11. Create ConnectionRecycler
            self._recycler = ConnectionRecycler(
                registry=self._registry,
                pool=self._pool,
            )
            logger.info("✓ ConnectionRecycler created")

            # 12. Start ConnectionRecycler (async)
            await self._recycler.start()
            logger.info("✓ ConnectionRecycler started")

            self._running = True
            logger.info("🚀 Application started successfully")

        except Exception as e:
            logger.error(f"❌ Application startup failed: {e}", exc_info=True)
            # Cleanup: stop already-started components
            await self._cleanup_on_startup_failure()
            raise

    async def _cleanup_on_startup_failure(self) -> None:
        """Stop any components that were started before failure."""
        logger.warning("Cleaning up after startup failure...")

        # Stop in reverse order, ignoring errors
        if self._recycler:
            try:
                await self._recycler.stop()
            except Exception as e:
                logger.error(f"Error stopping recycler during cleanup: {e}")

        if self._lifecycle:
            try:
                await self._lifecycle.stop()
            except Exception as e:
                logger.error(f"Error stopping lifecycle during cleanup: {e}")

        if self._pool:
            try:
                await self._pool.stop()
            except Exception as e:
                logger.error(f"Error stopping pool during cleanup: {e}")

        if self._router:
            try:
                await self._router.stop()
            except Exception as e:
                logger.error(f"Error stopping router during cleanup: {e}")

        if self._workers:
            try:
                self._workers.stop(timeout=5.0)
            except Exception as e:
                logger.error(f"Error stopping workers during cleanup: {e}")

        logger.warning("Cleanup completed")

    async def _on_new_market(self, event: str, asset_id: str) -> None:
        """Handle new market discovery."""
        logger.debug(f"New market discovered: {asset_id[:16]}...")

    async def _on_market_expired(self, event: str, asset_id: str) -> None:
        """Handle market expiration."""
        logger.debug(f"Market expired: {asset_id[:16]}...")
```

### Success Criteria:

#### Automated Verification:
- [ ] Unit test: Application.start() initializes all components
- [ ] Unit test: start() is idempotent (calling twice does nothing)
- [ ] Unit test: start() cleans up on failure (mock component failure)
- [ ] Integration test: Full startup sequence completes without errors
- [ ] Integration test: Worker processes are alive after start
- [ ] Integration test: Router is accepting messages after start

#### Manual Verification:
- [ ] Log messages appear in correct order during startup
- [ ] Each component logs successful initialization
- [ ] Startup completes in < 10 seconds
- [ ] ps aux shows worker processes after startup

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Phase 3: Shutdown Sequence Implementation

### Overview
Implement the complete shutdown sequence in reverse order with proper timeout handling.

### Changes Required:

#### 1. Implement `Application.stop()` method
**File**: `src/app.py`

```python
    async def stop(self) -> None:
        """
        Stop all components in reverse order.

        Completes within 30 seconds total (10s for WorkerManager graceful shutdown).
        """
        if not self._running:
            logger.warning("Application not running")
            return

        logger.info("Stopping application...")
        start_time = asyncio.get_event_loop().time()

        # Reverse order shutdown

        # 1. Stop ConnectionRecycler
        if self._recycler:
            try:
                await self._recycler.stop()
                logger.info("✓ ConnectionRecycler stopped")
            except Exception as e:
                logger.error(f"Error stopping recycler: {e}", exc_info=True)

        # 2. Stop LifecycleController
        if self._lifecycle:
            try:
                await self._lifecycle.stop()
                logger.info("✓ LifecycleController stopped")
            except Exception as e:
                logger.error(f"Error stopping lifecycle: {e}", exc_info=True)

        # 3. Stop ConnectionPool
        if self._pool:
            try:
                await self._pool.stop()
                logger.info("✓ ConnectionPool stopped")
            except Exception as e:
                logger.error(f"Error stopping pool: {e}", exc_info=True)

        # 4. Stop MessageRouter (sends None sentinels to workers)
        if self._router:
            try:
                await self._router.stop()
                logger.info("✓ MessageRouter stopped")
            except Exception as e:
                logger.error(f"Error stopping router: {e}", exc_info=True)

        # 5. Stop WorkerManager (SYNC method, 10s timeout)
        if self._workers:
            try:
                self._workers.stop(timeout=10.0)
                logger.info("✓ WorkerManager stopped")
            except Exception as e:
                logger.error(f"Error stopping workers: {e}", exc_info=True)

        self._running = False

        elapsed = asyncio.get_event_loop().time() - start_time
        logger.info(f"Application stopped (shutdown took {elapsed:.1f}s)")
```

### Success Criteria:

#### Automated Verification:
- [ ] Unit test: stop() is idempotent (calling twice is safe)
- [ ] Unit test: stop() completes even if components are None
- [ ] Unit test: stop() handles exceptions gracefully
- [ ] Integration test: Full startup → stop sequence completes
- [ ] Integration test: All worker processes terminate after stop
- [ ] Integration test: Shutdown completes within 30 seconds

#### Manual Verification:
- [ ] Log messages appear in reverse order during shutdown
- [ ] Each component logs successful shutdown
- [ ] ps aux shows no worker processes after shutdown
- [ ] No zombie processes left behind

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Phase 4: Signal Handling Implementation

### Overview
Implement the run() convenience method with SIGINT/SIGTERM signal handling.

### Changes Required:

#### 1. Implement `Application.run()` method
**File**: `src/app.py`

```python
    async def run(self) -> None:
        """
        Run application with automatic signal handling.

        Blocks until SIGINT (Ctrl+C) or SIGTERM received, then gracefully stops.
        """
        loop = asyncio.get_event_loop()

        def signal_handler() -> None:
            logger.info("Shutdown signal received (SIGINT/SIGTERM)")
            self._shutdown_event.set()

        # Register signal handlers
        for sig in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(sig, signal_handler)

        try:
            # Start all components
            await self.start()

            # Wait for shutdown signal
            await self._shutdown_event.wait()

        finally:
            # Stop all components
            await self.stop()

            # Remove signal handlers
            for sig in (signal.SIGINT, signal.SIGTERM):
                try:
                    loop.remove_signal_handler(sig)
                except Exception:
                    pass  # Already removed or loop closing
```

### Success Criteria:

#### Automated Verification:
- [ ] Unit test: run() registers signal handlers
- [ ] Unit test: run() removes signal handlers in finally block
- [ ] Unit test: Mock shutdown_event.set() triggers stop
- [ ] Integration test: SIGINT triggers graceful shutdown
- [ ] Integration test: SIGTERM triggers graceful shutdown

#### Manual Verification:
- [ ] Press Ctrl+C during run, see "Shutdown signal received" log
- [ ] Application stops gracefully after Ctrl+C
- [ ] kill -TERM <pid> triggers graceful shutdown
- [ ] Signal handlers are removed (no lingering handlers)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Phase 5: Stats Aggregation Implementation

### Overview
Implement the get_stats() method to collect comprehensive statistics from all components.

### Changes Required:

#### 1. Implement `Application.get_stats()` method
**File**: `src/app.py`

```python
    def get_stats(self) -> dict[str, Any]:
        """
        Get comprehensive application statistics.

        Returns:
            Dict with keys: running, registry, pool, router, workers, recycler
        """
        stats: dict[str, Any] = {
            "running": self._running,
            "registry": {},
            "pool": {},
            "router": {},
            "workers": {},
            "recycler": {},
        }

        # Registry stats
        if self._registry:
            from src.registry.asset_entry import AssetStatus

            stats["registry"] = {
                "total_markets": len(self._registry._assets),
                "pending": self._registry.get_pending_count(),
                "subscribed": len(self._registry.get_by_status(AssetStatus.SUBSCRIBED)),
                "expired": len(self._registry.get_by_status(AssetStatus.EXPIRED)),
            }

        # Pool stats
        if self._pool:
            stats["pool"] = {
                "connection_count": self._pool.connection_count,
                "active_connections": self._pool.active_connection_count,
                "total_capacity": self._pool.get_total_capacity(),
                "connections": self._pool.get_connection_stats(),
            }

        # Router stats
        if self._router:
            router_stats = self._router.stats
            stats["router"] = {
                "messages_routed": router_stats.messages_routed,
                "messages_dropped": router_stats.messages_dropped,
                "batches_sent": router_stats.batches_sent,
                "queue_full_events": router_stats.queue_full_events,
                "routing_errors": router_stats.routing_errors,
                "avg_latency_ms": router_stats.avg_latency_ms,
                "queue_depths": self._router.get_queue_depths(),
            }

        # Worker stats
        if self._workers:
            worker_stats_dict = self._workers.get_stats()
            stats["workers"] = {
                "alive_count": self._workers.get_alive_count(),
                "is_healthy": self._workers.is_healthy(),
                "worker_stats": {
                    worker_id: {
                        "messages_processed": ws.messages_processed,
                        "updates_applied": ws.updates_applied,
                        "snapshots_received": ws.snapshots_received,
                        "avg_processing_time_us": ws.avg_processing_time_us,
                        "orderbook_count": ws.orderbook_count,
                        "memory_usage_mb": ws.memory_usage_bytes / 1024 / 1024,
                    }
                    for worker_id, ws in worker_stats_dict.items()
                },
            }

        # Lifecycle stats (basic properties)
        if self._lifecycle:
            stats["lifecycle"] = {
                "is_running": self._lifecycle.is_running,
                "known_market_count": self._lifecycle.known_market_count,
            }

        # Recycler stats
        if self._recycler:
            recycler_stats = self._recycler.stats
            stats["recycler"] = {
                "recycles_initiated": recycler_stats.recycles_initiated,
                "recycles_completed": recycler_stats.recycles_completed,
                "recycles_failed": recycler_stats.recycles_failed,
                "success_rate": recycler_stats.success_rate,
                "markets_migrated": recycler_stats.markets_migrated,
                "avg_downtime_ms": recycler_stats.avg_downtime_ms,
                "active_recycles": list(self._recycler.get_active_recycles()),
            }

        return stats
```

### Success Criteria:

#### Automated Verification:
- [ ] Unit test: get_stats() returns dict with all keys
- [ ] Unit test: get_stats() handles None components gracefully
- [ ] Unit test: get_stats() returns valid data types
- [ ] Integration test: get_stats() returns accurate counts after adding markets
- [ ] Integration test: Stats reflect component state changes

#### Manual Verification:
- [ ] get_stats() output is valid JSON
- [ ] Stats match expected values for known system state
- [ ] Registry total_markets increases after lifecycle discovery
- [ ] Worker stats show messages_processed increasing
- [ ] Router avg_latency_ms is reasonable (< 10ms)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Phase 6: Health Check Implementation

### Overview
Implement the is_healthy() method to check overall system health.

### Changes Required:

#### 1. Implement `Application.is_healthy()` method
**File**: `src/app.py`

```python
    def is_healthy(self) -> bool:
        """
        Check overall application health.

        Returns:
            True if all critical components are healthy, False otherwise
        """
        if not self._running:
            return False

        # Check workers (critical)
        if self._workers and not self._workers.is_healthy():
            logger.warning("Health check failed: workers unhealthy")
            return False

        # Check pool has active connections (allow grace during startup)
        if self._pool and self._pool.active_connection_count == 0:
            # This might be OK during initial startup before lifecycle discovery
            # Don't fail health check, just log debug
            logger.debug("Health check: no active connections (may be during startup)")

        return True
```

### Success Criteria:

#### Automated Verification:
- [ ] Unit test: is_healthy() returns False when not running
- [ ] Unit test: is_healthy() returns False when workers unhealthy
- [ ] Unit test: is_healthy() returns True when all components healthy
- [ ] Integration test: is_healthy() is False before start()
- [ ] Integration test: is_healthy() is True after successful start()
- [ ] Integration test: is_healthy() becomes False if worker process dies

#### Manual Verification:
- [ ] is_healthy() returns True during normal operation
- [ ] is_healthy() returns False after stop()
- [ ] Kill a worker process (kill -9), verify is_healthy() becomes False
- [ ] Health check responds quickly (< 1ms)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Phase 7: Main Entry Point

### Overview
Create the main() function and update src/main.py to use the new Application class.

### Changes Required:

#### 1. Add main() function to `src/app.py`
**File**: `src/app.py`

```python
async def main() -> None:
    """
    Main entry point for the application.

    Configures logging and starts the Application orchestrator.
    """
    from src.core.logging import configure as configure_logging

    configure_logging()

    logger.info("Initializing Polymarket WebSocket Application")

    # Create application with default settings
    app = Application(num_workers=4)

    # Run with signal handling
    await app.run()


if __name__ == "__main__":
    asyncio.run(main())
```

#### 2. Update `src/main.py` to reference new application
**File**: `src/main.py`

Keep the current Phase 1 baseline but add a comment at the top:

```python
"""
Phase 1 Baseline - Single connection with synchronous processing.

For production use, see src/app.py (Application Orchestrator).

This file is kept for reference and learning purposes.
"""

# [Rest of existing code unchanged]
```

### Success Criteria:

#### Automated Verification:
- [ ] Can run: `uv run python src/app.py`
- [ ] Application starts without errors
- [ ] Lifecycle controller discovers markets from Gamma API
- [ ] Worker processes are created
- [ ] SIGINT (Ctrl+C) triggers graceful shutdown
- [ ] All linting passes: `uv run ruff check src/app.py`

#### Manual Verification:
- [ ] Run `uv run python src/app.py`, see all startup logs
- [ ] Application connects to Polymarket WebSocket
- [ ] Markets are discovered from Gamma API
- [ ] Messages are being routed and processed
- [ ] Press Ctrl+C, see graceful shutdown
- [ ] Run for 1 minute, verify no errors or warnings

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Phase 8: Comprehensive Testing

### Overview
Create comprehensive unit and integration tests for the Application class.

### Changes Required:

#### 1. Create `tests/test_app.py` - Unit tests
**File**: `tests/test_app.py` (new file)

```python
"""Unit tests for Application orchestrator."""

import asyncio
import signal
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from src.app import Application


class TestApplicationInit:
    """Test Application initialization."""

    def test_init_default_params(self):
        """Test initialization with default parameters."""
        app = Application()
        assert app._num_workers == 4
        assert not app.is_running
        assert app._registry is None

    def test_init_custom_num_workers(self):
        """Test initialization with custom num_workers."""
        app = Application(num_workers=8)
        assert app._num_workers == 8

    def test_init_invalid_num_workers(self):
        """Test initialization fails with invalid num_workers."""
        with pytest.raises(ValueError, match="num_workers must be at least 1"):
            Application(num_workers=0)


class TestApplicationStartStop:
    """Test Application start/stop lifecycle."""

    @pytest.mark.asyncio
    async def test_start_initializes_all_components(self):
        """Test that start() initializes all 6 components."""
        app = Application(num_workers=2)

        with patch("src.app.WorkerManager") as mock_worker_cls, \
             patch("src.app.MessageRouter") as mock_router_cls, \
             patch("src.app.ConnectionPool") as mock_pool_cls, \
             patch("src.app.LifecycleController") as mock_lifecycle_cls, \
             patch("src.app.ConnectionRecycler") as mock_recycler_cls:

            # Mock worker queues
            mock_workers = mock_worker_cls.return_value
            mock_workers.get_input_queues.return_value = [MagicMock(), MagicMock()]

            # Mock async start methods
            mock_router_cls.return_value.start = AsyncMock()
            mock_pool_cls.return_value.start = AsyncMock()
            mock_lifecycle_cls.return_value.start = AsyncMock()
            mock_recycler_cls.return_value.start = AsyncMock()

            await app.start()

            assert app.is_running
            assert app._registry is not None
            assert app._workers is not None
            assert app._router is not None
            assert app._pool is not None
            assert app._lifecycle is not None
            assert app._recycler is not None

    @pytest.mark.asyncio
    async def test_start_is_idempotent(self):
        """Test that calling start() twice is safe."""
        app = Application(num_workers=2)

        with patch("src.app.WorkerManager"), \
             patch("src.app.MessageRouter"), \
             patch("src.app.ConnectionPool"), \
             patch("src.app.LifecycleController"), \
             patch("src.app.ConnectionRecycler"):

            await app.start()
            await app.start()  # Should log warning and return

            assert app.is_running

    @pytest.mark.asyncio
    async def test_stop_is_idempotent(self):
        """Test that calling stop() twice is safe."""
        app = Application(num_workers=2)

        await app.stop()  # Not started
        await app.stop()  # Should log warning and return


class TestApplicationStats:
    """Test Application statistics."""

    def test_get_stats_returns_all_keys(self):
        """Test that get_stats() returns all expected keys."""
        app = Application()
        stats = app.get_stats()

        assert "running" in stats
        assert "registry" in stats
        assert "pool" in stats
        assert "router" in stats
        assert "workers" in stats
        assert "recycler" in stats

    def test_get_stats_handles_none_components(self):
        """Test that get_stats() handles uninitialized components."""
        app = Application()
        stats = app.get_stats()

        assert stats["running"] is False
        assert stats["registry"] == {}
        assert stats["pool"] == {}


class TestApplicationHealth:
    """Test Application health checks."""

    def test_is_healthy_false_when_not_running(self):
        """Test that is_healthy() returns False when not running."""
        app = Application()
        assert not app.is_healthy()

    def test_is_healthy_checks_workers(self):
        """Test that is_healthy() checks worker health."""
        app = Application()
        app._running = True

        # Mock unhealthy workers
        app._workers = MagicMock()
        app._workers.is_healthy.return_value = False

        assert not app.is_healthy()
```

#### 2. Create `tests/test_app_integration.py` - Integration tests
**File**: `tests/test_app_integration.py` (new file)

```python
"""Integration tests for Application orchestrator."""

import asyncio
import signal

import pytest

from src.app import Application


@pytest.mark.asyncio
@pytest.mark.timeout(30)
async def test_full_startup_shutdown_cycle():
    """Test complete startup and shutdown sequence."""
    app = Application(num_workers=2)

    try:
        # Start application
        await app.start()

        # Verify running
        assert app.is_running
        assert app.is_healthy()

        # Wait briefly for stabilization
        await asyncio.sleep(1.0)

        # Check stats
        stats = app.get_stats()
        assert stats["running"]
        assert stats["workers"]["alive_count"] == 2
        assert stats["workers"]["is_healthy"]

    finally:
        # Stop application
        await app.stop()

        # Verify stopped
        assert not app.is_running
        assert not app.is_healthy()


@pytest.mark.asyncio
@pytest.mark.timeout(60)
async def test_signal_handling():
    """Test SIGINT/SIGTERM signal handling."""
    app = Application(num_workers=2)

    # Create run task
    run_task = asyncio.create_task(app.run())

    # Wait for startup
    await asyncio.sleep(2.0)

    assert app.is_running

    # Trigger shutdown via event (simulates signal)
    app._shutdown_event.set()

    # Wait for shutdown
    await run_task

    assert not app.is_running


@pytest.mark.asyncio
@pytest.mark.timeout(30)
async def test_market_discovery():
    """Test that lifecycle controller discovers markets."""
    app = Application(num_workers=2)

    try:
        await app.start()

        # Wait for initial discovery
        await asyncio.sleep(5.0)

        # Check that markets were discovered
        stats = app.get_stats()

        if stats["lifecycle"]["known_market_count"] > 0:
            assert stats["registry"]["total_markets"] > 0
            pytest.skip("Market discovery working (live API test)")
        else:
            # Gamma API might be empty or unreachable
            pytest.skip("No markets discovered from Gamma API")

    finally:
        await app.stop()
```

### Success Criteria:

#### Automated Verification:
- [ ] All unit tests pass: `uv run pytest tests/test_app.py -v`
- [ ] All integration tests pass: `uv run pytest tests/test_app_integration.py -v`
- [ ] Test coverage > 80%: `uv run pytest --cov=src.app tests/`
- [ ] All tests complete within timeout
- [ ] No test failures or errors

#### Manual Verification:
- [ ] Run test suite multiple times, no flakiness
- [ ] Integration tests clean up worker processes
- [ ] No zombie processes after tests
- [ ] Tests run in < 60 seconds total

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful.

---

## Testing Strategy

### Unit Tests
- Application initialization and validation
- start() and stop() lifecycle methods
- Signal handler registration/removal
- Stats aggregation logic with None components
- Health check logic
- Idempotency of start/stop
- Error handling during startup failure

### Integration Tests
- Full startup → run → shutdown cycle
- Signal handling (SIGINT, SIGTERM)
- Component wiring (router receives messages from pool)
- Worker processes spawn and terminate correctly
- Stats accuracy with real components
- Health check reflects actual state
- Market discovery from Gamma API (if available)

### Manual Testing Checklist
- [ ] Run application for 5 minutes without errors
- [ ] Subscribe to 100+ markets successfully
- [ ] Press Ctrl+C, verify graceful shutdown
- [ ] kill -TERM, verify graceful shutdown
- [ ] Monitor memory usage (should be stable)
- [ ] Check logs for any warnings or errors
- [ ] Verify all worker processes terminate on shutdown

## Performance Considerations

- **Startup time**: Target < 10 seconds (includes Gamma API discovery)
- **Shutdown time**: Target < 30 seconds (with 10s worker graceful shutdown)
- **Stats collection**: Should be fast (< 10ms) as it's synchronous
- **Health check**: Should be instant (< 1ms) - just checks flags
- **Memory overhead**: Application class itself should be < 1KB (uses __slots__)

## Migration Notes

The current `src/main.py` is kept as `src/main.py` with a comment marking it as Phase 1 baseline. The new production entry point is `src/app.py`.

**To migrate existing code:**
```python
# Old (Phase 1):
from src.main import main
asyncio.run(main())

# New (Production):
from src.app import main
asyncio.run(main())
```

## Error Handling

- **Fast-fail on startup**: If any component fails during startup, immediately stop already-started components and raise the exception
- **Graceful errors during shutdown**: Log errors but continue stopping remaining components
- **Signal handling errors**: Ignore errors when removing signal handlers (may already be removed)
- **Stats collection errors**: Return empty dicts for components that fail stats collection

## References

- Original ticket: `thoughts/elliotsteene/tickets/ENG-006-application-orchestrator.md`
- System spec: `lessons/polymarket_websocket_system_spec.md:3041-3449`
- Research document: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:394-463`
- Current baseline: `src/main.py:20-113`

## Related Tickets

- ENG-001: Connection Pool Manager (complete)
- ENG-002: Async Message Router (complete)
- ENG-003: Worker Process Manager (complete)
- ENG-004: Market Lifecycle Controller (complete)
- ENG-005: Connection Recycler (complete)
- ENG-007: Add HTTP API for Stats and Health Endpoints (created, future work)
