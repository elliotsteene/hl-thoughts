---
date: 2025-12-08T17:15:34+05:30
researcher: Elliot Steene
git_commit: c48fbcf40e68c2428afe42ad22dd8b40009c450a
branch: phase-1-http-server-implementation
repository: polypy
topic: "Why only 291 active markets are fetched from Gamma API"
tags: [research, codebase, gamma-api, lifecycle, market-discovery]
status: complete
last_updated: 2025-12-08
last_updated_by: Elliot Steene
---

# Research: Why Only 291 Active Markets Are Fetched from Gamma API

**Date**: 2025-12-08T17:15:34+05:30
**Researcher**: Elliot Steene
**Git Commit**: c48fbcf40e68c2428afe42ad22dd8b40009c450a
**Branch**: phase-1-http-server-implementation
**Repository**: polypy

## Research Question

Why does the system fetch only 291 active markets from Polymarket's Gamma API when more markets were expected? The log message shows: "Fetched 291 active markets from Gamma API [src.lifecycle.api]"

## Summary

The system fetches exactly 291 active markets because that's the number of markets Polymarket's Gamma API returns when filtering by `active=true` AND `closed=false`. The client-side code properly implements pagination and will fetch all available markets matching these criteria. No additional client-side filters or limitations are being applied beyond the two query parameters sent to the API.

The second log message "Skipping connection creation: 0 pending (threshold: 50)" is unrelated to the market count - it indicates that WebSocket connection creation requires at least 50 pending markets to justify creating a new connection.

## Detailed Findings

### Gamma API Integration

**Location**: `/Users/elliotsteene/Documents/prediction_markets/polypy/src/lifecycle/api.py`

The system fetches markets from Polymarket's Gamma API using the following endpoint and parameters:

**API Endpoint** (`api.py:62`):
```
GET https://gamma-api.polymarket.com/markets
```

**Query Parameters** (`api.py:34-39`):
```python
params = {
    "limit": page_size,      # Default: 100
    "offset": offset,        # Increments by page_size
    "active": "true",        # Only active markets
    "closed": "false",       # Exclude closed markets
}
```

**Hardcoded Filters**:
- `active: "true"` - Markets currently active for trading
- `closed: "false"` - Markets not yet resolved/closed

These two filters are applied to every API request and cannot be disabled or configured.

### Pagination Implementation

**Function**: `fetch_active_markets()` (`api.py:21-54`)

The pagination logic properly handles offset-based pagination to fetch all available markets:

1. Starts with `offset=0`
2. Fetches pages of size 100 (configurable via `page_size` parameter)
3. Continues until either:
   - Empty response received
   - Response contains fewer items than `page_size` (indicating last page)
4. Increments offset by `page_size` for each iteration

**Configuration** (`api.py:15-18`):
```python
DEFAULT_PAGE_SIZE = 100
MAX_RETRIES = 3
RETRY_DELAY = 2.0
```

The pagination logic is correctly implemented and will fetch all pages until the API returns no more results. For 291 markets with page size 100, the system makes 3 API requests:
- Request 1: offset=0, returns 100 markets
- Request 2: offset=100, returns 100 markets
- Request 3: offset=200, returns 91 markets (stops here as 91 < 100)

### Market Validation and Parsing

**Validation Function**: `_is_valid_market()` (`api.py:84-87`)

Before parsing, markets must have all five required fields present in the API response:
```python
required = ["conditionId", "question", "outcomes", "clobTokenIds", "endDate"]
```

Markets missing any of these fields are silently filtered out. However, this validation only checks for field presence, not field values - empty strings, null values, or malformed data will pass validation.

**Parsing Function**: `_parse_market()` (`api.py:90-140`)

The parsing function handles malformed data gracefully with fallback values:
- Invalid `endDate` → Sets `end_timestamp = 0` and logs warning
- Malformed `outcomes` JSON → Defaults to `["Yes", "No"]`
- Malformed `clobTokenIds` JSON → Defaults to `[]`
- Missing optional fields → Use defaults (e.g., `active=True`, `closed=False`)

**Key insight**: Markets are NOT filtered out due to parsing errors. All markets that pass the initial field-presence check will be included in the results.

### Market Discovery Process

**Location**: `/Users/elliotsteene/Documents/prediction_markets/polypy/src/lifecycle/controller.py`

The `LifecycleController` manages periodic market discovery:

**Initial Discovery** (`controller.py:81-115`):
1. Creates `aiohttp.ClientSession` on startup
2. Calls `fetch_active_markets()` immediately
3. Registers discovered markets in the `AssetRegistry`
4. Starts background tasks for periodic polling

**Periodic Discovery Loop** (`controller.py:184-199`):
- Polls Gamma API every 60 seconds (`DISCOVERY_INTERVAL`)
- Checks for new markets not in `_known_conditions` set
- Registers each token (Yes/No outcomes) as separate assets
- Logs: "Discovered N new tokens from M markets"

**Market Tracking** (`controller.py:214-245`):
- Maintains `_known_conditions` set of condition IDs
- Skips markets already seen (line 216-217)
- Registers each token ID separately with the registry
- Triggers `on_new_market` callback for each new token

### WebSocket Connection Creation Threshold

**Location**: `/Users/elliotsteene/Documents/prediction_markets/polypy/src/connection/pool.py`

The log message "Skipping connection creation: 0 pending (threshold: 50)" comes from the connection pool's subscription management logic.

**Configuration** (`pool.py:26-32`):
```python
TARGET_MARKETS_PER_CONNECTION = 400
MAX_MARKETS_PER_CONNECTION = 500
POLLUTION_THRESHOLD = 0.30
MIN_AGE_FOR_RECYCLING = 300.0
BATCH_SUBSCRIPTION_INTERVAL = 30.0
MIN_PENDING_FOR_NEW_CONNECTION = 50  # This controls the threshold
```

**Connection Creation Logic** (`pool.py:171-179`):
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
```

**Why "0 pending"**:
When there are 0 markets in PENDING status (meaning all discovered markets are either already SUBSCRIBED or EXPIRED), the system logs this message. This is normal behavior when:
- All discovered markets have been subscribed to WebSocket connections
- No new markets have been discovered since the last subscription batch
- The system is waiting for the next discovery cycle (60s interval)

### Market Count Analysis

**291 Markets** is the actual number of markets returned by the Gamma API that meet both criteria:
1. `active=true` - Market is currently active
2. `closed=false` - Market has not been resolved

**No client-side limitations** are reducing this count:
- Pagination correctly fetches all pages
- Validation only checks for required fields (minimal filtering)
- Parsing errors use fallback values instead of dropping markets
- No hardcoded limits on market count
- No additional filtering beyond the two API parameters

**The number is determined entirely by Polymarket's backend**, which controls:
- Which markets are marked as "active"
- Which markets are marked as "closed"
- When markets transition between states

### Token vs Market Count

**Important distinction** (`controller.py:221-234`):

The system tracks **tokens** (individual outcomes) rather than **markets**:
- Each market typically has 2 outcomes: "Yes" and "No"
- Each outcome is a separate token with unique `token_id`
- 291 markets → approximately 582 tokens (if all are binary markets)

The WebSocket connections subscribe to **token IDs**, not market IDs.

## Code References

- `src/lifecycle/api.py:34-39` - API query parameters with hardcoded filters
- `src/lifecycle/api.py:21-54` - Pagination implementation
- `src/lifecycle/api.py:84-87` - Market validation logic
- `src/lifecycle/api.py:90-140` - Market parsing with fallback values
- `src/lifecycle/controller.py:200-245` - Market discovery and registration
- `src/connection/pool.py:171-179` - Connection creation threshold logic
- `src/lifecycle/types.py:26-29` - Configuration constants

## Architecture Documentation

### Data Flow

1. **API Fetch** → Gamma API returns markets matching `active=true` AND `closed=false`
2. **Pagination** → Client fetches all pages until exhausted
3. **Validation** → Filter markets missing required fields
4. **Parsing** → Convert to `MarketInfo` objects with fallback values
5. **Registration** → Add token IDs to `AssetRegistry` with PENDING status
6. **Subscription** → Connection pool creates WebSocket connections when ≥50 pending
7. **Monitoring** → Re-polls API every 60 seconds for new markets

### State Management

**Market States in Registry**:
- `PENDING` - Discovered but not yet subscribed
- `SUBSCRIBED` - Active WebSocket subscription
- `EXPIRED` - Past end_timestamp

**Lifecycle Transitions**:
- New market → PENDING (discovery)
- PENDING → SUBSCRIBED (connection creation)
- SUBSCRIBED → EXPIRED (expiration check every 30s)
- EXPIRED → Removed (cleanup after 1 hour)

## Historical Context (from thoughts/)

No historical documentation discusses specific market count expectations or issues with the Gamma API returning unexpected numbers of markets.

**Related documents**:
- `thoughts/shared/plans/2025-12-04-ENG-004-market-lifecycle-controller.md` - Implementation plan with API integration details
- `thoughts/elliotsteene/tickets/ENG-004-market-lifecycle-controller.md` - Original ticket with requirements
- `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md` - System architecture documentation

The system was designed to handle "whatever markets the API returns" through pagination, with capacity for 100+ markets mentioned as acceptance criteria. No specific expected market count was documented.

## Related Research

- Previous research on HTTP API integration: `thoughts/shared/research/2025-12-08-ENG-007-http-api-integration-analysis.md`

## Open Questions

None - the system is functioning as designed. The 291 market count reflects the actual state of Polymarket's active, non-closed markets at the time of the API call.

If more markets are expected, the investigation should focus on:
1. Verifying the Gamma API directly (e.g., via curl or browser)
2. Checking if the definitions of "active" and "closed" match expectations
3. Determining if there are alternative API endpoints with different filtering options
