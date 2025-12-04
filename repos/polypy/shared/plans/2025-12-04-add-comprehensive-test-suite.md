# Comprehensive Test Suite Implementation Plan

## Overview

Add comprehensive pytest-based unit tests for all core components in the polypy codebase. The project currently has only one test file (`tests/test_parser.py`) covering basic parser functionality. This plan adds tests for protocol conversions, orderbook state management, registry operations, connection statistics, and WebSocket connection handling with network I/O mocking.

## Current State Analysis

**Existing Tests:**
- `tests/test_parser.py` - Basic MessageParser tests with 3 parametrized cases (test_parser.py:15-114)
- Tests follow AAA pattern correctly
- Uses pytest parametrize for test data

**Missing Tests:**
- Protocol type conversions (PriceLevel, PriceChange, LastTradePrice)
- Parser edge cases (malformed data, missing fields, empty values)
- OrderbookState operations (snapshots, price changes, caching)
- OrderbookStore multi-book management
- AssetRegistry async operations with multi-index lookups
- ConnectionStats property calculations
- WebsocketConnection with network I/O mocking

**Dependencies Available:**
- `pytest-asyncio>=1.3.0` already in dev dependencies (pyproject.toml:21)
- `"pytest>=9.0.1"` already in dev dependencies (pyproject.toml:20)

## Desired End State

A complete test suite covering all core logic components with:
- 95%+ coverage of business logic (excluding main.py and connection/websocket.py I/O)
- All tests follow AAA (Arrange Act Assert) pattern
- Edge cases for parsers (malformed data, boundary conditions)
- Async tests for AssetRegistry operations
- Mocked network I/O for WebsocketConnection tests
- Clear test organization by module

**Verification:**
- Run `uv run pytest` - all tests pass
- Run `uv run pytest --verbose` - see clear test descriptions
- Run `uv run pytest tests/test_orderbook.py` - can run individual test modules

## What We're NOT Doing

- Integration tests (e.g., end-to-end WebSocket → Parser → Orderbook flow)
- Performance/benchmark tests
- Tests for main.py application entry point
- Tests for exercises/ module (reference implementations)
- Tests for core/config.py and core/logging.py (configuration only)
- Exhaustive edge case coverage (keeping it concise as requested)

## Implementation Approach

Add tests in 5 phases, organized by module. Each phase creates new test files following the existing pattern. Use pytest fixtures in conftest.py for common test data. Mock network I/O using unittest.mock for WebsocketConnection tests.

---

## Phase 1: Test Infrastructure Setup

### Overview
Set up pytest configuration and shared fixtures to support all subsequent test phases.

### Changes Required:


#### 1. Create pytest configuration
**File**: `pyproject.toml`
**Changes**: Add pytest configuration section at the end

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
python_classes = "Test*"
python_functions = "test_*"
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
```

#### 3. Create shared test fixtures
**File**: `tests/conftest.py` (new file)
**Changes**: Create fixtures for common test data

```python
"""Shared pytest fixtures for all test modules."""

import pytest

from src.messages.protocol import (
    BookSnapshot,
    EventType,
    LastTradePrice,
    ParsedMessage,
    PriceChange,
    PriceLevel,
    Side,
)


# Sample asset and market IDs
@pytest.fixture
def sample_asset_id() -> str:
    return "71321045679252212594626385532706912750332728571942532289631379312455583992563"


@pytest.fixture
def sample_market() -> str:
    return "0x5f65177b394277fd294cd75650044e32ba009a95022d88a0c1d565897d72f8f1"


# Sample price levels
@pytest.fixture
def sample_bid_levels() -> tuple[PriceLevel, ...]:
    return (
        PriceLevel(price=490, size=2000),  # 0.49 @ 20.00
        PriceLevel(price=480, size=3000),  # 0.48 @ 30.00
    )


@pytest.fixture
def sample_ask_levels() -> tuple[PriceLevel, ...]:
    return (
        PriceLevel(price=510, size=2500),  # 0.51 @ 25.00
        PriceLevel(price=520, size=1500),  # 0.52 @ 15.00
    )


# Sample book snapshot
@pytest.fixture
def sample_book_snapshot(
    sample_bid_levels: tuple[PriceLevel, ...],
    sample_ask_levels: tuple[PriceLevel, ...],
) -> BookSnapshot:
    return BookSnapshot(
        hash="0xabc123",
        bids=sample_bid_levels,
        asks=sample_ask_levels,
    )


# Sample price change
@pytest.fixture
def sample_price_change(sample_asset_id: str) -> PriceChange:
    return PriceChange(
        asset_id=sample_asset_id,
        price=500,
        size=1000,
        side=Side.BUY,
        hash="0xdef456",
        best_bid=500,
        best_ask=510,
    )
```

### Success Criteria:

#### Automated Verification:
- [x] Dependencies install successfully: `uv sync`
- [x] Pytest discovers test directory: `uv run pytest --collect-only`
- [x] Fixtures are available: `uv run pytest tests/test_parser.py -v`

#### Manual Verification:
- [ ] No pytest configuration warnings appear
- [ ] Test collection shows expected test count

**Implementation Note**: After completing this phase and all automated verification passes, confirm that pytest runs cleanly before proceeding to Phase 2.

---

## Phase 2: Protocol and Parser Tests

### Overview
Expand existing parser tests with edge cases and add comprehensive protocol conversion tests.

### Changes Required:

#### 1. Add protocol conversion tests
**File**: `tests/test_protocol.py` (new file)
**Changes**: Test string-to-integer conversions and struct creation

```python
"""Tests for protocol message type conversions."""

import pytest

from src.messages.protocol import (
    LastTradePrice,
    PriceChange,
    PriceLevel,
    Side,
    scale_price,
    scale_size,
)


class TestScalingFunctions:
    """Test price and size scaling utilities."""

    @pytest.mark.parametrize(
        "price_str,expected",
        [
            ("0.50", 500),
            ("0.456", 456),
            ("1.0", 1000),
            ("0.001", 1),
            ("0", 0),
        ],
    )
    def test_scale_price(self, price_str: str, expected: int) -> None:
        # Arrange & Act
        result = scale_price(price_str)

        # Assert
        assert result == expected

    @pytest.mark.parametrize(
        "size_str,expected",
        [
            ("100", 10000),
            ("10.25", 1025),
            ("219.217767", 21921),  # Truncates decimals beyond 2 places
            ("0", 0),
        ],
    )
    def test_scale_size(self, size_str: str, expected: int) -> None:
        # Arrange & Act
        result = scale_size(size_str)

        # Assert
        assert result == expected


class TestPriceLevel:
    """Test PriceLevel struct and conversion."""

    def test_from_strings_valid(self) -> None:
        # Arrange
        price_str = "0.48"
        size_str = "30"

        # Act
        level = PriceLevel.from_strings(price_str, size_str)

        # Assert
        assert level.price == 480
        assert level.size == 3000

    def test_price_level_frozen(self) -> None:
        # Arrange
        level = PriceLevel(price=500, size=1000)

        # Act & Assert
        with pytest.raises(AttributeError):
            level.price = 600  # type: ignore


class TestPriceChange:
    """Test PriceChange struct and conversion."""

    def test_from_strings_buy_side(self) -> None:
        # Arrange
        asset_id = "12345"
        price = "0.5"
        size = "200"
        side = "BUY"
        hash_val = "0xabc"
        best_bid = "0.5"
        best_ask = "0.51"

        # Act
        change = PriceChange.from_strings(
            asset_id=asset_id,
            price=price,
            size=size,
            side=side,
            hash=hash_val,
            best_bid=best_bid,
            best_ask=best_ask,
        )

        # Assert
        assert change.asset_id == asset_id
        assert change.price == 500
        assert change.size == 20000
        assert change.side == Side.BUY
        assert change.hash == hash_val
        assert change.best_bid == 500
        assert change.best_ask == 510

    def test_from_strings_sell_side(self) -> None:
        # Arrange
        asset_id = "67890"
        side = "SELL"

        # Act
        change = PriceChange.from_strings(
            asset_id=asset_id,
            price="0.49",
            size="100",
            side=side,
            hash="0xdef",
            best_bid="0.48",
            best_ask="0.49",
        )

        # Assert
        assert change.side == Side.SELL
        assert change.price == 490
        assert change.size == 10000


class TestLastTradePrice:
    """Test LastTradePrice struct and conversion."""

    def test_from_strings_valid(self) -> None:
        # Arrange
        price = "0.456"
        size = "219.217767"
        side = "BUY"

        # Act
        trade = LastTradePrice.from_strings(price=price, size=size, side=side)

        # Assert
        assert trade.price == 456
        assert trade.size == 21921
        assert trade.side == Side.BUY
```

#### 2. Expand parser tests with edge cases
**File**: `tests/test_parser.py`
**Changes**: Add edge case tests after existing test

```python
class TestParserEdgeCases:
    """Test parser behavior with malformed or edge case inputs."""

    def test_parse_empty_bids_asks(self) -> None:
        # Arrange
        parser = MessageParser()
        data = b'{"event_type":"book","asset_id":"123","market":"0xabc","bids":[],"asks":[],"timestamp":"1000","hash":"0x0"}'

        # Act
        messages = list(parser.parse_messages(data))

        # Assert
        assert len(messages) == 1
        assert messages[0].book is not None
        assert len(messages[0].book.bids) == 0
        assert len(messages[0].book.asks) == 0

    def test_parse_multiple_price_changes_empty(self) -> None:
        # Arrange
        parser = MessageParser()
        data = b'{"event_type":"price_change","market":"0xabc","price_changes":[],"timestamp":"1000"}'

        # Act
        messages = list(parser.parse_messages(data))

        # Assert
        assert len(messages) == 0

    def test_parse_unknown_event_type(self) -> None:
        # Arrange
        parser = MessageParser()
        data = b'{"event_type":"unknown_type","market":"0xabc","timestamp":"1000"}'

        # Act & Assert
        from src.exceptions import UnknownMessageType

        with pytest.raises(UnknownMessageType):
            list(parser.parse_messages(data))

    def test_parse_missing_market(self) -> None:
        # Arrange
        parser = MessageParser()
        data = b'{"event_type":"book","asset_id":"123","timestamp":"1000"}'

        # Act & Assert
        from src.exceptions import UnknownMarket

        with pytest.raises(UnknownMarket):
            list(parser.parse_messages(data))

    def test_parse_malformed_json(self) -> None:
        # Arrange
        parser = MessageParser()
        data = b'{"event_type":"book","invalid json'

        # Act & Assert
        with pytest.raises(Exception):
            list(parser.parse_messages(data))
```

### Success Criteria:

#### Automated Verification:
- [x] Protocol tests pass: `uv run pytest tests/test_protocol.py -v`
- [x] Expanded parser tests pass: `uv run pytest tests/test_parser.py -v`
- [x] All edge cases handled: `uv run pytest tests/test_parser.py::TestParserEdgeCases -v`

#### Manual Verification:
- [ ] Test output is clear and descriptive
- [ ] Edge cases cover common failure modes

**Implementation Note**: After completing this phase and all automated verification passes, confirm that protocol conversions and parser edge cases are properly tested before proceeding to Phase 3.

---

## Phase 3: Orderbook Tests

### Overview
Test orderbook state management including snapshot application, incremental updates, caching, and multi-book coordination.

### Changes Required:

#### 1. Test OrderbookState operations
**File**: `tests/test_orderbook.py` (new file)
**Changes**: Test snapshot, price changes, caching, and properties

```python
"""Tests for orderbook state management."""

import pytest

from src.messages.protocol import BookSnapshot, PriceChange, PriceLevel, Side
from src.orderbook.orderbook import OrderbookState


class TestOrderbookState:
    """Test individual orderbook state operations."""

    def test_apply_snapshot_initializes_book(
        self, sample_book_snapshot: BookSnapshot
    ) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")

        # Act
        book.apply_snapshot(sample_book_snapshot, timestamp=1000)

        # Assert
        assert book.best_bid == 490  # Highest bid
        assert book.best_ask == 510  # Lowest ask
        assert book.last_hash == "0xabc123"
        assert book.last_update_ts == 1000

    def test_apply_snapshot_clears_existing_state(
        self, sample_book_snapshot: BookSnapshot
    ) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        # Apply initial state
        initial_snapshot = BookSnapshot(
            hash="0xold",
            bids=(PriceLevel(price=100, size=1000),),
            asks=(PriceLevel(price=200, size=2000),),
        )
        book.apply_snapshot(initial_snapshot, timestamp=1000)

        # Act - apply new snapshot
        book.apply_snapshot(sample_book_snapshot, timestamp=2000)

        # Assert - old state completely replaced
        assert book.best_bid == 490
        assert book.best_ask == 510
        assert book.last_hash == "0xabc123"

    def test_apply_price_change_buy_adds_level(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        snapshot = BookSnapshot(
            hash="0x0", bids=(PriceLevel(price=490, size=1000),), asks=()
        )
        book.apply_snapshot(snapshot, timestamp=1000)

        change = PriceChange(
            asset_id="123",
            price=500,  # New higher bid
            size=2000,
            side=Side.BUY,
            hash="0x1",
            best_bid=500,
            best_ask=510,
        )

        # Act
        book.apply_price_change(change, timestamp=2000)

        # Assert
        assert book.best_bid == 500
        bids = book.get_bids(depth=5)
        assert len(bids) == 2
        assert bids[0].price == 500  # New level first

    def test_apply_price_change_removes_level_when_size_zero(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        snapshot = BookSnapshot(
            hash="0x0",
            bids=(PriceLevel(price=490, size=1000), PriceLevel(price=480, size=500)),
            asks=(),
        )
        book.apply_snapshot(snapshot, timestamp=1000)

        change = PriceChange(
            asset_id="123",
            price=490,
            size=0,  # Remove this level
            side=Side.BUY,
            hash="0x1",
            best_bid=480,
            best_ask=510,
        )

        # Act
        book.apply_price_change(change, timestamp=2000)

        # Assert
        assert book.best_bid == 480
        bids = book.get_bids(depth=5)
        assert len(bids) == 1

    def test_apply_price_change_sell_adds_level(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        snapshot = BookSnapshot(
            hash="0x0", bids=(), asks=(PriceLevel(price=510, size=1000),)
        )
        book.apply_snapshot(snapshot, timestamp=1000)

        change = PriceChange(
            asset_id="123",
            price=500,  # New lower ask
            size=2000,
            side=Side.SELL,
            hash="0x1",
            best_bid=490,
            best_ask=500,
        )

        # Act
        book.apply_price_change(change, timestamp=2000)

        # Assert
        assert book.best_ask == 500
        asks = book.get_asks(depth=5)
        assert len(asks) == 2
        assert asks[0].price == 500  # New level first (lowest)

    def test_best_bid_ask_cached_after_price_change(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        snapshot = BookSnapshot(
            hash="0x0",
            bids=(PriceLevel(price=490, size=1000),),
            asks=(PriceLevel(price=510, size=1000),),
        )
        book.apply_snapshot(snapshot, timestamp=1000)

        change = PriceChange(
            asset_id="123",
            price=495,
            size=500,
            side=Side.BUY,
            hash="0x1",
            best_bid=495,
            best_ask=510,
        )

        # Act
        book.apply_price_change(change, timestamp=2000)

        # Assert - cache should be valid
        assert book._cache_valid is True
        assert book.best_bid == 495
        assert book.best_ask == 510

    def test_spread_calculation(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        snapshot = BookSnapshot(
            hash="0x0",
            bids=(PriceLevel(price=490, size=1000),),
            asks=(PriceLevel(price=510, size=1000),),
        )
        book.apply_snapshot(snapshot, timestamp=1000)

        # Act
        spread = book.spread

        # Assert
        assert spread == 20  # 510 - 490

    def test_mid_price_calculation(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        snapshot = BookSnapshot(
            hash="0x0",
            bids=(PriceLevel(price=490, size=1000),),
            asks=(PriceLevel(price=510, size=1000),),
        )
        book.apply_snapshot(snapshot, timestamp=1000)

        # Act
        mid = book.mid_price

        # Assert
        assert mid == 500  # (490 + 510) // 2

    def test_get_bids_respects_depth(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        bids = tuple(PriceLevel(price=500 - i * 10, size=1000) for i in range(5))
        snapshot = BookSnapshot(hash="0x0", bids=bids, asks=())
        book.apply_snapshot(snapshot, timestamp=1000)

        # Act
        top_2 = book.get_bids(depth=2)

        # Assert
        assert len(top_2) == 2
        assert top_2[0].price == 500  # Highest
        assert top_2[1].price == 490

    def test_get_asks_respects_depth(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")
        asks = tuple(PriceLevel(price=500 + i * 10, size=1000) for i in range(5))
        snapshot = BookSnapshot(hash="0x0", bids=(), asks=asks)
        book.apply_snapshot(snapshot, timestamp=1000)

        # Act
        top_2 = book.get_asks(depth=2)

        # Assert
        assert len(top_2) == 2
        assert top_2[0].price == 500  # Lowest
        assert top_2[1].price == 510

    def test_empty_book_properties(self) -> None:
        # Arrange
        book = OrderbookState(asset_id="123", market="0xabc")

        # Act & Assert
        assert book.best_bid is None
        assert book.best_ask is None
        assert book.spread is None
        assert book.mid_price is None


class TestOrderbookStore:
    """Test multi-book registry management."""

    def test_register_asset_creates_new_state(self) -> None:
        # Arrange
        from src.orderbook.orderbook_store import Asset, OrderbookStore

        store = OrderbookStore()
        asset = Asset(asset_id="123", market="0xabc")

        # Act
        state = store.register_asset(asset)

        # Assert
        assert state is not None
        assert state.asset_id == "123"
        assert state.market == "0xabc"

    def test_register_asset_returns_existing_state(self) -> None:
        # Arrange
        from src.orderbook.orderbook_store import Asset, OrderbookStore

        store = OrderbookStore()
        asset = Asset(asset_id="123", market="0xabc")
        first_state = store.register_asset(asset)

        # Act
        second_state = store.register_asset(asset)

        # Assert
        assert second_state is first_state  # Same object

    def test_get_state_returns_none_for_unknown_asset(self) -> None:
        # Arrange
        from src.orderbook.orderbook_store import OrderbookStore

        store = OrderbookStore()

        # Act
        state = store.get_state("nonexistent")

        # Assert
        assert state is None

    def test_remove_state_removes_from_books(self) -> None:
        # Arrange
        from src.orderbook.orderbook_store import Asset, OrderbookStore

        store = OrderbookStore()
        asset = Asset(asset_id="123", market="0xabc")
        store.register_asset(asset)

        # Act
        removed = store.remove_state("123")

        # Assert
        assert removed is True
        assert store.get_state("123") is None

    def test_remove_state_updates_market_index(self) -> None:
        # Arrange
        from src.orderbook.orderbook_store import Asset, OrderbookStore

        store = OrderbookStore()
        asset1 = Asset(asset_id="123", market="0xabc")
        asset2 = Asset(asset_id="456", market="0xabc")
        store.register_asset(asset1)
        store.register_asset(asset2)

        # Act
        store.remove_state("123")

        # Assert - market index should still have asset2
        assert store.get_state("456") is not None
```

### Success Criteria:

#### Automated Verification:
- [x] OrderbookState tests pass: `uv run pytest tests/test_orderbook.py::TestOrderbookState -v`
- [x] OrderbookStore tests pass: `uv run pytest tests/test_orderbook.py::TestOrderbookStore -v`
- [x] All orderbook tests pass: `uv run pytest tests/test_orderbook.py -v`

#### Manual Verification:
- [ ] Test descriptions clearly indicate what's being tested
- [ ] Edge cases for empty books are covered

**Implementation Note**: After completing this phase and all automated verification passes, confirm that orderbook operations are properly tested before proceeding to Phase 4.

---

## Phase 4: Registry Tests

### Overview
Test asset registry operations including async mutations, multi-index lookups, and connection lifecycle management.

### Changes Required:

#### 1. Test AssetEntry and AssetRegistry
**File**: `tests/test_registry.py` (new file)
**Changes**: Test entry properties and async registry operations

```python
"""Tests for asset registry and entry management."""

import pytest

from src.registry.asset_entry import AssetEntry, AssetStatus
from src.registry.asset_registry import AssetRegistry


class TestAssetEntry:
    """Test AssetEntry properties and status."""

    def test_is_active_when_subscribed(self) -> None:
        # Arrange
        entry = AssetEntry(
            asset_id="123",
            condition_id="condition_1",
            status=AssetStatus.SUBSCRIBED,
        )

        # Act & Assert
        assert entry.is_active is True

    def test_is_not_active_when_pending(self) -> None:
        # Arrange
        entry = AssetEntry(
            asset_id="123",
            condition_id="condition_1",
            status=AssetStatus.PENDING,
        )

        # Act & Assert
        assert entry.is_active is False

    def test_is_not_active_when_expired(self) -> None:
        # Arrange
        entry = AssetEntry(
            asset_id="123",
            condition_id="condition_1",
            status=AssetStatus.EXPIRED,
        )

        # Act & Assert
        assert entry.is_active is False


class TestAssetRegistry:
    """Test async asset registry operations."""

    @pytest.fixture
    def registry(self) -> AssetRegistry:
        return AssetRegistry()

    async def test_add_asset_creates_entry(self, registry: AssetRegistry) -> None:
        # Arrange
        asset_id = "123"
        condition_id = "condition_1"

        # Act
        added = await registry.add_asset(asset_id, condition_id)

        # Assert
        assert added is True
        entry = registry.get(asset_id)
        assert entry is not None
        assert entry.asset_id == asset_id
        assert entry.condition_id == condition_id
        assert entry.status == AssetStatus.PENDING

    async def test_add_asset_duplicate_returns_false(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        asset_id = "123"
        condition_id = "condition_1"
        await registry.add_asset(asset_id, condition_id)

        # Act
        added_again = await registry.add_asset(asset_id, condition_id)

        # Assert
        assert added_again is False

    async def test_mark_subscribed_updates_status(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        asset_id = "123"
        await registry.add_asset(asset_id, "condition_1")
        connection_id = "conn_1"

        # Act
        count = await registry.mark_subscribed([asset_id], connection_id)

        # Assert
        assert count == 1
        entry = registry.get(asset_id)
        assert entry is not None
        assert entry.status == AssetStatus.SUBSCRIBED
        assert entry.connection_id == connection_id

    async def test_mark_expired_updates_status(self, registry: AssetRegistry) -> None:
        # Arrange
        asset_id = "123"
        await registry.add_asset(asset_id, "condition_1")
        await registry.mark_subscribed([asset_id], "conn_1")

        # Act
        count = await registry.mark_expired([asset_id])

        # Assert
        assert count == 1
        entry = registry.get(asset_id)
        assert entry is not None
        assert entry.status == AssetStatus.EXPIRED

    async def test_get_by_condition_returns_assets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        condition_id = "condition_1"
        await registry.add_asset("123", condition_id)
        await registry.add_asset("456", condition_id)
        await registry.add_asset("789", "condition_2")  # Different condition

        # Act
        assets = registry.get_by_condition(condition_id)

        # Assert
        assert len(assets) == 2
        assert "123" in assets
        assert "456" in assets
        assert "789" not in assets

    async def test_get_by_connection_returns_assets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        await registry.add_asset("123", "condition_1")
        await registry.add_asset("456", "condition_1")
        await registry.mark_subscribed(["123", "456"], "conn_1")

        # Act
        assets = registry.get_by_connection("conn_1")

        # Assert
        assert len(assets) == 2
        assert "123" in assets
        assert "456" in assets

    async def test_take_pending_batch_respects_max_size(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        for i in range(10):
            await registry.add_asset(f"asset_{i}", "condition_1")

        # Act
        batch = await registry.take_pending_batch(max_size=5)

        # Assert
        assert len(batch) == 5
        # Next batch should have remaining 5
        batch2 = await registry.take_pending_batch(max_size=5)
        assert len(batch2) == 5

    async def test_reassign_connection_moves_assets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        await registry.add_asset("123", "condition_1")
        await registry.mark_subscribed(["123"], "conn_1")

        # Act
        count = await registry.reassign_connection(["123"], "conn_1", "conn_2")

        # Assert
        assert count == 1
        entry = registry.get("123")
        assert entry is not None
        assert entry.connection_id == "conn_2"
        # Check indices updated
        assert "123" in registry.get_by_connection("conn_2")
        assert "123" not in registry.get_by_connection("conn_1")

    async def test_remove_connection_orphans_assets(
        self, registry: AssetRegistry
    ) -> None:
        # Arrange
        await registry.add_asset("123", "condition_1")
        await registry.mark_subscribed(["123"], "conn_1")

        # Act
        assets = await registry.remove_connection("conn_1")

        # Assert
        assert "123" in assets
        entry = registry.get("123")
        assert entry is not None
        assert entry.connection_id is None  # Orphaned

    async def test_remove_asset_deletes_fully(self, registry: AssetRegistry) -> None:
        # Arrange
        await registry.add_asset("123", "condition_1")
        await registry.mark_subscribed(["123"], "conn_1")

        # Act
        removed = await registry.remove_asset("123")

        # Assert
        assert removed is True
        assert registry.get("123") is None
        assert "123" not in registry.get_by_condition("condition_1")
        assert "123" not in registry.get_by_connection("conn_1")
```

### Success Criteria:

#### Automated Verification:
- [x] AssetEntry tests pass: `uv run pytest tests/test_registry.py::TestAssetEntry -v`
- [x] AssetRegistry async tests pass: `uv run pytest tests/test_registry.py::TestAssetRegistry -v`
- [x] All registry tests pass: `uv run pytest tests/test_registry.py -v`

#### Manual Verification:
- [ ] Async operations complete without warnings
- [ ] Multi-index consistency is maintained

**Implementation Note**: After completing this phase and all automated verification passes, confirm that registry operations are properly tested before proceeding to Phase 5.

---

## Phase 5: Connection Stats and WebSocket Tests

### Overview
Test connection statistics calculations and WebSocket connection lifecycle with mocked network I/O.

### Changes Required:

#### 1. Test ConnectionStats properties
**File**: `tests/test_connection.py` (new file)
**Changes**: Test stats calculations and WebSocket mocking

```python
"""Tests for connection statistics and WebSocket handling."""

import time
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from src.connection.stats import ConnectionStats


class TestConnectionStats:
    """Test connection statistics calculations."""

    def test_uptime_calculation(self) -> None:
        # Arrange
        stats = ConnectionStats()
        initial_time = stats.created_at

        # Act - simulate time passing
        with patch("time.monotonic", return_value=initial_time + 10.0):
            uptime = stats.uptime

        # Assert
        assert uptime == 10.0

    def test_message_rate_with_messages(self) -> None:
        # Arrange
        stats = ConnectionStats()
        stats.messages_received = 100
        initial_time = stats.created_at

        # Act
        with patch("time.monotonic", return_value=initial_time + 10.0):
            rate = stats.message_rate

        # Assert
        assert rate == 10.0  # 100 messages / 10 seconds

    def test_message_rate_zero_when_no_time_elapsed(self) -> None:
        # Arrange
        stats = ConnectionStats()
        stats.messages_received = 100

        # Act - immediately after creation
        with patch("time.monotonic", return_value=stats.created_at):
            rate = stats.message_rate

        # Assert
        assert rate == 0.0

    def test_message_rate_zero_when_no_messages(self) -> None:
        # Arrange
        stats = ConnectionStats()
        initial_time = stats.created_at

        # Act
        with patch("time.monotonic", return_value=initial_time + 10.0):
            rate = stats.message_rate

        # Assert
        assert rate == 0.0


class TestWebsocketConnection:
    """Test WebSocket connection with mocked I/O."""

    @pytest.fixture
    def mock_websocket(self):
        """Mock websocket connection."""
        ws = AsyncMock()
        ws.recv = AsyncMock()
        ws.send = AsyncMock()
        ws.close = AsyncMock()
        return ws

    async def test_connect_initializes_connection(self, mock_websocket) -> None:
        # Arrange
        from src.connection.websocket import WebsocketConnection

        with patch("websockets.connect", return_value=mock_websocket):
            conn = WebsocketConnection(
                url="wss://example.com", asset_ids=["123", "456"]
            )

            # Act
            await conn.connect()

            # Assert
            assert conn._ws is not None
            assert conn.stats.reconnect_count == 0

    async def test_subscribe_sends_message(self, mock_websocket) -> None:
        # Arrange
        from src.connection.websocket import WebsocketConnection

        with patch("websockets.connect", return_value=mock_websocket):
            conn = WebsocketConnection(
                url="wss://example.com", asset_ids=["123", "456"]
            )
            await conn.connect()

            # Act
            await conn.subscribe()

            # Assert
            mock_websocket.send.assert_called_once()
            # Verify subscribe message format
            call_args = mock_websocket.send.call_args[0][0]
            assert "subscribe" in call_args or b"subscribe" in call_args

    async def test_receive_updates_stats(self, mock_websocket) -> None:
        # Arrange
        from src.connection.websocket import WebsocketConnection

        test_data = b'{"event_type":"book","market":"0xabc","asset_id":"123","bids":[],"asks":[],"timestamp":"1000","hash":"0x0"}'
        mock_websocket.recv.return_value = test_data

        with patch("websockets.connect", return_value=mock_websocket):
            conn = WebsocketConnection(url="wss://example.com", asset_ids=["123"])
            await conn.connect()

            # Act
            data = await conn.receive()

            # Assert
            assert data == test_data
            assert conn.stats.messages_received == 1
            assert conn.stats.bytes_received == len(test_data)

    async def test_disconnect_closes_connection(self, mock_websocket) -> None:
        # Arrange
        from src.connection.websocket import WebsocketConnection

        with patch("websockets.connect", return_value=mock_websocket):
            conn = WebsocketConnection(url="wss://example.com", asset_ids=["123"])
            await conn.connect()

            # Act
            await conn.disconnect()

            # Assert
            mock_websocket.close.assert_called_once()
```

### Success Criteria:

#### Automated Verification:
- [x] ConnectionStats tests pass: `uv run pytest tests/test_connection.py::TestConnectionStats -v`
- [x] WebSocket mocking tests pass: `uv run pytest tests/test_connection.py::TestWebsocketConnection -v`
- [x] All connection tests pass: `uv run pytest tests/test_connection.py -v`
- [x] Full test suite passes: `uv run pytest -v`
- [x] Tests run quickly (no actual network I/O): `uv run pytest --durations=10`

#### Manual Verification:
- [ ] Mock assertions are clear and verify behavior
- [ ] No actual WebSocket connections are made during tests

**Implementation Note**: After completing this phase and all automated verification passes, confirm that all tests in the suite pass and the codebase has comprehensive test coverage.

---

## Testing Strategy

### Unit Tests
Each component is tested in isolation:
- **Protocol**: String conversions and struct creation
- **Parser**: Message decoding and edge cases (malformed, missing fields)
- **Orderbook**: State mutations, caching, property calculations
- **Registry**: Async operations, multi-index consistency
- **Connection**: Stats calculations, mocked WebSocket I/O

### Edge Cases Covered
- Empty collections (bids/asks, price_changes)
- Unknown event types
- Missing required fields
- Malformed JSON
- Zero values (size=0 removes level)
- Cache invalidation and recomputation
- Concurrent async operations (implicit via asyncio)

### Test Organization
```
tests/
├── conftest.py           # Shared fixtures
├── test_parser.py        # Parser tests (existing + expanded)
├── test_protocol.py      # Protocol conversion tests
├── test_orderbook.py     # Orderbook state and store tests
├── test_registry.py      # Registry and entry tests
└── test_connection.py    # Stats and WebSocket tests
```

### Running Tests
```bash
# All tests
uv run pytest

# Specific module
uv run pytest tests/test_orderbook.py

# Specific test class
uv run pytest tests/test_orderbook.py::TestOrderbookState

# Specific test
uv run pytest tests/test_parser.py::test_parse_messages

# Verbose output
uv run pytest -v

# Show test durations
uv run pytest --durations=10
```

## Performance Considerations

- Fixtures use `pytest.fixture` scope="function" (default) for test isolation
- Async tests use `pytest-asyncio` with auto mode (configured in pyproject.toml)
- Mocked WebSocket avoids slow network I/O in tests
- No actual file I/O or database operations in unit tests
- Test suite should complete in <5 seconds

## References

- Existing test: `tests/test_parser.py`
- Parser implementation: `src/messages/parser.py`
- Protocol definitions: `src/messages/protocol.py`
- Orderbook state: `src/orderbook/orderbook.py`
- Orderbook store: `src/orderbook/orderbook_store.py`
- Asset registry: `src/registry/asset_registry.py`
- Connection stats: `src/connection/stats.py`
- WebSocket connection: `src/connection/websocket.py`
