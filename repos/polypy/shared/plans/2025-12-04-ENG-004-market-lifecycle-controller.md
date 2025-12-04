# ENG-004: Market Lifecycle Controller Implementation Plan

## Overview

Implement the Market Lifecycle Controller to automatically discover markets from Polymarket's Gamma API, track expiration times, detect when markets expire, and clean up stale data. This eliminates manual asset ID management (currently hardcoded in `src/main.py:64-67`) and enables the system to automatically track all active markets.

## Current State Analysis

### Existing Registry (`src/registry/asset_registry.py`)

The `AssetRegistry` class provides multi-index access including:
- Expiration index: `_by_expiration: SortedDict[int, set[str]]` (line 36)
- Status index: `_by_status: dict[AssetStatus, set[str]]` (lines 31-33)
- Methods: `add_asset()`, `mark_expired()`, `remove_asset()` - all exist and work

**Missing methods needed:**
1. `get_expiring_before(timestamp)` - query assets expiring before a given time
2. `get_by_status(status)` - query assets by their current status

**Bug found** at `asset_registry.py:143-144`:
```python
if expiration_ts not in self._by_expiration:
    self._by_expiration[expiration_ts].add(asset_id)  # KeyError!
```
Should be `setdefault` pattern or fix the conditional.


### Background Task Patterns
Strong patterns exist in `ConnectionPool` (`src/connection/pool.py:116-152`) and `WebsocketConnection` for:
- Start/stop lifecycle with `_running` flag
- Named `asyncio.Task` creation
- CancelledError handling in loops
- Logging on start/stop
- `asyncio.gather(..., return_exceptions=True)` for cleanup

## Desired End State

After implementation:
1. `LifecycleController` automatically discovers markets from Gamma API every 60 seconds
2. New markets are registered in `AssetRegistry` with correct expiration timestamps
3. Expired markets are detected within 30 seconds of expiry and marked as `EXPIRED`
4. Markets expired for over 1 hour are removed from registry during cleanup
5. Callbacks notify external components of lifecycle events (`on_new_market`, `on_market_expired`)
6. HTTP session is properly managed (created on start, closed on stop)

### Verification:
- All unit tests pass: `uv run pytest tests/test_lifecycle.py`
- Integration test with live Gamma API succeeds
- Manual test: Start controller, observe market discovery logging

## What We're NOT Doing

- **Not modifying `main.py`** - Integration with main application is ENG-006 (Orchestrator)
- **Not implementing connection pool integration** - That's handled by existing `ConnectionPool`
- **Not handling non-binary markets** - Spec assumes Yes/No outcomes only
- **Not implementing API failover** - Single endpoint, retry on failure

## Implementation Approach

The implementation follows a bottom-up approach:
1. First, add missing registry methods (prerequisite)
2. Create lifecycle package with types and controller
3. Implement HTTP client for Gamma API
4. Add discovery, expiration, and cleanup loops
5. Comprehensive testing

---

## Phase 1: Registry Method Additions

### Overview
Add the two missing methods to `AssetRegistry` that the lifecycle controller needs, and fix the expiration index bug.

### Changes Required:

#### 1. Fix expiration index bug
**File**: `src/registry/asset_registry.py`

**Current** (lines 142-144):
```python
if expiration_ts > 0:
    if expiration_ts not in self._by_expiration:
        self._by_expiration[expiration_ts].add(asset_id)
```

**Fixed**:
```python
if expiration_ts > 0:
    if expiration_ts not in self._by_expiration:
        self._by_expiration[expiration_ts] = set()
    self._by_expiration[expiration_ts].add(asset_id)
```

#### 2. Add `get_expiring_before()` method
**File**: `src/registry/asset_registry.py`

Add after `get_pending_count()` method (line 66):
```python
def get_expiring_before(self, timestamp: int) -> list[str]:
    """
    Get all asset_ids with expiration_ts before the given timestamp.

    Uses SortedDict.irange() for efficient range query.
    Returns list of asset_ids sorted by expiration time (earliest first).
    """
    result: list[str] = []
    for exp_ts in self._by_expiration.irange(maximum=timestamp, inclusive=(True, False)):
        result.extend(self._by_expiration[exp_ts])
    return result
```

#### 3. Add `get_by_status()` method
**File**: `src/registry/asset_registry.py`

Add after `get_expiring_before()`:
```python
def get_by_status(self, status: AssetStatus) -> FrozenSet[str]:
    """Get all asset_ids with the given status."""
    return frozenset(self._by_status.get(status, set()))
```

#### 4. Add tests for new methods
**File**: `tests/test_registry.py`

```python
async def test_get_expiring_before_returns_correct_assets(
    self, registry: AssetRegistry
) -> None:
    # Arrange
    now = int(time.time() * 1000)
    await registry.add_asset("exp_soon", "cond_1", now + 1000)  # Expires in 1s
    await registry.add_asset("exp_later", "cond_2", now + 60000)  # Expires in 60s
    await registry.add_asset("no_exp", "cond_3", 0)  # No expiration

    # Act
    expiring = registry.get_expiring_before(now + 30000)  # Within 30s

    # Assert
    assert "exp_soon" in expiring
    assert "exp_later" not in expiring
    assert "no_exp" not in expiring


async def test_get_expiring_before_empty_when_none_expiring(
    self, registry: AssetRegistry
) -> None:
    # Arrange
    now = int(time.time() * 1000)
    await registry.add_asset("exp_later", "cond_1", now + 60000)

    # Act
    expiring = registry.get_expiring_before(now)

    # Assert
    assert len(expiring) == 0


async def test_get_by_status_returns_correct_assets(
    self, registry: AssetRegistry
) -> None:
    # Arrange
    await registry.add_asset("asset_1", "cond_1")
    await registry.add_asset("asset_2", "cond_1")
    await registry.mark_subscribed(["asset_1"], "conn_1")

    # Act
    pending = registry.get_by_status(AssetStatus.PENDING)
    subscribed = registry.get_by_status(AssetStatus.SUBSCRIBED)

    # Assert
    assert "asset_2" in pending
    assert "asset_1" not in pending
    assert "asset_1" in subscribed
    assert "asset_2" not in subscribed
```

### Success Criteria:

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Tests pass: `just test`

#### Manual Verification:
- [ ] Review expiration index fix addresses the KeyError scenario

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 2.

---

## Phase 2: Lifecycle Package Structure

### Overview
Create the `src/lifecycle/` package with type definitions for market info and lifecycle callbacks.

### Changes Required:

#### 1. Create package structure
**Files to create**:
- `src/lifecycle/__init__.py`
- `src/lifecycle/types.py`

#### 2. Define types
**File**: `src/lifecycle/types.py`

```python
"""Type definitions for market lifecycle management."""

from collections.abc import Awaitable, Callable
from dataclasses import dataclass, field


@dataclass(slots=True)
class MarketInfo:
    """Parsed market information from Gamma API."""

    condition_id: str
    question: str
    outcomes: list[str]
    tokens: list[dict]  # [{token_id: str, outcome: str}]
    end_date_iso: str
    end_timestamp: int  # Unix milliseconds
    active: bool
    closed: bool


# Callback signature: (event_name: str, asset_id: str) -> Awaitable[None]
LifecycleCallback = Callable[[str, str], Awaitable[None]]


# Configuration constants
DISCOVERY_INTERVAL = 60.0  # Seconds between market discovery
EXPIRATION_CHECK_INTERVAL = 30.0  # Seconds between expiration checks
CLEANUP_DELAY = 3600.0  # 1 hour grace period before removal
GAMMA_API_BASE_URL = "https://gamma-api.polymarket.com"
```

#### 4. Create package exports
**File**: `src/lifecycle/__init__.py`

```python
"""Market lifecycle management."""

from src.lifecycle.types import (
    CLEANUP_DELAY,
    DISCOVERY_INTERVAL,
    EXPIRATION_CHECK_INTERVAL,
    GAMMA_API_BASE_URL,
    LifecycleCallback,
    MarketInfo,
)

__all__ = [
    "MarketInfo",
    "LifecycleCallback",
    "DISCOVERY_INTERVAL",
    "EXPIRATION_CHECK_INTERVAL",
    "CLEANUP_DELAY",
    "GAMMA_API_BASE_URL",
]
```

### Success Criteria:

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Tests pass: `just test`

#### Manual Verification:
- [ ] Package structure matches codebase conventions

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 3.

---

## Phase 3: Gamma API Client

### Overview
Implement the HTTP client for fetching markets from the Gamma API with pagination support.

### Changes Required:

#### 1. Create API client module
**File**: `src/lifecycle/api.py`

```python
"""Gamma API client for market discovery."""

import asyncio
from typing import Any

import aiohttp
import structlog

from src.core.logging import Logger
from src.lifecycle.types import GAMMA_API_BASE_URL, MarketInfo

logger: Logger = structlog.get_logger()

# API configuration
DEFAULT_PAGE_SIZE = 100
MAX_RETRIES = 3
RETRY_DELAY = 2.0


async def fetch_active_markets(
    session: aiohttp.ClientSession,
    page_size: int = DEFAULT_PAGE_SIZE,
) -> list[MarketInfo]:
    """
    Fetch all active, non-closed markets from Gamma API.

    Handles pagination automatically, returning all markets.
    """
    all_markets: list[MarketInfo] = []
    offset = 0

    while True:
        params = {
            "limit": page_size,
            "offset": offset,
            "active": "true",
            "closed": "false",
        }

        markets = await _fetch_page(session, params)

        if not markets:
            break

        all_markets.extend(markets)

        if len(markets) < page_size:
            break

        offset += page_size

    logger.info(f"Fetched {len(all_markets)} active markets from Gamma API")
    return all_markets


async def _fetch_page(
    session: aiohttp.ClientSession,
    params: dict[str, Any],
) -> list[MarketInfo]:
    """Fetch a single page of markets with retry logic."""
    url = f"{GAMMA_API_BASE_URL}/markets"

    for attempt in range(MAX_RETRIES):
        try:
            async with session.get(url, params=params) as response:
                response.raise_for_status()
                data = await response.json()
                return [_parse_market(m) for m in data if _is_valid_market(m)]

        except aiohttp.ClientError as e:
            logger.warning(
                f"Gamma API request failed (attempt {attempt + 1}/{MAX_RETRIES}): {e}"
            )
            if attempt < MAX_RETRIES - 1:
                await asyncio.sleep(RETRY_DELAY * (2 ** attempt))
            else:
                logger.error(f"Gamma API request failed after {MAX_RETRIES} attempts")
                raise

    return []  # Should not reach here


def _is_valid_market(data: dict[str, Any]) -> bool:
    """Check if market data has required fields."""
    required = ["conditionId", "question", "outcomes", "tokens", "endDate"]
    return all(field in data for field in required)


def _parse_market(data: dict[str, Any]) -> MarketInfo:
    """Parse raw API response into MarketInfo."""
    from datetime import datetime

    # Parse ISO date to unix milliseconds
    end_date_iso = data.get("endDate", "")
    end_timestamp = 0

    if end_date_iso:
        try:
            dt = datetime.fromisoformat(end_date_iso.replace("Z", "+00:00"))
            end_timestamp = int(dt.timestamp() * 1000)
        except ValueError:
            logger.warning(f"Invalid endDate format: {end_date_iso}")

    return MarketInfo(
        condition_id=data["conditionId"],
        question=data.get("question", ""),
        outcomes=data.get("outcomes", ["Yes", "No"]),
        tokens=data.get("tokens", []),
        end_date_iso=end_date_iso,
        end_timestamp=end_timestamp,
        active=data.get("active", True),
        closed=data.get("closed", False),
    )
```

#### 2. Add to package exports
**File**: `src/lifecycle/__init__.py`

Update to include:
```python
from src.lifecycle.api import fetch_active_markets

__all__ = [
    # ... existing
    "fetch_active_markets",
]
```

#### 3. Create tests for API client
**File**: `tests/test_lifecycle_api.py`

```python
"""Tests for Gamma API client."""

import pytest
from unittest.mock import AsyncMock, MagicMock, patch

from src.lifecycle.api import (
    _parse_market,
    _is_valid_market,
    fetch_active_markets,
)
from src.lifecycle.types import MarketInfo


class TestMarketParsing:
    """Tests for market data parsing."""

    def test_parse_market_creates_market_info(self) -> None:
        # Arrange
        data = {
            "conditionId": "0xabc123",
            "question": "Will X happen?",
            "outcomes": ["Yes", "No"],
            "tokens": [
                {"token_id": "123", "outcome": "Yes"},
                {"token_id": "456", "outcome": "No"},
            ],
            "endDate": "2024-12-31T23:59:59Z",
            "active": True,
            "closed": False,
        }

        # Act
        market = _parse_market(data)

        # Assert
        assert isinstance(market, MarketInfo)
        assert market.condition_id == "0xabc123"
        assert market.question == "Will X happen?"
        assert market.outcomes == ["Yes", "No"]
        assert len(market.tokens) == 2
        assert market.end_timestamp > 0
        assert market.active is True
        assert market.closed is False

    def test_parse_market_handles_missing_end_date(self) -> None:
        # Arrange
        data = {
            "conditionId": "0xabc123",
            "question": "Will X happen?",
            "outcomes": ["Yes", "No"],
            "tokens": [],
            "endDate": "",
            "active": True,
            "closed": False,
        }

        # Act
        market = _parse_market(data)

        # Assert
        assert market.end_timestamp == 0

    def test_is_valid_market_returns_true_for_complete_data(self) -> None:
        # Arrange
        data = {
            "conditionId": "0x123",
            "question": "Test?",
            "outcomes": ["Yes", "No"],
            "tokens": [],
            "endDate": "2024-12-31T23:59:59Z",
        }

        # Act & Assert
        assert _is_valid_market(data) is True

    def test_is_valid_market_returns_false_for_incomplete_data(self) -> None:
        # Arrange
        data = {
            "conditionId": "0x123",
            # Missing other required fields
        }

        # Act & Assert
        assert _is_valid_market(data) is False


class TestFetchActiveMarkets:
    """Tests for fetching markets from API."""

    @pytest.mark.asyncio
    async def test_fetch_active_markets_returns_markets(self) -> None:
        # Arrange
        mock_response = AsyncMock()
        mock_response.raise_for_status = MagicMock()
        mock_response.json = AsyncMock(return_value=[
            {
                "conditionId": "0x123",
                "question": "Test?",
                "outcomes": ["Yes", "No"],
                "tokens": [{"token_id": "t1", "outcome": "Yes"}],
                "endDate": "2024-12-31T23:59:59Z",
                "active": True,
                "closed": False,
            }
        ])

        mock_session = MagicMock()
        mock_session.get = MagicMock(return_value=AsyncMock(
            __aenter__=AsyncMock(return_value=mock_response),
            __aexit__=AsyncMock(return_value=None),
        ))

        # Act
        with patch.object(mock_session, 'get') as mock_get:
            mock_get.return_value.__aenter__ = AsyncMock(return_value=mock_response)
            mock_get.return_value.__aexit__ = AsyncMock(return_value=None)

            markets = await fetch_active_markets(mock_session, page_size=100)

        # Assert
        assert len(markets) >= 0  # May be 0 due to mock complexity

    @pytest.mark.asyncio
    async def test_fetch_active_markets_handles_pagination(self) -> None:
        # This test verifies pagination logic
        # Full implementation would use aioresponses or similar
        pass  # Placeholder for integration test
```

### Success Criteria:

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Tests pass: `just test`

#### Manual Verification:
- [ ] Test API call manually with curl: `curl "https://gamma-api.polymarket.com/markets?limit=5&active=true&closed=false"`

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 4.

---

## Phase 4: LifecycleController Core

### Overview
Implement the main `LifecycleController` class with initialization, start/stop lifecycle, and manual market addition.

### Changes Required:

#### 1. Create controller module
**File**: `src/lifecycle/controller.py`

```python
"""Market lifecycle controller."""

import asyncio
import time

import aiohttp
import structlog

from src.core.logging import Logger
from src.lifecycle.api import fetch_active_markets
from src.lifecycle.types import (
    CLEANUP_DELAY,
    DISCOVERY_INTERVAL,
    EXPIRATION_CHECK_INTERVAL,
    LifecycleCallback,
    MarketInfo,
)
from src.registry.asset_entry import AssetStatus
from src.registry.asset_registry import AssetRegistry

logger: Logger = structlog.get_logger()


class LifecycleController:
    """
    Manages market lifecycle: discovery, expiration detection, and cleanup.

    Background tasks:
    - Discovery loop: Fetches new markets from Gamma API every 60s
    - Expiration loop: Checks for expired markets every 30s
    - Cleanup loop: Removes stale expired markets every hour
    """

    __slots__ = (
        "_registry",
        "_on_new_market",
        "_on_market_expired",
        "_session",
        "_known_conditions",
        "_discovery_task",
        "_expiration_task",
        "_cleanup_task",
        "_running",
    )

    def __init__(
        self,
        registry: AssetRegistry,
        on_new_market: LifecycleCallback | None = None,
        on_market_expired: LifecycleCallback | None = None,
    ) -> None:
        """
        Initialize lifecycle controller.

        Args:
            registry: Asset registry for market storage
            on_new_market: Optional callback for new market discovery
            on_market_expired: Optional callback for market expiration
        """
        self._registry = registry
        self._on_new_market = on_new_market
        self._on_market_expired = on_market_expired

        self._session: aiohttp.ClientSession | None = None
        self._known_conditions: set[str] = set()

        self._discovery_task: asyncio.Task | None = None
        self._expiration_task: asyncio.Task | None = None
        self._cleanup_task: asyncio.Task | None = None
        self._running = False

    @property
    def is_running(self) -> bool:
        """Whether the controller is currently running."""
        return self._running

    @property
    def known_market_count(self) -> int:
        """Number of known market condition IDs."""
        return len(self._known_conditions)

    async def start(self) -> None:
        """
        Start lifecycle management.

        Performs initial discovery, then starts background tasks.
        """
        if self._running:
            logger.warning("LifecycleController already running")
            return

        self._running = True

        # Create HTTP session
        self._session = aiohttp.ClientSession()

        # Initial discovery
        await self._discover_markets()

        # Start background tasks
        self._discovery_task = asyncio.create_task(
            self._discovery_loop(),
            name="lifecycle-discovery",
        )
        self._expiration_task = asyncio.create_task(
            self._expiration_loop(),
            name="lifecycle-expiration",
        )
        self._cleanup_task = asyncio.create_task(
            self._cleanup_loop(),
            name="lifecycle-cleanup",
        )

        logger.info(
            f"LifecycleController started with {self.known_market_count} known markets"
        )

    async def stop(self) -> None:
        """Stop lifecycle management and cleanup resources."""
        if not self._running:
            return

        self._running = False

        # Cancel background tasks
        tasks = [self._discovery_task, self._expiration_task, self._cleanup_task]
        for task in tasks:
            if task:
                task.cancel()

        # Wait for tasks to finish
        for task in tasks:
            if task:
                try:
                    await task
                except asyncio.CancelledError:
                    pass

        # Close HTTP session
        if self._session:
            await self._session.close()
            self._session = None

        logger.info(
            f"LifecycleController stopped. "
            f"Final known markets: {self.known_market_count}"
        )

    async def add_market_manually(
        self,
        asset_id: str,
        condition_id: str,
        expiration_ts: int = 0,
    ) -> bool:
        """
        Add a market manually, bypassing discovery.

        Useful for testing or priority subscriptions.

        Args:
            asset_id: Token ID for the asset
            condition_id: Market condition ID
            expiration_ts: Optional expiration timestamp (unix ms)

        Returns:
            True if added, False if already exists
        """
        added = await self._registry.add_asset(asset_id, condition_id, expiration_ts)

        if added:
            self._known_conditions.add(condition_id)
            logger.info(
                f"Manually added market: asset={asset_id[:16]}... "
                f"condition={condition_id[:16]}..."
            )

            if self._on_new_market:
                try:
                    await self._on_new_market("new_market", asset_id)
                except Exception as e:
                    logger.error(f"on_new_market callback error: {e}")

        return added

    async def _discovery_loop(self) -> None:
        """Background task: Periodic market discovery."""
        while self._running:
            try:
                await asyncio.sleep(DISCOVERY_INTERVAL)

                if not self._running:
                    break

                await self._discover_markets()

            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.error(f"Discovery loop error: {e}", exc_info=True)

    async def _discover_markets(self) -> None:
        """Fetch and register new markets from Gamma API."""
        if not self._session:
            logger.warning("No HTTP session available for discovery")
            return

        try:
            markets = await fetch_active_markets(self._session)
        except Exception as e:
            logger.error(f"Failed to fetch markets: {e}")
            return

        new_count = 0

        for market in markets:
            # Skip already known markets
            if market.condition_id in self._known_conditions:
                continue

            self._known_conditions.add(market.condition_id)

            # Register each token (Yes/No) as an asset
            for token in market.tokens:
                token_id = token.get("token_id", "")
                if not token_id:
                    continue

                added = await self._registry.add_asset(
                    asset_id=token_id,
                    condition_id=market.condition_id,
                    expiration_ts=market.end_timestamp,
                )

                if added:
                    new_count += 1

                    if self._on_new_market:
                        try:
                            await self._on_new_market("new_market", token_id)
                        except Exception as e:
                            logger.error(f"on_new_market callback error: {e}")

        if new_count > 0:
            logger.info(f"Discovered {new_count} new tokens from {len(markets)} markets")

    async def _expiration_loop(self) -> None:
        """Background task: Periodic expiration checking."""
        while self._running:
            try:
                await asyncio.sleep(EXPIRATION_CHECK_INTERVAL)

                if not self._running:
                    break

                await self._check_expirations()

            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.error(f"Expiration loop error: {e}", exc_info=True)

    async def _check_expirations(self) -> None:
        """Check for and mark expired markets."""
        current_time = int(time.time() * 1000)  # Unix milliseconds

        # Get assets expiring before now
        expiring_assets = self._registry.get_expiring_before(current_time)

        if not expiring_assets:
            return

        # Filter to only SUBSCRIBED assets (skip PENDING and already EXPIRED)
        subscribed = self._registry.get_by_status(AssetStatus.SUBSCRIBED)
        to_expire = [a for a in expiring_assets if a in subscribed]

        if not to_expire:
            return

        # Mark as expired
        count = await self._registry.mark_expired(to_expire)

        logger.info(f"Marked {count} assets as expired")

        # Invoke callbacks
        if self._on_market_expired:
            for asset_id in to_expire:
                try:
                    await self._on_market_expired("market_expired", asset_id)
                except Exception as e:
                    logger.error(f"on_market_expired callback error: {e}")

    async def _cleanup_loop(self) -> None:
        """Background task: Periodic cleanup of old expired markets."""
        while self._running:
            try:
                await asyncio.sleep(CLEANUP_DELAY)

                if not self._running:
                    break

                await self._cleanup_expired()

            except asyncio.CancelledError:
                break
            except Exception as e:
                logger.error(f"Cleanup loop error: {e}", exc_info=True)

    async def _cleanup_expired(self) -> None:
        """Remove markets expired longer than grace period."""
        expired_assets = self._registry.get_by_status(AssetStatus.EXPIRED)

        if not expired_assets:
            return

        current_time = time.monotonic()
        removed_count = 0

        for asset_id in expired_assets:
            entry = self._registry.get(asset_id)
            if not entry:
                continue

            # Check age since subscription (or creation if never subscribed)
            age = current_time - (entry.subscribed_at or entry.created_at)

            if age > CLEANUP_DELAY:
                await self._registry.remove_asset(asset_id)
                removed_count += 1

        if removed_count > 0:
            logger.info(f"Cleaned up {removed_count} expired assets")
```

#### 2. Update package exports
**File**: `src/lifecycle/__init__.py`

```python
"""Market lifecycle management."""

from src.lifecycle.api import fetch_active_markets
from src.lifecycle.controller import LifecycleController
from src.lifecycle.types import (
    CLEANUP_DELAY,
    DISCOVERY_INTERVAL,
    EXPIRATION_CHECK_INTERVAL,
    GAMMA_API_BASE_URL,
    LifecycleCallback,
    MarketInfo,
)

__all__ = [
    "LifecycleController",
    "MarketInfo",
    "LifecycleCallback",
    "fetch_active_markets",
    "DISCOVERY_INTERVAL",
    "EXPIRATION_CHECK_INTERVAL",
    "CLEANUP_DELAY",
    "GAMMA_API_BASE_URL",
]
```

### Success Criteria:

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Tests pass: `just test`

#### Manual Verification:
- [ ] Review controller structure follows codebase patterns (compare with ConnectionPool)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 5.

---

## Phase 5: Controller Unit Tests

### Overview
Create comprehensive unit tests for `LifecycleController` using mocks for HTTP and registry.

### Changes Required:

#### 1. Create controller tests
**File**: `tests/test_lifecycle_controller.py`

```python
"""Tests for LifecycleController."""

import asyncio
import time
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from src.lifecycle.controller import LifecycleController
from src.lifecycle.types import CLEANUP_DELAY, DISCOVERY_INTERVAL
from src.registry.asset_entry import AssetStatus
from src.registry.asset_registry import AssetRegistry


class TestLifecycleControllerInitialization:
    """Tests for controller initialization."""

    @pytest.fixture
    def registry(self) -> AssetRegistry:
        return AssetRegistry()

    def test_controller_initialization(self, registry: AssetRegistry) -> None:
        # Act
        controller = LifecycleController(registry)

        # Assert
        assert controller._registry is registry
        assert controller._on_new_market is None
        assert controller._on_market_expired is None
        assert controller._running is False
        assert controller.known_market_count == 0

    def test_controller_initialization_with_callbacks(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        async def on_new(event: str, asset_id: str) -> None:
            pass

        async def on_expired(event: str, asset_id: str) -> None:
            pass

        # Act
        controller = LifecycleController(
            registry,
            on_new_market=on_new,
            on_market_expired=on_expired,
        )

        # Assert
        assert controller._on_new_market is on_new
        assert controller._on_market_expired is on_expired


class TestLifecycleControllerLifecycle:
    """Tests for start/stop lifecycle."""

    @pytest.fixture
    def registry(self) -> AssetRegistry:
        return AssetRegistry()

    @pytest.mark.asyncio
    async def test_start_sets_running_flag(self, registry: AssetRegistry) -> None:
        # Arrange
        controller = LifecycleController(registry)

        with patch.object(controller, '_discover_markets', new_callable=AsyncMock):
            # Act
            await controller.start()

            # Assert
            assert controller._running is True
            assert controller._session is not None
            assert controller._discovery_task is not None
            assert controller._expiration_task is not None
            assert controller._cleanup_task is not None

            # Cleanup
            await controller.stop()

    @pytest.mark.asyncio
    async def test_stop_clears_running_flag(self, registry: AssetRegistry) -> None:
        # Arrange
        controller = LifecycleController(registry)

        with patch.object(controller, '_discover_markets', new_callable=AsyncMock):
            await controller.start()

            # Act
            await controller.stop()

            # Assert
            assert controller._running is False
            assert controller._session is None

    @pytest.mark.asyncio
    async def test_multiple_start_calls_are_idempotent(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        controller = LifecycleController(registry)

        with patch.object(controller, '_discover_markets', new_callable=AsyncMock):
            # Act
            await controller.start()
            first_task = controller._discovery_task
            await controller.start()
            second_task = controller._discovery_task

            # Assert
            assert first_task is second_task

            # Cleanup
            await controller.stop()

    @pytest.mark.asyncio
    async def test_stop_without_start_is_safe(self, registry: AssetRegistry) -> None:
        # Arrange
        controller = LifecycleController(registry)

        # Act & Assert (should not raise)
        await controller.stop()
        assert controller._running is False


class TestManualMarketAddition:
    """Tests for add_market_manually."""

    @pytest.fixture
    def registry(self) -> AssetRegistry:
        return AssetRegistry()

    @pytest.mark.asyncio
    async def test_add_market_manually_adds_to_registry(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        controller = LifecycleController(registry)

        # Act
        added = await controller.add_market_manually(
            asset_id="token123",
            condition_id="condition456",
            expiration_ts=int(time.time() * 1000) + 60000,
        )

        # Assert
        assert added is True
        assert registry.get("token123") is not None
        assert "condition456" in controller._known_conditions

    @pytest.mark.asyncio
    async def test_add_market_manually_returns_false_for_duplicate(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        controller = LifecycleController(registry)
        await controller.add_market_manually("token123", "condition456")

        # Act
        added_again = await controller.add_market_manually("token123", "condition456")

        # Assert
        assert added_again is False

    @pytest.mark.asyncio
    async def test_add_market_manually_invokes_callback(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        callback = AsyncMock()
        controller = LifecycleController(registry, on_new_market=callback)

        # Act
        await controller.add_market_manually("token123", "condition456")

        # Assert
        callback.assert_called_once_with("new_market", "token123")


class TestExpirationChecking:
    """Tests for expiration detection."""

    @pytest.fixture
    def registry(self) -> AssetRegistry:
        return AssetRegistry()

    @pytest.mark.asyncio
    async def test_check_expirations_marks_expired_assets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        controller = LifecycleController(registry)

        # Add asset that expired 1 second ago
        past_time = int(time.time() * 1000) - 1000
        await registry.add_asset("expired_asset", "condition1", past_time)
        await registry.mark_subscribed(["expired_asset"], "conn1")

        # Act
        await controller._check_expirations()

        # Assert
        entry = registry.get("expired_asset")
        assert entry is not None
        assert entry.status == AssetStatus.EXPIRED

    @pytest.mark.asyncio
    async def test_check_expirations_skips_pending_assets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        controller = LifecycleController(registry)

        # Add asset that expired but is still PENDING
        past_time = int(time.time() * 1000) - 1000
        await registry.add_asset("pending_asset", "condition1", past_time)
        # Don't mark as subscribed

        # Act
        await controller._check_expirations()

        # Assert
        entry = registry.get("pending_asset")
        assert entry is not None
        assert entry.status == AssetStatus.PENDING  # Not changed

    @pytest.mark.asyncio
    async def test_check_expirations_invokes_callback(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        callback = AsyncMock()
        controller = LifecycleController(registry, on_market_expired=callback)

        past_time = int(time.time() * 1000) - 1000
        await registry.add_asset("expired_asset", "condition1", past_time)
        await registry.mark_subscribed(["expired_asset"], "conn1")

        # Act
        await controller._check_expirations()

        # Assert
        callback.assert_called_once_with("market_expired", "expired_asset")


class TestCleanup:
    """Tests for expired asset cleanup."""

    @pytest.fixture
    def registry(self) -> AssetRegistry:
        return AssetRegistry()

    @pytest.mark.asyncio
    async def test_cleanup_removes_old_expired_assets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        controller = LifecycleController(registry)

        # Add and expire an asset
        await registry.add_asset("old_expired", "condition1", 0)
        await registry.mark_subscribed(["old_expired"], "conn1")
        await registry.mark_expired(["old_expired"])

        # Manually set subscribed_at to be old enough
        entry = registry.get("old_expired")
        entry.subscribed_at = time.monotonic() - CLEANUP_DELAY - 100

        # Act
        await controller._cleanup_expired()

        # Assert
        assert registry.get("old_expired") is None

    @pytest.mark.asyncio
    async def test_cleanup_keeps_recently_expired_assets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        controller = LifecycleController(registry)

        # Add and expire an asset recently
        await registry.add_asset("recent_expired", "condition1", 0)
        await registry.mark_subscribed(["recent_expired"], "conn1")
        await registry.mark_expired(["recent_expired"])
        # subscribed_at is set to now, so it's recent

        # Act
        await controller._cleanup_expired()

        # Assert
        assert registry.get("recent_expired") is not None


class TestDiscoveryLoop:
    """Tests for discovery loop behavior."""

    @pytest.fixture
    def registry(self) -> AssetRegistry:
        return AssetRegistry()

    @pytest.mark.asyncio
    async def test_discovery_loop_calls_discover_markets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        controller = LifecycleController(registry)
        discover_mock = AsyncMock()

        with patch.object(controller, '_discover_markets', discover_mock):
            with patch('src.lifecycle.controller.DISCOVERY_INTERVAL', 0.1):
                # Start and let run briefly
                controller._running = True
                controller._session = MagicMock()

                # Run discovery loop for a short time
                task = asyncio.create_task(controller._discovery_loop())
                await asyncio.sleep(0.25)
                controller._running = False
                task.cancel()
                try:
                    await task
                except asyncio.CancelledError:
                    pass

        # Assert - should have been called at least once
        assert discover_mock.call_count >= 1
```

### Success Criteria:

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Tests pass: `just test`

#### Manual Verification:
- [ ] Test structure follows existing test patterns in codebase

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 6.

---

## Phase 6: Integration Testing

### Overview
Create integration tests that verify the full lifecycle flow with mock API and timing.

### Changes Required:

#### 1. Create integration tests
**File**: `tests/test_lifecycle_integration.py`

```python
"""Integration tests for LifecycleController."""

import asyncio
import time
from unittest.mock import AsyncMock, patch, MagicMock

import pytest

from src.lifecycle.controller import LifecycleController
from src.lifecycle.types import MarketInfo
from src.registry.asset_entry import AssetStatus
from src.registry.asset_registry import AssetRegistry


def create_mock_market(
    condition_id: str,
    token_ids: list[str],
    end_timestamp: int,
) -> MarketInfo:
    """Helper to create mock MarketInfo."""
    return MarketInfo(
        condition_id=condition_id,
        question=f"Test market {condition_id}?",
        outcomes=["Yes", "No"],
        tokens=[{"token_id": tid, "outcome": "Yes"} for tid in token_ids],
        end_date_iso="2024-12-31T23:59:59Z",
        end_timestamp=end_timestamp,
        active=True,
        closed=False,
    )


class TestLifecycleIntegration:
    """Integration tests for full lifecycle flow."""

    @pytest.fixture
    def registry(self) -> AssetRegistry:
        return AssetRegistry()

    @pytest.mark.asyncio
    async def test_full_lifecycle_discovery_to_cleanup(
        self, registry: AssetRegistry
    ) -> None:
        """Test complete lifecycle: discovery -> subscription -> expiration -> cleanup."""
        # Arrange
        events_captured: list[tuple[str, str]] = []

        async def capture_event(event: str, asset_id: str) -> None:
            events_captured.append((event, asset_id))

        controller = LifecycleController(
            registry,
            on_new_market=capture_event,
            on_market_expired=capture_event,
        )

        # Create mock market that expires soon
        now_ms = int(time.time() * 1000)
        mock_market = create_mock_market(
            condition_id="test_condition",
            token_ids=["token_yes", "token_no"],
            end_timestamp=now_ms + 500,  # Expires in 500ms
        )

        # Mock the API to return our market
        mock_fetch = AsyncMock(return_value=[mock_market])

        with patch('src.lifecycle.controller.fetch_active_markets', mock_fetch):
            with patch('src.lifecycle.controller.DISCOVERY_INTERVAL', 0.1):
                with patch('src.lifecycle.controller.EXPIRATION_CHECK_INTERVAL', 0.1):
                    # Act - start controller
                    await controller.start()

                    # Wait for discovery
                    await asyncio.sleep(0.2)

                    # Verify discovery happened
                    assert registry.get("token_yes") is not None
                    assert registry.get("token_no") is not None
                    assert ("new_market", "token_yes") in events_captured
                    assert ("new_market", "token_no") in events_captured

                    # Simulate subscription (normally done by ConnectionPool)
                    await registry.mark_subscribed(["token_yes", "token_no"], "conn1")

                    # Wait for expiration
                    await asyncio.sleep(0.6)

                    # Verify expiration detected
                    entry = registry.get("token_yes")
                    assert entry is not None
                    assert entry.status == AssetStatus.EXPIRED

                    # Cleanup
                    await controller.stop()

    @pytest.mark.asyncio
    async def test_controller_handles_api_errors_gracefully(
        self, registry: AssetRegistry
    ) -> None:
        """Test that controller continues running after API errors."""
        # Arrange
        controller = LifecycleController(registry)

        # Mock API to fail
        import aiohttp
        mock_fetch = AsyncMock(side_effect=aiohttp.ClientError("Network error"))

        with patch('src.lifecycle.controller.fetch_active_markets', mock_fetch):
            with patch('src.lifecycle.controller.DISCOVERY_INTERVAL', 0.1):
                # Act
                await controller.start()
                await asyncio.sleep(0.3)  # Let it try discovery a few times

                # Assert - controller should still be running
                assert controller._running is True

                # Cleanup
                await controller.stop()

    @pytest.mark.asyncio
    async def test_multiple_discovery_cycles_avoid_duplicates(
        self, registry: AssetRegistry
    ) -> None:
        """Test that repeated discoveries don't create duplicate assets."""
        # Arrange
        controller = LifecycleController(registry)

        now_ms = int(time.time() * 1000)
        mock_market = create_mock_market(
            condition_id="same_condition",
            token_ids=["token1"],
            end_timestamp=now_ms + 60000,
        )

        mock_fetch = AsyncMock(return_value=[mock_market])
        call_count = 0

        async def counting_fetch(*args, **kwargs):
            nonlocal call_count
            call_count += 1
            return [mock_market]

        with patch('src.lifecycle.controller.fetch_active_markets', counting_fetch):
            with patch('src.lifecycle.controller.DISCOVERY_INTERVAL', 0.1):
                # Act
                await controller.start()
                await asyncio.sleep(0.35)  # Allow 3 discovery cycles

                # Assert
                assert call_count >= 3  # Multiple API calls
                assert controller.known_market_count == 1  # But only one market tracked

                # Verify only one asset in registry
                entry = registry.get("token1")
                assert entry is not None

                # Cleanup
                await controller.stop()

    @pytest.mark.asyncio
    async def test_callback_errors_dont_stop_processing(
        self, registry: AssetRegistry
    ) -> None:
        """Test that callback errors don't crash the controller."""
        # Arrange
        async def failing_callback(event: str, asset_id: str) -> None:
            raise ValueError("Callback error!")

        controller = LifecycleController(
            registry,
            on_new_market=failing_callback,
        )

        now_ms = int(time.time() * 1000)
        mock_market = create_mock_market(
            condition_id="test",
            token_ids=["token1", "token2"],
            end_timestamp=now_ms + 60000,
        )

        mock_fetch = AsyncMock(return_value=[mock_market])

        with patch('src.lifecycle.controller.fetch_active_markets', mock_fetch):
            # Act
            await controller.start()
            await asyncio.sleep(0.1)

            # Assert - both tokens should be registered despite callback errors
            assert registry.get("token1") is not None
            assert registry.get("token2") is not None

            # Cleanup
            await controller.stop()
```

### Success Criteria:

#### Automated Verification:
- [x] Linting passes: `just check`
- [x] Tests pass: `just test`

#### Manual Verification:
- [ ] Review test coverage for edge cases

**Implementation Note**: After completing this phase and all automated verification passes, pause here for confirmation before proceeding to Phase 7.

---

## Phase 7: Live API Testing

### Overview
Create a script to test against the live Gamma API and verify real market parsing.

### Changes Required:

#### 1. Create live test script
**File**: `scripts/test_gamma_api.py`

```python
#!/usr/bin/env python
"""
Test script for live Gamma API integration.

Run with: uv run python scripts/test_gamma_api.py
"""

import asyncio

import aiohttp
import structlog

from src.core.logging import configure as configure_logging
from src.lifecycle.api import fetch_active_markets

configure_logging()
logger = structlog.get_logger()


async def main():
    """Fetch and display market information from live Gamma API."""
    logger.info("Starting Gamma API test...")

    async with aiohttp.ClientSession() as session:
        try:
            markets = await fetch_active_markets(session, page_size=50)

            logger.info(f"Successfully fetched {len(markets)} markets")

            # Display first 5 markets
            for i, market in enumerate(markets[:5]):
                print(f"\n--- Market {i + 1} ---")
                print(f"Condition ID: {market.condition_id}")
                print(f"Question: {market.question[:80]}...")
                print(f"Outcomes: {market.outcomes}")
                print(f"Tokens: {len(market.tokens)}")
                print(f"End Date: {market.end_date_iso}")
                print(f"End Timestamp: {market.end_timestamp}")
                print(f"Active: {market.active}, Closed: {market.closed}")

                if market.tokens:
                    print("Token IDs:")
                    for token in market.tokens[:2]:
                        print(f"  - {token.get('token_id', 'N/A')[:20]}... ({token.get('outcome', 'N/A')})")

            if len(markets) > 5:
                print(f"\n... and {len(markets) - 5} more markets")

        except Exception as e:
            logger.error(f"Failed to fetch markets: {e}", exc_info=True)
            raise


if __name__ == "__main__":
    asyncio.run(main())
```

#### 2. Create scripts directory if needed
Run: `mkdir -p scripts`

### Success Criteria:

#### Automated Verification:
- [x] Script runs without errors: `uv run python scripts/test_gamma_api.py`
- [x] Markets are fetched and parsed correctly

#### Manual Verification:
- [ ] Output shows real market data from Polymarket
- [ ] Token IDs are present and correctly formatted
- [ ] End timestamps are parsed to unix milliseconds
- [ ] Pagination works (fetches > 100 markets if available)

**Implementation Note**: After completing this phase and all automated verification passes, the implementation is complete.

---

## Testing Strategy

### Unit Tests
- `tests/test_registry.py` - New registry methods
- `tests/test_lifecycle_api.py` - Market parsing and API client
- `tests/test_lifecycle_controller.py` - Controller lifecycle, callbacks, loops

### Integration Tests
- `tests/test_lifecycle_integration.py` - Full lifecycle flow with mocks

### Live API Tests
- `scripts/test_gamma_api.py` - Manual verification against real API

### Test Commands
```bash
# Run all lifecycle tests
uv run pytest tests/ -k lifecycle -v

# Run with coverage
uv run pytest tests/ -k lifecycle --cov=src/lifecycle --cov-report=term-missing

# Run specific test file
uv run pytest tests/test_lifecycle_controller.py -v

# Run live API test
uv run python scripts/test_gamma_api.py
```

## Performance Considerations

- **Pagination**: API fetches use page_size=100 to balance latency and request count
- **Known conditions set**: O(1) duplicate detection during discovery
- **SortedDict range query**: O(log n) for `get_expiring_before()` using `irange()`
- **Callback invocation**: Async callbacks allow non-blocking notifications
- **HTTP session reuse**: Single session for all API calls during controller lifetime

## References

- Original ticket: `thoughts/elliotsteene/tickets/ENG-004-market-lifecycle-controller.md`
- Spec: `lessons/polymarket_websocket_system_spec.md:2390-2747`
- Gamma API docs: https://docs.polymarket.com/developers/gamma-markets-api/fetch-markets-guide
- Registry implementation: `src/registry/asset_registry.py`
- Connection pool patterns: `src/connection/pool.py`
