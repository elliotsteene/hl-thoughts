---
date: 2025-12-03 20:31:10 GMT
researcher: Elliot Steene
git_commit: 1759ee69c612a851d7b5e2307740cfc73feea17c
branch: main
repository: polypy
topic: "How is orderbook state maintained? What are the optimisations used to reduce time complexity?"
tags: [research, codebase, orderbook, state-management, performance, sorteddict, time-complexity]
status: complete
last_updated: 2025-12-03
last_updated_by: Elliot Steene
---

# Research: Orderbook State Management and Time Complexity Optimizations

**Date**: 2025-12-03 20:31:10 GMT
**Researcher**: Elliot Steene
**Git Commit**: 1759ee69c612a851d7b5e2307740cfc73feea17c
**Branch**: main
**Repository**: polypy

## Research Question

How is orderbook state maintained? What are the optimisations used to reduce time complexity?

## Summary

Orderbook state is maintained using `SortedDict` from the `sortedcontainers` library with a clever **key negation pattern** for bid ordering. The system uses two separate `SortedDict` instances - one for bids (with negated prices for descending sort) and one for asks (with natural prices for ascending sort). Key optimizations include:

1. **O(log n) inserts/deletes and O(1) lookups** via SortedDict's hybrid balanced tree + hash table structure
2. **Best bid/ask caching** to avoid repeated lookups during spread/mid-price calculations
3. **Memory-efficient slots** on all data classes to reduce per-instance overhead
4. **Integer price representation** (scaled by 1000) to avoid floating-point precision issues
5. **Zero-copy parsing** with msgspec.Struct for protocol messages
6. **Direct registry lookup** via `dict[str, OrderbookState]` for O(1) asset_id → state mapping

The system supports full book snapshots (complete replacement) and incremental price changes (single level updates) with consistent O(log n) or better performance.

## Detailed Findings

### OrderbookState: Core State Container

**Location**: `src/orderbook/orderbook.py:9-171`

OrderbookState maintains a single orderbook using two `SortedDict` instances for bids and asks.

#### Data Structure: SortedDict with Key Negation

**Bids - Descending Order** (`orderbook.py:18-21, 46, 64`):
```python
# Bids: highest price first (negative keys for reverse sort)
# Key: -price (negated for descending order)
# Value: size at that price level
_bids: SortedDict = field(default_factory=SortedDict)

# Example usage:
self._bids[-level.price] = level.size  # Line 46 - negate for desc sort
key = -price_change.price              # Line 64 - negated for bid ordering
```

**Why negation works**:
- SortedDict naturally sorts keys in ascending order
- By negating prices, higher prices become more negative
- Example: prices [100, 99, 98] become [-100, -99, -98]
- When sorted ascending: [-100, -99, -98] represents prices [100, 99, 98] descending
- First element (`keys()[0]`) is the best (highest) bid

**Asks - Ascending Order** (`orderbook.py:23-26, 50, 71`):
```python
# Asks: lowest price first (natural ascending order)
# Key: price
# Value: size at that price level
_asks: SortedDict = field(default_factory=SortedDict)

# Example usage:
self._asks[level.price] = level.size  # Line 50 - natural key
key = price_change.price              # Line 71 - natural key
```

**Why natural keys work**:
- SortedDict's ascending sort matches desired ask order
- Lower prices naturally appear first
- First element is the best (lowest) ask

#### Update Operations

**Full Snapshot Replacement** (`orderbook.py:37-55`):
```python
def apply_snapshot(self, snapshot: BookSnapshot, timestamp: int) -> None:
    self._bids.clear()
    self._asks.clear()

    for level in snapshot.bids:
        if level.size > 0:
            self._bids[-level.price] = level.size

    for level in snapshot.asks:
        if level.size > 0:
            self._asks[level.price] = level.size

    self.last_hash = snapshot.hash
    self.last_update_ts = timestamp
    self.local_update_ts = time.monotonic()
    self._invalidate_cache()
```

**Behavior**:
- Completely replaces existing state (lines 41-42)
- Filters out zero-size levels during snapshot
- Updates both exchange timestamp and local monotonic time
- Invalidates cached best bid/ask values

**Incremental Price Change** (`orderbook.py:57-84`):
```python
def apply_price_change(self, price_change: PriceChange, timestamp: int) -> None:
    if price_change.side == Side.BUY:
        key = -price_change.price  # Negated for bid ordering
        self._update_price_level(key, price_change.size, self._bids)
    else:
        key = price_change.price
        self._update_price_level(key, price_change.size, self._asks)

    self._set_best_bid(price_change.best_bid)
    self._set_best_ask(price_change.best_ask)
    self._cache_valid = True
```

**Core update logic** (`orderbook.py:86-91`):
```python
@staticmethod
def _update_price_level(key: int, size: int, price_map: SortedDict) -> None:
    if size == 0:
        price_map.pop(key, None)  # Remove level
    else:
        price_map[key] = size     # Insert or update
```

**Behavior**:
- Size=0 removes the price level entirely
- Size>0 inserts new or updates existing level
- Best bid/ask received from message (not recomputed)
- Cache marked valid after update

#### Caching Mechanism

**Cache Fields** (`orderbook.py:32-35`):
```python
_cached_best_bid: int | None = None
_cached_best_ask: int | None = None
_cache_valid: bool = False
```

**Cache Invalidation** (`orderbook.py:99-102`):
```python
def _invalidate_cache(self) -> None:
    self._cache_valid = False
    self._cached_best_bid = None
    self._cached_best_ask = None
```
Called when `apply_snapshot()` rebuilds the book.

**Cache Population from Message** (`orderbook.py:78-80`):
```python
self._set_best_bid(price_change.best_bid)
self._set_best_ask(price_change.best_ask)
self._cache_valid = True
```
Called during `apply_price_change()` when message contains authoritative best prices.

**Cache Recomputation** (`orderbook.py:152-160`):
```python
def _recompute_cache(self) -> None:
    # First key is most negative = highest price
    best_bid: int | None = -self._bids.keys()[0] if self._bids else None
    self._set_best_bid(best_bid)

    best_ask: int | None = self._asks.keys()[0] if self._asks else None
    self._set_best_ask(best_ask)

    self._cache_valid = True
```

**Cache Access** (`orderbook.py:104-114`):
```python
@property
def best_bid(self) -> int | None:
    if not self._cache_valid:
        self._recompute_cache  # Note: method not invoked (missing parentheses)
    return self._cached_best_bid
```

**Derived Properties** (`orderbook.py:116-130`):
```python
@property
def spread(self) -> int | None:
    bid, ask = self.best_bid, self.best_ask
    if bid is not None and ask is not None:
        return ask - bid
    return None

@property
def mid_price(self) -> int | None:
    bid, ask = self.best_bid, self.best_ask
    if bid is not None and ask is not None:
        return (bid + ask) // 2
    return None
```

Both rely on cached best bid/ask for efficient computation.

#### Depth Queries

**Get Top N Levels** (`orderbook.py:132-150`):
```python
def get_bids(self, depth: int = 10) -> list[PriceLevel]:
    if depth == 0:
        return []

    result = []
    for neg_price, size in self._bids.items()[:depth]:
        result.append(PriceLevel(price=-neg_price, size=size))

    return result

def get_asks(self, depth: int = 10) -> list[PriceLevel]:
    if depth == 0:
        return []

    result = []
    for price, size in self._asks.items()[:depth]:
        result.append(PriceLevel(price=price, size=size))

    return result
```

**Behavior**:
- Default depth of 10 levels
- SortedDict slicing provides efficient top-k access
- Bid prices un-negated when creating PriceLevel
- Ask prices used as-is

### OrderbookStore: Multi-Orderbook Registry

**Location**: `src/orderbook/orderbook_store.py:11-68`

OrderbookStore manages multiple OrderbookState instances with efficient lookup.

#### Primary Registry

**Data Structure** (`orderbook_store.py:21`):
```python
self._books: dict[str, OrderbookState] = {}  # asset_id -> state
```

**Lookup** (`orderbook_store.py:34-35`):
```python
def get_state(self, asset_id: str) -> OrderbookState | None:
    return self._books.get(asset_id)
```

**Time Complexity**: O(1) - direct dictionary lookup

**Usage**: In `main.py:34`, the store is queried with `self.store.get_state(message.asset_id)` to route messages to the correct orderbook.

#### Market Grouping Index

**Data Structure** (`orderbook_store.py:22`):
```python
self._by_market: dict[str, list[str]] = {}  # market -> [asset_ids]
```

**Population** (`orderbook_store.py:58-64`):
```python
def _add_market(self, asset: Asset) -> None:
    if asset.market not in self._by_market:
        self._by_market[asset.market] = []

    curr_assets = set(self._by_market[asset.market])
    if asset.asset_id not in curr_assets:
        self._by_market[asset.market].append(asset.asset_id)
```

**Time Complexity**: O(n) where n = number of assets already in that market (due to set conversion for duplicate check)

#### Asset Registration

**Get-or-Create Pattern** (`orderbook_store.py:24-32`):
```python
def register_asset(self, asset: Asset) -> OrderbookState:
    orderbook_state = self.get_state(asset.asset_id)
    if not orderbook_state:
        orderbook_state = self._add_state(asset=asset)

    return orderbook_state
```

**Usage** (`main.py:21-26`):
```python
class App:
    def __init__(self, asset_ids: list[str]) -> None:
        self.store: OrderbookStore = OrderbookStore()

        for id in asset_ids:
            asset = Asset(id, "16r9tvhjks98r")
            self.store.register_asset(asset)
```

**Behavior**:
- Idempotent - returns existing state if already registered
- Creates new state if not found
- Automatically updates both indices

#### State Creation

**Internal Method** (`orderbook_store.py:47-56`):
```python
def _add_state(self, asset: Asset) -> OrderbookState:
    state = OrderbookState(
        asset_id=asset.asset_id,
        market=asset.market,
    )
    self._books[asset.asset_id] = state

    self._add_market(asset)

    return state
```

**Behavior**:
- Creates OrderbookState with asset_id and market
- Adds to primary `_books` dictionary
- Updates market index via `_add_market()`

#### Asset Removal

**Remove Pattern** (`orderbook_store.py:37-45`):
```python
def remove_state(self, asset_id: str) -> bool:
    book = self._books.pop(asset_id, None)

    if book and book.market:
        for i, m_asset_id in enumerate(self._by_market.get(book.market, [])):
            if asset_id == m_asset_id:
                self._by_market[book.market].pop(i)

    return book is not None
```

**Time Complexity**: O(1) for primary removal + O(n) for market index removal = O(n) where n = assets in that market

#### Memory Tracking

**Aggregation** (`orderbook_store.py:66-67`):
```python
def memory_usage(self) -> int:
    return sum(book.__sizeof__() for book in self._books.values())
```

**Per-Orderbook Calculation** (`orderbook.py:162-170`):
```python
def __sizeof__(self) -> int:
    base = object.__sizeof__(self) + 8 * 10

    bids_size = 64 + len(self._bids) * 16
    ask_size = 64 + len(self._asks) * 16
    str_size = len(self.asset_id) + len(self.market) + len(self.last_hash)

    return base + bids_size + ask_size + str_size
```

**Usage** (`main.py:104`):
```python
logger.info(f"Memory usage: {app.store.memory_usage()}")
```

**Time Complexity**: O(m) where m = total number of orderbooks

## Time Complexity Analysis

### OrderbookState Operations

| Operation | Complexity | Implementation |
|-----------|------------|----------------|
| **Insert level** | O(log n) | SortedDict binary search insert (`orderbook.py:91`) |
| **Update level** | O(log n) | SortedDict key lookup + rebalance (`orderbook.py:91`) |
| **Delete level** | O(log n) | SortedDict key removal + rebalance (`orderbook.py:89`) |
| **Lookup by price** | O(1) | Dictionary semantics of SortedDict |
| **Best bid/ask (cached)** | O(1) | Cached value access (`orderbook.py:105-114`) |
| **Best bid/ask (uncached)** | O(1) | First element access via `keys()[0]` (`orderbook.py:154, 157`) |
| **Spread** | O(1) | Relies on cached best bid/ask (`orderbook.py:117-122`) |
| **Mid-price** | O(1) | Relies on cached best bid/ask (`orderbook.py:124-130`) |
| **Depth query (top k)** | O(k) | SortedDict slicing (`orderbook.py:137, 147`) |
| **Full snapshot** | O(n log n) | n inserts at O(log n) each (`orderbook.py:44-50`) |

*where n = number of price levels, k = requested depth*

### OrderbookStore Operations

| Operation | Complexity | Implementation |
|-----------|------------|----------------|
| **Get state by asset_id** | O(1) | Direct dict lookup (`orderbook_store.py:35`) |
| **Register asset** | O(n)* | O(1) lookup + O(n) market indexing (`orderbook_store.py:24-32`) |
| **Remove asset** | O(n) | O(1) dict removal + O(n) market index scan (`orderbook_store.py:37-45`) |
| **Memory usage** | O(m) | Iterate all orderbooks (`orderbook_store.py:66-67`) |

*where n = assets in same market, m = total orderbooks*

## Code References

### Core Implementation
- `src/orderbook/orderbook.py:9-171` - OrderbookState class with SortedDict-based state management
- `src/orderbook/orderbook.py:18-26` - Key negation pattern for bid/ask ordering
- `src/orderbook/orderbook.py:37-55` - Full snapshot replacement logic
- `src/orderbook/orderbook.py:57-84` - Incremental price change updates
- `src/orderbook/orderbook.py:86-91` - Core update helper with size=0 deletion
- `src/orderbook/orderbook.py:99-160` - Caching mechanism for best bid/ask
- `src/orderbook/orderbook_store.py:11-68` - Multi-orderbook registry

### Integration Points
- `src/main.py:21-26` - Asset registration at startup
- `src/main.py:28-57` - Message routing and state updates
- `src/main.py:34` - Asset ID lookup: `self.store.get_state(message.asset_id)`
- `src/main.py:40-43` - Conditional dispatch: snapshot vs price change
- `src/main.py:104` - Memory usage tracking at shutdown

### Protocol & Parsing
- `src/messages/protocol.py:18-19` - Price/size scaling constants (1000x, 100x)
- `src/messages/protocol.py:65-83` - PriceLevel msgspec.Struct definition
- `src/messages/protocol.py:85-94` - BookSnapshot with frozen tuples
- `src/messages/protocol.py:101-132` - PriceChange with best bid/ask
- `src/messages/parser.py:92-117` - BookSnapshot parsing
- `src/messages/parser.py:119-145` - PriceChange parsing (can contain multiple changes)

### WebSocket Integration
- `src/connection/websocket.py:200-226` - Raw message reception and parsing
- `src/connection/websocket.py:219-220` - Callback invocation per parsed message

## Architecture Documentation

### Key Design Patterns

#### 1. Key Negation for Descending Sort
Instead of using custom comparators or reverse sorting, bids use negated prices as keys in SortedDict. This leverages the library's natural ascending sort to achieve descending price order.

**Trade-off**: Requires un-negating when reading (line 138: `price=-neg_price`), but avoids performance overhead of custom comparison functions.

#### 2. Best Bid/Ask Caching Strategy
Two cache population paths:
- **From exchange messages**: Price change messages include authoritative best bid/ask (lines 78-80)
- **Local recomputation**: Snapshot updates require recomputing from SortedDict first elements (lines 152-160)

**Trade-off**: Relies on exchange providing accurate best prices in updates. Cache invalidation on snapshots ensures consistency but requires O(1) recomputation.

#### 3. Hybrid Message Processing
Single `ParsedMessage` container with tagged union pattern:
```python
ParsedMessage(
    event_type: EventType,
    asset_id: str,
    market: str,
    raw_timestamp: int,
    book: BookSnapshot | None = None,
    price_change: PriceChange | None = None,
    last_trade: LastTradePrice | None = None
)
```
Only one of `book`, `price_change`, or `last_trade` is populated based on `event_type`.

**Trade-off**: Single type simplifies callback signatures, but unused fields are always None.

#### 4. Integer Price Representation
All prices scaled by 1000, sizes by 100 (`protocol.py:18-19`):
```python
PRICE_SCALE: Final[int] = 1000
SIZE_SCALE: Final[int] = 100
```

**Benefits**:
- Avoids floating-point precision errors
- Faster integer arithmetic
- Smaller memory footprint

**Trade-off**: Must scale back for display, but exchange provides string prices anyway.

#### 5. Generator-Based Parsing
Parser yields messages instead of returning lists:
```python
def parse_messages(self, data: bytes) -> Generator[ParsedMessage]:
    # ...
    yield ParsedMessage(...)
```

**Benefits**:
- Single WebSocket message can contain multiple price changes
- Memory-efficient streaming processing
- Callback invoked per parsed message

**Current usage**: Immediately materialized in `main.py:28-57` loop, so streaming benefits not fully utilized yet.

#### 6. Memory Optimization via Slots
All major classes use `__slots__` or `slots=True`:
- `OrderbookState`: `@dataclass(slots=True)` (`orderbook.py:9`)
- `OrderbookStore`: `__slots__ = ("_books", "_by_market")` (`orderbook_store.py:18`)
- `WebsocketConnection`: `__slots__ = (...)` (`websocket.py:35-47`)
- Protocol messages: `msgspec.Struct` with `frozen=True` (`protocol.py:65-70`)

**Benefits**:
- Reduces per-instance memory overhead (no `__dict__`)
- Prevents accidental attribute creation
- Slight performance improvement for attribute access

#### 7. Dual Index Registry
OrderbookStore maintains two parallel data structures:
- **Primary**: `asset_id` → `OrderbookState` for O(1) message routing
- **Secondary**: `market` → `[asset_ids]` for market-level operations

**Current usage**: Market index populated but not actively used in `main.py`. Likely intended for future phases (multi-market queries, cross-asset analytics).

### Data Flow Summary

```
WebSocket bytes
  → MessageParser.parse_messages()
  → Generator[ParsedMessage]
  → App.message_callback(connection_id, message)
  → OrderbookStore.get_state(asset_id)
  → OrderbookState.apply_snapshot() OR apply_price_change()
  → SortedDict insert/update/delete
```

### Optimization Highlights

1. **SortedDict Choice**: Provides O(log n) inserts with O(1) lookups, avoiding O(n) list operations or O(log n) heap operations for all accesses
2. **Key Negation Trick**: Eliminates need for custom comparators or reverse iteration
3. **Best Bid/Ask Caching**: Prevents repeated O(1) lookups during spread/mid-price calculations
4. **Integer Scaling**: Avoids floating-point arithmetic overhead and precision issues
5. **msgspec for Parsing**: Zero-copy deserialization significantly faster than standard JSON
6. **Frozen Immutable Structs**: Enables hashing and prevents accidental mutation
7. **Direct Asset Lookup**: O(1) dictionary access for message routing instead of linear search
8. **Memory Tracking**: Custom `__sizeof__` enables accurate memory profiling

## Historical Context (from thoughts/)

**No historical documents found.**

The thoughts/ directory is properly structured but currently contains no documents related to orderbook implementation, state management, performance optimizations, or data structure choices. This appears to be a new project without prior design documentation captured in the thoughts system.

## Related Research

**No prior research documents found.**

This is the first research document created for the orderbook implementation in `thoughts/shared/research/`.

## Open Questions

1. **Cache Invocation Bug**: Lines `orderbook.py:107` and `orderbook.py:113` reference `self._recompute_cache` without parentheses, meaning the method is never actually invoked. This appears to be a bug since the cache validation check precedes the access but recomputation never happens. However, the cache is populated via `apply_price_change()` receiving best prices from messages, so this may not affect current functionality if all updates come through price change messages (not just snapshots).

2. **Market Index Usage**: The `_by_market` index in OrderbookStore is populated but not currently used in `main.py` or `connection/websocket.py`. Likely intended for future phases of the learning project where multi-market queries or cross-asset analytics are needed.

3. **Generator Materialization**: MessageParser returns a generator, but it's immediately consumed in a list comprehension in the main callback loop. The streaming benefits are not currently utilized, but this pattern is likely intentional for future optimization when processing high-throughput messages.

4. **Memory Usage Calculation**: The `__sizeof__` implementation provides approximations (e.g., 16 bytes per price level, 64 bytes SortedDict overhead). These are rough estimates and don't account for Python object overhead, hash table load factors, or balanced tree node structures in SortedDict's implementation.

5. **Snapshot Cache Invalidation**: When a snapshot is applied, the cache is invalidated but never explicitly recomputed. The next access to `best_bid`/`best_ask` properties should trigger recomputation, but the missing parentheses bug means this doesn't happen. The system relies on subsequent `apply_price_change()` calls to populate the cache with exchange-provided values.
