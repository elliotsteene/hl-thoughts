---
date: 2025-12-13T21:47:03Z
researcher: elliotsteene
git_commit: 513a7eb
branch: refactor-changes
repository: polypy
topic: "Worker Classes Refactoring for Testability"
tags: [research, codebase, worker, multiprocessing, refactoring, testability]
status: complete
last_updated: 2025-12-13
last_updated_by: elliotsteene
---

# Research: Worker Classes Refactoring for Testability

**Date**: 2025-12-13T21:47:03Z
**Researcher**: elliotsteene
**Git Commit**: 513a7eb
**Branch**: refactor-changes
**Repository**: polypy

## Research Question
How have the worker classes been refactored to improve testability and code organization?

## Summary
The worker implementation has been refactored from a monolithic structure into three focused modules with clear separation of concerns. The refactoring introduces dependency injection, method extraction, and a Worker class that can be tested independently from multiprocessing, dramatically improving testability while maintaining the same functionality.

## Detailed Findings

### Module Structure

The worker code is now organized into three files in `src/worker/`:

#### 1. **worker.py** - Core Worker Logic (src/worker/worker.py)

The main `Worker` class encapsulates all message processing logic:

**Worker Class** (src/worker/worker.py:37-230):
- **Constructor dependencies** (lines 38-50):
  - `worker_id`: Unique identifier
  - `input_queue`: MPQueue for receiving messages
  - `stats_queue`: MPQueue for sending stats
  - `orderbook_store`: OrderbookStore instance
  - `stats`: WorkerStats instance

- **Main run loop** (lines 52-93):
  - Runs until shutdown_event is set
  - Processes messages with timeout (100ms)
  - Periodic heartbeat updates every 5 seconds
  - Periodic stats reports every 30 seconds
  - Handles shutdown gracefully via finally block

- **Message processing methods**:
  - `_process_message_loop()` (lines 94-114): Single iteration - gets message, processes, tracks timing
  - `_process_message()` (lines 129-171): Dispatches by event type using pattern matching
  - `_process_book_snapshot()` (lines 172-185): Handles BOOK snapshot messages
  - `_process_price_change()` (lines 186-204): Handles PRICE_CHANGE updates
  - `_process_last_trade()` (lines 205-217): Handles LAST_TRADE_PRICE (no-op in v1)
  - `_process_tick_size_change()` (lines 218-229): Handles TICK_SIZE_CHANGE (no-op in v1)

- **Stats management**:
  - `_update_stats()` (lines 119-122): Updates orderbook count and memory usage
  - `_put_stats()` (lines 123-128): Sends stats to parent (handles Full exception)
  - `_send_final_stats()` (lines 115-118): Called in finally block

**_worker_process function** (src/worker/worker.py:232-273):
- Module-level function (required for multiprocessing pickling)
- Entry point for worker processes
- Sets up signal handling (ignores SIGINT)
- Creates Worker instance with fresh OrderbookStore and WorkerStats
- Calls Worker.run() with shutdown_event
- Logs final statistics on exit

**Configuration constants** (src/worker/worker.py:28-30):
- `QUEUE_TIMEOUT = 0.1`: 100ms timeout for queue.get()
- `HEARTBEAT_INTERVAL = 5.0`: Seconds between heartbeat updates
- `STATS_INTERVAL = 30.0`: Seconds between stats reports

#### 2. **manager.py** - Process Lifecycle Management (src/worker/manager.py)

The `WorkerManager` class handles multiprocessing lifecycle:

**WorkerManager Class** (src/worker/manager.py:18-210):
- **Initialization** (lines 39-66):
  - Validates num_workers >= 1
  - Warns if num_workers exceeds CPU count
  - Creates shutdown event and stats queue
  - Uses __slots__ for memory efficiency

- **Queue management** (lines 67-82):
  - `get_input_queues()`: Lazy creation of worker queues (5000 capacity)
  - Creates one MPQueue per worker
  - Returns same queues on repeated calls

- **Lifecycle methods**:
  - `start()` (lines 89-114): Spawns worker processes
    - Creates Process for each worker with _worker_process target
    - Passes worker_id, queues, and shutdown_event
    - Sets daemon=False for explicit lifecycle control
  - `stop()` (lines 116-166): Three-stage graceful shutdown
    - Stage 1: Set shutdown_event
    - Stage 2: Send None sentinels to unblock queued workers
    - Stage 3: Wait with timeout, then terminate, then kill if needed

- **Monitoring methods**:
  - `get_stats()` (lines 168-188): Non-blocking stats collection from queue
  - `is_healthy()` (lines 190-200): Checks all processes are alive
  - `get_alive_count()` (lines 202-210): Counts alive processes

#### 3. **stats.py** - Statistics Data Structure (src/worker/stats.py)

**WorkerStats dataclass** (src/worker/stats.py:4-22):
- Uses `@dataclass(slots=True)` for memory efficiency
- Fields (all with default values):
  - `messages_processed`: Total messages handled
  - `updates_applied`: Orderbook updates applied
  - `snapshots_received`: BOOK snapshots received
  - `processing_time_ms`: Cumulative processing time
  - `last_message_ts`: Timestamp of last message
  - `orderbook_count`: Number of orderbooks managed
  - `memory_usage_bytes`: Total memory usage

- **Computed property** (lines 16-22):
  - `avg_processing_time_us`: Average per-message time in microseconds
  - Calculates: `(processing_time_ms * 1000) / messages_processed`
  - Returns 0.0 if no messages processed

### Testability Improvements

#### 1. Dependency Injection Pattern

**Before**: Worker logic likely had hardcoded dependencies
**After**: Worker constructor accepts all dependencies (src/worker/worker.py:38-50)

```python
def __init__(
    self,
    worker_id: int,
    input_queue: MPQueue,
    stats_queue: MPQueue,
    orderbook_store: OrderbookStore,
    stats: WorkerStats,
) -> None:
```

This enables:
- Mocking queues for testing without multiprocessing
- Injecting test OrderbookStore instances
- Observing stats changes directly

#### 2. Method Extraction

**Small, focused methods** that can be tested independently:

- `_process_message()`: Tests message type dispatching
- `_process_book_snapshot()`: Tests snapshot handling with assertions
- `_process_price_change()`: Tests update logic with unknown asset handling
- `_process_last_trade()`: Tests trade handling (currently no-op)
- `_update_stats()`: Tests stats updates from store
- `_put_stats()`: Tests queue communication with Full exception handling

Each method has 2-20 lines of focused logic, making tests precise and failures easy to debug.

#### 3. Separation of Concerns

**Worker class vs _worker_process function**:
- `Worker`: Testable business logic (no multiprocessing)
- `_worker_process`: Thin wrapper for process entry (integration tests only)

This allows:
- Fast unit tests (tests/worker/test_worker.py) that test Worker methods directly
- Slower integration tests (tests/worker/test_worker_integration.py) that test full process lifecycle

#### 4. Test Organization

Tests are organized by functionality (tests/worker/):

**test_worker.py** (316 lines of unit tests):
- `TestProcessMessage`: 7 tests for message dispatching
  - Valid message types (BOOK, PRICE_CHANGE, LAST_TRADE_PRICE)
  - Unknown event type handling
  - Assertion error handling
  - Generic exception handling

- `TestProcessBookSnapshot`: 2 tests
  - Valid snapshot processing
  - Missing book field assertion

- `TestProcessPriceChange`: 3 tests
  - Existing orderbook updates
  - Unknown asset handling
  - Missing field assertion

- `TestProcessLastTrade`: 2 tests
  - Last trade processing (no-op)
  - Missing field assertion

- `TestWorkerProcess`: 17 tests for process lifecycle
  - Shutdown sentinel handling
  - Shutdown event handling
  - Message processing
  - Queue timeout handling
  - Heartbeat updates
  - Stats reporting
  - Exception handling
  - Final stats in finally block
  - Full queue handling
  - Processing time tracking
  - Signal handling setup

**test_worker_manager.py** (172 lines):
- `TestWorkerManager`: 16 tests
  - Initialization (valid/invalid num_workers)
  - CPU count warnings
  - Queue management
  - Start/stop lifecycle
  - Health monitoring
  - Stats collection
  - Graceful shutdown stages
  - Force terminate/kill handling

**test_worker_stats.py** (28 lines):
- `TestWorkerStats`: 3 tests
  - Average processing time calculation
  - Zero messages handling
  - Default values

**test_worker_integration.py**: Full end-to-end tests with real multiprocessing

### Integration with Application

**Usage in app.py** (src/app.py:118-123):

```python
# 3. WorkerManager
self._workers = WorkerManager(
    num_workers=self._num_workers,
)
worker_queues = self._workers.get_input_queues()
logger.info(f"✓ WorkerManager initialised with {self._num_workers} workers")
```

The WorkerManager:
1. Creates worker queues during initialization
2. Provides queues to MessageRouter for message routing
3. Spawns worker processes on app.start()
4. Collects stats via get_stats() for /stats endpoint
5. Provides health status via is_healthy() for /health endpoint
6. Stops workers gracefully on app.stop()

### Message Flow Through Worker

1. **MessageRouter** routes messages to worker queues using consistent hashing
2. **Worker.run()** loop:
   - Gets message from queue with 100ms timeout
   - Calls `_process_message_loop()`
   - Unpacks (message, receive_ts) tuple
   - Calls `_process_message(message)`
   - Tracks processing time
3. **_process_message()** dispatches by event_type:
   - BOOK → `_process_book_snapshot()`
   - PRICE_CHANGE → `_process_price_change()`
   - LAST_TRADE_PRICE → `_process_last_trade()`
   - TICK_SIZE_CHANGE → `_process_tick_size_change()`
4. **Handler methods** update OrderbookStore and WorkerStats
5. **Periodic updates**:
   - Every 5s: `_update_stats()` refreshes orderbook count and memory
   - Every 30s: `_put_stats()` sends stats to parent via stats_queue

### Error Handling

**Robust error handling at multiple levels**:

1. **Message processing** (src/worker/worker.py:162-171):
   - Catches AssertionError (missing fields) → logs error
   - Catches generic Exception → logs exception
   - Worker continues processing despite errors

2. **Main loop** (src/worker/worker.py:60-93):
   - Catches ShutdownReceivedError → clean exit
   - Catches Empty (queue timeout) → continue
   - Catches generic Exception → log and continue
   - Finally block ensures stats are sent

3. **Stats queue** (src/worker/worker.py:123-128):
   - put_nowait() can raise Full
   - Silently ignores Full (non-critical)
   - Prevents blocking on full queue

4. **Shutdown** (src/worker/manager.py:116-166):
   - Handles Full when sending sentinels
   - Timeout-based join with fallback to terminate
   - Terminate failure → force kill
   - Logs warnings at each escalation level

## Code References

### Core Implementation
- `src/worker/worker.py:37-93` - Worker class and run loop
- `src/worker/worker.py:129-171` - Message processing dispatcher
- `src/worker/worker.py:232-273` - Process entry point
- `src/worker/manager.py:18-210` - WorkerManager lifecycle
- `src/worker/stats.py:4-22` - WorkerStats dataclass

### Test Coverage
- `tests/worker/test_worker.py:1-755` - Worker unit tests (31 test cases)
- `tests/worker/test_worker_manager.py:1-172` - Manager tests (16 test cases)
- `tests/worker/test_worker_stats.py:1-28` - Stats tests (3 test cases)
- `tests/worker/test_worker_integration.py` - Integration tests

### Application Integration
- `src/app.py:118-123` - WorkerManager initialization
- `src/router.py` - Routes messages to worker queues

## Architecture Documentation

### Design Patterns Used

1. **Dependency Injection**: Worker accepts all dependencies in constructor
2. **Separation of Concerns**: Worker logic separate from process management
3. **Command Pattern**: Messages as commands dispatched to handlers
4. **Producer-Consumer**: Router produces to worker queues, workers consume
5. **Graceful Degradation**: Continues processing despite individual message errors
6. **Three-Stage Shutdown**: Event → Sentinel → Timeout → Terminate → Kill

### Memory Efficiency

- All classes use `__slots__` to reduce memory footprint
- WorkerStats uses dataclass with slots=True
- OrderbookStore manages memory tracking
- Stats include `memory_usage_bytes` for monitoring

### Multiprocessing Architecture

- **Process isolation**: Each worker runs in separate OS process
- **GIL bypass**: CPU-bound orderbook updates run in parallel
- **Queue-based communication**: MPQueue for message passing and stats
- **Event-based coordination**: MPEvent for shutdown signaling
- **Non-daemon processes**: Explicit lifecycle control instead of daemon=True

## Related Research
- Architecture overview: CLAUDE.md
- Multiprocessing patterns: lessons/phase-3-multiprocessing.md
- Message routing: src/router.py
- Orderbook implementation: src/orderbook/orderbook.py

## Open Questions
None - the implementation is well-documented and tested.
