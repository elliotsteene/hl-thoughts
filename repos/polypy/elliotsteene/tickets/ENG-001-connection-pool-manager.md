# ENG-001: Implement Connection Pool Manager

**Status**: Open
**Priority**: High
**Component**: Phase 2 - Connection Pool Manager
**Assignee**: Unassigned
**Created**: 2025-12-03

## Overview

Implement the Connection Pool Manager (Component 5) to manage multiple WebSocket connections with capacity tracking and automatic subscription batching. This enables the system to subscribe to more than 500 assets by distributing them across multiple connections.

## Context

- **Current State**: System uses a single WebSocket connection hardcoded to 2 assets (src/main.py:21-26)
- **Target State**: Support thousands of assets across multiple connections (500 assets per connection)
- **Spec Location**: `lessons/polymarket_websocket_system_spec.md:1417-1754`
- **Research Doc**: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:127-168`

## Technical Requirements

### 1. Capacity Management
- Track 400 active markets per connection (leaving 100 slot buffer under 500 hard limit)
- Automatically create new connections when capacity exhausted
- Assign pending markets to connections with available capacity
- Track total capacity across all connections

### 2. Connection Tracking
Create `ConnectionInfo` dataclass:
```python
@dataclass(slots=True)
class ConnectionInfo:
    connection: WebSocketConnection
    created_at: float = field(default_factory=time.monotonic)
    is_draining: bool = False
```

Maintain registry: `dict[str, ConnectionInfo]` mapping connection_id → info

### 3. Subscription Management
- Periodic loop (30 second interval) checking for pending markets in MarketRegistry
- Batch size: 400 markets per connection
- Minimum threshold: 50 pending markets before creating new connection
- Call `registry.take_pending_batch()` to retrieve markets
- Create WebSocketConnection with batch
- Mark markets as subscribed via `registry.mark_subscribed()`

### 4. Recycling Detection
Monitor connections for recycling triggers:
- Pollution ratio ≥ 30% (expired markets / total markets)
- Connection age ≥ 5 minutes (don't recycle brand new connections)
- Delegate actual recycling to Component 9 (ConnectionRecycler)

### 5. Key Methods to Implement

```python
class ConnectionPool:
    async def start() -> None
        """Start subscription management loop"""

    async def stop() -> None
        """Stop all connections concurrently"""

    async def force_subscribe(asset_ids: list[str]) -> str
        """Immediately create connection for priority assets, returns connection_id"""

    def get_connection_stats() -> list[dict]
        """Aggregate stats from all connections"""

    async def _process_pending_markets() -> None
        """Internal: Create connections for pending market batches"""

    async def _check_for_recycling() -> None
        """Internal: Check connections for recycling needs"""

    async def _remove_connection(connection_id: str) -> None
        """Internal: Cleanup connection and registry"""
```

## Dependencies

**Required Before Starting**:
- ✅ Component 1: Protocol Messages (src/messages/protocol.py)
- ✅ Component 3: Market Registry (src/registry/asset_registry.py)
- ✅ Component 4: WebSocket Connection (src/connection/websocket.py)

**Blocks**:
- ❌ Component 9: Connection Recycler (needs ConnectionPool.get_connection_stats())
- ❌ Component 10: Application Orchestrator (needs ConnectionPool)

**Registry Methods Needed** (may need to add to Component 3):
- `get_pending_count()` - Check how many markets awaiting subscription
- `take_pending_batch(max_size)` - Retrieve batch of pending markets
- `mark_subscribed(asset_ids, connection_id)` - Mark markets as subscribed
- `connection_stats(connection_id)` - Get pollution ratio and counts

## Implementation Plan

### Phase 1: Core Structure (2-3 days)
1. Create `src/pool.py`
2. Define ConnectionInfo dataclass
3. Implement ConnectionPool.__init__() with storage structures
4. Implement start() and stop() methods
5. Write unit tests for lifecycle

### Phase 2: Subscription Loop (2-3 days)
1. Implement `_subscription_loop()` background task
2. Implement `_process_pending_markets()` logic
3. Add connection creation and registry integration
4. Test with multiple pending market batches
5. Verify correct batch sizes and timing

### Phase 3: Recycling Detection (1-2 days)
1. Implement `_check_for_recycling()` logic
2. Calculate pollution ratios via registry.connection_stats()
3. Add age checking (skip connections < 5 min old)
4. Create stub for `_initiate_recycling()` (full implementation in ENG-005)
5. Test detection triggers

### Phase 4: Stats and Monitoring (1 day)
1. Implement `get_connection_stats()` aggregation
2. Add `get_total_capacity()` calculation
3. Implement `force_subscribe()` for priority markets
4. Add logging for key events
5. Test stats accuracy

### Phase 5: Integration Testing (2 days)
1. Test with MarketRegistry integration
2. Test with multiple connections (simulate 1000+ markets)
3. Verify capacity tracking accuracy
4. Test graceful shutdown of all connections
5. Load test with concurrent market additions

## Acceptance Criteria

- [ ] Can create and manage multiple WebSocket connections
- [ ] Correctly tracks capacity (400 markets per connection)
- [ ] Respects 500 market hard limit per connection
- [ ] Periodic subscription loop processes pending markets every 30s
- [ ] Minimum 50 pending threshold works correctly
- [ ] Pollution ratio calculation accurate
- [ ] Age checking prevents recycling of connections < 5 min old
- [ ] `force_subscribe()` creates immediate connection
- [ ] Graceful shutdown stops all connections within 10s
- [ ] Stats aggregation returns correct data
- [ ] All unit tests pass
- [ ] Integration tests pass with MarketRegistry

## Testing Strategy

### Unit Tests
- ConnectionInfo dataclass creation and properties
- Connection tracking (add, remove, lookup)
- Capacity calculation logic
- Batch size calculations
- Pollution ratio calculation

### Integration Tests
- Create pool with MarketRegistry
- Add 1000 pending markets, verify connections created
- Test subscription batch processing
- Test concurrent connection management
- Test graceful shutdown

### Performance Tests
- Measure subscription loop latency
- Verify < 1s to process 400 market batch
- Test with 10+ connections (5000+ markets)

## Notes

- Reference implementation may exist in `src/exercises/` but should follow spec exactly
- The spec shows some recycling logic in ConnectionPool (lines 1637-1688) but full recycling should be delegated to Component 9
- Need to decide whether to implement `_initiate_recycling()` stub or wait for ENG-005

## Open Questions

1. Should we implement the full `_initiate_recycling()` workflow in this component or just detection + stub?
  Just add a stub for now an dimplement in ENG-005.
2. Do we need to add missing registry methods (`get_pending_count`, `connection_stats`) as part of this ticket or create separate ticket?
3. Should force_subscribe() bypass the 50 minimum threshold or use same logic?
  Use the same logic

## Related Tickets

- ENG-002: Async Message Router (depends on this for message callback)
- ENG-005: Connection Recycler (depends on get_connection_stats())
- ENG-006: Application Orchestrator (depends on full pool functionality)

## References

- Spec: `lessons/polymarket_websocket_system_spec.md:1417-1754`
- Research: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:127-168`
- Current Code: `src/connection/websocket.py`, `src/registry/asset_registry.py`
