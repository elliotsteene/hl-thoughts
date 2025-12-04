# Connection Pool Manager Implementation Plan

## Overview

Implement the Connection Pool Manager (Component 5) to manage multiple WebSocket connections with capacity tracking, automatic subscription batching, and recycling detection. This enables the system to subscribe to more than 500 assets by distributing them across multiple connections.

## Current State Analysis

**Existing Components (Working):**
- `src/connection/websocket.py` - WebsocketConnection with connect/disconnect, auto-reconnect, validates 500 asset limit
- `src/registry/asset_registry.py` - AssetRegistry with multi-index storage, PENDING/SUBSCRIBED/EXPIRED states
- `src/messages/protocol.py` - Protocol message definitions
- `src/messages/parser.py` - MessageParser for zero-copy parsing (stateless, can be shared)

**Current System Limitations:**
- `src/main.py:21-26` - Hardcoded to 2 assets with single connection
- No connection pooling or capacity management
- No automatic subscription batching
- No connection recycling detection

**Missing Registry Methods (needed by ConnectionPool):**
- `get_pending_count()` - Query count of pending assets
- `connection_stats(connection_id)` - Get pollution ratio and counts for recycling detection
- `get_active_by_connection(connection_id)` - Get only SUBSCRIBED (not EXPIRED) assets for a connection

**Missing WebSocketConnection Method:**
- `mark_draining()` - Set status to DRAINING for recycling workflow

## Desired End State

**System Capabilities:**
- Manages multiple WebSocket connections (each handling up to 500 assets)
- Automatically creates connections when pending markets exceed threshold (50)
- Tracks capacity per connection (target 400 active, 500 hard limit)
- Detects connections needing recycling (30% pollution, 5+ min age)
- Gracefully shuts down all connections
- Provides aggregated statistics across all connections
- Supports force subscription for priority assets

**Verification:**
1. Run integration tests showing multiple connections created for 1000+ pending markets
2. Verify capacity tracking prevents over-subscription (>400 per connection)
3. Verify recycling detection triggers at 30% pollution threshold
4. Verify graceful shutdown stops all connections within timeout
5. Verify stats aggregation returns correct data from all connections

## What We're NOT Doing

- Full recycling implementation (that's ENG-005: Connection Recycler) - only detection and stub
- Message routing to workers (that's ENG-002: Async Message Router)
- Market discovery or lifecycle management (that's ENG-004: Market Lifecycle Controller)
- Multiprocessing or worker management (that's ENG-003: Worker Process Manager)

## Implementation Approach

**Architecture Pattern:**
- Connection Pool manages a registry of ConnectionInfo wrappers around WebSocketConnection instances
- Periodic subscription loop (30s interval) checks for pending markets and creates connections
- Recycling detection loop identifies polluted connections but delegates actual recycling to Component 9
- All methods are async and use asyncio.Lock for thread-safety within a single event loop

**Key Design Decisions:**
1. **Shared MessageParser**: Create once and pass to all connections (stateless, reduces memory)
2. **Dependency Injection**: Pool accepts registry and message_callback via constructor for testability
3. **Stub for Recycling**: `_initiate_recycling()` contains basic logic but will be refined in ENG-005
4. **Target vs Hard Limit**: Use 400 as target (buffer), 500 as absolute max per Polymarket's limits

---

## Phase 1: Registry Enhancements

### Overview
Add missing methods to AssetRegistry required by ConnectionPool and fix existing bugs.

### Changes Required:

#### 1. Add `get_pending_count()` Method
**File**: `src/registry/asset_registry.py`
**Location**: After line 51 (after `get_by_connection()`)
**Changes**: Add lock-free read method to query pending queue length

```python
def get_pending_count(self) -> int:
    """Get count of assets awaiting subscription."""
    return len(self._pending_queue)
```

#### 2. Add `connection_stats()` Method
**File**: `src/registry/asset_registry.py`
**Location**: After `get_pending_count()`
**Changes**: Calculate pollution ratio and market counts for a connection

```python
def connection_stats(self, connection_id: str) -> dict[str, int | float]:
    """
    Get statistics for a connection.

    Returns dict with:
        - total: total assets assigned to connection
        - subscribed: count of SUBSCRIBED assets
        - expired: count of EXPIRED assets
        - pollution_ratio: expired / total (0.0 if total == 0)
    """
    asset_ids = self._by_connection.get(connection_id, set())
    total = len(asset_ids)

    if total == 0:
        return {
            "total": 0,
            "subscribed": 0,
            "expired": 0,
            "pollution_ratio": 0.0,
        }

    subscribed = 0
    expired = 0

    for asset_id in asset_ids:
        entry = self._assets.get(asset_id)
        if entry:
            if entry.status == AssetStatus.SUBSCRIBED:
                subscribed += 1
            elif entry.status == AssetStatus.EXPIRED:
                expired += 1

    return {
        "total": total,
        "subscribed": subscribed,
        "expired": expired,
        "pollution_ratio": expired / total if total > 0 else 0.0,
    }
```

#### 3. Add `get_active_by_connection()` Method
**File**: `src/registry/asset_registry.py`
**Location**: After `get_by_connection()`
**Changes**: Return only SUBSCRIBED (not EXPIRED) assets for a connection

```python
def get_active_by_connection(self, connection_id: str) -> FrozenSet[str]:
    """Get only active (SUBSCRIBED) assets for a connection, excluding expired."""
    asset_ids = self._by_connection.get(connection_id, set())
    active = {
        asset_id
        for asset_id in asset_ids
        if (entry := self._assets.get(asset_id))
        and entry.status == AssetStatus.SUBSCRIBED
    }
    return frozenset(active)
```

#### 4. Add `mark_draining()` Method to WebSocketConnection
**File**: `src/connection/websocket.py`
**Location**: After `is_healthy` property (after line 92)
**Changes**: Add method to set DRAINING status

```python
def mark_draining(self) -> None:
    """Mark connection as draining (preparing for recycling)."""
    if self._status == ConnectionStatus.CONNECTED:
        self._status = ConnectionStatus.DRAINING
```

### Success Criteria:

#### Automated Verification:
- [x] All unit tests pass: `uv run pytest tests/test_registry.py -v`
- [x] Type checking passes: `uv run ruff check .`
- [x] Linting passes: `uv run ruff format .`
- [x] New methods callable without errors
- [x] `connection_stats()` returns correct pollution ratio for test data

#### Manual Verification:
- [ ] Registry methods return expected data types and values
- [ ] Bug fix in `reassign_connection()` correctly moves assets between connections
- [ ] `mark_draining()` changes connection status appropriately

---

## Phase 2: Core Pool Structure

### Overview
Create the ConnectionPool class with basic initialization, storage structures, and lifecycle methods (start/stop).

### Changes Required:

#### 1. Create Pool Module
**File**: `src/pool.py` (new file)
**Changes**: Create module with imports and configuration constants

```python
"""
Connection Pool Manager - Orchestrates multiple WebSocket connections.

Responsibilities:
- Create/destroy connections based on demand
- Track capacity and pollution per connection
- Assign markets to appropriate connections
- Coordinate recycling of polluted connections
"""

import asyncio
import logging
import time
import uuid
from dataclasses import dataclass, field
from typing import Callable, Awaitable

from src.connection.websocket import WebsocketConnection
from src.connection.types import ConnectionStatus, MessageCallback
from src.messages.parser import MessageParser
from src.messages.protocol import ParsedMessage
from src.registry.asset_registry import AssetRegistry

logger = logging.getLogger(__name__)

# Configuration
TARGET_MARKETS_PER_CONNECTION = 400  # Leave headroom below 500 limit
MAX_MARKETS_PER_CONNECTION = 500
POLLUTION_THRESHOLD = 0.30  # 30% expired triggers recycling
MIN_AGE_FOR_RECYCLING = 300.0  # Don't recycle connections younger than 5 min
BATCH_SUBSCRIPTION_INTERVAL = 30.0  # Check for pending markets every 30s
MIN_PENDING_FOR_NEW_CONNECTION = 50  # Min pending to justify new connection


@dataclass(slots=True)
class ConnectionInfo:
    """Metadata about a managed connection."""
    connection: WebsocketConnection
    created_at: float = field(default_factory=time.monotonic)
    is_draining: bool = False

    @property
    def age(self) -> float:
        return time.monotonic() - self.created_at
```

#### 2. Define ConnectionPool Class
**File**: `src/pool.py`
**Changes**: Add class definition with `__init__`, properties, and storage

```python
class ConnectionPool:
    """
    Manages pool of WebSocket connections.

    Thread-safety: All methods are async and safe for concurrent calls
    from the same event loop.
    """

    __slots__ = (
        '_registry',
        '_message_callback',
        '_message_parser',
        '_connections',
        '_subscription_task',
        '_running',
        '_lock',
    )

    def __init__(
        self,
        registry: AssetRegistry,
        message_callback: MessageCallback,
        message_parser: MessageParser | None = None,
    ) -> None:
        """
        Initialize pool.

        Args:
            registry: Market registry for coordination
            message_callback: Callback for all received messages
            message_parser: Optional parser instance (creates one if not provided)
        """
        self._registry = registry
        self._message_callback = message_callback
        self._message_parser = message_parser or MessageParser()
        self._connections: dict[str, ConnectionInfo] = {}
        self._subscription_task: asyncio.Task | None = None
        self._running = False
        self._lock = asyncio.Lock()

    @property
    def connection_count(self) -> int:
        """Total number of connections (including draining)."""
        return len(self._connections)

    @property
    def active_connection_count(self) -> int:
        """Number of healthy, non-draining connections."""
        return sum(
            1 for info in self._connections.values()
            if info.connection.status == ConnectionStatus.CONNECTED
            and not info.is_draining
        )

    def get_total_capacity(self) -> int:
        """Total subscription slots available across all non-draining connections."""
        total = 0
        for cid, info in self._connections.items():
            if info.is_draining:
                continue

            used = len(self._registry.get_by_connection(cid))
            available = TARGET_MARKETS_PER_CONNECTION - used
            total += max(0, available)

        return total
```

#### 3. Implement start() and stop() Methods
**File**: `src/pool.py`
**Changes**: Add lifecycle management methods

```python
    async def start(self) -> None:
        """Start the pool and subscription manager."""
        if self._running:
            logger.warning("Connection pool already running")
            return

        self._running = True

        # Start subscription management task
        self._subscription_task = asyncio.create_task(
            self._subscription_loop(),
            name="pool-subscription-manager"
        )

        logger.info("Connection pool started")

    async def stop(self) -> None:
        """Stop all connections and cleanup."""
        if not self._running:
            return

        self._running = False

        # Stop subscription task
        if self._subscription_task:
            self._subscription_task.cancel()
            try:
                await self._subscription_task
            except asyncio.CancelledError:
                pass

        # Stop all connections concurrently
        stop_tasks = [
            info.connection.stop()
            for info in self._connections.values()
        ]
        if stop_tasks:
            await asyncio.gather(*stop_tasks, return_exceptions=True)

        self._connections.clear()
        logger.info("Connection pool stopped")
```

### Success Criteria:

#### Automated Verification:
- [x] Module imports successfully: `python -c "from src.pool import ConnectionPool"`
- [x] Type checking passes: `uv run ruff check .`
- [x] Can instantiate ConnectionPool with registry and callback
- [x] `start()` and `stop()` complete without errors
- [x] Properties return correct default values (0 connections initially)

#### Manual Verification:
- [x] ConnectionInfo dataclass correctly tracks age
- [x] Pool initializes with all required dependencies
- [x] Graceful shutdown completes within expected time

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Subscription Loop

### Overview
Implement the periodic subscription loop that checks for pending markets and creates connections with batches.

### Changes Required:

#### 1. Implement `_subscription_loop()` Background Task
**File**: `src/pool.py`
**Changes**: Add periodic loop that processes pending markets and checks for recycling

```python
    async def _subscription_loop(self) -> None:
        """Periodically process pending markets and check for recycling."""
        while self._running:
            try:
                await asyncio.sleep(BATCH_SUBSCRIPTION_INTERVAL)

                if not self._running:
                    break

                await self._process_pending_markets()
                await self._check_for_recycling()

            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.error(f"Subscription loop error: {e}", exc_info=True)
```

#### 2. Implement `_process_pending_markets()` Logic
**File**: `src/pool.py`
**Changes**: Create connections for pending market batches

```python
    async def _process_pending_markets(self) -> None:
        """Create connections for pending markets."""
        pending_count = self._registry.get_pending_count()

        if pending_count < MIN_PENDING_FOR_NEW_CONNECTION:
            logger.debug(
                f"Skipping connection creation: {pending_count} pending "
                f"(threshold: {MIN_PENDING_FOR_NEW_CONNECTION})"
            )
            return

        async with self._lock:
            # Take batch of pending markets
            batch = await self._registry.take_pending_batch(TARGET_MARKETS_PER_CONNECTION)

            if not batch:
                return

            # Create new connection
            connection_id = f"conn-{uuid.uuid4().hex[:8]}"
            connection = WebsocketConnection(
                connection_id=connection_id,
                asset_ids=batch,
                message_parser=self._message_parser,
                on_message=self._message_callback,
            )

            self._connections[connection_id] = ConnectionInfo(connection=connection)

            # Start connection
            await connection.start()

            # Mark markets as subscribed
            await self._registry.mark_subscribed(batch, connection_id)

            logger.info(
                f"Created connection {connection_id} with {len(batch)} markets "
                f"(total connections: {len(self._connections)}, "
                f"pending remaining: {self._registry.get_pending_count()})"
            )
```

### Success Criteria:

#### Automated Verification:
- [x] Unit tests pass: `uv run pytest tests/test_pool.py::test_subscription_loop -v`
- [x] Mock registry returns pending markets correctly
- [x] Subscription loop creates connections when pending >= 50
- [x] Subscription loop skips creation when pending < 50
- [x] Type checking passes: `uv run ruff check .`

#### Manual Verification:
- [ ] Add 100 pending assets to registry, verify connection created after 30s interval
- [ ] Add 40 pending assets, verify no connection created (below threshold)
- [ ] Verify `mark_subscribed()` called with correct asset_ids and connection_id
- [ ] Check logs show correct pending count and connection creation messages

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 4: Recycling Detection

### Overview
Implement connection monitoring for recycling triggers (pollution ratio, age) and create stub for actual recycling workflow.

### Changes Required:

#### 1. Implement `_check_for_recycling()` Logic
**File**: `src/pool.py`
**Changes**: Monitor connections and detect recycling triggers

```python
    async def _check_for_recycling(self) -> None:
        """Check connections for recycling needs."""
        for connection_id, info in list(self._connections.items()):
            # Skip connections already draining
            if info.is_draining:
                continue

            # Skip young connections (don't recycle if < 5 min old)
            if info.age < MIN_AGE_FOR_RECYCLING:
                continue

            # Get stats from registry
            stats = self._registry.connection_stats(connection_id)

            # Check pollution threshold
            if stats["pollution_ratio"] >= POLLUTION_THRESHOLD:
                logger.info(
                    f"Connection {connection_id} needs recycling: "
                    f"{stats['pollution_ratio']:.1%} pollution "
                    f"({stats['expired']}/{stats['total']} expired)"
                )
                # Create recycling task (stub implementation)
                asyncio.create_task(
                    self._initiate_recycling(connection_id),
                    name=f"recycle-{connection_id}"
                )
```

#### 2. Implement `_initiate_recycling()` Stub
**File**: `src/pool.py`
**Changes**: Create basic recycling workflow (refined in ENG-005)

```python
    async def _initiate_recycling(self, connection_id: str) -> None:
        """
        Initiate connection recycling.

        NOTE: This is a stub implementation. Full recycling workflow
        will be implemented in ENG-005 (Connection Recycler).
        """
        info = self._connections.get(connection_id)
        if not info:
            logger.warning(f"Cannot recycle {connection_id}: not found")
            return

        # Mark as draining
        info.is_draining = True
        info.connection.mark_draining()

        # Get active (non-expired) markets to migrate
        active_assets = list(self._registry.get_active_by_connection(connection_id))

        if not active_assets:
            # No active markets, just close
            logger.info(f"Connection {connection_id} has no active markets, closing")
            await self._remove_connection(connection_id)
            return

        # Create replacement connection
        new_connection_id = f"conn-{uuid.uuid4().hex[:8]}"
        new_connection = WebsocketConnection(
            connection_id=new_connection_id,
            asset_ids=active_assets,
            message_parser=self._message_parser,
            on_message=self._message_callback,
        )

        async with self._lock:
            self._connections[new_connection_id] = ConnectionInfo(connection=new_connection)

        # Start new connection
        await new_connection.start()

        # Wait for connection to stabilize
        await asyncio.sleep(2.0)

        # Reassign markets in registry
        await self._registry.reassign_connection(
            active_assets,
            connection_id,
            new_connection_id,
        )

        # Remove old connection
        await self._remove_connection(connection_id)

        logger.info(
            f"Recycled {connection_id} -> {new_connection_id} "
            f"with {len(active_assets)} active markets"
        )
```

#### 3. Implement `_remove_connection()` Helper
**File**: `src/pool.py`
**Changes**: Clean up connection from pool and registry

```python
    async def _remove_connection(self, connection_id: str) -> None:
        """Remove and cleanup a connection."""
        async with self._lock:
            info = self._connections.pop(connection_id, None)

        if info:
            await info.connection.stop()
            await self._registry.remove_connection(connection_id)
            logger.debug(f"Removed connection {connection_id}")
```

### Success Criteria:

#### Automated Verification:
- [x] Unit tests pass: `uv run pytest tests/test_pool.py::test_recycling_detection -v`
- [x] Recycling triggered when pollution >= 30%
- [x] Recycling skipped when age < 5 minutes
- [x] Recycling skipped for draining connections
- [x] Type checking passes: `uv run ruff check .`

#### Manual Verification:
- [ ] Create connection with 70% expired assets (age > 5 min), verify recycling triggered
- [ ] Create connection with 70% expired assets (age < 5 min), verify no recycling
- [ ] Verify new connection created with only active assets
- [ ] Verify old connection marked as draining and removed
- [ ] Check logs show correct pollution ratios

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 5: Stats and Force Subscribe

### Overview
Implement stats aggregation, capacity calculation, and force subscription for priority assets.

### Changes Required:

#### 1. Implement `get_connection_stats()` Aggregation
**File**: `src/pool.py`
**Changes**: Collect and return stats from all connections

```python
    def get_connection_stats(self) -> list[dict]:
        """Get stats for all connections."""
        result = []

        for cid, info in self._connections.items():
            conn_stats = info.connection.stats
            registry_stats = self._registry.connection_stats(cid)

            result.append({
                "connection_id": cid,
                "status": info.connection.status.name,
                "is_draining": info.is_draining,
                "age_seconds": info.age,
                "messages_received": conn_stats.messages_received,
                "bytes_received": conn_stats.bytes_received,
                "parse_errors": conn_stats.parse_errors,
                "reconnect_count": conn_stats.reconnect_count,
                "is_healthy": info.connection.is_healthy,
                "total_markets": registry_stats["total"],
                "subscribed_markets": registry_stats["subscribed"],
                "expired_markets": registry_stats["expired"],
                "pollution_ratio": registry_stats["pollution_ratio"],
            })

        return result
```

#### 2. Implement `force_subscribe()` for Priority Assets
**File**: `src/pool.py`
**Changes**: Create immediate connection bypassing pending queue

```python
    async def force_subscribe(self, asset_ids: list[str]) -> str:
        """
        Immediately create connection for given assets.

        Bypasses pending queue - used for priority subscriptions.
        Returns connection_id.
        """
        if len(asset_ids) > MAX_MARKETS_PER_CONNECTION:
            raise ValueError(
                f"Cannot subscribe to more than {MAX_MARKETS_PER_CONNECTION} markets, "
                f"got {len(asset_ids)}"
            )

        if not asset_ids:
            raise ValueError("Cannot force subscribe to empty asset list")

        connection_id = f"conn-{uuid.uuid4().hex[:8]}"
        connection = WebsocketConnection(
            connection_id=connection_id,
            asset_ids=asset_ids,
            message_parser=self._message_parser,
            on_message=self._message_callback,
        )

        async with self._lock:
            self._connections[connection_id] = ConnectionInfo(connection=connection)

        await connection.start()
        await self._registry.mark_subscribed(asset_ids, connection_id)

        logger.info(
            f"Force subscribed {len(asset_ids)} assets on connection {connection_id}"
        )

        return connection_id
```

### Success Criteria:

#### Automated Verification:
- [x] Unit tests pass: `uv run pytest tests/test_pool.py::TestStatsAndForceSubscribe -v`
- [x] `get_connection_stats()` returns list of dicts with correct keys
- [x] `force_subscribe()` raises ValueError for >500 assets
- [x] `force_subscribe()` raises ValueError for empty list
- [x] `force_subscribe()` returns connection_id string
- [x] Type checking passes: `uv run ruff check .`

#### Manual Verification:
- [ ] Create multiple connections, verify stats show accurate data
- [ ] Force subscribe 100 assets, verify immediate connection created
- [ ] Attempt force subscribe >500 assets, verify error raised
- [ ] Verify stats include pollution ratios, message counts, health status

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 6: Integration Testing

### Overview
Comprehensive testing with MarketRegistry integration, multiple connections, and capacity tracking.

### Changes Required:

#### 1. Create Integration Test Suite
**File**: `tests/test_pool_integration.py` (new file)
**Changes**: Test with real registry and multiple connections

```python
import asyncio
import pytest
from src.pool import ConnectionPool
from src.registry.asset_registry import AssetRegistry
from src.messages.parser import MessageParser


@pytest.mark.asyncio
async def test_pool_with_registry_integration():
    """Test pool with real registry managing 1000+ markets."""
    registry = AssetRegistry()
    parser = MessageParser()

    # Mock callback
    async def message_callback(connection_id, message):
        pass

    pool = ConnectionPool(registry, message_callback, parser)

    # Add 1000 pending markets
    for i in range(1000):
        await registry.add_asset(
            asset_id=f"asset-{i}",
            condition_id=f"market-{i // 2}",  # 2 assets per market
            expiration_ts=0,
        )

    # Start pool
    await pool.start()

    # Wait for subscription loop to process
    await asyncio.sleep(35)  # Wait for first interval + processing

    # Verify connections created (1000 / 400 = 3 connections)
    assert pool.connection_count >= 2
    assert registry.get_pending_count() < 1000

    # Verify capacity tracking
    stats = pool.get_connection_stats()
    total_subscribed = sum(s["subscribed_markets"] for s in stats)
    assert total_subscribed > 800  # Most should be subscribed

    # Cleanup
    await pool.stop()


@pytest.mark.asyncio
async def test_capacity_tracking():
    """Verify capacity tracking with multiple connections."""
    registry = AssetRegistry()
    parser = MessageParser()

    async def message_callback(connection_id, message):
        pass

    pool = ConnectionPool(registry, message_callback, parser)

    # Add exactly 800 pending markets
    for i in range(800):
        await registry.add_asset(f"asset-{i}", f"market-{i}", 0)

    await pool.start()
    await asyncio.sleep(35)

    # Should create 2 connections (400 each)
    assert pool.connection_count == 2

    # Check total capacity
    capacity = pool.get_total_capacity()
    assert capacity == 0  # All slots used

    await pool.stop()


@pytest.mark.asyncio
async def test_graceful_shutdown():
    """Verify all connections stop within timeout."""
    registry = AssetRegistry()
    parser = MessageParser()

    async def message_callback(connection_id, message):
        pass

    pool = ConnectionPool(registry, message_callback, parser)

    # Create 5 connections via force_subscribe
    for i in range(5):
        await pool.force_subscribe([f"asset-{i}"])

    assert pool.connection_count == 5

    # Stop and measure time
    import time
    start = time.monotonic()
    await pool.stop()
    elapsed = time.monotonic() - start

    assert pool.connection_count == 0
    assert elapsed < 10.0  # Should complete within 10 seconds
```

#### 2. Update Main Application (Optional Demo)
**File**: `src/main.py`
**Changes**: Demonstrate pool usage (optional, for testing)

```python
# Example integration (add to main.py for manual testing):
# from src.pool import ConnectionPool

# async def run_with_pool():
#     registry = AssetRegistry()
#     store = OrderbookStore()
#     parser = MessageParser()
#
#     # Add test assets
#     for i in range(100):
#         await registry.add_asset(f"test-asset-{i}", f"market-{i}", 0)
#
#     async def callback(connection_id, message):
#         # Handle message...
#         pass
#
#     pool = ConnectionPool(registry, callback, parser)
#     await pool.start()
#
#     # Let it run
#     await asyncio.sleep(60)
#
#     # Print stats
#     stats = pool.get_connection_stats()
#     for s in stats:
#         print(f"Connection {s['connection_id']}: "
#               f"{s['subscribed_markets']} markets, "
#               f"{s['pollution_ratio']:.1%} pollution")
#
#     await pool.stop()
```

### Success Criteria:

#### Automated Verification:
- [x] All integration tests pass: `uv run pytest tests/test_pool_integration.py -v`
- [x] Pool creates multiple connections for 1000+ markets
- [x] Capacity tracking accurate across connections
- [x] Graceful shutdown completes within 10s
- [x] No resource leaks (all connections closed)
- [x] Type checking passes: `uv run ruff check .`
- [x] Linting passes: `uv run ruff format .`

#### Manual Verification:
- [ ] Run integration test and observe log output
- [ ] Verify connections created at appropriate intervals
- [ ] Verify stats show accurate market counts
- [ ] Test with live WebSocket by manually adding real asset IDs
- [ ] Verify no exceptions during normal operation
- [ ] Verify clean shutdown with no hanging tasks

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Testing Strategy

### Unit Tests

**Registry Method Tests** (`tests/test_registry.py`):
- `test_get_pending_count()` - Verify count matches queue length
- `test_connection_stats()` - Verify pollution ratio calculation
- `test_connection_stats_empty()` - Verify 0.0 ratio for empty connection
- `test_get_active_by_connection()` - Verify only SUBSCRIBED assets returned
- `test_reassign_connection_bug_fix()` - Verify bug fix works correctly

**Connection Tests** (`tests/test_connection.py`):
- `test_mark_draining()` - Verify status changes to DRAINING

**Pool Tests** (`tests/test_pool.py`):
- `test_pool_initialization()` - Verify pool starts with 0 connections
- `test_start_stop()` - Verify lifecycle methods work
- `test_subscription_loop_threshold()` - Verify 50 minimum threshold
- `test_process_pending_markets()` - Verify connection creation
- `test_recycling_detection()` - Verify pollution triggers recycling
- `test_recycling_age_check()` - Verify young connections not recycled
- `test_force_subscribe()` - Verify immediate connection creation
- `test_get_connection_stats()` - Verify stats aggregation

### Integration Tests

**Pool Integration** (`tests/test_pool_integration.py`):
- `test_pool_with_registry_integration()` - 1000+ markets across connections
- `test_capacity_tracking()` - Verify capacity calculations
- `test_graceful_shutdown()` - Verify shutdown within timeout
- `test_concurrent_operations()` - Test with concurrent market additions

### Performance Tests

**Benchmarks** (optional, `tests/test_pool_performance.py`):
- Measure subscription loop latency (should be < 1s for 400 markets)
- Test with 10+ connections (5000+ markets)
- Memory usage per connection
- Shutdown time with many connections

## Performance Considerations

1. **Subscription Loop Interval**: 30 seconds balances responsiveness with API load
2. **Batch Size**: 400 markets per connection leaves 100-slot buffer under hard limit
3. **Minimum Threshold**: 50 pending markets prevents creating connections for tiny batches
4. **Shared Parser**: Single MessageParser instance reduces memory footprint
5. **Concurrent Shutdown**: Use `asyncio.gather()` to stop all connections in parallel

## Migration Notes

**From Current Single-Connection System:**
1. Replace hardcoded asset_ids in `main.py` with AssetRegistry
2. Replace single WebsocketConnection with ConnectionPool
3. Add pending assets to registry instead of directly to connection
4. Let pool manage connection lifecycle automatically

**Backward Compatibility:**
- Existing WebsocketConnection API unchanged
- Existing AssetRegistry methods still work
- New methods are additive only

## Open Questions & Decisions

All open questions from the ticket have been resolved:

1. **Recycling Implementation**: Using stub approach - detection in this component, full workflow in ENG-005 ✓
2. **Missing Registry Methods**: Adding as part of this ticket since they're required ✓
3. **Force Subscribe Threshold**: Uses same logic (creates connection immediately, no special bypass) ✓

## References

- **Original Ticket**: `thoughts/elliotsteene/tickets/ENG-001-connection-pool-manager.md`
- **System Spec**: `lessons/polymarket_websocket_system_spec.md:1417-1754`
- **Research Doc**: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:127-168`
- **Current WebSocket**: `src/connection/websocket.py:24`
- **Current Registry**: `src/registry/asset_registry.py:10`
- **Protocol Messages**: `src/messages/protocol.py`
- **Message Parser**: `src/messages/parser.py:22`
