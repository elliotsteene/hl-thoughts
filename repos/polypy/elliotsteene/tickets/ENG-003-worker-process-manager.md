# ENG-003: Implement Worker Process Manager

**Status**: Open
**Priority**: High
**Component**: Phase 3 - Worker Process Manager
**Assignee**: Unassigned
**Created**: 2025-12-03

## Overview

Implement the Worker Process Manager (Component 7) to spawn and manage separate OS processes that maintain orderbook state. Each worker processes messages from its dedicated queue and bypasses Python's GIL for true CPU parallelism.

## Context

- **Current State**: Orderbook updates processed synchronously in main event loop (src/main.py:34-43)
- **Target State**: Multiple worker processes handling orderbook updates in parallel
- **Spec Location**: `lessons/polymarket_websocket_system_spec.md:2060-2387`
- **Research Doc**: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:224-278`
- **Reference**: `src/exercises/worker.py` (may not match spec exactly)

## Technical Requirements

### 1. Process Management
- Spawn separate OS processes using multiprocessing.Process
- Each worker owns subset of orderbooks determined by consistent hash
- Uses multiprocessing.Queue for message input
- Uses multiprocessing.Queue for stats output
- Uses multiprocessing.Event for shutdown coordination

### 2. Worker Process Function

Must be **module-level function** (pickleable):

```python
def _worker_process(
    worker_id: int,
    input_queue: MPQueue,
    stats_queue: MPQueue,
    shutdown_event: Event,
    handler_factory: Optional[Callable[[], Any]] = None
) -> None:
    """Worker process entry point - runs in separate process"""
```

**Responsibilities**:
- Initialize OrderbookStore for this worker's assets
- Process messages from input_queue (QUEUE_TIMEOUT = 100ms)
- Send heartbeat every 5 seconds
- Send stats report every 30 seconds
- Handle shutdown sentinel (None message)
- Ignore SIGINT (parent handles it)

### 3. Message Processing

```python
def _process_message(
    store: OrderbookStore,
    message: ParsedMessage,
    stats: WorkerStats,
    handler: Optional[Any]
) -> None:
    """Process single message and update state"""
```

**Handle three event types**:
1. **BOOK** (snapshot):
   - Get or create OrderbookState
   - Call `orderbook.apply_snapshot(message.book)`
   - Register market via `store.register_market()`
   - Invoke `handler.on_snapshot()` if handler exists

2. **PRICE_CHANGE** (delta):
   - For each change in message.price_change.changes:
     - Get existing orderbook (skip if not found)
     - Call `orderbook.apply_price_change(change)`
     - Invoke `handler.on_price_change()` if handler exists

3. **LAST_TRADE_PRICE**:
   - Get orderbook
   - Invoke `handler.on_trade()` if handler exists
   - (No state update - just callback)

### 4. Health Monitoring

**Heartbeat** (every 5 seconds):
- Update stats.orderbook_count = len(store)
- Update stats.memory_usage_bytes = store.memory_usage()

**Stats Report** (every 30 seconds):
- Send (worker_id, stats) tuple via stats_queue
- Non-blocking put (drop if queue full)

### 5. WorkerManager Class

```python
class WorkerManager:
    def __init__(
        num_workers: int,
        handler_factory: Optional[Callable[[], Any]] = None
    )
        """Initialize manager"""

    def get_input_queues() -> list[MPQueue]
        """Get input queues for workers"""

    def start() -> None
        """Start all worker processes"""

    def stop(timeout: float = 10.0) -> None
        """Graceful shutdown with force terminate fallback"""

    def get_stats() -> dict[int, WorkerStats]
        """Collect latest stats from all workers"""

    def is_healthy() -> bool
        """Check if all workers are alive"""

    def get_alive_count() -> int
        """Count of alive worker processes"""
```

### 6. Stats Tracking

```python
@dataclass(slots=True)
class WorkerStats:
    messages_processed: int = 0
    updates_applied: int = 0
    snapshots_received: int = 0
    processing_time_ms: float = 0.0
    last_message_ts: float = 0.0
    orderbook_count: int = 0
    memory_usage_bytes: int = 0

    @property
    def avg_processing_time_us(self) -> float:
        """Average per-message processing time in microseconds"""
```

## Dependencies

**Required Before Starting**:
- ✅ Component 1: Protocol Messages (ParsedMessage, EventType)
- ✅ Component 2: Orderbook State (OrderbookStore, OrderbookState)
- ⚠️ Component 6: Message Router (provides input queues) - Can develop in parallel

**Blocks**:
- ❌ Component 10: Application Orchestrator (needs workers for message processing)

## Implementation Plan

### Phase 1: Core Structure (2 days)
1. Create `src/worker.py`
2. Define WorkerStats dataclass
3. Implement WorkerManager.__init__()
4. Create input queues and stats queue
5. Implement get_input_queues()
6. Write unit tests for manager initialization

### Phase 2: Message Processing Logic (2 days)
1. Implement `_process_message()` function
2. Handle BOOK snapshot updates
3. Handle PRICE_CHANGE delta updates
4. Handle LAST_TRADE_PRICE events
5. Add stats tracking to each path
6. Test message processing with mock OrderbookStore

### Phase 3: Worker Process Function (3 days)
1. Implement `_worker_process()` entry point
2. Add signal handling (ignore SIGINT)
3. Initialize OrderbookStore per worker
4. Add message receive loop with timeout
5. Handle None sentinel for shutdown
6. Add error handling and logging
7. Test worker process in isolation

### Phase 4: Health Monitoring (1 day)
1. Add heartbeat timer (5 second interval)
2. Update orderbook_count and memory_usage
3. Add stats report timer (30 second interval)
4. Implement non-blocking stats queue put
5. Test timing accuracy

### Phase 5: Process Lifecycle (2 days)
1. Implement WorkerManager.start()
2. Spawn all worker processes with proper naming
3. Implement WorkerManager.stop() with timeout
4. Add shutdown event signaling
5. Add force terminate fallback
6. Test graceful shutdown
7. Test force terminate on hung worker

### Phase 6: Stats Collection (1 day)
1. Implement get_stats() to collect from stats_queue
2. Implement is_healthy() to check process.is_alive()
3. Implement get_alive_count()
4. Test stats aggregation
5. Test with dead worker detection

### Phase 7: Integration Testing (2 days)
1. Test with MessageRouter feeding messages
2. Test with multiple workers (4+)
3. Verify consistent hash routing to correct worker
4. Test orderbook state persistence across messages
5. Load test with high message volume
6. Verify no memory leaks over time

## Acceptance Criteria

- [ ] Worker processes start and run independently
- [ ] Messages processed and orderbook state updated correctly
- [ ] BOOK snapshots applied correctly
- [ ] PRICE_CHANGE deltas applied correctly
- [ ] LAST_TRADE_PRICE events handled
- [ ] Custom handlers receive callbacks on state changes
- [ ] Graceful shutdown completes within 10 second timeout
- [ ] Force terminate works if graceful shutdown fails
- [ ] Stats collected via stats_queue
- [ ] Health check detects dead workers
- [ ] Memory usage per worker < 100MB for 1000 orderbooks
- [ ] Processing latency < 5ms per update (performance target)
- [ ] All unit tests pass
- [ ] Integration tests with router pass

## Testing Strategy

### Unit Tests
- WorkerStats dataclass and computed properties
- _process_message() logic for each event type
- Stats tracking accuracy
- Handler callback invocation

### Integration Tests (with multiprocessing)
- Spawn 4 workers
- Send 1000 messages through queues
- Verify orderbook states updated
- Test graceful shutdown
- Test force terminate
- Collect and verify stats

### Performance Tests
- Measure per-message processing time
- Test with 100 orderbooks per worker
- Verify < 5ms average processing time
- Monitor memory usage over 1 hour
- Test with 10,000+ messages/second load

## Notes

- Reference implementation exists at `src/exercises/worker.py` but may not match spec
- Worker process function MUST be at module level (not nested) for pickling
- Each worker maintains its own OrderbookStore with subset of assets
- Workers are stateless - can be restarted without data loss (snapshots rebuild state)
- Signal handling: workers ignore SIGINT, parent process handles graceful shutdown

## Multiprocessing Considerations

- Queue serialization uses pickle (automatic for msgspec.Struct)
- Queue.get() with timeout prevents blocking forever
- Queue.put_nowait() prevents blocking on stats reporting
- Process.is_alive() checks if worker crashed
- Process.join(timeout) allows bounded wait for shutdown
- Process.terminate() forces kill if needed

## Handler Factory Pattern

Optional `handler_factory` allows per-worker custom callbacks:
```python
def my_handler_factory():
    return MyHandler()  # Create fresh instance per worker

manager = WorkerManager(num_workers=4, handler_factory=my_handler_factory)
```

Handlers must be pickleable (function reference, not lambda).

## Open Questions

1. Should we implement handler_factory in initial version or defer?
2. How to handle worker crashes mid-message? (Probably restart worker)
3. Should stats_queue be bounded or unbounded?
4. What to do if a worker consistently lags behind? (Probably alert/metric)
5. Should we implement worker restart on crash in this component or Component 10?

## Related Tickets

- ENG-002: Async Message Router (provides message source)
- ENG-006: Application Orchestrator (manages worker lifecycle)

## References

- Spec: `lessons/polymarket_websocket_system_spec.md:2060-2387`
- Research: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:224-278`
- Reference: `src/exercises/worker.py`
- Current Code: `src/orderbook/orderbook.py`, `src/orderbook/orderbook_store.py`
