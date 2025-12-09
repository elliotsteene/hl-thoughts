---
date: 2025-12-08T17:15:34+05:30
researcher: Elliot Steene
git_commit: c48fbcf40e68c2428afe42ad22dd8b40009c450a
branch: phase-1-http-server-implementation
repository: polypy
topic: "Why only 291 active markets are fetched from Gamma API"
tags: [research, codebase, gamma-api, lifecycle, market-discovery, bug]
status: complete
last_updated: 2025-12-08
last_updated_by: Elliot Steene
last_updated_note: "Added root cause analysis - pagination bug identified"
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

**UPDATE**: Root cause identified - the system stops pagination prematurely due to a bug in the break condition logic.

The pagination loop breaks when `len(markets) < page_size` (line 52-56 in `src/lifecycle/api.py`), but this check happens **after** filtering out invalid markets (line 74). If the API returns a full page (e.g., 500 markets) but some are filtered out due to missing required fields, the filtered list will be smaller than `page_size`, causing the loop to incorrectly assume it has reached the last page.

**Example scenario:**
1. API returns 500 markets (full page)
2. 5 markets are filtered out due to missing required fields
3. `markets` list now has 495 items
4. `495 < 500` → loop breaks
5. Remaining markets are never fetched

This explains why only 291 markets are fetched when more are likely available from the API.

The second log message "Skipping connection creation: 0 pending (threshold: 50)" is unrelated to the market count - it indicates that WebSocket connection creation requires at least 50 pending markets to justify creating a new connection.

## Detailed Findings

### Gamma API Integration

**Location**: `/Users/elliotsteene/Documents/prediction_markets/polypy/src/lifecycle/api.py`

The system fetches markets from Polymarket's Gamma API using the following endpoint and parameters:

**API Endpoint** (`api.py:62`):
```
GET https://gamma-api.polymarket.com/markets
```

**Query Parameters** (`api.py:36-42`):
```python
params = {
    "limit": page_size,        # Default: 500
    "offset": offset,          # Increments by page_size
    "closed": "false",         # Exclude closed markets
    "order": "id",             # Order by ID
    "ascending": "false",      # Descending order
}
```

**Note**: The `active: "true"` filter was removed from the current implementation. Only `closed: "false"` is applied.

### Pagination Implementation

**Function**: `fetch_active_markets()` (`api.py:23-61`)

The pagination logic uses offset-based pagination but contains a bug:

**Configuration** (`api.py:17-20`):
```python
DEFAULT_PAGE_SIZE = 500
MAX_RETRIES = 3
RETRY_DELAY = 2.0
MARKETS_URL = f"{GAMMA_API_BASE_URL}/markets"
```

**Pagination Flow** (`api.py:35-58`):
```python
while True:
    params = {
        "limit": page_size,
        "offset": offset,
        "closed": "false",
        "order": "id",
        "ascending": "false",
    }

    markets = await _fetch_page(session, params)  # Returns FILTERED list

    if not markets:
        logger.warning("No more markets")
        break

    all_markets.extend(markets)

    if len(markets) < page_size:  # BUG: checks filtered count, not API response
        logger.warning(
            f"markets less than page_size: {len(markets)} and {len(all_markets)} and {page_size}"
        )
        break

    offset += page_size
```

**The Bug** (`api.py:52-56`):

The break condition `len(markets) < page_size` checks the **filtered** market count after `_is_valid_market()` has removed invalid markets (line 74):

```python
# In _fetch_page() at line 74:
return [_parse_market(m) for m in data if _is_valid_market(m)]
```

**Problem**: If the API returns a full page (500 markets) but some are filtered out due to missing required fields, the filtered list will be smaller than `page_size`, causing premature termination.

**Example:**
- API returns 500 markets (full page at offset=0)
- 5 markets fail validation (missing `conditionId`, `question`, etc.)
- `_fetch_page()` returns 495 markets
- `495 < 500` → loop breaks incorrectly
- Markets at offset=500 and beyond are never fetched

### Market Validation and Parsing

**Validation Function**: `_is_valid_market()` (`api.py:89-92`)

Before parsing, markets must have all five required fields present in the API response:
```python
required = ["conditionId", "question", "outcomes", "clobTokenIds", "endDate"]
return all(field in data for field in required)
```

Markets missing any of these fields are **silently filtered out** at line 74 in `_fetch_page()`. This filtering is what causes the pagination bug - when markets are removed, the filtered count becomes smaller than the page size, triggering early loop termination.

**Parsing Function**: `_parse_market()` (`api.py:95-144`)

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

**291 Markets** represents the number fetched before the pagination bug caused early termination.

**Root cause**: The pagination loop breaks prematurely when:
1. API returns a full page (500 markets)
2. Some markets are filtered out due to missing required fields
3. Filtered count < page_size triggers incorrect "last page" detection
4. Loop stops, leaving remaining markets unfetched

**Actual available markets**: Unknown - potentially much higher than 291. The true count depends on:
- How many markets Polymarket has with `closed=false`
- How many pages of results exist beyond the point where pagination stopped

**The bug affects**:
- Initial market discovery on startup
- Periodic re-discovery (every 60 seconds)
- Any call to `fetch_active_markets()`

### Token vs Market Count

**Important distinction** (`controller.py:221-234`):

The system tracks **tokens** (individual outcomes) rather than **markets**:
- Each market typically has 2 outcomes: "Yes" and "No"
- Each outcome is a separate token with unique `token_id`
- 291 markets → approximately 582 tokens (if all are binary markets)

The WebSocket connections subscribe to **token IDs**, not market IDs.

## Code References

- `src/lifecycle/api.py:36-42` - API query parameters (only `closed=false` filter)
- `src/lifecycle/api.py:23-61` - Pagination implementation with bug
- `src/lifecycle/api.py:52-56` - **BUG**: Break condition checks filtered count
- `src/lifecycle/api.py:74` - Validation filter applied before pagination check
- `src/lifecycle/api.py:89-92` - Market validation logic (required fields)
- `src/lifecycle/api.py:95-144` - Market parsing with fallback values
- `src/lifecycle/api.py:17` - `DEFAULT_PAGE_SIZE = 500`
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

**Resolved**: The 291 market count is due to a pagination bug, not API limitations.

## Follow-up Research (2025-12-08 17:25 IST)

### Bug Discovery

User identified that the pagination break condition at `src/lifecycle/api.py:52-56` incorrectly checks the filtered market count instead of the raw API response count.

**The Issue**:
```python
# Line 52-56
if len(markets) < page_size:  # markets is AFTER filtering
    logger.warning(
        f"markets less than page_size: {len(markets)} and {len(all_markets)} and {page_size}"
    )
    break
```

But `markets` comes from `_fetch_page()` which filters at line 74:
```python
# Line 74
return [_parse_market(m) for m in data if _is_valid_market(m)]
```

**Impact**: If any markets are filtered out due to missing required fields (`conditionId`, `question`, `outcomes`, `clobTokenIds`, `endDate`), the loop terminates early, leaving potentially hundreds of markets unfetched.

**Fix needed**: The break condition should check the raw API response length before filtering, not the filtered result length. This would require `_fetch_page()` to return both the filtered markets and the original response count.
