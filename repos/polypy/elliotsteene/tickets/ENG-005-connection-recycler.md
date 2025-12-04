# ENG-005: Implement Connection Recycler

**Status**: Open
**Priority**: Medium
**Component**: Phase 4 - Connection Recycler
**Assignee**: Unassigned
**Created**: 2025-12-03

## Overview

Implement the Connection Recycler (Component 9) to handle zero-downtime connection recycling. Monitors connections for recycling triggers (pollution, age, health) and orchestrates seamless migration of active markets from old connections to new ones without message loss.

## Context

- **Current State**: No connection recycling - connections run until restart
- **Target State**: Automatic recycling when connections become polluted (>30% expired markets) or aged (>24 hours)
- **Spec Location**: `lessons/polymarket_websocket_system_spec.md:2750-3038`
- **Research Doc**: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:335-391`

## Technical Requirements

### 1. Recycling Triggers

Monitor connections and trigger recycling when:
1. **Pollution Ratio ≥ 30%**: `expired_markets / total_markets >= 0.30`
2. **Age ≥ 24 hours**: `connection_age >= 86400.0` seconds
3. **Unhealthy**: `connection.is_healthy == False`

### 2. Zero-Downtime Recycling Flow

Critical: **Both connections receive messages during transition**

```
1. Detect trigger (pollution/age/health)
2. Mark old connection as DRAINING (no new subscriptions)
3. Get list of active markets from old connection
4. Create new connection with those active markets
5. Wait STABILIZATION_DELAY = 3.0 seconds for new connection
6. Verify new connection is healthy
7. Atomically update registry (reassign markets)
8. Close old connection
```

**Result**: Zero message loss, brief message duplication (during overlap)

### 3. Concurrency Control

- MAX_CONCURRENT_RECYCLES = 2 via asyncio.Semaphore
- Track `_active_recycles: set[str]` to prevent duplicate recycling
- Skip connections already being recycled
- Skip connections already marked as DRAINING

### 4. ConnectionRecycler Class

```python
class ConnectionRecycler:
    def __init__(
        registry: MarketRegistry,
        pool: ConnectionPool
    )
        """Initialize recycler"""

    async def start() -> None
        """Start monitoring loop"""

    async def stop() -> None
        """Stop monitoring"""

    async def force_recycle(connection_id: str) -> bool
        """Manual intervention for specific connection"""

    def get_active_recycles() -> set[str]
        """Get connection IDs currently being recycled"""

    async def _monitor_loop() -> None
        """Internal: Check connections every 60s"""

    async def _check_all_connections() -> None
        """Internal: Iterate connections and check triggers"""

    async def _recycle_connection(connection_id: str) -> bool
        """Internal: Perform full recycling workflow"""
```

### 5. Stats Tracking

```python
@dataclass(slots=True)
class RecycleStats:
    recycles_initiated: int = 0
    recycles_completed: int = 0
    recycles_failed: int = 0
    markets_migrated: int = 0
    total_downtime_ms: float = 0.0

    @property
    def success_rate(self) -> float:
        """Percentage of successful recycles"""
```

### 6. Configuration Constants

```python
POLLUTION_THRESHOLD = 0.30        # 30% expired triggers recycling
AGE_THRESHOLD = 86400.0           # 24 hours max connection age
HEALTH_CHECK_INTERVAL = 60.0      # Check health every minute
STABILIZATION_DELAY = 3.0         # Wait for new connection stability
MAX_CONCURRENT_RECYCLES = 2       # Limit concurrent operations
```

## Dependencies

**Required Before Starting**:
- ✅ Component 3: Market Registry (MarketRegistry, MarketStatus)
- ✅ Component 5: Connection Pool (ConnectionPool)

**Registry Methods Needed**:
- `get_active_by_connection(connection_id)` - Get only non-expired markets
- `reassign_connection(asset_ids, old_id, new_id)` - Atomic reassignment
- `connection_stats(connection_id)` - Get pollution ratio

**Pool Methods Needed**:
- `get_connection_stats()` - Get all connection metadata
- `force_subscribe(asset_ids)` - Create new connection immediately
- `_remove_connection(connection_id)` - Clean up old connection

**Blocks**:
- ❌ Component 10: Application Orchestrator (needs recycler for production readiness)

## Implementation Plan

### Phase 1: Core Structure (1 day)
1. Create `src/lifecycle/recycler.py`
2. Define RecycleStats dataclass
3. Implement ConnectionRecycler.__init__()
4. Add _active_recycles set
5. Add _recycle_semaphore
6. Write unit tests for initialization

### Phase 2: Trigger Detection (2 days)
1. Implement `_check_all_connections()` logic
2. Add pollution ratio checking
3. Add age checking
4. Add health checking
5. Test trigger detection
6. Test skip logic (already recycling, draining)

### Phase 3: Recycling Workflow (3 days)
1. Implement `_recycle_connection()` full workflow
2. Add semaphore acquisition
3. Get active markets from registry
4. Create new connection via pool.force_subscribe()
5. Add stabilization delay
6. Verify new connection health
7. Test workflow with mock connections

### Phase 4: Registry Integration (2 days)
1. Add atomic reassignment via registry.reassign_connection()
2. Remove old connection via pool._remove_connection()
3. Track migration count
4. Measure downtime
5. Test with real ConnectionPool and MarketRegistry

### Phase 5: Monitoring Loop (1 day)
1. Implement start() method
2. Create _monitor_loop() task
3. Add HEALTH_CHECK_INTERVAL timing
4. Implement stop() method
5. Test monitoring lifecycle

### Phase 6: Manual Operations (1 day)
1. Implement force_recycle() for manual intervention
2. Add get_active_recycles() for visibility
3. Add comprehensive logging
4. Test manual recycling

### Phase 7: Integration Testing (2 days)
1. Test full recycling flow with active connections
2. Verify zero message loss during recycling
3. Test concurrent recycles (2 simultaneous)
4. Test failure scenarios (new connection fails)
5. Load test with multiple recycling events
6. Verify pollution ratio accuracy

## Acceptance Criteria

- [ ] Recycling triggers on pollution ≥ 30%
- [ ] Recycling triggers on age ≥ 24 hours
- [ ] Recycling triggers on unhealthy connection
- [ ] No message loss during recycling (both connections active)
- [ ] Registry atomically updated
- [ ] Concurrent recycles limited to MAX_CONCURRENT_RECYCLES (2)
- [ ] Stats accurately track recycles and migrations
- [ ] Force recycle works for manual intervention
- [ ] Failed recycling handled gracefully (old connection kept)
- [ ] New connection health verified before swap
- [ ] Downtime tracking accurate
- [ ] All unit tests pass
- [ ] Integration tests pass

## Testing Strategy

### Unit Tests
- RecycleStats dataclass and computed properties
- Trigger detection logic (pollution, age, health)
- Skip logic (active recycles, draining)
- Semaphore limiting

### Integration Tests
- Full recycling flow with mock pool/registry
- Test with polluted connection (40% expired)
- Test with aged connection (25+ hours)
- Test with unhealthy connection
- Test concurrent recycling (2 simultaneous)
- Test recycling failure (new connection fails)

### End-to-End Tests
- Recycle live connection with active markets
- Verify messages flowing during transition
- Verify no dropped messages
- Measure actual downtime
- Test with 100+ markets being migrated

## Notes

- Recycling is **destructive** - old connection is closed, cannot be recovered
- Both connections receive messages briefly (during STABILIZATION_DELAY)
- Registry reassignment is atomic - either succeeds completely or fails completely
- Failed recycling keeps old connection running (safe failure mode)
- Pollution naturally increases over time as markets expire
- 24 hour age threshold prevents indefinite connection lifetime

## Zero-Downtime Pattern

The key to zero message loss:
1. New connection starts receiving messages **before** old connection stops
2. Registry update is atomic - race condition handled by connection_id checks
3. Brief message duplication is acceptable (idempotency in orderbook updates)
4. 3 second stabilization ensures new connection is stable

## Error Handling

**New Connection Fails**:
- Abort recycling
- Keep old connection running
- Log failure
- Increment recycles_failed

**Registry Reassignment Fails**:
- Close new connection
- Keep old connection
- Log failure

**Old Connection Close Fails**:
- Log warning
- Consider already closed
- Still count as success

## Open Questions

1. What if pool.force_subscribe() fails? (Currently aborts recycling)
2. Should we implement retry logic for failed recycling?
3. What's the ideal STABILIZATION_DELAY? (spec says 3s, but may need tuning)
4. Should we drain old connection (stop accepting new markets) before recycling?
5. How to handle connection recycling during low-traffic periods? (May not detect failures quickly)

## Related Tickets

- ENG-001: Connection Pool Manager (provides connections to recycle)
- ENG-006: Application Orchestrator (starts recycler)
- Prerequisite: Add missing registry methods (get_active_by_connection, reassign_connection)

## References

- Spec: `lessons/polymarket_websocket_system_spec.md:2750-3038`
- Research: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:335-391`
- Current Code: `src/connection/websocket.py`, `src/registry/asset_registry.py`
