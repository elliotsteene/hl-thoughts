# ENG-002: Async Message Router Implementation Plan

## Overview

Implement the Async Message Router to route messages from WebSocket connections (async domain) to worker processes (process domain) via multiprocessing queues. The router uses consistent hashing to ensure the same asset always routes to the same worker, with batching for efficiency and backpressure handling.

## Current State Analysis

### What Exists Now
- **src/main.py:28-58**: Messages processed synchronously in `App.message_callback()` - directly applies updates to OrderbookState
- **src/connection/pool.py:68-69**: ConnectionPool accepts a `MessageCallback` that gets invoked for each message
- **src/messages/protocol.py:163-178**: `ParsedMessage` uses msgspec.Struct with tagged union pattern
- **src/exercises/message_router.py**: Reference implementation using capacity-based distribution (different from our consistent hashing requirement)
- **lessons/polymarket_websocket_system_spec.md:1773-2046**: Full specification with code examples

### Key Constraints Discovered
1. **Parser yields one ParsedMessage per asset** (src/messages/parser.py:138-145) - no need for multi-asset fan-out logic in router
2. **MessageCallback type**: `Callable[[str, ParsedMessage], Awaitable[None]]` (src/connection/types.py:17)
3. **ConnectionPool passes callback at init** - router.route_message will become the callback
4. **Pickle serialization**: multiprocessing.Queue uses pickle automatically - need to verify msgspec.Struct compatibility

### Spec Location
Full implementation specification at `lessons/polymarket_websocket_system_spec.md:1773-2046`

## Desired End State

A `MessageRouter` class in `src/router.py` that:
1. Receives messages from connection callbacks via `route_message(connection_id, message)`
2. Buffers in bounded asyncio.Queue with backpressure handling
3. Routes to worker queues using consistent hashing on asset_id
4. Batches messages for efficiency (100 msgs or 10ms timeout)
5. Tracks routing stats (messages routed, dropped, latency)
6. Provides worker queues for ENG-003 to consume

### Verification
- Unit tests for hashing, batching, stats
- Integration tests with mock workers
- `uv run pytest tests/test_router.py` passes
- `just check` passes

## What We're NOT Doing

1. **Worker process implementation** - ENG-003 will implement workers that consume from queues
2. **Multi-asset fan-out logic** - Parser already yields one message per asset
3. **Custom serialization** - Using pickle implicitly via Queue
4. **Application orchestrator integration** - ENG-006 will wire router into main.py
5. **Dynamic worker scaling** - Fixed worker count at initialization

## Implementation Approach

Create the router in phases, building from core data structures up to the full routing loop:

1. **Phase 1**: Stats dataclass and hash function
2. **Phase 2**: MessageRouter initialization and queue setup
3. **Phase 3**: Message reception with backpressure
4. **Phase 4**: Batching and routing loop
5. **Phase 5**: Lifecycle management (start/stop)
6. **Phase 6**: Unit and integration tests

---

## Phase 1: Core Data Structures

### Overview
Create RouterStats dataclass and consistent hash function as foundation.

### Changes Required

#### 1. Create src/router.py with Stats and Hash Function
**File**: `src/router.py`
**Changes**: New file with configuration constants, RouterStats dataclass, and hash function

```python
"""
Async Message Router - Bridges async domain to multiprocessing workers.

Routes ParsedMessage from WebSocket connections to worker processes via
multiprocessing queues using consistent hashing for worker assignment.
"""

import hashlib
import logging
import time
from dataclasses import dataclass
from typing import Final

logger = logging.getLogger(__name__)

# Configuration constants
ASYNC_QUEUE_SIZE: Final[int] = 10_000  # Max pending in async queue
WORKER_QUEUE_SIZE: Final[int] = 5_000  # Max pending per worker
BATCH_SIZE: Final[int] = 100  # Messages to batch before routing
BATCH_TIMEOUT: Final[float] = 0.01  # 10ms max wait for batch
PUT_TIMEOUT: Final[float] = 0.001  # 1ms timeout for queue put


@dataclass(slots=True)
class RouterStats:
    """Routing performance metrics."""

    messages_routed: int = 0
    messages_dropped: int = 0
    batches_sent: int = 0
    queue_full_events: int = 0
    routing_errors: int = 0
    total_latency_ms: float = 0.0

    @property
    def avg_latency_ms(self) -> float:
        """Average routing latency in milliseconds."""
        if self.messages_routed > 0:
            return self.total_latency_ms / self.messages_routed
        return 0.0


def _hash_to_worker(asset_id: str, num_workers: int) -> int:
    """
    Consistent hash of asset_id to worker index.

    Uses MD5 for speed (not security-sensitive). The same asset_id
    will always hash to the same worker index.

    Args:
        asset_id: The asset identifier to hash
        num_workers: Number of workers to distribute across

    Returns:
        Worker index in range [0, num_workers)
    """
    hash_bytes = hashlib.md5(asset_id.encode()).digest()
    hash_int = int.from_bytes(hash_bytes[:8], "little")
    return hash_int % num_workers
```

### Success Criteria

#### Automated Verification:
- [x] File created: `ls src/router.py`
- [x] Linting passes: `just check`
- [x] Import works: `uv run python -c "from src.router import RouterStats, _hash_to_worker; print('OK')"`

#### Manual Verification:
- [ ] Review code matches spec lines 1810-1835

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 2.

---

## Phase 2: MessageRouter Initialization

### Overview
Create MessageRouter class with queue initialization and basic properties.

### Changes Required

#### 1. Add MessageRouter Class
**File**: `src/router.py`
**Changes**: Add MessageRouter class with __init__, properties, and queue accessors

```python
import asyncio
from multiprocessing import Queue as MPQueue
from typing import Any

from src.messages.protocol import ParsedMessage

# Type alias for worker queue (will be consumed by ENG-003)
WorkerQueue = MPQueue[tuple[ParsedMessage, float] | None]


class MessageRouter:
    """
    Routes messages from async domain to worker processes.

    Usage:
        router = MessageRouter(num_workers=4)
        queues = router.get_worker_queues()  # Pass to worker processes
        await router.start()

        # In connection callback:
        await router.route_message(connection_id, message)
    """

    __slots__ = (
        "_num_workers",
        "_worker_queues",
        "_async_queue",
        "_routing_task",
        "_running",
        "_stats",
        "_asset_worker_cache",
    )

    def __init__(self, num_workers: int) -> None:
        """
        Initialize router.

        Args:
            num_workers: Number of worker processes to route to
        """
        if num_workers < 1:
            raise ValueError("num_workers must be at least 1")

        self._num_workers = num_workers
        self._worker_queues: list[WorkerQueue] = [
            MPQueue(maxsize=WORKER_QUEUE_SIZE) for _ in range(num_workers)
        ]
        self._async_queue: asyncio.Queue[tuple[str, ParsedMessage, float]] = (
            asyncio.Queue(maxsize=ASYNC_QUEUE_SIZE)
        )
        self._routing_task: asyncio.Task[None] | None = None
        self._running = False
        self._stats = RouterStats()

        # Cache asset_id -> worker_index mapping
        self._asset_worker_cache: dict[str, int] = {}

    @property
    def stats(self) -> RouterStats:
        """Get current routing statistics."""
        return self._stats

    @property
    def num_workers(self) -> int:
        """Number of worker processes."""
        return self._num_workers

    def get_worker_queues(self) -> list[WorkerQueue]:
        """Get queues to pass to worker processes."""
        return self._worker_queues

    def get_worker_for_asset(self, asset_id: str) -> int:
        """Get worker index for an asset (cached)."""
        if asset_id not in self._asset_worker_cache:
            self._asset_worker_cache[asset_id] = _hash_to_worker(
                asset_id, self._num_workers
            )
        return self._asset_worker_cache[asset_id]

    def get_queue_depths(self) -> dict[str, int]:
        """Get current queue depths for monitoring."""
        return {
            "async_queue": self._async_queue.qsize(),
            **{f"worker_{i}": q.qsize() for i, q in enumerate(self._worker_queues)},
        }
```

### Success Criteria

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Import works: `uv run python -c "from src.router import MessageRouter; r = MessageRouter(4); print(r.num_workers)"`
- [x] Queues created: `uv run python -c "from src.router import MessageRouter; r = MessageRouter(4); print(len(r.get_worker_queues()))"`

#### Manual Verification:
- [ ] __slots__ matches spec
- [ ] Queue sizes use configuration constants

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 3.

---

## Phase 3: Message Reception with Backpressure

### Overview
Implement `route_message()` method that receives messages from connection callbacks and handles backpressure.

### Changes Required

#### 1. Add route_message Method
**File**: `src/router.py`
**Changes**: Add route_message method to MessageRouter class

```python
    async def route_message(
        self,
        connection_id: str,
        message: ParsedMessage,
    ) -> bool:
        """
        Route a message to appropriate worker.

        Called from connection callbacks. This is the entry point
        for all messages from WebSocket connections.

        Args:
            connection_id: ID of the connection that received the message
            message: Parsed message to route

        Returns:
            True if queued successfully, False if dropped due to backpressure
        """
        try:
            # Add receive timestamp for latency tracking
            receive_ts = time.monotonic()
            self._async_queue.put_nowait((connection_id, message, receive_ts))
            return True
        except asyncio.QueueFull:
            self._stats.messages_dropped += 1
            logger.warning(
                f"Async queue full, message dropped for asset {message.asset_id}"
            )
            return False
```

### Success Criteria

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Method signature matches MessageCallback type

#### Manual Verification:
- [ ] Backpressure logs warning with asset_id
- [ ] Returns bool indicating success/failure

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 4.

---

## Phase 4: Batching and Routing Loop

### Overview
Implement the core routing loop with batching and worker distribution.

### Changes Required

#### 1. Add Routing Loop Methods
**File**: `src/router.py`
**Changes**: Add _routing_loop and _route_batch methods

```python
    async def _routing_loop(self) -> None:
        """
        Main routing loop - batches messages and routes to workers.

        Collects messages up to BATCH_SIZE or BATCH_TIMEOUT, then
        groups by worker and sends to multiprocessing queues.
        """
        batch: list[tuple[str, ParsedMessage, float]] = []

        while self._running:
            try:
                # Collect batch
                batch.clear()
                deadline = time.monotonic() + BATCH_TIMEOUT

                while len(batch) < BATCH_SIZE:
                    remaining = deadline - time.monotonic()
                    if remaining <= 0:
                        break

                    try:
                        item = await asyncio.wait_for(
                            self._async_queue.get(),
                            timeout=remaining,
                        )
                        batch.append(item)
                    except asyncio.TimeoutError:
                        break

                if batch:
                    await self._route_batch(batch)

            except asyncio.CancelledError:
                # Drain remaining messages before exit
                while not self._async_queue.empty():
                    try:
                        item = self._async_queue.get_nowait()
                        batch.append(item)
                    except asyncio.QueueEmpty:
                        break
                if batch:
                    await self._route_batch(batch)
                break
            except Exception as e:
                logger.error(f"Routing loop error: {e}", exc_info=True)
                self._stats.routing_errors += 1

    async def _route_batch(
        self,
        batch: list[tuple[str, ParsedMessage, float]],
    ) -> None:
        """
        Route batch of messages to workers.

        Groups messages by worker for efficiency, then sends to
        multiprocessing queues with non-blocking puts.
        """
        from queue import Full

        # Group by worker
        worker_batches: dict[int, list[tuple[ParsedMessage, float]]] = {
            i: [] for i in range(self._num_workers)
        }

        for connection_id, message, receive_ts in batch:
            worker_idx = self.get_worker_for_asset(message.asset_id)
            worker_batches[worker_idx].append((message, receive_ts))

        # Send to workers
        now = time.monotonic()

        for worker_idx, messages in worker_batches.items():
            if not messages:
                continue

            queue = self._worker_queues[worker_idx]

            for message, receive_ts in messages:
                try:
                    # Put with timeout - non-blocking to prevent router stall
                    queue.put((message, receive_ts), timeout=PUT_TIMEOUT)

                    self._stats.messages_routed += 1
                    self._stats.total_latency_ms += (now - receive_ts) * 1000

                except Full:
                    self._stats.queue_full_events += 1
                    self._stats.messages_dropped += 1
                    logger.warning(
                        f"Worker {worker_idx} queue full, message dropped"
                    )

        self._stats.batches_sent += 1
```

### Success Criteria

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] No syntax errors: `uv run python -c "from src.router import MessageRouter"`

#### Manual Verification:
- [ ] Batch collection respects BATCH_SIZE limit
- [ ] Batch collection respects BATCH_TIMEOUT
- [ ] Worker grouping uses consistent hash
- [ ] Graceful handling of CancelledError drains queue

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 5.

---

## Phase 5: Lifecycle Management

### Overview
Implement start/stop methods for router lifecycle management.

### Changes Required

#### 1. Add Lifecycle Methods
**File**: `src/router.py`
**Changes**: Add start and stop methods

```python
    async def start(self) -> None:
        """Start the routing task."""
        if self._running:
            logger.warning("Router already running")
            return

        self._running = True
        self._routing_task = asyncio.create_task(
            self._routing_loop(),
            name="message-router",
        )
        logger.info(f"Message router started with {self._num_workers} workers")

    async def stop(self) -> None:
        """Stop routing and cleanup."""
        if not self._running:
            return

        self._running = False

        if self._routing_task:
            self._routing_task.cancel()
            try:
                await self._routing_task
            except asyncio.CancelledError:
                pass

        # Signal workers to stop (send None sentinel)
        from queue import Full

        for i, q in enumerate(self._worker_queues):
            try:
                q.put_nowait(None)
            except Full:
                logger.warning(f"Could not send shutdown sentinel to worker {i}")

        logger.info(
            f"Message router stopped. Stats: "
            f"routed={self._stats.messages_routed}, "
            f"dropped={self._stats.messages_dropped}, "
            f"avg_latency={self._stats.avg_latency_ms:.2f}ms"
        )
```

### Success Criteria

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Full module imports: `uv run python -c "from src.router import MessageRouter, RouterStats, WorkerQueue"`

#### Manual Verification:
- [ ] start() creates task with name "message-router"
- [ ] stop() sends None sentinel to all workers
- [ ] stop() logs final stats

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 6.

---

## Phase 6: Unit and Integration Tests

### Overview
Create comprehensive tests for the router implementation.

### Changes Required

#### 1. Create Test File
**File**: `tests/test_router.py`
**Changes**: New test file with unit and integration tests

```python
"""Tests for MessageRouter."""

import asyncio
import time

import pytest

from src.messages.protocol import EventType, ParsedMessage, PriceChange, Side
from src.router import (
    ASYNC_QUEUE_SIZE,
    MessageRouter,
    RouterStats,
    _hash_to_worker,
)


class TestRouterStats:
    """Tests for RouterStats dataclass."""

    def test_default_values(self):
        stats = RouterStats()
        assert stats.messages_routed == 0
        assert stats.messages_dropped == 0
        assert stats.batches_sent == 0
        assert stats.queue_full_events == 0
        assert stats.routing_errors == 0
        assert stats.total_latency_ms == 0.0

    def test_avg_latency_no_messages(self):
        stats = RouterStats()
        assert stats.avg_latency_ms == 0.0

    def test_avg_latency_with_messages(self):
        stats = RouterStats(messages_routed=10, total_latency_ms=50.0)
        assert stats.avg_latency_ms == 5.0


class TestHashFunction:
    """Tests for consistent hash function."""

    def test_deterministic(self):
        """Same asset_id always hashes to same worker."""
        asset_id = "test-asset-123"
        worker1 = _hash_to_worker(asset_id, 4)
        worker2 = _hash_to_worker(asset_id, 4)
        assert worker1 == worker2

    def test_range(self):
        """Hash result is within valid range."""
        for i in range(100):
            worker = _hash_to_worker(f"asset-{i}", 4)
            assert 0 <= worker < 4

    def test_distribution(self):
        """Hash distributes reasonably across workers."""
        counts = {i: 0 for i in range(4)}
        for i in range(1000):
            worker = _hash_to_worker(f"asset-{i}", 4)
            counts[worker] += 1

        # Each worker should get at least some messages (>10%)
        for count in counts.values():
            assert count > 100


class TestMessageRouterInit:
    """Tests for MessageRouter initialization."""

    def test_init_creates_queues(self):
        router = MessageRouter(num_workers=4)
        assert len(router.get_worker_queues()) == 4
        assert router.num_workers == 4

    def test_init_zero_workers_raises(self):
        with pytest.raises(ValueError):
            MessageRouter(num_workers=0)

    def test_init_negative_workers_raises(self):
        with pytest.raises(ValueError):
            MessageRouter(num_workers=-1)

    def test_stats_initial(self):
        router = MessageRouter(num_workers=2)
        assert router.stats.messages_routed == 0

    def test_queue_depths_initial(self):
        router = MessageRouter(num_workers=2)
        depths = router.get_queue_depths()
        assert depths["async_queue"] == 0
        assert depths["worker_0"] == 0
        assert depths["worker_1"] == 0


class TestWorkerAssignment:
    """Tests for asset-to-worker assignment."""

    def test_get_worker_for_asset_cached(self):
        router = MessageRouter(num_workers=4)
        asset_id = "test-asset"

        worker1 = router.get_worker_for_asset(asset_id)
        worker2 = router.get_worker_for_asset(asset_id)

        assert worker1 == worker2
        assert asset_id in router._asset_worker_cache


def _make_price_change_message(asset_id: str) -> ParsedMessage:
    """Helper to create a test message."""
    return ParsedMessage(
        event_type=EventType.PRICE_CHANGE,
        asset_id=asset_id,
        market="test-market",
        raw_timestamp=int(time.time() * 1000),
        price_change=PriceChange(
            asset_id=asset_id,
            price=500,
            size=100,
            side=Side.BUY,
            hash="test-hash",
            best_bid=500,
            best_ask=510,
        ),
    )


class TestRouteMessage:
    """Tests for route_message method."""

    @pytest.mark.asyncio
    async def test_route_message_success(self):
        router = MessageRouter(num_workers=2)
        message = _make_price_change_message("asset-1")

        result = await router.route_message("conn-1", message)

        assert result is True
        assert router.get_queue_depths()["async_queue"] == 1

    @pytest.mark.asyncio
    async def test_route_message_backpressure(self):
        """Test that messages are dropped when queue is full."""
        # Create router with tiny queue for testing
        router = MessageRouter(num_workers=1)
        # Fill the async queue
        for i in range(ASYNC_QUEUE_SIZE):
            message = _make_price_change_message(f"asset-{i}")
            await router.route_message("conn-1", message)

        # Next message should be dropped
        message = _make_price_change_message("asset-overflow")
        result = await router.route_message("conn-1", message)

        assert result is False
        assert router.stats.messages_dropped == 1


class TestRoutingLoop:
    """Integration tests for routing loop."""

    @pytest.mark.asyncio
    async def test_start_stop(self):
        router = MessageRouter(num_workers=2)
        await router.start()
        assert router._running is True
        assert router._routing_task is not None

        await router.stop()
        assert router._running is False

    @pytest.mark.asyncio
    async def test_messages_routed_to_workers(self):
        router = MessageRouter(num_workers=2)
        await router.start()

        # Send messages
        for i in range(10):
            message = _make_price_change_message(f"asset-{i}")
            await router.route_message("conn-1", message)

        # Give routing loop time to process
        await asyncio.sleep(0.05)

        await router.stop()

        # Check stats
        assert router.stats.messages_routed == 10
        assert router.stats.batches_sent >= 1

    @pytest.mark.asyncio
    async def test_consistent_routing(self):
        """Same asset always goes to same worker."""
        router = MessageRouter(num_workers=4)
        await router.start()

        asset_id = "test-asset"
        expected_worker = router.get_worker_for_asset(asset_id)

        # Send multiple messages for same asset
        for _ in range(5):
            message = _make_price_change_message(asset_id)
            await router.route_message("conn-1", message)

        await asyncio.sleep(0.05)
        await router.stop()

        # All messages should be in the same worker queue
        # (Others should have received sentinel None only)
        queues = router.get_worker_queues()
        for i, q in enumerate(queues):
            if i == expected_worker:
                # Should have 5 messages + None sentinel
                assert q.qsize() >= 5
            else:
                # Should only have None sentinel
                assert q.qsize() <= 1

    @pytest.mark.asyncio
    async def test_stop_sends_sentinels(self):
        router = MessageRouter(num_workers=2)
        await router.start()
        await router.stop()

        # Each worker queue should have None sentinel
        for q in router.get_worker_queues():
            item = q.get_nowait()
            assert item is None
```

### Success Criteria

#### Automated Verification:
- [x] Tests pass: `uv run pytest tests/test_router.py -v`
- [x] All tests green: `uv run pytest tests/test_router.py --tb=short`

#### Manual Verification:
- [ ] Test coverage appears reasonable
- [ ] Integration tests verify end-to-end flow

**Implementation Note**: After completing this phase and all automated verification passes, pause here for final review.

---

## Testing Strategy

### Unit Tests
- RouterStats dataclass and computed properties
- Consistent hash function distribution
- Worker cache hit/miss behavior
- Batch collection timing
- Queue full exception handling

### Integration Tests
- Route 10+ messages to multiple workers
- Verify distribution matches hash function
- Test worker queue overflow behavior
- Test graceful shutdown with pending messages

### Performance Tests (Future)
- Measure routing latency under load
- Test with 100,000+ messages/second
- Verify < 1ms average latency
- Test queue depth monitoring accuracy

## Migration Notes

This component can be developed independently. Integration with the application will happen in ENG-006 (Application Orchestrator) which will:
1. Replace `app.message_callback` with `router.route_message`
2. Pass worker queues to worker processes (ENG-003)
3. Wire up start/stop lifecycle

## References

- Ticket: `thoughts/elliotsteene/tickets/ENG-002-async-message-router.md`
- Spec: `lessons/polymarket_websocket_system_spec.md:1758-2057`
- Reference implementation: `src/exercises/message_router.py`
- Protocol messages: `src/messages/protocol.py`
- Current callback usage: `src/main.py:28-58`
