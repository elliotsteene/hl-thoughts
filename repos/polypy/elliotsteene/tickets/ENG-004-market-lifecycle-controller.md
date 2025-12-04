# ENG-004: Implement Market Lifecycle Controller

**Status**: Open
**Priority**: Medium
**Component**: Phase 4 - Market Lifecycle Controller
**Assignee**: Unassigned
**Created**: 2025-12-03

## Overview

Implement the Market Lifecycle Controller (Component 8) to automatically discover markets from Polymarket's Gamma API, track expiration times, detect when markets expire, and clean up stale data. This eliminates manual asset ID management and enables the system to automatically track all active markets.

## Context

- **Current State**: Asset IDs hardcoded in src/main.py (lines 21-26), manual specification required
- **Target State**: Automatic market discovery and lifecycle management
- **Spec Location**: `lessons/polymarket_websocket_system_spec.md:2390-2747`
- **Research Doc**: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:281-332`

## Technical Requirements

### 1. Market Discovery from Gamma API

**API Endpoint**: `https://gamma-api.polymarket.com/markets`

**Query Parameters**:
```python
params = {
    "limit": 100,
    "offset": 0,
    "active": "true",
    "closed": "false"
}
```

**Discovery Loop**:
- Poll API every DISCOVERY_INTERVAL = 60.0 seconds
- Fetch all active, non-closed markets with pagination
- Parse response to extract market metadata
- Register new markets in MarketRegistry

**Market Metadata** (MarketInfo):
```python
@dataclass(slots=True)
class MarketInfo:
    condition_id: str          # Market identifier
    question: str              # Human-readable question
    outcomes: list[str]        # ["Yes", "No"] typically
    tokens: list[dict]         # [{token_id, outcome}]
    end_date_iso: str          # ISO format: "2024-12-31T23:59:59Z"
    end_timestamp: int         # Parsed to unix ms
    active: bool
    closed: bool
```

### 2. Market Registration

**Process**:
1. Track known condition_ids in set to avoid duplicates
2. For each new market:
   - Parse endDate ISO string to unix timestamp
   - Extract all tokens (Yes/No outcomes)
   - For each token:
     - Call `registry.add_market(token_id, condition_id, end_timestamp)`
     - Invoke `on_new_market` callback if provided
3. Log count of newly discovered markets

**Date Parsing**:
```python
from datetime import datetime
dt = datetime.fromisoformat(end_date.replace("Z", "+00:00"))
end_ts = int(dt.timestamp() * 1000)  # Unix milliseconds
```

### 3. Expiration Checking

**Expiration Loop**:
- Check every EXPIRATION_CHECK_INTERVAL = 30.0 seconds
- Get current time in unix ms: `int(time.time() * 1000)`
- Query registry: `registry.get_expiring_before(current_time)`
- Filter to only SUBSCRIBED markets (skip PENDING and already EXPIRED)
- Mark as EXPIRED: `registry.mark_expired(asset_ids)`
- Invoke `on_market_expired` callback for each asset

### 4. Cleanup

**Cleanup Loop**:
- Run every CLEANUP_DELAY = 3600.0 seconds (1 hour)
- Get all EXPIRED markets: `registry.get_by_status(MarketStatus.EXPIRED)`
- For each expired market:
  - Check age: `time.monotonic() - entry.subscribed_at`
  - If age > CLEANUP_DELAY:
    - Call `registry.remove_market(asset_id)`
- Log count of cleaned up markets

### 5. LifecycleController Class

```python
class LifecycleController:
    def __init__(
        registry: MarketRegistry,
        on_new_market: Optional[LifecycleCallback] = None,
        on_market_expired: Optional[LifecycleCallback] = None
    )
        """Initialize controller"""

    async def start() -> None
        """Initial discovery + start background tasks"""

    async def stop() -> None
        """Cancel tasks + close HTTP session"""

    async def add_market_manually(
        asset_id: str,
        condition_id: str,
        expiration_ts: int = 0
    ) -> bool
        """Bypass discovery for specific markets"""

    async def _discovery_loop() -> None
        """Internal: Periodic market discovery"""

    async def _discover_markets() -> None
        """Internal: Fetch and register markets from API"""

    async def _expiration_loop() -> None
        """Internal: Periodic expiration checking"""

    async def _check_expirations() -> None
        """Internal: Mark expired markets"""

    async def _cleanup_loop() -> None
        """Internal: Periodic cleanup"""

    async def _cleanup_expired() -> None
        """Internal: Remove old expired markets"""
```

**Callback Type**:
```python
LifecycleCallback = Callable[[str, str], Awaitable[None]]
# Args: (event_name, asset_id)
```

### 6. HTTP Session Management

Use aiohttp.ClientSession:
- Create on start()
- Reuse for all API calls
- Close on stop()

## Dependencies

**Required Before Starting**:
- ✅ Component 3: Market Registry (MarketRegistry, MarketStatus)

**Registry Methods Needed** (may need to add):
- `add_market(asset_id, condition_id, expiration_ts)` - Add new market
- `get_expiring_before(timestamp)` - Time-range query
- `get_by_status(status)` - Query by MarketStatus
- `mark_expired(asset_ids)` - Batch status update
- `remove_market(asset_id)` - Full removal

**Blocks**:
- ❌ Component 10: Application Orchestrator (needs lifecycle for market management)

## Implementation Plan

### Phase 1: API Client (2 days)
1. Create `src/lifecycle.py`
2. Define MarketInfo dataclass
3. Implement `fetch_active_markets()` async function
4. Add aiohttp session management
5. Add pagination logic
6. Test API calls against live Gamma API
7. Test error handling (network failures, timeouts)

### Phase 2: Market Discovery (2 days)
1. Implement LifecycleController.__init__()
2. Add known_conditions set tracking
3. Implement `_discover_markets()` logic
4. Add date parsing (ISO to unix ms)
5. Integrate with MarketRegistry
6. Test new market registration
7. Test duplicate detection

### Phase 3: Background Tasks (2 days)
1. Implement `start()` method
2. Create three background tasks:
   - `_discovery_loop()`
   - `_expiration_loop()`
   - `_cleanup_loop()`
3. Implement `stop()` method
4. Test task lifecycle
5. Test graceful cancellation

### Phase 4: Expiration Detection (1 day)
1. Implement `_check_expirations()` logic
2. Add current time calculation
3. Filter to SUBSCRIBED markets only
4. Test expiration marking
5. Test callback invocation

### Phase 5: Cleanup Logic (1 day)
1. Implement `_cleanup_expired()` logic
2. Add age checking
3. Test removal of old markets
4. Test grace period (1 hour)

### Phase 6: Manual Operations (1 day)
1. Implement `add_market_manually()` for testing
2. Add logging for all major events
3. Add error handling for API failures
4. Test manual market addition

### Phase 7: Integration Testing (2 days)
1. Test full lifecycle: discovery → subscription → expiration → cleanup
2. Test with live Gamma API
3. Test callback invocations
4. Test with MarketRegistry integration
5. Verify timing intervals (60s, 30s, 1h)
6. Test API error scenarios

## Acceptance Criteria

- [ ] New markets discovered from Gamma API
- [ ] All tokens (Yes/No) registered for each market
- [ ] Expiration detection triggers within 30 seconds of expiry time
- [ ] Callbacks invoked for lifecycle events (new market, expired)
- [ ] Cleanup removes markets after 1 hour grace period
- [ ] Manual market addition works (`add_market_manually()`)
- [ ] API errors handled gracefully (retry, logging)
- [ ] HTTP session properly managed (created/closed)
- [ ] Known conditions tracked to avoid duplicates
- [ ] Pagination works for large market lists
- [ ] All unit tests pass
- [ ] Integration tests with live API pass

## Testing Strategy

### Unit Tests
- MarketInfo dataclass creation
- Date parsing (ISO to unix ms)
- Known conditions duplicate detection
- Age calculation for cleanup
- Callback invocation logic

### Integration Tests (with mock API)
- Mock Gamma API responses
- Test pagination with 250 markets
- Test duplicate market handling
- Test expiration detection timing
- Test cleanup timing
- Test callback invocations

### Live API Tests
- Fetch real markets from Gamma API
- Verify response parsing
- Test with actual market data
- Verify token extraction

### Performance Tests
- Measure API call latency
- Test with 1000+ markets
- Verify discovery loop timing (60s)
- Verify expiration loop timing (30s)

## Notes

- Discovery runs immediately on start(), then every 60 seconds
- Expiration checking is independent of market creation time
- Cleanup provides 1 hour grace period for expired markets (useful for late data)
- Markets can be manually added for testing without waiting for discovery
- API may rate limit - consider adding retry with backoff

## API Considerations

- Gamma API documentation can be found here - <https://docs.polymarket.com/developers/gamma-markets-api/fetch-markets-guide#3-fetch-all-active-markets>
- May need to handle new market types in future
- Rate limits unknown - be conservative
- Consider caching API responses if discovery becomes expensive
- Network failures should not crash the controller

## Open Questions

1. What to do if Gamma API is down for extended period? (Keep retrying?)
2. Should we support multiple API endpoints (failover)?
3. How to handle markets with more than 2 outcomes? (spec assumes binary)
4. Should cleanup grace period be configurable?
5. What if a market expires but is still trading? (Exchange issue, but should we handle?)

## Related Tickets

- ENG-001: Connection Pool Manager (lifecycle feeds pending markets to pool)
- ENG-006: Application Orchestrator (starts lifecycle controller)
- Prerequisite: Add missing methods to MarketRegistry (separate ticket?)

## References

- Spec: `lessons/polymarket_websocket_system_spec.md:2390-2747`
- Research: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:281-332`
- Gamma API: `https://gamma-api.polymarket.com/markets`
- Current Code: `src/registry/asset_registry.py`
