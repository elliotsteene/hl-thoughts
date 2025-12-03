# ENG-006: Implement Full Application Orchestrator

**Status**: Open
**Priority**: High
**Component**: Phase 5 - Application Orchestrator
**Assignee**: Unassigned
**Created**: 2025-12-03

## Overview

Implement the full Application Orchestrator (Component 10) to tie all components together into a production-ready system. Provides unified startup/shutdown sequences, signal handling, stats endpoints, and health checks.

## Context

- **Current State**: Simplified main.py with single connection and synchronous processing
- **Target State**: Production orchestrator managing 6 major subsystems with proper lifecycle
- **Spec Location**: `lessons/polymarket_websocket_system_spec.md:3041-3449`
- **Research Doc**: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:394-463`

## Technical Requirements

### 1. Application Class

```python
class Application:
    def __init__(
        num_workers: int = 4,
        handler_factory: Optional[Callable[[], Any]] = None
    )
        """Initialize application"""

    async def start() -> None
        """Start all components in correct order"""

    async def stop() -> None
        """Stop all components in reverse order"""

    async def run() -> None
        """Run with signal handling (convenience method)"""

    def get_stats() -> dict[str, Any]
        """Get comprehensive statistics"""

    def is_healthy() -> bool
        """Check overall application health"""
```

### 2. Startup Sequence (CRITICAL: Must be in order)

```
1. Initialize MarketRegistry
   └─ Create empty registry instance

2. Start WorkerManager (spawn processes)
   └─ Create and start worker processes
   └─ Get input queues for router

3. Create MessageRouter (wire to worker queues)
   └─ Initialize with num_workers
   └─ Connect to worker queues
   └─ Start routing task

4. Start ConnectionPool (with message callback to router)
   └─ Wire router.route_message as callback
   └─ Start subscription management loop

5. Start LifecycleController (market discovery)
   └─ Initial market discovery
   └─ Start background tasks (discovery, expiration, cleanup)

6. Start ConnectionRecycler (monitoring)
   └─ Start connection monitoring loop
```

### 3. Shutdown Sequence (REVERSE order)

```
1. Stop ConnectionRecycler
   └─ Cancel monitoring loop

2. Stop LifecycleController
   └─ Cancel background tasks
   └─ Close HTTP session

3. Stop ConnectionPool
   └─ Stop all connections concurrently
   └─ Clean up registry assignments

4. Stop MessageRouter
   └─ Send None sentinel to all workers
   └─ Cancel routing task

5. Stop WorkerManager
   └─ Signal shutdown event
   └─ Wait for graceful shutdown (10s timeout)
   └─ Force terminate if needed

6. Cleanup
   └─ Log final statistics
```

### 4. Signal Handling

```python
async def run() -> None:
    """Run with automatic signal handling"""
    loop = asyncio.get_event_loop()

    def signal_handler():
        logger.info("Shutdown signal received")
        shutdown_event.set()

    # Register handlers
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, signal_handler)

    try:
        await self.start()
        await shutdown_event.wait()
    finally:
        await self.stop()

        # Remove handlers
        for sig in (signal.SIGINT, signal.SIGTERM):
            loop.remove_signal_handler(sig)
```

### 5. Stats Endpoint

Return comprehensive dict with all subsystem stats:

```python
{
    "running": bool,

    "registry": {
        "total_markets": int,
        "pending": int,
        "subscribed": int,
        "expired": int
    },

    "pool": {
        "connection_count": int,
        "active_connections": int,
        "connections": [
            {
                "connection_id": str,
                "status": str,
                "age_seconds": float,
                "messages_received": int,
                "total": int,
                "active": int,
                "expired": int,
                "pollution_ratio": float
            }
        ]
    },

    "router": {
        "messages_routed": int,
        "messages_dropped": int,
        "avg_latency_ms": float,
        "queue_depths": {
            "async_queue": int,
            "worker_0": int,
            ...
        }
    },

    "workers": {
        "alive_count": int,
        "is_healthy": bool,
        "worker_stats": {
            0: WorkerStats,
            1: WorkerStats,
            ...
        }
    },

    "recycler": {
        "recycles_completed": int,
        "recycles_failed": int,
        "markets_migrated": int,
        "active_recycles": [str]
    }
}
```

### 6. Health Check

```python
def is_healthy() -> bool:
    """Check overall system health"""
    if not self._running:
        return False

    # Check workers
    if self._workers and not self._workers.is_healthy():
        return False

    # Check active connections (allow grace during startup)
    if self._pool and self._pool.active_connection_count == 0:
        # May be OK during startup
        pass

    return True
```

### 7. Lifecycle Callbacks

Wire lifecycle events to application:

```python
async def _on_new_market(event: str, asset_id: str) -> None:
    """Handle new market discovery"""
    logger.debug(f"New market discovered: {asset_id}")
    # Could trigger immediate subscription if desired

async def _on_market_expired(event: str, asset_id: str) -> None:
    """Handle market expiration"""
    logger.debug(f"Market expired: {asset_id}")
```

## Dependencies

**Required Before Starting**:
- ✅ Component 3: Market Registry
- ✅ Component 5: Connection Pool
- ✅ Component 6: Message Router
- ✅ Component 7: Worker Manager
- ✅ Component 8: Lifecycle Controller
- ✅ Component 9: Connection Recycler

**This is the final component** - blocks nothing, requires everything else.

## Implementation Plan

### Phase 1: Core Structure (1 day)
1. Create `src/app.py`
2. Implement Application.__init__() with __slots__
3. Add component storage fields
4. Add _running flag and _shutdown_event
5. Write unit tests for initialization

### Phase 2: Startup Sequence (2 days)
1. Implement start() method
2. Initialize all 6 components in correct order
3. Wire message callback: pool → router → workers
4. Wire lifecycle callbacks
5. Add detailed logging at each step
6. Test startup sequence
7. Test startup failures at each stage

### Phase 3: Shutdown Sequence (2 days)
1. Implement stop() method
2. Stop all components in reverse order
3. Add timeout handling
4. Add error handling for shutdown failures
5. Test graceful shutdown
6. Test shutdown with hung component

### Phase 4: Signal Handling (1 day)
1. Implement run() convenience method
2. Add SIGINT and SIGTERM handlers
3. Test Ctrl+C handling
4. Test kill signal handling
5. Verify handler cleanup

### Phase 5: Stats Endpoint (2 days)
1. Implement get_stats() method
2. Collect stats from all 6 subsystems
3. Format into unified dict
4. Handle missing/uninitialized components
5. Test stats accuracy
6. Test with components in various states

### Phase 6: Health Check (1 day)
1. Implement is_healthy() method
2. Check workers.is_healthy()
3. Check pool connection count
4. Handle startup grace period
5. Test health check accuracy
6. Test with failed components

### Phase 7: Migration from Current main.py (2 days)
1. Create new main() function using Application
2. Migrate logging setup
3. Add command-line arguments (num_workers, etc.)
4. Test full system with real connections
5. Compare behavior to current main.py
6. Update documentation

### Phase 8: Integration Testing (3 days)
1. Test full startup → run → shutdown cycle
2. Test with live market data
3. Test with 1000+ markets
4. Test signal handling (SIGINT, SIGTERM)
5. Load test with high message volume
6. Endurance test (run for 1+ hour)
7. Test failure scenarios (component crashes)

## Acceptance Criteria

- [ ] Clean startup sequence with logging
- [ ] Clean shutdown sequence (reverse order)
- [ ] Shutdown completes within reasonable time (< 30s)
- [ ] SIGINT/SIGTERM handled gracefully
- [ ] Stats endpoint returns comprehensive data
- [ ] Stats accurate for all subsystems
- [ ] Health check reflects actual system state
- [ ] All components properly wired together
- [ ] Application can run for extended periods without memory leaks
- [ ] Handles component failures gracefully
- [ ] All unit tests pass
- [ ] Integration tests pass
- [ ] Can subscribe to 1000+ markets successfully

## Testing Strategy

### Unit Tests
- Application initialization
- Component storage
- Stats aggregation logic
- Health check logic
- Callback wiring

### Integration Tests
- Full startup sequence
- Full shutdown sequence
- Stats collection from all components
- Health check with various states
- Signal handling (mock signals)

### End-to-End Tests
- Run with live Gamma API
- Subscribe to 100+ markets
- Run for 5+ minutes
- Test graceful shutdown
- Verify no memory leaks
- Verify no resource leaks

### Performance Tests
- Startup time (should be < 10s)
- Shutdown time (should be < 30s)
- Stats collection overhead
- Health check latency
- Message throughput with full system

## Notes

- Current src/main.py is Phase 1 baseline - will be replaced/renamed
- Keep current main.py as main_phase1.py for reference?
- New application entry point becomes production version
- Consider command-line arguments: --num-workers, --log-level, etc.
- Stats endpoint could be exposed via HTTP API in future
- Health check could be exposed via HTTP API in future

## Configuration

Consider adding configuration options:
- `num_workers`: Number of worker processes (default 4)
- `handler_factory`: Custom per-worker handlers (optional)
- `log_level`: Logging verbosity
- `enable_lifecycle`: Enable/disable market discovery (default True)
- `enable_recycling`: Enable/disable connection recycling (default True)

## Observability

Consider adding:
- Prometheus metrics export
- Structured logging to stdout
- Performance tracing
- Error alerting hooks
- Stats export to time-series DB

## Open Questions

1. Should we add HTTP API for stats/health endpoints?
2. Should configuration be via environment variables or config file?
3. How to handle partial startup failure? (Currently aborts)
4. Should we add --dry-run mode for testing?
5. Should we implement graceful degradation (run with fewer components)?
6. What's the ideal shutdown timeout? (Currently unbounded)

## Related Tickets

- ALL previous tickets (ENG-001 through ENG-005) must be complete
- This is the final integration ticket

## References

- Spec: `lessons/polymarket_websocket_system_spec.md:3041-3449`
- Research: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:394-463`
- Current Code: `src/main.py` (Phase 1 baseline to be replaced)

## Success Criteria

This ticket is complete when:
1. All components wire together correctly
2. System can run for extended periods (24+ hours)
3. Graceful shutdown works reliably
4. Stats and health checks work
5. System handles 1000+ markets successfully
6. Message throughput meets targets (< 1ms routing, < 5ms processing)
7. Memory usage is stable (no leaks)
8. Production ready for deployment
