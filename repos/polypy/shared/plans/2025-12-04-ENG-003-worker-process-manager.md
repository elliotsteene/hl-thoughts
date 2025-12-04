# Worker Process Manager Implementation Plan

## Overview

Implement the Worker Process Manager (Component 7) to spawn and manage separate OS processes that maintain orderbook state. Each worker processes messages from its dedicated queue and bypasses Python's GIL for true CPU parallelism.

## Current State Analysis

### What Exists:
- **MessageRouter** (`src/router.py:76-318`): Routes messages via consistent hashing to multiprocessing queues
- **OrderbookState** (`src/orderbook/orderbook.py:9-171`): Manages individual orderbook with `apply_snapshot()`, `apply_price_change()`
- **OrderbookStore** (`src/orderbook/orderbook_store.py:11-68`): Registry managing multiple orderbook states with `register_asset()`, `get_state()`, `memory_usage()`
- **Protocol Messages** (`src/messages/protocol.py`): Complete ParsedMessage, EventType enum, BookSnapshot, PriceChange, LastTradePrice
- **Reference Implementation** (`src/exercises/worker.py:16-62`): Simplified worker pattern (doesn't match full spec)

### What's Missing:
- `src/worker.py` - Worker Process Manager with process lifecycle, health monitoring, and stats collection
- Integration between WorkerManager and MessageRouter

### Key Architecture Decisions:
1. **Queue Ownership**: WorkerManager creates and owns multiprocessing queues, injects them into MessageRouter
2. **Message Processing**: Each ParsedMessage contains a single PriceChange (router yields separately), no iteration needed
3. **API Compatibility**: Worker calls `apply_snapshot(snapshot, timestamp)` and `apply_price_change(price_change, timestamp)` with both parameters
4. **Handler Factory**: Deferred to future enhancement (v1 focuses on core functionality)

## Desired End State

A production-ready Worker Process Manager that:
- Spawns N worker processes with dedicated input queues and shared stats queue
- Processes ParsedMessage objects and updates OrderbookStore state
- Tracks health via heartbeats and reports stats periodically
- Integrates with MessageRouter for message distribution
- Supports graceful shutdown with timeout and force-terminate fallback

### Verification:
```bash
# Unit tests pass
uv run pytest tests/test_worker.py -v

# Integration tests pass
uv run pytest tests/test_worker_integration.py -v
```

## What We're NOT Doing

- Handler factory pattern for custom callbacks (deferred to v2)
- Worker crash recovery and automatic restart (handled by Component 10: Application Orchestrator)
- Persistent state storage (workers are stateless, rebuild from snapshots)
- Dynamic worker scaling (fixed pool size)
- Advanced monitoring (Prometheus metrics, distributed tracing) - deferred to Phase 8

## Implementation Approach

Incremental approach building from dataclasses → helper functions → worker process → manager class:

1. **Phase 1**: Define WorkerStats dataclass and WorkerManager skeleton
2. **Phase 2**: Implement `_process_message()` helper function (unit testable)
3. **Phase 3**: Implement `_worker_process()` entry point with message loop
4. **Phase 4**: Add heartbeat and stats reporting timers
5. **Phase 5**: Implement WorkerManager lifecycle (start/stop)
6. **Phase 6**: Implement stats collection and health monitoring
7. **Phase 7**: Integration testing with MessageRouter

Each phase is independently testable and incrementally adds functionality.

### Worker Count Guidelines

**Choosing num_workers**:
- Typically set to `cpu_count()` or `cpu_count() - 1` (leave one core for main process/OS)
- Example: 8-core machine → use 7 or 8 workers
- Check available cores: `python -c "import os; print(os.cpu_count())"`

**Practical Limits**:
- Hard limit: System CPU core count
- Soft limit: Available memory (each worker can use 50-200MB depending on orderbook count)
- Recommended max: 16 workers (diminishing returns beyond this due to queue contention)

**Performance Considerations**:
- More workers = better CPU parallelism but higher overhead
- Fewer workers = lower overhead but potential bottleneck
- Optimal: Match or slightly exceed core count for CPU-bound workloads

---

## Phase 1: Core Structure & Stats Dataclass

### Overview
Create the basic structure for `src/worker.py` with WorkerStats dataclass and WorkerManager skeleton. This establishes the foundation for all subsequent phases.

### Changes Required

#### 1. Create `src/worker.py`

**New File**: `src/worker.py`

```python
"""
Worker Process Manager - Manages CPU-intensive orderbook workers.

Each worker:
- Runs in separate OS process (bypasses GIL)
- Owns subset of orderbooks (determined by consistent hash in router)
- Processes messages from dedicated multiprocessing queue
- Reports health and stats periodically
"""

import logging
import os
import signal
import time
from dataclasses import dataclass
from multiprocessing import Event as MPEvent
from multiprocessing import Process
from multiprocessing import Queue as MPQueue
from multiprocessing.synchronize import Event
from queue import Empty, Full

from src.messages.protocol import EventType, ParsedMessage
from src.orderbook.orderbook_store import Asset, OrderbookStore

logger = logging.getLogger(__name__)

# Configuration constants
QUEUE_TIMEOUT: float = 0.1  # 100ms timeout for queue.get()
HEARTBEAT_INTERVAL: float = 5.0  # Seconds between heartbeat updates
STATS_INTERVAL: float = 30.0  # Seconds between stats reports


@dataclass(slots=True)
class WorkerStats:
    """Per-worker performance statistics."""

    messages_processed: int = 0
    updates_applied: int = 0
    snapshots_received: int = 0
    processing_time_ms: float = 0.0
    last_message_ts: float = 0.0
    orderbook_count: int = 0
    memory_usage_bytes: int = 0

    @property
    def avg_processing_time_us(self) -> float:
        """Average per-message processing time in microseconds."""
        if self.messages_processed > 0:
            return (self.processing_time_ms * 1000) / self.messages_processed
        return 0.0


class WorkerManager:
    """
    Manages pool of worker processes.

    Usage:
        manager = WorkerManager(num_workers=4)
        queues = manager.get_input_queues()  # Pass to router
        manager.start()
        # ... run ...
        manager.stop()
    """

    __slots__ = (
        "_num_workers",
        "_processes",
        "_input_queues",
        "_stats_queue",
        "_shutdown_event",
        "_running",
    )

    def __init__(self, num_workers: int) -> None:
        """
        Initialize manager.

        Args:
            num_workers: Number of worker processes to spawn

        Raises:
            ValueError: If num_workers < 1
        """
        if num_workers < 1:
            raise ValueError("num_workers must be at least 1")

        # Warn if num_workers exceeds CPU count
        cpu_count = os.cpu_count() or 1
        if num_workers > cpu_count:
            logger.warning(
                f"num_workers ({num_workers}) exceeds CPU count ({cpu_count}). "
                f"This may cause performance degradation due to context switching."
            )

        self._num_workers = num_workers
        self._processes: list[Process] = []
        self._input_queues: list[MPQueue] = []
        self._stats_queue: MPQueue = MPQueue()
        self._shutdown_event: Event = MPEvent()
        self._running = False

    def get_input_queues(self) -> list[MPQueue]:
        """
        Get input queues for workers.

        Creates queues on first call. Router should use these queues
        and validate count matches num_workers.

        Returns:
            List of multiprocessing.Queue, one per worker
        """
        if not self._input_queues:
            # Match router's WORKER_QUEUE_SIZE (5000)
            self._input_queues = [
                MPQueue(maxsize=5000) for _ in range(self._num_workers)
            ]
        return self._input_queues

    @property
    def num_workers(self) -> int:
        """Number of worker processes."""
        return self._num_workers

    def start(self) -> None:
        """Start all worker processes (to be implemented in Phase 5)."""
        raise NotImplementedError

    def stop(self, timeout: float = 10.0) -> None:
        """Stop all workers gracefully (to be implemented in Phase 5)."""
        raise NotImplementedError

    def get_stats(self) -> dict[int, WorkerStats]:
        """Collect latest stats from all workers (to be implemented in Phase 6)."""
        raise NotImplementedError

    def is_healthy(self) -> bool:
        """Check if all workers are alive (to be implemented in Phase 6)."""
        raise NotImplementedError

    def get_alive_count(self) -> int:
        """Count of alive worker processes (to be implemented in Phase 6)."""
        raise NotImplementedError
```

#### 2. Update MessageRouter to Require Queues

**File**: `src/router.py:99-122`

**Current Code**:
```python
def __init__(self, num_workers: int) -> None:
    """Initialize router."""
    if num_workers < 1:
        raise ValueError("num_workers must be at least 1")

    self._num_workers = num_workers
    self._worker_queues: list[WorkerQueue] = [
        MPQueue(maxsize=WORKER_QUEUE_SIZE) for _ in range(num_workers)
    ]
    # ... rest of init ...
```

**New Code**:
```python
def __init__(
    self,
    num_workers: int,
    worker_queues: list[WorkerQueue],
) -> None:
    """
    Initialize router.

    Args:
        num_workers: Number of worker processes to route to
        worker_queues: Pre-created queues from WorkerManager.
                      Must have exactly one queue per worker.

    Raises:
        ValueError: If num_workers < 1 or queue count doesn't match num_workers
    """
    if num_workers < 1:
        raise ValueError("num_workers must be at least 1")

    if len(worker_queues) != num_workers:
        raise ValueError(
            f"worker_queues length ({len(worker_queues)}) must match "
            f"num_workers ({num_workers})"
        )

    self._num_workers = num_workers
    self._worker_queues = worker_queues

    self._async_queue: asyncio.Queue[tuple[str, ParsedMessage, float]] = (
        asyncio.Queue(maxsize=ASYNC_QUEUE_SIZE)
    )
    self._routing_task: asyncio.Task[None] | None = None
    self._running = False
    self._stats = RouterStats()
    self._asset_worker_cache: dict[str, int] = {}
```

**Key Changes**:
- `worker_queues` is now **required** (not optional)
- Router never creates its own queues
- Validation ensures exactly one queue per worker

### Success Criteria

#### Automated Verification:
- [ ] File `src/worker.py` exists and imports successfully: `uv run python -c "from src.worker import WorkerManager, WorkerStats"`
- [ ] No linting errors: `just check`
- [ ] All tests pass: `just test`

#### Manual Verification:
- [ ] WorkerStats slot count is correct (should have 7 fields + property)
- [ ] Queue maxsize matches router's WORKER_QUEUE_SIZE constant (5000)

**Implementation Note**: After all automated tests pass, proceed to Phase 2.

---

## Phase 2: Message Processing Logic

### Overview
Implement `_process_message()` helper function that handles BOOK, PRICE_CHANGE, and LAST_TRADE_PRICE events. This function is pure (no I/O) and easily unit testable.

### Changes Required

#### 1. Add `_process_message()` Function

**File**: `src/worker.py` (add after WorkerStats definition, before WorkerManager)

```python
def _process_message(
    store: OrderbookStore,
    message: ParsedMessage,
    stats: WorkerStats,
) -> None:
    """
    Process single message and update orderbook state.

    Args:
        store: OrderbookStore instance for this worker
        message: Parsed message to process
        stats: Stats tracker to update
    """
    stats.messages_processed += 1

    try:
        match message.event_type:
            case EventType.BOOK:
                _process_book_snapshot(store, message, stats)

            case EventType.PRICE_CHANGE:
                _process_price_change(store, message, stats)

            case EventType.LAST_TRADE_PRICE:
                _process_last_trade(store, message, stats)

            case _:
                logger.warning(f"Unknown event type: {message.event_type}")

    except AssertionError as e:
        logger.error(
            f"Assertion failed processing {message.event_type}: {e}",
            exc_info=True,
        )
    except Exception as e:
        logger.exception(
            f"Error processing {message.event_type} for asset {message.asset_id}: {e}"
        )


def _process_book_snapshot(
    store: OrderbookStore,
    message: ParsedMessage,
    stats: WorkerStats,
) -> None:
    """Process BOOK snapshot message."""
    assert message.book is not None, "BOOK message missing book field"

    asset = Asset(asset_id=message.asset_id, market=message.market)
    orderbook = store.register_asset(asset)
    orderbook.apply_snapshot(message.book, message.raw_timestamp)

    stats.snapshots_received += 1
    stats.updates_applied += 1


def _process_price_change(
    store: OrderbookStore,
    message: ParsedMessage,
    stats: WorkerStats,
) -> None:
    """Process PRICE_CHANGE message."""
    assert message.price_change is not None, "PRICE_CHANGE message missing price_change field"

    orderbook = store.get_state(message.asset_id)
    if orderbook:
        orderbook.apply_price_change(message.price_change, message.raw_timestamp)
        stats.updates_applied += 1
    else:
        logger.debug(
            f"Skipping price_change for unknown asset {message.asset_id}"
        )


def _process_last_trade(
    store: OrderbookStore,
    message: ParsedMessage,
    stats: WorkerStats,
) -> None:
    """Process LAST_TRADE_PRICE message."""
    assert message.last_trade is not None, "LAST_TRADE_PRICE message missing last_trade field"

    # No state update for trades in v1 (handler_factory deferred)
    # In future, custom handlers would receive callback here
    pass
```

**Key Design Notes**:
- Uses structural pattern matching (`match`/`case`) for clean event type dispatch
- Separate handler functions for each event type improve testability
- Assertions validate message structure; caught and logged on failure
- Uses `message.asset_id` and `message.market` from top-level ParsedMessage (not nested in book)
- Calls `apply_snapshot(snapshot, timestamp)` with both parameters
- Processes single PriceChange directly (no iteration over `changes` list)
- Skips LAST_TRADE_PRICE updates (no handler_factory in v1)
- Only updates stats.updates_applied when state actually changes
- All imports at module level (no function-level imports)

### Success Criteria

#### Automated Verification:
- [ ] No linting errors: `just check`
- [ ] All tests pass: `just test`

#### Manual Verification:
- [ ] Message processing logic matches actual OrderbookState API signatures
- [ ] Stats tracking accurately reflects applied updates vs received messages

**Implementation Note**: After all automated tests pass, proceed to Phase 3.

---

## Phase 3: Worker Process Entry Point

### Overview
Implement `_worker_process()` module-level function that serves as the entry point for spawned processes. Includes message receive loop, signal handling, and graceful shutdown via sentinel.

### Changes Required

#### 1. Add `_worker_process()` Function

**File**: `src/worker.py` (add after `_process_message()`, before WorkerManager)

```python
def _worker_process(
    worker_id: int,
    input_queue: MPQueue,
    stats_queue: MPQueue,
    shutdown_event: Event,
) -> None:
    """
    Worker process entry point.

    Runs in separate process - must be pickleable (module-level function).

    Args:
        worker_id: Unique identifier for this worker (0-indexed)
        input_queue: Queue to receive (ParsedMessage, timestamp) tuples
        stats_queue: Queue to send periodic stats updates
        shutdown_event: Multiprocessing event for shutdown coordination
    """
    # Ignore SIGINT - parent process handles graceful shutdown
    signal.signal(signal.SIGINT, signal.SIG_IGN)

    # Initialize worker state
    store = OrderbookStore()
    stats = WorkerStats()

    logger.info(f"Worker {worker_id} started")

    try:
        while not shutdown_event.is_set():
            try:
                # Get message with timeout (prevents infinite blocking)
                item = input_queue.get(timeout=QUEUE_TIMEOUT)

                if item is None:
                    # None is shutdown sentinel from router.stop()
                    logger.info(f"Worker {worker_id} received shutdown sentinel")
                    break

                # Unpack message and timestamp
                message, receive_ts = item
                process_start = time.monotonic()

                # Process the message
                _process_message(store, message, stats)

                # Track processing time
                process_time_ms = (time.monotonic() - process_start) * 1000
                stats.processing_time_ms += process_time_ms
                stats.last_message_ts = time.monotonic()

            except Empty:
                # Timeout - check shutdown_event and continue
                continue

            except Exception as e:
                logger.exception(f"Worker {worker_id} error processing message: {e}")
                # Continue processing despite errors

    except Exception as e:
        logger.exception(f"Worker {worker_id} fatal error: {e}")

    finally:
        logger.info(
            f"Worker {worker_id} stopping. "
            f"Processed {stats.messages_processed} messages, "
            f"managing {len(store._books)} orderbooks"
        )
```

**Key Design Notes**:
- Module-level function (required for multiprocessing pickle)
- Signal handling: ignore SIGINT (parent handles it via shutdown_event)
- Timeout prevents infinite blocking if queue empty
- None sentinel triggers immediate shutdown
- Errors logged but don't crash worker (resilience)
- Final log shows lifetime stats

### Success Criteria

#### Automated Verification:
- [ ] No linting errors: `just check`
- [ ] All tests pass: `just test`

#### Manual Verification:
- [ ] Worker process logs appear correctly in output
- [ ] Processing latency is reasonable (<5ms for typical updates)
- [ ] Worker cleanup happens even on unexpected exceptions

**Implementation Note**: After all automated tests pass, proceed to Phase 4.

---

## Phase 4: Health Monitoring

### Overview
Add heartbeat and periodic stats reporting to `_worker_process()`. Heartbeat updates orderbook_count and memory_usage every 5 seconds. Stats reports sent to stats_queue every 30 seconds.

### Changes Required

#### 1. Update `_worker_process()` with Timers

**File**: `src/worker.py` - Replace existing `_worker_process()` implementation

**Updated Code**:
```python
def _worker_process(
    worker_id: int,
    input_queue: MPQueue,
    stats_queue: MPQueue,
    shutdown_event: Event,
) -> None:
    """
    Worker process entry point with health monitoring.

    Runs in separate process - must be pickleable (module-level function).

    Args:
        worker_id: Unique identifier for this worker (0-indexed)
        input_queue: Queue to receive (ParsedMessage, timestamp) tuples
        stats_queue: Queue to send periodic stats updates
        shutdown_event: Multiprocessing event for shutdown coordination
    """
    # Ignore SIGINT - parent process handles graceful shutdown
    signal.signal(signal.SIGINT, signal.SIG_IGN)

    # Initialize worker state
    store = OrderbookStore()
    stats = WorkerStats()

    # Initialize timers
    last_heartbeat = time.monotonic()
    last_stats_report = time.monotonic()

    logger.info(f"Worker {worker_id} started")

    try:
        while not shutdown_event.is_set():
            try:
                # Get message with timeout (prevents infinite blocking)
                item = input_queue.get(timeout=QUEUE_TIMEOUT)

                if item is None:
                    # None is shutdown sentinel from router.stop()
                    logger.info(f"Worker {worker_id} received shutdown sentinel")
                    break

                # Unpack message and timestamp
                message, receive_ts = item
                process_start = time.monotonic()

                # Process the message
                _process_message(store, message, stats)

                # Track processing time
                process_time_ms = (time.monotonic() - process_start) * 1000
                stats.processing_time_ms += process_time_ms
                stats.last_message_ts = time.monotonic()

            except Empty:
                # Timeout - check shutdown_event and continue
                pass

            except Exception as e:
                logger.exception(f"Worker {worker_id} error processing message: {e}")
                # Continue processing despite errors

            # Periodic heartbeat - update orderbook and memory stats
            now = time.monotonic()
            if now - last_heartbeat >= HEARTBEAT_INTERVAL:
                stats.orderbook_count = len(store._books)
                stats.memory_usage_bytes = store.memory_usage()
                last_heartbeat = now

            # Periodic stats report - send to parent process
            if now - last_stats_report >= STATS_INTERVAL:
                try:
                    # Non-blocking put - drop if queue full (don't block worker)
                    stats_queue.put_nowait((worker_id, stats))
                except Full:
                    # Stats queue full - skip this report
                    pass
                last_stats_report = now

    except Exception as e:
        logger.exception(f"Worker {worker_id} fatal error: {e}")

    finally:
        # Send final stats before exit
        try:
            stats.orderbook_count = len(store._books)
            stats.memory_usage_bytes = store.memory_usage()
            stats_queue.put_nowait((worker_id, stats))
        except Full:
            pass

        logger.info(
            f"Worker {worker_id} stopping. "
            f"Processed {stats.messages_processed} messages, "
            f"managing {stats.orderbook_count} orderbooks, "
            f"memory: {stats.memory_usage_bytes / 1024 / 1024:.2f}MB"
        )
```

**Key Design Notes**:
- Heartbeat runs every 5 seconds (HEARTBEAT_INTERVAL)
- Stats report every 30 seconds (STATS_INTERVAL)
- Non-blocking `put_nowait()` prevents worker stall if stats queue full
- Final stats sent on shutdown
- Memory usage formatted in MB for readability

### Success Criteria

#### Automated Verification:
- [ ] No linting errors: `just check`
- [ ] All tests pass: `just test`

#### Manual Verification:
- [ ] Stats appear in queue at expected intervals
- [ ] Memory usage calculation is reasonable
- [ ] Heartbeat doesn't impact message processing performance

**Implementation Note**: After all automated tests pass, proceed to Phase 5.

---

## Phase 5: Process Lifecycle Management

### Overview
Implement WorkerManager.start() and stop() methods to spawn processes and manage graceful shutdown. Includes shutdown event signaling, sentinel values, timeout handling, and force termination fallback.

### Changes Required

#### 1. Implement `WorkerManager.start()`

**File**: `src/worker.py` - Replace NotImplementedError in start()

```python
def start(self) -> None:
    """Start all worker processes."""
    if self._running:
        logger.warning("WorkerManager already running")
        return

    self._shutdown_event.clear()
    queues = self.get_input_queues()

    for worker_id in range(self._num_workers):
        process = Process(
            target=_worker_process,
            args=(
                worker_id,
                queues[worker_id],
                self._stats_queue,
                self._shutdown_event,
            ),
            name=f"orderbook-worker-{worker_id}",
            daemon=False,  # Not daemon - we want explicit lifecycle control
        )
        process.start()
        self._processes.append(process)

    self._running = True
    logger.info(f"WorkerManager started {self._num_workers} worker processes")
```

#### 2. Implement `WorkerManager.stop()`

**File**: `src/worker.py` - Replace NotImplementedError in stop()

```python
def stop(self, timeout: float = 10.0) -> None:
    """
    Stop all workers gracefully with timeout.

    Uses three-stage shutdown:
    1. Set shutdown_event to signal workers to stop
    2. Send None sentinel to each queue (in case worker blocked on queue)
    3. Wait for graceful exit with timeout, then force terminate

    Args:
        timeout: Max seconds to wait for graceful shutdown per worker
    """
    if not self._running:
        logger.warning("WorkerManager not running")
        return

    logger.info(f"Stopping {self._num_workers} worker processes...")

    # Stage 1: Signal shutdown via event
    self._shutdown_event.set()

    # Stage 2: Send sentinel values to unblock any waiting workers
    for i, queue in enumerate(self._input_queues):
        try:
            queue.put_nowait(None)
        except Full:
            logger.warning(f"Could not send shutdown sentinel to worker {i}")

    # Stage 3: Wait for graceful exit, then force terminate
    deadline = time.monotonic() + timeout

    for process in self._processes:
        remaining = max(0.0, deadline - time.monotonic())
        process.join(timeout=remaining)

        if process.is_alive():
            logger.warning(
                f"Worker {process.name} did not stop gracefully, force terminating"
            )
            process.terminate()
            process.join(timeout=1.0)

            if process.is_alive():
                logger.error(f"Worker {process.name} did not terminate, killing")
                process.kill()
                process.join(timeout=1.0)

    self._processes.clear()
    self._running = False

    logger.info("All worker processes stopped")
```

**Key Design Notes**:
- Three-stage shutdown ensures workers stop even if hung
- Non-blocking sentinel put (don't block on full queue)
- Shared timeout budget across all workers
- Escalates: join → terminate → kill
- Clears process list after shutdown

### Success Criteria

#### Automated Verification:
- [ ] No linting errors: `just check`
- [ ] All tests pass: `just test`

#### Manual Verification:
- [ ] Worker processes visible in system process list while running
- [ ] Worker processes disappear after stop()
- [ ] Graceful shutdown completes in reasonable time (<5 seconds for 4 workers)
- [ ] No zombie processes left behind

**Implementation Note**: After all automated tests pass, proceed to Phase 6.

---

## Phase 6: Stats Collection & Health Checks

### Overview
Implement WorkerManager.get_stats(), is_healthy(), and get_alive_count() for monitoring worker health and collecting performance metrics.

### Changes Required

#### 1. Implement `get_stats()`

**File**: `src/worker.py` - Replace NotImplementedError in get_stats()

```python
def get_stats(self) -> dict[int, WorkerStats]:
    """
    Collect latest stats from all workers.

    Non-blocking - drains available stats from queue without waiting.
    Returns most recent stats received from each worker.

    Returns:
        Dict mapping worker_id to WorkerStats
    """
    stats: dict[int, WorkerStats] = {}

    # Drain all available stats from queue
    while True:
        try:
            worker_id, worker_stats = self._stats_queue.get_nowait()
            stats[worker_id] = worker_stats
        except Empty:
            break

    return stats
```

#### 2. Implement `is_healthy()`

**File**: `src/worker.py` - Replace NotImplementedError in is_healthy()

```python
def is_healthy(self) -> bool:
    """
    Check if all worker processes are alive.

    Returns:
        True if all workers are running, False if any worker died
    """
    if not self._running:
        return False

    return all(p.is_alive() for p in self._processes)
```

#### 3. Implement `get_alive_count()`

**File**: `src/worker.py` - Replace NotImplementedError in get_alive_count()

```python
def get_alive_count(self) -> int:
    """
    Count of alive worker processes.

    Returns:
        Number of workers currently running
    """
    return sum(1 for p in self._processes if p.is_alive())
```

### Success Criteria

#### Automated Verification:
- [ ] No linting errors: `just check`
- [ ] All tests pass: `just test`

#### Manual Verification:
- [ ] Stats appear after STATS_INTERVAL (30 seconds)
- [ ] Health check accurately reflects worker state
- [ ] get_stats() doesn't block main thread

**Implementation Note**: After all automated tests pass, proceed to Phase 7.

---

## Phase 7: Integration Testing

### Overview
Test complete integration between WorkerManager, MessageRouter, OrderbookStore, and message processing. Verify end-to-end message flow from router to worker to state update.

### Changes Required

#### 1. Create Integration Test Suite

**New File**: `tests/test_worker_integration.py`

```python
"""Integration tests for WorkerManager with MessageRouter."""

import asyncio
import time
from multiprocessing import Queue as MPQueue

import pytest

from src.messages.parser import MessageParser
from src.messages.protocol import EventType
from src.router import MessageRouter
from src.worker import WorkerManager


@pytest.fixture
def worker_manager():
    """Create WorkerManager for testing."""
    manager = WorkerManager(num_workers=2)
    yield manager
    if manager._running:
        manager.stop()


@pytest.fixture
def message_router(worker_manager):
    """Create MessageRouter with WorkerManager queues."""
    queues = worker_manager.get_input_queues()
    router = MessageRouter(num_workers=2, worker_queues=queues)
    yield router
    if router._running:
        asyncio.run(router.stop())


@pytest.mark.asyncio
async def test_integration_router_to_worker(worker_manager, message_router):
    """Test message flow from router to worker to orderbook state."""
    # Start components
    worker_manager.start()
    await message_router.start()

    # Create test message (book snapshot)
    parser = MessageParser()
    raw_message = b'{"event_type":"book","asset_id":"123","market":"0xabc","timestamp":1234567890,"hash":"test","bids":[["0.50","100"]],"asks":[["0.51","100"]]}'

    messages = list(parser.parse_messages(raw_message))
    assert len(messages) == 1

    # Route message
    success = await message_router.route_message("conn1", messages[0])
    assert success

    # Wait for processing
    await asyncio.sleep(0.5)

    # Verify stats show processing
    stats = worker_manager.get_stats()
    assert len(stats) > 0

    # At least one worker processed messages
    total_processed = sum(s.messages_processed for s in stats.values())
    assert total_processed >= 1

    # Cleanup
    await message_router.stop()
    worker_manager.stop()


@pytest.mark.asyncio
async def test_integration_multiple_workers(worker_manager, message_router):
    """Test messages distributed across multiple workers."""
    worker_manager.start()
    await message_router.start()

    parser = MessageParser()

    # Send messages for different assets
    for asset_id in ["asset1", "asset2", "asset3", "asset4"]:
        raw = (
            f'{{"event_type":"book","asset_id":"{asset_id}",'
            f'"market":"0xabc","timestamp":1234567890,"hash":"test",'
            f'"bids":[["0.50","100"]],"asks":[["0.51","100"]]}}'
        ).encode()

        messages = list(parser.parse_messages(raw))
        await message_router.route_message("conn1", messages[0])

    # Wait for processing
    await asyncio.sleep(1.0)

    # Both workers should have processed messages
    stats = worker_manager.get_stats()
    assert len(stats) == 2

    worker0_count = stats[0].messages_processed if 0 in stats else 0
    worker1_count = stats[1].messages_processed if 1 in stats else 0

    assert worker0_count > 0
    assert worker1_count > 0

    # Cleanup
    await message_router.stop()
    worker_manager.stop()


def test_integration_worker_crash_detection(worker_manager):
    """Test health monitoring detects crashed worker."""
    worker_manager.start()

    # Initially healthy
    assert worker_manager.is_healthy()
    assert worker_manager.get_alive_count() == 2

    # Kill a worker
    worker_manager._processes[0].terminate()
    worker_manager._processes[0].join(timeout=2.0)

    # Should detect unhealthy
    assert not worker_manager.is_healthy()
    assert worker_manager.get_alive_count() == 1

    # Cleanup
    worker_manager.stop()
```

#### 2. Update Main Application (Optional Demo)

**File**: `src/main.py` (add after connection pool setup)

Example integration (not required for tests):

```python
# Create worker manager and router
worker_manager = WorkerManager(num_workers=4)
worker_queues = worker_manager.get_input_queues()
router = MessageRouter(num_workers=4, worker_queues=worker_queues)

# Start workers and router
worker_manager.start()
await router.start()

# Use router in connection callback
async def on_message(connection_id: str, message: ParsedMessage):
    await router.route_message(connection_id, message)

# ... existing connection code ...

# Cleanup
await router.stop()
worker_manager.stop()
```

### Success Criteria

#### Automated Verification:
- [ ] No linting errors: `just check`
- [ ] All tests pass: `just test`

#### Manual Verification:
- [ ] Run application with workers: `uv run src/main.py` (should spawn workers and process messages)
- [ ] Check logs show worker startup messages
- [ ] Verify orderbook state updates correctly via debug output
- [ ] Confirm graceful shutdown on Ctrl+C
- [ ] Monitor memory usage stays under 100MB per worker with typical load

**Implementation Note**: After all tests pass and manual verification succeeds, the implementation is complete.

---

## Testing Strategy

### Unit Tests (`tests/test_worker.py`)

**Coverage Areas**:
- WorkerStats dataclass and computed properties
- _process_message() logic for each event type
- _worker_process() lifecycle (spawn, run, stop)
- Stats tracking accuracy
- Health monitoring (heartbeat, stats reporting)
- WorkerManager lifecycle (start, stop, cleanup)
- Queue creation and management
- Error handling and resilience

**Test Patterns**:
```python
# Example: Test WorkerStats property
def test_worker_stats_avg_processing_time():
    stats = WorkerStats()
    stats.messages_processed = 10
    stats.processing_time_ms = 50.0
    assert stats.avg_processing_time_us == pytest.approx(5000.0)

# Example: Test message processing
def test_process_message_book():
    store = OrderbookStore()
    stats = WorkerStats()
    # ... create test message ...
    _process_message(store, message, stats)
    assert stats.messages_processed == 1
    assert stats.snapshots_received == 1
```

### Integration Tests (`tests/test_worker_integration.py`)

**Coverage Areas**:
- Router → Worker message flow
- Multiple workers with hash distribution
- Stats collection across processes
- Health monitoring detects crashes
- Graceful shutdown coordination
- Performance under load

**Test Patterns**:
```python
# Example: End-to-end flow
@pytest.mark.asyncio
async def test_e2e_message_processing():
    manager = WorkerManager(num_workers=2)
    queues = manager.get_input_queues()
    router = MessageRouter(num_workers=2, worker_queues=queues)

    manager.start()
    await router.start()

    # Send message
    await router.route_message("conn", test_message)

    # Verify processing
    await asyncio.sleep(0.5)
    stats = manager.get_stats()
    assert sum(s.messages_processed for s in stats.values()) >= 1

    await router.stop()
    manager.stop()
```

### Performance Tests

**Metrics to Validate**:
- Per-message processing time < 5ms (avg_processing_time_us < 5000)
- Memory usage < 100MB per worker for 1000 orderbooks
- Message throughput > 10,000 messages/second across 4 workers
- Graceful shutdown completes within 10 seconds

**Test Approach**:
```python
def test_performance_processing_latency():
    """Verify processing latency meets target."""
    # Process 10,000 messages
    # Check stats.avg_processing_time_us < 5000
    pass

def test_performance_memory_usage():
    """Verify memory stays within bounds."""
    # Process 1000 unique assets
    # Check stats.memory_usage_bytes < 100 * 1024 * 1024
    pass
```

---

## Performance Considerations

### Message Processing Efficiency
- Use `apply_price_change()` for deltas (faster than full snapshot)
- SortedDict provides O(log n) inserts, O(1) best bid/ask updates
- Integer arithmetic avoids float precision issues
- Cached best prices reduce computation

### Memory Management
- Each OrderbookState uses slots (reduced overhead)
- SortedDict memory: ~64 bytes base + 16 bytes per level
- Target: <100KB per orderbook average
- Monitor via `stats.memory_usage_bytes`

### Process Communication
- Non-blocking stats puts prevent worker stall
- Queue timeout prevents infinite blocking
- Batch message processing in router reduces context switches

### Optimization Opportunities (Future)
- Shared memory for orderbook state (Phase 4 lesson)
- Zero-copy message passing (Phase 6 lesson)
- Actor model for cross-worker queries (Phase 5 lesson)

---

## Migration Notes

### Updating Existing Code

If any code currently creates MessageRouter directly:

**Before**:
```python
router = MessageRouter(num_workers=4)
```

**After**:
```python
worker_manager = WorkerManager(num_workers=4)
queues = worker_manager.get_input_queues()
router = MessageRouter(num_workers=4, worker_queues=queues)

worker_manager.start()
await router.start()
```

### Breaking Change

**Important**: MessageRouter now **requires** `worker_queues` parameter. This is a breaking change:

- Old code: `MessageRouter(num_workers=4)` will **fail**
- New code: Must create WorkerManager first and pass its queues to router
- Reason: WorkerManager owns multiprocessing resources and their lifecycle

**Migration is required** for any existing code using MessageRouter.

---

## Open Questions (Resolved)

1. ~~Should we implement handler_factory in initial version?~~ **Deferred to v2**
2. ~~How to handle worker crashes mid-message?~~ **Handled by Component 10 (Application Orchestrator)**
3. ~~Should stats_queue be bounded or unbounded?~~ **Unbounded (default MPQueue) - non-blocking puts prevent backpressure**
4. ~~What to do if a worker consistently lags?~~ **Alert/metric in Component 10**

---

## Related Tickets

- ENG-002: Async Message Router (provides message source) - `src/router.py`
- ENG-006: Application Orchestrator (manages worker lifecycle, restart on crash)

---

## References

- **Specification**: `lessons/polymarket_websocket_system_spec.md:2060-2387`
- **Current Router**: `src/router.py:76-318`
- **OrderbookState API**: `src/orderbook/orderbook.py:37-84`
- **OrderbookStore API**: `src/orderbook/orderbook_store.py:24-67`
- **Message Protocol**: `src/messages/protocol.py:163-178`
- **Reference Implementation**: `src/exercises/worker.py:16-62`
- **Pattern Examples**: ConnectionPool (`src/connection/pool.py`), WebsocketConnection (`src/connection/websocket.py`)
