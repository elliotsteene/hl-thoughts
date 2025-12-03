# ENG-002: Implement Async Message Router

**Status**: Open
**Priority**: High
**Component**: Phase 3 - Async Message Router
**Assignee**: Unassigned
**Created**: 2025-12-03

## Overview

Implement the Async Message Router (Component 6) to route messages from WebSocket connections (async domain) to worker processes (process domain) via multiprocessing queues. Uses consistent hashing to ensure the same asset always routes to the same worker.

## Context

- **Current State**: Messages processed synchronously in main event loop (src/main.py:28-58)
- **Target State**: Async router distributing messages to multiple worker processes for parallel CPU-intensive orderbook updates
- **Spec Location**: `lessons/polymarket_websocket_system_spec.md:1758-2057`
- **Research Doc**: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:171-221`

## Technical Requirements

### 1. Queue Architecture
Two-stage queuing system:

**Stage 1 - Async Queue**:
- Bounded asyncio.Queue (ASYNC_QUEUE_SIZE = 10,000)
- Receives messages from connection callbacks
- Provides backpressure handling

**Stage 2 - Worker Queues**:
- One multiprocessing.Queue per worker (WORKER_QUEUE_SIZE = 5,000)
- Routes to process domain
- Non-blocking puts with timeout

### 2. Consistent Hashing
```python
def _hash_to_worker(asset_id: str, num_workers: int) -> int:
    """Hash asset_id to worker index using MD5"""
    hash_bytes = hashlib.md5(asset_id.encode()).digest()
    hash_int = int.from_bytes(hash_bytes[:8], 'little')
    return hash_int % num_workers
```

Cache mappings in `_asset_worker_cache: dict[str, int]` for performance.

### 3. Batching Logic
- Collect messages up to BATCH_SIZE = 100
- Timeout after BATCH_TIMEOUT = 10ms
- Group by target worker before sending
- Reduces queue operation overhead

### 4. Backpressure Handling
- Non-blocking puts with PUT_TIMEOUT = 1ms
- Drop messages when worker queue full
- Track dropped count in RouterStats
- Log queue full events

### 5. Special Routing: Multi-Asset Messages
PRICE_CHANGE events can contain multiple assets (spec lines 2002-2008):
```python
if message.event_type == EventType.PRICE_CHANGE:
    seen_workers = set()
    for change in message.price_change.changes:
        worker_idx = get_worker_for_asset(change.asset_id)
        if worker_idx not in seen_workers:
            # Fan out to each affected worker
            worker_batches[worker_idx].append((message, receive_ts))
            seen_workers.add(worker_idx)
```

### 6. Key Methods to Implement

```python
class MessageRouter:
    def __init__(num_workers: int)
        """Initialize with worker count"""

    async def start() -> None
        """Start routing task"""

    async def stop() -> None
        """Stop routing, send None sentinel to all workers"""

    async def route_message(connection_id: str, message: ParsedMessage) -> bool
        """Entry point from connection callback, returns True if queued"""

    def get_worker_queues() -> list[MPQueue]
        """Get queues to pass to worker processes"""

    def get_worker_for_asset(asset_id: str) -> int
        """Get worker index for asset (cached)"""

    def get_queue_depths() -> dict[str, int]
        """Get current queue depths for monitoring"""

    async def _routing_loop() -> None
        """Internal: Batch and route messages"""

    async def _route_batch(batch: list) -> None
        """Internal: Group by worker and send"""
```

### 7. Stats Tracking

```python
@dataclass(slots=True)
class RouterStats:
    messages_routed: int = 0
    messages_dropped: int = 0
    batches_sent: int = 0
    queue_full_events: int = 0
    routing_errors: int = 0
    total_latency_ms: float = 0.0

    @property
    def avg_latency_ms(self) -> float:
        """Compute average routing latency"""
```

## Dependencies

**Required Before Starting**:
- ✅ Component 1: Protocol Messages (ParsedMessage, EventType)
- ⚠️ Component 5: Connection Pool (provides message callback) - Can develop in parallel
- ⚠️ Component 7: Worker Manager (provides queues) - Can develop in parallel

**Blocks**:
- ❌ Component 10: Application Orchestrator (needs router for message flow)

## Implementation Plan

### Phase 1: Core Structure (2 days)
1. Create `src/router.py`
2. Define RouterStats dataclass
3. Implement MessageRouter.__init__()
4. Create async and worker queues
5. Implement get_worker_queues() and get_queue_depths()
6. Write unit tests for initialization

### Phase 2: Consistent Hashing (1 day)
1. Implement `_hash_to_worker()` function
2. Add `_asset_worker_cache` dict
3. Implement `get_worker_for_asset()` with caching
4. Test hash distribution uniformity
5. Test cache hit rate

### Phase 3: Message Reception (2 days)
1. Implement `route_message()` entry point
2. Add message to async queue with receive timestamp
3. Handle QueueFull exception
4. Track dropped messages in stats
5. Test backpressure behavior

### Phase 4: Batching and Routing (3 days)
1. Implement `_routing_loop()` task
2. Add batch collection logic with timeout
3. Implement `_route_batch()` with worker grouping
4. Handle multi-asset PRICE_CHANGE fan-out
5. Add non-blocking puts to worker queues
6. Test batch timing and sizes

### Phase 5: Lifecycle Management (1 day)
1. Implement start() to spawn routing task
2. Implement stop() to cancel task
3. Send None sentinel to all worker queues
4. Test graceful shutdown
5. Verify no message loss during shutdown

### Phase 6: Integration Testing (2 days)
1. Create mock workers with multiprocessing.Queue
2. Test message distribution to workers
3. Verify consistent hashing (same asset → same worker)
4. Test multi-asset message fan-out
5. Load test with high message volume
6. Measure routing latency

## Acceptance Criteria

- [ ] Messages route to correct worker based on asset_id hash
- [ ] Same asset_id always routes to same worker (consistency verified)
- [ ] Backpressure handled gracefully (drops with logging, no crashes)
- [ ] Multi-asset PRICE_CHANGE events fan out correctly to multiple workers
- [ ] Latency tracking accurate to within 1ms
- [ ] Stop sends None sentinel to all workers
- [ ] Queue depths reportable for monitoring
- [ ] Batch sizes respect BATCH_SIZE limit (100 messages)
- [ ] Batch timeout works correctly (10ms max wait)
- [ ] Stats accurately track routed/dropped messages
- [ ] All unit tests pass
- [ ] Integration tests with mock workers pass
- [ ] Routing latency < 1ms average (performance target)

## Testing Strategy

### Unit Tests
- RouterStats dataclass and computed properties
- Consistent hash function distribution
- Worker cache hit/miss behavior
- Batch collection timing
- Queue full exception handling

### Integration Tests
- Route 10,000 messages to 4 workers
- Verify distribution matches hash function
- Test multi-asset message handling
- Test worker queue overflow behavior
- Test graceful shutdown with pending messages

### Performance Tests
- Measure routing latency under load
- Test with 100,000+ messages/second
- Verify < 1ms average latency
- Test queue depth monitoring accuracy
- Measure cache hit rate (should be >99%)

## Notes

- Reference implementation exists at `src/exercises/message_router.py`
- Can be developed in parallel with ENG-001 (Connection Pool) and ENG-003 (Worker Manager)
- The router acts as a bridge between async and multiprocessing domains
- Multiprocessing.Queue uses pickle for serialization (automatic)

## Performance Considerations

- Pre-compile hash function to avoid repeated MD5 setup
- Cache asset→worker mappings aggressively
- Use batch sending to reduce queue operation overhead
- Non-blocking puts prevent router from blocking on slow workers
- Consider queue depth monitoring to detect worker slowdown

## Open Questions

1. Should we implement custom serialization for ParsedMessage or rely on pickle?
2. What should happen if all workers are full? (Currently spec says drop)
3. Should we add a priority queue for certain event types?
4. How to handle worker crashes? (Probably Component 7's responsibility)

## Related Tickets

- ENG-001: Connection Pool Manager (provides message source)
- ENG-003: Worker Process Manager (provides queue consumers)
- ENG-006: Application Orchestrator (wires router into message flow)

## References

- Spec: `lessons/polymarket_websocket_system_spec.md:1758-2057`
- Research: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:171-221`
- Reference: `src/exercises/message_router.py`
- Current Code: `src/messages/protocol.py` (ParsedMessage)
