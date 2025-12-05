# Connection Recycler Implementation Plan

## Overview

Implement the ConnectionRecycler component to enable zero-downtime connection recycling. The recycler monitors WebSocket connections for health issues (pollution ≥30%, age ≥24h, unhealthy status) and orchestrates seamless migration of active markets from old connections to fresh ones without message loss.

## Current State Analysis

### Existing Implementation

**AssetRegistry** (`src/registry/asset_registry.py`):
- ✅ Has `get_active_by_connection(connection_id)` - line 53
- ✅ Has `reassign_connection(asset_ids, old_id, new_id)` - line 245
- ✅ Has `connection_stats(connection_id)` - line 86
- All required registry methods exist

**ConnectionPool** (`src/connection/pool.py`):
- ✅ Has `get_connection_stats()` - line 311
- ✅ Has `force_subscribe(asset_ids)` - line 339
- ⚠️ Has stub recycling implementation (`_initiate_recycling` - line 241) that:
  - Lacks concurrency control (no semaphore)
  - Doesn't track stats
  - Doesn't verify new connection health
  - Only checks pollution, not age or health triggers
  - Hardcoded 2s stabilization (spec requires 3s)

**LifecycleController** (`src/lifecycle/controller.py`):
- Provides pattern for start/stop lifecycle with multiple background tasks
- Shows how to manage asyncio tasks with cancellation handling

### Key Discoveries

**Pattern: Start/Stop Lifecycle** (from `LifecycleController:81-146`):
- Guard clause prevents double-start
- Create resources first (HTTP session, etc)
- Start background tasks with named tasks
- Cancel and await tasks in stop() with CancelledError handling
- Clean up resources in specific order

**Pattern: Background Monitoring Loop** (from `ConnectionPool:154-169`):
- Sleep first, then check running flag
- Explicit `asyncio.CancelledError` handler
- Catch-all exception handler for resilience
- Spawn tasks from monitoring loop using `asyncio.create_task()`

**Pattern: Semaphore for Concurrency** (from spec):
- `asyncio.Semaphore(MAX_CONCURRENT_RECYCLES)`
- Use `async with semaphore:` context manager
- Track active operations in set for deduplication

## Desired End State

A production-ready `ConnectionRecycler` class that:
1. Automatically recycles connections when pollution ≥30%, age ≥24h, or unhealthy
2. Maintains zero message loss during transitions (both connections receive messages briefly)
3. Limits concurrent recycles to 2 using semaphore
4. Tracks comprehensive stats (initiated, completed, failed, migrations, downtime)
5. Supports manual intervention via `force_recycle()`
6. Integrates seamlessly with existing ConnectionPool

### Verification

**Automated:**
- All unit tests pass: `just test`
- No linting errors: `just check`

**Manual:**
- Recycling triggers correctly on pollution threshold
- Zero message loss during recycling (verified by watching logs)
- Concurrent recycles limited to 2 (watch logs during load test)
- Stats accurately reflect operations

## What We're NOT Doing

- **NOT** implementing retry logic for failed recycling (fails safely by keeping old connection)
- **NOT** implementing gradual draining of old connection before recycling
- **NOT** tuning STABILIZATION_DELAY (using spec default of 3.0s)
- **NOT** implementing automatic health recovery (unhealthy connections recycled, not healed)
- **NOT** implementing connection pooling optimization (separate concern)
- **NOT** removing the stub implementation from ConnectionPool yet (will integrate in final phase)

## Implementation Approach

Create a standalone `ConnectionRecycler` class in `src/lifecycle/recycler.py` that:
1. Monitors connections via periodic health checks (every 60s)
2. Detects triggers: pollution ratio, age, or health status
3. Uses semaphore to limit concurrent recycles to 2
4. Executes zero-downtime recycling: create new → stabilize → atomic swap → remove old
5. Tracks detailed stats for observability

The recycler will be instantiated by ConnectionPool and started/stopped with the pool lifecycle.

---

## Phase 1: Core Structure & Stats

### Overview

Create the foundation: RecycleStats dataclass and ConnectionRecycler class skeleton with initialization, properties, and basic structure. Includes comprehensive unit tests for stats calculations.

### Changes Required

#### 1. Create `src/lifecycle/recycler.py`

**File**: `src/lifecycle/recycler.py`
**Changes**: New file with RecycleStats and ConnectionRecycler skeleton

```python
"""Connection recycler for zero-downtime connection migration."""

import asyncio
import time
from dataclasses import dataclass

import structlog

from src.connection.pool import ConnectionPool
from src.core.logging import Logger
from src.registry.asset_registry import AssetRegistry

logger: Logger = structlog.get_logger()

# Configuration constants
POLLUTION_THRESHOLD = 0.30  # 30% expired triggers recycling
AGE_THRESHOLD = 86400.0  # 24 hours max connection age (seconds)
HEALTH_CHECK_INTERVAL = 60.0  # Check health every 60 seconds
STABILIZATION_DELAY = 3.0  # Wait 3 seconds for new connection stability
MAX_CONCURRENT_RECYCLES = 2  # Limit concurrent operations


@dataclass(slots=True)
class RecycleStats:
    """Recycling performance metrics."""

    recycles_initiated: int = 0
    recycles_completed: int = 0
    recycles_failed: int = 0
    markets_migrated: int = 0
    total_downtime_ms: float = 0.0

    @property
    def success_rate(self) -> float:
        """
        Percentage of successful recycles.

        Returns 1.0 (100%) when no recycles initiated.
        """
        if self.recycles_initiated > 0:
            return self.recycles_completed / self.recycles_initiated
        return 1.0

    @property
    def avg_downtime_ms(self) -> float:
        """
        Average downtime per completed recycle.

        Returns 0.0 if no recycles completed.
        """
        if self.recycles_completed > 0:
            return self.total_downtime_ms / self.recycles_completed
        return 0.0


class ConnectionRecycler:
    """
    Monitors connections and orchestrates zero-downtime recycling.

    Responsibilities:
    - Detect recycling triggers (pollution, age, health)
    - Coordinate seamless migration of markets to new connections
    - Track recycling statistics
    - Limit concurrent recycling operations
    """

    __slots__ = (
        "_registry",
        "_pool",
        "_running",
        "_monitor_task",
        "_active_recycles",
        "_stats",
        "_recycle_semaphore",
    )

    def __init__(
        self,
        registry: AssetRegistry,
        pool: ConnectionPool,
    ) -> None:
        """
        Initialize recycler.

        Args:
            registry: Asset registry for market coordination
            pool: Connection pool to monitor and recycle
        """
        self._registry = registry
        self._pool = pool
        self._running = False
        self._monitor_task: asyncio.Task | None = None
        self._active_recycles: set[str] = set()
        self._stats = RecycleStats()
        self._recycle_semaphore = asyncio.Semaphore(MAX_CONCURRENT_RECYCLES)

    @property
    def stats(self) -> RecycleStats:
        """Read-only access to recycling statistics."""
        return self._stats

    @property
    def is_running(self) -> bool:
        """Whether the recycler is currently running."""
        return self._running

    def get_active_recycles(self) -> set[str]:
        """Get connection IDs currently being recycled."""
        return self._active_recycles.copy()
```

#### 2. Create Unit Tests

**File**: `tests/test_recycler.py`
**Changes**: New test file

```python
"""Unit tests for ConnectionRecycler."""

import pytest

from src.lifecycle.recycler import RecycleStats


class TestRecycleStats:
    """Test RecycleStats dataclass and computed properties."""

    def test_default_values(self):
        """All fields default to 0."""
        stats = RecycleStats()
        assert stats.recycles_initiated == 0
        assert stats.recycles_completed == 0
        assert stats.recycles_failed == 0
        assert stats.markets_migrated == 0
        assert stats.total_downtime_ms == 0.0

    def test_success_rate_no_recycles(self):
        """Success rate is 1.0 (100%) when no recycles initiated."""
        stats = RecycleStats()
        assert stats.success_rate == 1.0

    def test_success_rate_all_success(self):
        """Success rate is 1.0 when all recycles completed."""
        stats = RecycleStats(
            recycles_initiated=10,
            recycles_completed=10,
        )
        assert stats.success_rate == 1.0

    def test_success_rate_partial_success(self):
        """Success rate calculated correctly with failures."""
        stats = RecycleStats(
            recycles_initiated=10,
            recycles_completed=7,
            recycles_failed=3,
        )
        assert stats.success_rate == 0.7

    def test_success_rate_all_failures(self):
        """Success rate is 0.0 when all recycles failed."""
        stats = RecycleStats(
            recycles_initiated=5,
            recycles_completed=0,
            recycles_failed=5,
        )
        assert stats.success_rate == 0.0

    def test_avg_downtime_no_recycles(self):
        """Average downtime is 0.0 when no recycles completed."""
        stats = RecycleStats()
        assert stats.avg_downtime_ms == 0.0

    def test_avg_downtime_single_recycle(self):
        """Average downtime calculated correctly for single recycle."""
        stats = RecycleStats(
            recycles_completed=1,
            total_downtime_ms=150.5,
        )
        assert stats.avg_downtime_ms == 150.5

    def test_avg_downtime_multiple_recycles(self):
        """Average downtime calculated correctly for multiple recycles."""
        stats = RecycleStats(
            recycles_completed=4,
            total_downtime_ms=600.0,
        )
        assert stats.avg_downtime_ms == 150.0


class TestConnectionRecyclerInit:
    """Test ConnectionRecycler initialization."""

    def test_initialization(self, mock_registry, mock_pool):
        """Recycler initializes with correct defaults."""
        from src.lifecycle.recycler import ConnectionRecycler

        recycler = ConnectionRecycler(mock_registry, mock_pool)

        assert recycler.is_running is False
        assert recycler.get_active_recycles() == set()
        assert recycler.stats.recycles_initiated == 0
        assert recycler.stats.recycles_completed == 0
        assert recycler.stats.recycles_failed == 0

    def test_stats_property_readonly(self, mock_registry, mock_pool):
        """Stats property returns the stats object."""
        from src.lifecycle.recycler import ConnectionRecycler

        recycler = ConnectionRecycler(mock_registry, mock_pool)
        stats = recycler.stats

        assert isinstance(stats, RecycleStats)
        assert stats is recycler._stats


@pytest.fixture
def mock_registry():
    """Mock AssetRegistry."""
    from unittest.mock import Mock
    return Mock()


@pytest.fixture
def mock_pool():
    """Mock ConnectionPool."""
    from unittest.mock import Mock
    return Mock()
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `just test`
- [ ] Formatting correct: `just check`

#### Manual Verification:
- [ ] RecycleStats dataclass has correct default values
- [ ] Computed properties handle edge cases (division by zero)
- [ ] ConnectionRecycler initializes without errors

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Trigger Detection

### Overview

Implement `_check_all_connections()` to detect recycling triggers: pollution ratio ≥30%, age ≥24h, or unhealthy status. Includes skip logic for connections already being recycled or marked as draining.

### Changes Required

#### 1. Add Trigger Detection Method

**File**: `src/lifecycle/recycler.py`
**Changes**: Add `_check_all_connections()` method to ConnectionRecycler

```python
async def _check_all_connections(self) -> None:
    """
    Check all connections for recycling triggers.

    Triggers:
    - Pollution ratio >= POLLUTION_THRESHOLD (30%)
    - Age >= AGE_THRESHOLD (24 hours)
    - Unhealthy status (is_healthy == False)

    Skips connections already being recycled or marked as draining.
    """
    connection_stats = self._pool.get_connection_stats()

    for stats in connection_stats:
        connection_id = stats["connection_id"]

        # Skip if already recycling
        if connection_id in self._active_recycles:
            logger.debug(
                f"Skipping {connection_id}: already recycling"
            )
            continue

        # Skip if draining
        if stats.get("is_draining"):
            logger.debug(
                f"Skipping {connection_id}: already draining"
            )
            continue

        # Check triggers
        should_recycle = False
        reason = ""

        pollution_ratio = stats.get("pollution_ratio", 0.0)
        age_seconds = stats.get("age_seconds", 0.0)
        is_healthy = stats.get("is_healthy", True)

        if pollution_ratio >= POLLUTION_THRESHOLD:
            should_recycle = True
            reason = f"pollution={pollution_ratio:.1%}"
        elif age_seconds >= AGE_THRESHOLD:
            should_recycle = True
            reason = f"age={age_seconds/3600:.1f}h"
        elif not is_healthy:
            should_recycle = True
            reason = "unhealthy"

        if should_recycle:
            logger.info(
                f"Recycling trigger detected for {connection_id}: {reason}"
            )
            # TODO: Spawn recycling task (implemented in Phase 3)
```

#### 2. Add Unit Tests for Trigger Detection

**File**: `tests/test_recycler.py`
**Changes**: Add test class for trigger detection

```python
class TestTriggerDetection:
    """Test connection recycling trigger detection."""

    @pytest.mark.asyncio
    async def test_pollution_trigger(self, recycler, mock_pool):
        """Trigger detected when pollution >= 30%."""
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-1",
                "pollution_ratio": 0.35,
                "age_seconds": 100.0,
                "is_healthy": True,
                "is_draining": False,
            }
        ]

        # Should detect trigger (will be stubbed for now)
        await recycler._check_all_connections()

        # Verify log output or other side effects
        mock_pool.get_connection_stats.assert_called_once()

    @pytest.mark.asyncio
    async def test_age_trigger(self, recycler, mock_pool):
        """Trigger detected when age >= 24 hours."""
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-1",
                "pollution_ratio": 0.10,
                "age_seconds": 86500.0,  # Over 24 hours
                "is_healthy": True,
                "is_draining": False,
            }
        ]

        await recycler._check_all_connections()
        mock_pool.get_connection_stats.assert_called_once()

    @pytest.mark.asyncio
    async def test_health_trigger(self, recycler, mock_pool):
        """Trigger detected when connection unhealthy."""
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-1",
                "pollution_ratio": 0.10,
                "age_seconds": 100.0,
                "is_healthy": False,
                "is_draining": False,
            }
        ]

        await recycler._check_all_connections()
        mock_pool.get_connection_stats.assert_called_once()

    @pytest.mark.asyncio
    async def test_skip_already_recycling(self, recycler, mock_pool):
        """Skip connections already being recycled."""
        recycler._active_recycles.add("conn-1")

        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-1",
                "pollution_ratio": 0.40,
                "age_seconds": 100.0,
                "is_healthy": True,
                "is_draining": False,
            }
        ]

        await recycler._check_all_connections()

        # Should not trigger recycling (already in progress)
        assert "conn-1" in recycler._active_recycles

    @pytest.mark.asyncio
    async def test_skip_draining(self, recycler, mock_pool):
        """Skip connections already marked as draining."""
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-1",
                "pollution_ratio": 0.40,
                "age_seconds": 100.0,
                "is_healthy": True,
                "is_draining": True,
            }
        ]

        await recycler._check_all_connections()

        # Should skip draining connection
        assert "conn-1" not in recycler._active_recycles

    @pytest.mark.asyncio
    async def test_no_trigger(self, recycler, mock_pool):
        """No trigger when connection is healthy and young."""
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-1",
                "pollution_ratio": 0.10,
                "age_seconds": 1000.0,
                "is_healthy": True,
                "is_draining": False,
            }
        ]

        await recycler._check_all_connections()

        # Should not trigger
        assert "conn-1" not in recycler._active_recycles

    @pytest.mark.asyncio
    async def test_multiple_connections(self, recycler, mock_pool):
        """Handle multiple connections correctly."""
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-1",
                "pollution_ratio": 0.05,
                "age_seconds": 100.0,
                "is_healthy": True,
                "is_draining": False,
            },
            {
                "connection_id": "conn-2",
                "pollution_ratio": 0.40,
                "age_seconds": 100.0,
                "is_healthy": True,
                "is_draining": False,
            },
        ]

        await recycler._check_all_connections()

        # Only conn-2 should trigger
        mock_pool.get_connection_stats.assert_called_once()


@pytest.fixture
def recycler(mock_registry, mock_pool):
    """Create ConnectionRecycler instance."""
    from src.lifecycle.recycler import ConnectionRecycler
    return ConnectionRecycler(mock_registry, mock_pool)
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `just test`
- [ ] Formatting correct: `just check`

#### Manual Verification:
- [ ] Pollution trigger correctly identifies connections above 30%
- [ ] Age trigger correctly identifies connections older than 24 hours
- [ ] Health trigger correctly identifies unhealthy connections
- [ ] Skip logic prevents duplicate recycling attempts

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 3.

---

## Phase 3: Recycling Workflow

### Overview

Implement the full `_recycle_connection()` workflow: acquire semaphore, get active markets, create new connection, wait for stabilization, verify health, atomic registry update, remove old connection, update stats. Includes comprehensive error handling.

### Changes Required

#### 1. Implement Recycling Workflow

**File**: `src/lifecycle/recycler.py`
**Changes**: Add `_recycle_connection()` method

```python
async def _recycle_connection(self, connection_id: str) -> bool:
    """
    Perform connection recycling with zero-downtime migration.

    Workflow:
    1. Acquire semaphore for concurrency control
    2. Get active markets from old connection
    3. Create new connection with those markets
    4. Wait STABILIZATION_DELAY for new connection
    5. Verify new connection is healthy
    6. Atomically update registry (reassign markets)
    7. Close old connection
    8. Update stats

    Args:
        connection_id: ID of connection to recycle

    Returns:
        True if successful, False otherwise
    """
    start_time = time.monotonic()

    try:
        async with self._recycle_semaphore:
            self._active_recycles.add(connection_id)
            self._stats.recycles_initiated += 1

            logger.info(f"Starting recycle for {connection_id}")

            # Step 1: Get active markets from old connection
            active_assets = list(
                self._registry.get_active_by_connection(connection_id)
            )

            # Step 2: Handle empty connection
            if not active_assets:
                logger.info(
                    f"Connection {connection_id} has no active markets, "
                    "removing without replacement"
                )
                await self._pool._remove_connection(connection_id)
                self._stats.recycles_completed += 1
                return True

            logger.info(
                f"Recycling {connection_id} with {len(active_assets)} active markets"
            )

            # Step 3: Create new connection
            try:
                new_connection_id = await self._pool.force_subscribe(active_assets)
            except Exception as e:
                logger.error(
                    f"Failed to create replacement connection for {connection_id}: {e}",
                    exc_info=True,
                )
                self._stats.recycles_failed += 1
                return False

            logger.info(
                f"Created replacement connection {new_connection_id}, "
                f"waiting {STABILIZATION_DELAY}s for stabilization"
            )

            # Step 4: Stabilization delay (both connections receiving messages)
            await asyncio.sleep(STABILIZATION_DELAY)

            # Step 5: Verify new connection health
            new_stats = None
            for stats in self._pool.get_connection_stats():
                if stats["connection_id"] == new_connection_id:
                    new_stats = stats
                    break

            if not new_stats or not new_stats.get("is_healthy", False):
                logger.error(
                    f"New connection {new_connection_id} is not healthy, "
                    f"aborting recycle of {connection_id}"
                )
                self._stats.recycles_failed += 1
                return False

            logger.info(f"New connection {new_connection_id} is healthy")

            # Step 6: Atomic registry update
            try:
                migrated_count = await self._registry.reassign_connection(
                    active_assets,
                    connection_id,
                    new_connection_id,
                )
            except Exception as e:
                logger.error(
                    f"Failed to reassign markets from {connection_id} "
                    f"to {new_connection_id}: {e}",
                    exc_info=True,
                )
                self._stats.recycles_failed += 1
                return False

            logger.info(
                f"Reassigned {migrated_count} markets from {connection_id} "
                f"to {new_connection_id}"
            )

            # Step 7: Remove old connection
            try:
                await self._pool._remove_connection(connection_id)
            except Exception as e:
                logger.warning(
                    f"Error removing old connection {connection_id}: {e}",
                    exc_info=True,
                )
                # Continue - still count as success since markets are migrated

            # Step 8: Update stats
            duration_ms = (time.monotonic() - start_time) * 1000
            self._stats.recycles_completed += 1
            self._stats.markets_migrated += migrated_count
            self._stats.total_downtime_ms += duration_ms

            logger.info(
                f"Successfully recycled {connection_id} -> {new_connection_id} "
                f"in {duration_ms:.1f}ms with {migrated_count} markets"
            )

            return True

    except Exception as e:
        logger.error(
            f"Unexpected error recycling {connection_id}: {e}",
            exc_info=True,
        )
        self._stats.recycles_failed += 1
        return False
    finally:
        self._active_recycles.discard(connection_id)
```

#### 2. Update Trigger Detection to Spawn Tasks

**File**: `src/lifecycle/recycler.py`
**Changes**: Update `_check_all_connections()` to spawn recycling tasks

```python
# In _check_all_connections(), replace the TODO comment with:
        if should_recycle:
            logger.info(
                f"Recycling trigger detected for {connection_id}: {reason}"
            )
            asyncio.create_task(
                self._recycle_connection(connection_id),
                name=f"recycle-{connection_id}",
            )
```

#### 3. Add Unit Tests for Recycling Workflow

**File**: `tests/test_recycler.py`
**Changes**: Add test class for recycling workflow

```python
class TestRecyclingWorkflow:
    """Test full recycling workflow."""

    @pytest.mark.asyncio
    async def test_successful_recycle(self, recycler, mock_registry, mock_pool):
        """Full recycle workflow succeeds."""
        # Setup
        connection_id = "conn-old"
        new_connection_id = "conn-new"
        active_assets = ["asset-1", "asset-2", "asset-3"]

        mock_registry.get_active_by_connection.return_value = frozenset(active_assets)
        mock_pool.force_subscribe.return_value = new_connection_id
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": new_connection_id,
                "is_healthy": True,
            }
        ]
        mock_registry.reassign_connection.return_value = len(active_assets)

        # Execute
        result = await recycler._recycle_connection(connection_id)

        # Verify
        assert result is True
        assert recycler.stats.recycles_initiated == 1
        assert recycler.stats.recycles_completed == 1
        assert recycler.stats.recycles_failed == 0
        assert recycler.stats.markets_migrated == 3
        assert recycler.stats.total_downtime_ms > 0

        # Verify calls
        mock_registry.get_active_by_connection.assert_called_once_with(connection_id)
        mock_pool.force_subscribe.assert_called_once_with(active_assets)
        mock_registry.reassign_connection.assert_called_once_with(
            active_assets,
            connection_id,
            new_connection_id,
        )
        mock_pool._remove_connection.assert_called_once_with(connection_id)

    @pytest.mark.asyncio
    async def test_recycle_empty_connection(self, recycler, mock_registry, mock_pool):
        """Recycle connection with no active markets."""
        connection_id = "conn-old"

        mock_registry.get_active_by_connection.return_value = frozenset()

        result = await recycler._recycle_connection(connection_id)

        assert result is True
        assert recycler.stats.recycles_completed == 1
        assert recycler.stats.markets_migrated == 0

        # Should remove without creating replacement
        mock_pool.force_subscribe.assert_not_called()
        mock_pool._remove_connection.assert_called_once_with(connection_id)

    @pytest.mark.asyncio
    async def test_recycle_new_connection_fails(
        self, recycler, mock_registry, mock_pool
    ):
        """Recycle aborts if new connection creation fails."""
        connection_id = "conn-old"
        active_assets = ["asset-1", "asset-2"]

        mock_registry.get_active_by_connection.return_value = frozenset(active_assets)
        mock_pool.force_subscribe.side_effect = Exception("Connection failed")

        result = await recycler._recycle_connection(connection_id)

        assert result is False
        assert recycler.stats.recycles_initiated == 1
        assert recycler.stats.recycles_failed == 1
        assert recycler.stats.recycles_completed == 0

        # Old connection should remain (not removed)
        mock_pool._remove_connection.assert_not_called()

    @pytest.mark.asyncio
    async def test_recycle_new_connection_unhealthy(
        self, recycler, mock_registry, mock_pool
    ):
        """Recycle aborts if new connection is unhealthy."""
        connection_id = "conn-old"
        new_connection_id = "conn-new"
        active_assets = ["asset-1"]

        mock_registry.get_active_by_connection.return_value = frozenset(active_assets)
        mock_pool.force_subscribe.return_value = new_connection_id
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": new_connection_id,
                "is_healthy": False,  # Unhealthy
            }
        ]

        result = await recycler._recycle_connection(connection_id)

        assert result is False
        assert recycler.stats.recycles_failed == 1

        # Should not reassign or remove old connection
        mock_registry.reassign_connection.assert_not_called()

    @pytest.mark.asyncio
    async def test_recycle_reassignment_fails(
        self, recycler, mock_registry, mock_pool
    ):
        """Recycle aborts if registry reassignment fails."""
        connection_id = "conn-old"
        new_connection_id = "conn-new"
        active_assets = ["asset-1"]

        mock_registry.get_active_by_connection.return_value = frozenset(active_assets)
        mock_pool.force_subscribe.return_value = new_connection_id
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": new_connection_id,
                "is_healthy": True,
            }
        ]
        mock_registry.reassign_connection.side_effect = Exception("Reassignment failed")

        result = await recycler._recycle_connection(connection_id)

        assert result is False
        assert recycler.stats.recycles_failed == 1

    @pytest.mark.asyncio
    async def test_semaphore_limits_concurrency(
        self, recycler, mock_registry, mock_pool
    ):
        """Semaphore limits concurrent recycles to MAX_CONCURRENT_RECYCLES."""
        from src.lifecycle.recycler import MAX_CONCURRENT_RECYCLES

        # Setup slow recycle
        active_assets = ["asset-1"]
        mock_registry.get_active_by_connection.return_value = frozenset(active_assets)

        async def slow_force_subscribe(assets):
            await asyncio.sleep(0.5)
            return f"conn-{len(assets)}"

        mock_pool.force_subscribe.side_effect = slow_force_subscribe
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-1",
                "is_healthy": True,
            }
        ]
        mock_registry.reassign_connection.return_value = 1

        # Start MAX_CONCURRENT_RECYCLES + 1 recycles
        tasks = [
            asyncio.create_task(recycler._recycle_connection(f"conn-{i}"))
            for i in range(MAX_CONCURRENT_RECYCLES + 1)
        ]

        # Wait a bit - some should be blocked by semaphore
        await asyncio.sleep(0.1)

        # Check active recycles (should be <= MAX_CONCURRENT_RECYCLES)
        assert len(recycler.get_active_recycles()) <= MAX_CONCURRENT_RECYCLES

        # Wait for all to complete
        await asyncio.gather(*tasks)

    @pytest.mark.asyncio
    async def test_active_recycles_tracking(self, recycler, mock_registry, mock_pool):
        """Connection ID added/removed from active_recycles correctly."""
        connection_id = "conn-1"
        active_assets = ["asset-1"]

        mock_registry.get_active_by_connection.return_value = frozenset(active_assets)
        mock_pool.force_subscribe.return_value = "conn-new"
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-new",
                "is_healthy": True,
            }
        ]
        mock_registry.reassign_connection.return_value = 1

        # Before recycle
        assert connection_id not in recycler.get_active_recycles()

        # During recycle (check with task)
        task = asyncio.create_task(recycler._recycle_connection(connection_id))
        await asyncio.sleep(0.01)  # Let it start
        assert connection_id in recycler.get_active_recycles()

        # After recycle
        await task
        assert connection_id not in recycler.get_active_recycles()
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `just test`
- [ ] Formatting correct: `just check`

#### Manual Verification:
- [ ] Recycling workflow completes without errors in happy path
- [ ] Error handling preserves old connection on failures
- [ ] Semaphore correctly limits concurrent operations
- [ ] Stats accurately track all operations

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 4.

---

## Phase 4: Monitoring Loop & Lifecycle

### Overview

Implement start/stop lifecycle methods and the background monitoring loop that periodically checks connections for recycling triggers. Follows established patterns from LifecycleController.

### Changes Required

#### 1. Implement Lifecycle Methods

**File**: `src/lifecycle/recycler.py`
**Changes**: Add start(), stop(), and _monitor_loop() methods

```python
async def start(self) -> None:
    """
    Start the recycler monitoring loop.

    Begins checking connections every HEALTH_CHECK_INTERVAL seconds.
    """
    if self._running:
        logger.warning("ConnectionRecycler already running")
        return

    self._running = True

    self._monitor_task = asyncio.create_task(
        self._monitor_loop(),
        name="connection-recycler-monitor",
    )

    logger.info(
        f"ConnectionRecycler started (check interval: {HEALTH_CHECK_INTERVAL}s, "
        f"pollution threshold: {POLLUTION_THRESHOLD:.0%}, "
        f"age threshold: {AGE_THRESHOLD/3600:.0f}h)"
    )

async def stop(self) -> None:
    """
    Stop the recycler and cleanup.

    Cancels monitoring task and waits for active recycles to complete.
    """
    if not self._running:
        return

    self._running = False

    # Cancel monitor task
    if self._monitor_task:
        self._monitor_task.cancel()
        try:
            await self._monitor_task
        except asyncio.CancelledError:
            pass

    # Wait for active recycles to complete (with timeout)
    if self._active_recycles:
        logger.info(
            f"Waiting for {len(self._active_recycles)} active recycles to complete"
        )

        timeout = 30.0  # Max 30 seconds
        deadline = time.monotonic() + timeout

        while self._active_recycles and time.monotonic() < deadline:
            await asyncio.sleep(0.5)

        if self._active_recycles:
            logger.warning(
                f"Timed out waiting for recycles: {self._active_recycles}"
            )

    logger.info(
        f"ConnectionRecycler stopped. Stats: "
        f"initiated={self._stats.recycles_initiated}, "
        f"completed={self._stats.recycles_completed}, "
        f"failed={self._stats.recycles_failed}, "
        f"success_rate={self._stats.success_rate:.1%}"
    )

async def _monitor_loop(self) -> None:
    """
    Monitor connections for recycling needs.

    Runs every HEALTH_CHECK_INTERVAL seconds while running.
    """
    while self._running:
        try:
            await asyncio.sleep(HEALTH_CHECK_INTERVAL)

            if not self._running:
                break

            await self._check_all_connections()

        except asyncio.CancelledError:
            break
        except Exception as e:
            logger.error(f"Monitor loop error: {e}", exc_info=True)
```

#### 2. Add Unit Tests for Lifecycle

**File**: `tests/test_recycler.py`
**Changes**: Add test class for lifecycle management

```python
class TestLifecycle:
    """Test start/stop lifecycle management."""

    @pytest.mark.asyncio
    async def test_start(self, recycler):
        """Start initializes monitoring loop."""
        await recycler.start()

        assert recycler.is_running is True
        assert recycler._monitor_task is not None

        # Cleanup
        await recycler.stop()

    @pytest.mark.asyncio
    async def test_start_idempotent(self, recycler):
        """Starting already-running recycler is no-op."""
        await recycler.start()
        task1 = recycler._monitor_task

        await recycler.start()  # Second start
        task2 = recycler._monitor_task

        assert task1 is task2  # Same task

        await recycler.stop()

    @pytest.mark.asyncio
    async def test_stop(self, recycler):
        """Stop cancels monitoring task."""
        await recycler.start()
        await recycler.stop()

        assert recycler.is_running is False
        assert recycler._monitor_task.cancelled() or recycler._monitor_task.done()

    @pytest.mark.asyncio
    async def test_stop_idempotent(self, recycler):
        """Stopping already-stopped recycler is no-op."""
        await recycler.start()
        await recycler.stop()
        await recycler.stop()  # Second stop

        assert recycler.is_running is False

    @pytest.mark.asyncio
    async def test_stop_waits_for_active_recycles(
        self, recycler, mock_registry, mock_pool
    ):
        """Stop waits for active recycles to complete."""
        await recycler.start()

        # Simulate active recycle
        recycler._active_recycles.add("conn-1")

        # Stop in background
        stop_task = asyncio.create_task(recycler.stop())

        # Wait a bit
        await asyncio.sleep(0.1)

        # Stop should be waiting
        assert not stop_task.done()

        # Complete the recycle
        recycler._active_recycles.discard("conn-1")

        # Now stop should complete
        await asyncio.wait_for(stop_task, timeout=1.0)
        assert recycler.is_running is False

    @pytest.mark.asyncio
    async def test_monitor_loop_checks_connections(
        self, recycler, mock_pool, monkeypatch
    ):
        """Monitor loop calls _check_all_connections periodically."""
        from src.lifecycle import recycler as recycler_module

        # Speed up for testing
        monkeypatch.setattr(recycler_module, "HEALTH_CHECK_INTERVAL", 0.1)

        check_count = 0

        async def mock_check():
            nonlocal check_count
            check_count += 1

        recycler._check_all_connections = mock_check

        await recycler.start()
        await asyncio.sleep(0.35)  # Should trigger ~3 checks
        await recycler.stop()

        assert check_count >= 2  # At least 2 checks

    @pytest.mark.asyncio
    async def test_monitor_loop_handles_exceptions(
        self, recycler, mock_pool, monkeypatch
    ):
        """Monitor loop continues after exceptions."""
        from src.lifecycle import recycler as recycler_module

        monkeypatch.setattr(recycler_module, "HEALTH_CHECK_INTERVAL", 0.05)

        call_count = 0

        async def failing_check():
            nonlocal call_count
            call_count += 1
            if call_count == 1:
                raise Exception("Test error")

        recycler._check_all_connections = failing_check

        await recycler.start()
        await asyncio.sleep(0.15)
        await recycler.stop()

        # Should have recovered and continued
        assert call_count >= 2
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `just test`
- [ ] Formatting correct: `just check`

#### Manual Verification:
- [ ] Monitoring loop starts and runs periodically
- [ ] Stop method cleanly cancels monitoring task
- [ ] Exception in check doesn't crash monitor loop
- [ ] Active recycles tracked correctly during lifecycle

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 5.

---

## Phase 5: Manual Operations & Visibility

### Overview

Implement `force_recycle()` for manual intervention and enhance visibility with logging. Provides operators with tools to manually trigger recycling and observe system behavior.

### Changes Required

#### 1. Implement Manual Operations

**File**: `src/lifecycle/recycler.py`
**Changes**: Add force_recycle() method

```python
async def force_recycle(self, connection_id: str) -> bool:
    """
    Manually trigger recycling for a specific connection.

    Bypasses trigger checks - used for manual intervention.

    Args:
        connection_id: ID of connection to recycle

    Returns:
        True if successful, False otherwise

    Raises:
        ValueError: If connection_id is already being recycled
    """
    if connection_id in self._active_recycles:
        raise ValueError(
            f"Connection {connection_id} is already being recycled"
        )

    logger.info(f"Manual recycle triggered for {connection_id}")

    result = await self._recycle_connection(connection_id)

    if result:
        logger.info(f"Manual recycle of {connection_id} completed successfully")
    else:
        logger.error(f"Manual recycle of {connection_id} failed")

    return result
```

#### 2. Add Comprehensive Logging

**File**: `src/lifecycle/recycler.py`
**Changes**: Enhance logging throughout (already present in previous phases)

All methods should have appropriate log levels:
- **INFO**: Normal operations (start, stop, recycle trigger, completion)
- **WARNING**: Abnormal but recoverable (unhealthy connection, timeout)
- **ERROR**: Failures (recycle failed, exception in loop)
- **DEBUG**: Verbose details (skip reasons, intermediate steps)

#### 3. Add Unit Tests for Manual Operations

**File**: `tests/test_recycler.py`
**Changes**: Add test class for manual operations

```python
class TestManualOperations:
    """Test manual intervention and visibility."""

    @pytest.mark.asyncio
    async def test_force_recycle_success(self, recycler, mock_registry, mock_pool):
        """force_recycle successfully triggers recycling."""
        connection_id = "conn-1"
        active_assets = ["asset-1"]

        mock_registry.get_active_by_connection.return_value = frozenset(active_assets)
        mock_pool.force_subscribe.return_value = "conn-new"
        mock_pool.get_connection_stats.return_value = [
            {
                "connection_id": "conn-new",
                "is_healthy": True,
            }
        ]
        mock_registry.reassign_connection.return_value = 1

        result = await recycler.force_recycle(connection_id)

        assert result is True
        assert recycler.stats.recycles_completed == 1

    @pytest.mark.asyncio
    async def test_force_recycle_already_recycling(self, recycler):
        """force_recycle raises ValueError if already recycling."""
        connection_id = "conn-1"
        recycler._active_recycles.add(connection_id)

        with pytest.raises(ValueError, match="already being recycled"):
            await recycler.force_recycle(connection_id)

    @pytest.mark.asyncio
    async def test_force_recycle_failure(self, recycler, mock_registry, mock_pool):
        """force_recycle returns False on failure."""
        connection_id = "conn-1"

        mock_registry.get_active_by_connection.return_value = frozenset(["asset-1"])
        mock_pool.force_subscribe.side_effect = Exception("Failed")

        result = await recycler.force_recycle(connection_id)

        assert result is False
        assert recycler.stats.recycles_failed == 1

    @pytest.mark.asyncio
    async def test_get_active_recycles(self, recycler):
        """get_active_recycles returns copy of active set."""
        recycler._active_recycles.add("conn-1")
        recycler._active_recycles.add("conn-2")

        active = recycler.get_active_recycles()

        assert active == {"conn-1", "conn-2"}

        # Should be a copy, not the original
        active.add("conn-3")
        assert "conn-3" not in recycler._active_recycles

    def test_stats_property(self, recycler):
        """stats property returns stats object."""
        stats = recycler.stats

        assert stats is recycler._stats
        assert isinstance(stats, RecycleStats)
```

### Success Criteria

#### Automated Verification:
- [x] Unit tests pass: `just test`
- [x] Formatting correct: `just check`

#### Manual Verification:
- [x] force_recycle() successfully triggers recycling on demand
- [x] ValueError raised when trying to recycle connection already in progress
- [x] Logging provides clear visibility into recycler operations
- [x] get_active_recycles() shows current state correctly

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 6.

---

## Phase 6: System Integration & Testing

### Overview

Wire the ConnectionRecycler into the system: update ConnectionPool to instantiate and manage the recycler, add integration tests to verify end-to-end recycling behavior with real components.

### Changes Required

#### 1. Integrate Recycler into ConnectionPool

**File**: `src/connection/pool.py`
**Changes**: Add recycler initialization and lifecycle management

```python
# Add import at top
from src.lifecycle.recycler import ConnectionRecycler

# In ConnectionPool.__init__(), add to __slots__:
__slots__ = (
    "_registry",
    "_message_callback",
    "_message_parser",
    "_connections",
    "_subscription_task",
    "_recycler",  # NEW
    "_running",
    "_lock",
)

# In ConnectionPool.__init__(), after setting _lock:
def __init__(
    self,
    registry: AssetRegistry,
    message_callback: MessageCallback,
    message_parser: MessageParser | None = None,
) -> None:
    # ... existing init code ...
    self._lock = asyncio.Lock()

    # Initialize recycler
    self._recycler = ConnectionRecycler(registry, self)

# In ConnectionPool.start(), after subscription task:
async def start(self) -> None:
    if self._running:
        logger.warning("Connection pool already running")
        return

    self._running = True

    # Start subscription management task
    self._subscription_task = asyncio.create_task(
        self._subscription_loop(), name="pool-subscription-manager"
    )

    # Start recycler
    await self._recycler.start()

    logger.info("Connection pool started")

# In ConnectionPool.stop(), after subscription task cleanup:
async def stop(self) -> None:
    if not self._running:
        return

    self._running = False

    # Stop recycler first (before stopping connections)
    await self._recycler.stop()

    # Stop subscription task
    if self._subscription_task:
        self._subscription_task.cancel()
        try:
            await self._subscription_task
        except asyncio.CancelledError:
            pass

    # ... rest of stop() ...

# Add property for visibility:
@property
def recycler_stats(self) -> RecycleStats:
    """Get recycler statistics."""
    return self._recycler.stats
```

#### 2. Remove/Deprecate Stub Implementation

**File**: `src/connection/pool.py`
**Changes**: Remove `_check_for_recycling()` call from `_subscription_loop()`

```python
async def _subscription_loop(self) -> None:
    """Periodically process pending markets."""
    while self._running:
        try:
            await asyncio.sleep(BATCH_SUBSCRIPTION_INTERVAL)

            if not self._running:
                break

            await self._process_pending_markets()
            # REMOVED: await self._check_for_recycling()

        except asyncio.CancelledError:
            break
        except Exception as e:
            logger.error(f"Subscription loop error: {e}", exc_info=True)
```

**Note**: Keep `_initiate_recycling()` and `_check_for_recycling()` methods for now but log deprecation warnings. Can remove in future cleanup.

#### 3. Create Integration Tests

**File**: `tests/test_recycler_integration.py`
**Changes**: New integration test file

```python
"""Integration tests for ConnectionRecycler with real components."""

import asyncio

import pytest

from src.connection.pool import ConnectionPool
from src.lifecycle.recycler import ConnectionRecycler, POLLUTION_THRESHOLD
from src.registry.asset_registry import AssetRegistry


@pytest.mark.asyncio
async def test_recycler_integrates_with_pool():
    """Recycler can be created with real pool and registry."""
    registry = AssetRegistry()

    def mock_callback(msg):
        pass

    pool = ConnectionPool(registry, mock_callback)
    recycler = ConnectionRecycler(registry, pool)

    assert recycler.is_running is False
    assert recycler.stats.recycles_initiated == 0


@pytest.mark.asyncio
async def test_full_recycling_flow_with_real_components(monkeypatch):
    """
    End-to-end recycling with real AssetRegistry and mocked pool.

    Simulates:
    1. Connection with high pollution (40%)
    2. Recycler detects trigger
    3. Creates new connection
    4. Registry reassigns markets
    5. Old connection removed
    """
    from unittest.mock import AsyncMock, Mock
    from src.lifecycle import recycler as recycler_module

    # Speed up for testing
    monkeypatch.setattr(recycler_module, "HEALTH_CHECK_INTERVAL", 0.1)
    monkeypatch.setattr(recycler_module, "STABILIZATION_DELAY", 0.1)

    # Create real registry
    registry = AssetRegistry()

    # Add markets to registry
    await registry.add_asset("asset-1", "condition-1", expiration_ts=9999999999000)
    await registry.add_asset("asset-2", "condition-1", expiration_ts=9999999999000)
    await registry.add_asset("asset-3", "condition-1", expiration_ts=1000)  # Expired
    await registry.add_asset("asset-4", "condition-1", expiration_ts=1000)  # Expired
    await registry.add_asset("asset-5", "condition-1", expiration_ts=9999999999000)

    # Mark as subscribed to old connection
    await registry.mark_subscribed(
        ["asset-1", "asset-2", "asset-3", "asset-4", "asset-5"],
        "conn-old"
    )

    # Mark some as expired (creates 40% pollution)
    await registry.mark_expired(["asset-3", "asset-4"])

    # Verify pollution
    stats = registry.connection_stats("conn-old")
    assert stats["pollution_ratio"] >= POLLUTION_THRESHOLD

    # Mock pool
    mock_pool = Mock()
    mock_pool.get_connection_stats = Mock(return_value=[
        {
            "connection_id": "conn-old",
            "pollution_ratio": stats["pollution_ratio"],
            "age_seconds": 100.0,
            "is_healthy": True,
            "is_draining": False,
        }
    ])
    mock_pool.force_subscribe = AsyncMock(return_value="conn-new")
    mock_pool._remove_connection = AsyncMock()

    # Update mock to return healthy new connection
    def get_stats_dynamic():
        return [
            {
                "connection_id": "conn-old",
                "pollution_ratio": stats["pollution_ratio"],
                "age_seconds": 100.0,
                "is_healthy": True,
                "is_draining": False,
            },
            {
                "connection_id": "conn-new",
                "is_healthy": True,
            }
        ]

    mock_pool.get_connection_stats = Mock(side_effect=get_stats_dynamic)

    # Create recycler
    recycler = ConnectionRecycler(registry, mock_pool)

    # Start recycler
    await recycler.start()

    # Wait for recycling to trigger
    await asyncio.sleep(0.3)

    # Stop recycler
    await recycler.stop()

    # Verify recycling occurred
    assert recycler.stats.recycles_initiated >= 1
    assert recycler.stats.recycles_completed >= 1
    assert recycler.stats.markets_migrated == 3  # Only active markets

    # Verify registry state
    active_on_new = registry.get_active_by_connection("conn-new")
    assert len(active_on_new) == 3
    assert "asset-1" in active_on_new
    assert "asset-2" in active_on_new
    assert "asset-5" in active_on_new

    # Verify old connection removed from pool
    mock_pool._remove_connection.assert_called_with("conn-old")


@pytest.mark.asyncio
async def test_concurrent_recycles_limited(monkeypatch):
    """Concurrent recycles limited to MAX_CONCURRENT_RECYCLES."""
    from unittest.mock import AsyncMock, Mock
    from src.lifecycle import recycler as recycler_module

    monkeypatch.setattr(recycler_module, "STABILIZATION_DELAY", 0.2)

    registry = AssetRegistry()

    # Add assets for multiple connections
    for i in range(5):
        await registry.add_asset(f"asset-{i}", "condition-1", expiration_ts=1000)
        await registry.mark_subscribed([f"asset-{i}"], f"conn-{i}")
        await registry.mark_expired([f"asset-{i}"])

    # Mock pool
    mock_pool = Mock()

    connections = []
    for i in range(5):
        connections.append({
            "connection_id": f"conn-{i}",
            "pollution_ratio": 1.0,  # 100% expired
            "age_seconds": 100.0,
            "is_healthy": True,
            "is_draining": False,
        })

    mock_pool.get_connection_stats = Mock(return_value=connections)
    mock_pool._remove_connection = AsyncMock()

    # Create recycler
    recycler = ConnectionRecycler(registry, mock_pool)

    # Trigger all recycles manually
    tasks = [
        asyncio.create_task(recycler.force_recycle(f"conn-{i}"))
        for i in range(5)
    ]

    # Wait a bit and check active
    await asyncio.sleep(0.05)

    # Should have at most MAX_CONCURRENT_RECYCLES active
    active = recycler.get_active_recycles()
    assert len(active) <= recycler_module.MAX_CONCURRENT_RECYCLES

    # Wait for all to complete
    await asyncio.gather(*tasks, return_exceptions=True)


@pytest.mark.asyncio
async def test_age_based_recycling(monkeypatch):
    """Recycling triggers on connection age >= 24 hours."""
    from unittest.mock import AsyncMock, Mock
    from src.lifecycle import recycler as recycler_module

    monkeypatch.setattr(recycler_module, "HEALTH_CHECK_INTERVAL", 0.1)
    monkeypatch.setattr(recycler_module, "STABILIZATION_DELAY", 0.05)

    registry = AssetRegistry()
    await registry.add_asset("asset-1", "condition-1", expiration_ts=9999999999000)
    await registry.mark_subscribed(["asset-1"], "conn-old")

    mock_pool = Mock()
    mock_pool.get_connection_stats = Mock(return_value=[
        {
            "connection_id": "conn-old",
            "pollution_ratio": 0.0,  # Not polluted
            "age_seconds": 86500.0,  # Over 24 hours
            "is_healthy": True,
            "is_draining": False,
        }
    ])
    mock_pool.force_subscribe = AsyncMock(return_value="conn-new")
    mock_pool._remove_connection = AsyncMock()

    # Update mock for new connection
    def get_stats_dynamic():
        return [
            {
                "connection_id": "conn-old",
                "pollution_ratio": 0.0,
                "age_seconds": 86500.0,
                "is_healthy": True,
                "is_draining": False,
            },
            {
                "connection_id": "conn-new",
                "is_healthy": True,
            }
        ]

    mock_pool.get_connection_stats = Mock(side_effect=get_stats_dynamic)

    recycler = ConnectionRecycler(registry, mock_pool)
    await recycler.start()
    await asyncio.sleep(0.3)
    await recycler.stop()

    # Should have triggered recycling due to age
    assert recycler.stats.recycles_initiated >= 1


@pytest.mark.asyncio
async def test_unhealthy_connection_recycling(monkeypatch):
    """Recycling triggers on unhealthy connection."""
    from unittest.mock import AsyncMock, Mock
    from src.lifecycle import recycler as recycler_module

    monkeypatch.setattr(recycler_module, "HEALTH_CHECK_INTERVAL", 0.1)
    monkeypatch.setattr(recycler_module, "STABILIZATION_DELAY", 0.05)

    registry = AssetRegistry()
    await registry.add_asset("asset-1", "condition-1", expiration_ts=9999999999000)
    await registry.mark_subscribed(["asset-1"], "conn-old")

    mock_pool = Mock()
    mock_pool.get_connection_stats = Mock(return_value=[
        {
            "connection_id": "conn-old",
            "pollution_ratio": 0.0,
            "age_seconds": 100.0,
            "is_healthy": False,  # Unhealthy
            "is_draining": False,
        }
    ])
    mock_pool.force_subscribe = AsyncMock(return_value="conn-new")
    mock_pool._remove_connection = AsyncMock()

    def get_stats_dynamic():
        return [
            {
                "connection_id": "conn-old",
                "pollution_ratio": 0.0,
                "age_seconds": 100.0,
                "is_healthy": False,
                "is_draining": False,
            },
            {
                "connection_id": "conn-new",
                "is_healthy": True,
            }
        ]

    mock_pool.get_connection_stats = Mock(side_effect=get_stats_dynamic)

    recycler = ConnectionRecycler(registry, mock_pool)
    await recycler.start()
    await asyncio.sleep(0.3)
    await recycler.stop()

    assert recycler.stats.recycles_initiated >= 1
```

#### 4. Update ConnectionPool Tests

**File**: `tests/test_pool.py`
**Changes**: Update tests to account for recycler

```python
# Add test for recycler integration
@pytest.mark.asyncio
async def test_pool_starts_recycler():
    """Pool starts recycler on start()."""
    registry = AssetRegistry()

    def mock_callback(msg):
        pass

    pool = ConnectionPool(registry, mock_callback)

    assert pool._recycler.is_running is False

    await pool.start()

    assert pool._recycler.is_running is True

    await pool.stop()

    assert pool._recycler.is_running is False
```

### Success Criteria

#### Automated Verification:
- [ ] Unit tests pass: `just test`
- [ ] Formatting correct: `just check`

#### Manual Verification:
- [ ] Pool starts and stops recycler correctly
- [ ] Recycler triggers on real pollution ratios
- [ ] End-to-end recycling completes without errors
- [ ] Stats visible via pool.recycler_stats property
- [ ] No message loss during recycling (check logs)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation. This is the final phase.

---

## Testing Strategy

### Unit Tests (Phases 1-5)

**Coverage**:
- RecycleStats dataclass and computed properties (success_rate, avg_downtime)
- ConnectionRecycler initialization and properties
- Trigger detection logic (pollution, age, health thresholds)
- Skip logic (already recycling, draining)
- Full recycling workflow (happy path and error cases)
- Semaphore concurrency limiting
- Start/stop lifecycle management
- Monitor loop behavior (periodic checks, exception handling)
- Manual operations (force_recycle, get_active_recycles)

**Test Files**:
- `tests/test_recycler.py` - All unit tests

**Execution**: `uv run pytest tests/test_recycler.py -v`

### Integration Tests (Phase 6)

**Coverage**:
- End-to-end recycling with real AssetRegistry
- Pollution-based trigger detection
- Age-based trigger detection
- Unhealthy connection trigger detection
- Concurrent recycle limiting with real semaphore
- Registry reassignment correctness
- Pool integration (start/stop lifecycle)

**Test Files**:
- `tests/test_recycler_integration.py` - Integration tests
- `tests/test_pool.py` - Updated pool tests

**Execution**: `uv run pytest tests/test_recycler_integration.py -v`

### Manual Testing Checklist

After all phases complete:

1. **Start Application**: `just run`
2. **Observe Recycler Startup**: Check logs for recycler initialization
3. **Wait for Natural Recycling**: Let markets expire naturally, watch for:
   - Pollution ratio increasing over time
   - Recycling trigger when 30% pollution reached
   - Zero-downtime migration (both connections receiving messages)
   - Successful completion with stats logged
4. **Manual Recycle**: Use force_recycle() via API/script
5. **Concurrent Load**: Trigger multiple recycles simultaneously, verify limited to 2
6. **Verify Stats**: Check recycler_stats show accurate counts
7. **Stop Application**: Graceful shutdown with recycler stopping cleanly

---

## Performance Considerations

**Monitoring Overhead**:
- Health checks every 60 seconds (configurable via HEALTH_CHECK_INTERVAL)
- `get_connection_stats()` is O(n) where n = number of connections
- Expected: <10 connections, negligible overhead

**Recycling Overhead**:
- 3 second stabilization delay per recycle
- Brief message duplication during transition (acceptable for idempotent updates)
- Expected downtime: 3-5 seconds per recycle (tracked in stats)

**Concurrency Impact**:
- Max 2 concurrent recycles prevents resource exhaustion
- Semaphore blocking is async (no thread blocking)

**Memory Impact**:
- RecycleStats: <100 bytes
- Active recycles set: ~50 bytes per connection
- Minimal memory footprint

---

## Migration Notes

**Backward Compatibility**:
- Existing ConnectionPool stub implementation remains functional
- Recycler adds monitoring and stats on top
- No breaking API changes

**Gradual Rollout**:
1. Deploy with recycler but keep stub for safety
2. Monitor recycler stats in production
3. Remove stub in future cleanup if recycler performs well

**Configuration Tuning**:
- Start with spec defaults (30% pollution, 24h age, 3s stabilization)
- Monitor stats: success_rate, avg_downtime_ms
- Adjust thresholds if needed (requires code change, not config yet)

---

## References

- **Original Ticket**: `thoughts/elliotsteene/tickets/ENG-005-connection-recycler.md`
- **System Spec**: `lessons/polymarket_websocket_system_spec.md:2750-3038`
- **Research Doc**: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:335-391`
- **AssetRegistry**: `src/registry/asset_registry.py:53-275` (required methods)
- **ConnectionPool**: `src/connection/pool.py:311-373` (integration points)
- **LifecycleController**: `src/lifecycle/controller.py:81-146` (lifecycle pattern)
