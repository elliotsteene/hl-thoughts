---
date: 2025-12-14T23:11:54+0000
researcher: elliotsteene
git_commit: 704a022e79448524bb95829f62c5afda799d2dee
branch:
repository: polypy
topic: "UI Visualization Options for PolyPy Market Data"
tags: [research, codebase, ui, websocket, orderbook, visualization, real-time-updates]
status: complete
last_updated: 2025-12-14
last_updated_by: elliotsteene
---

# Research: UI Visualization Options for PolyPy Market Data

**Date**: 2025-12-14T23:11:54+0000
**Researcher**: elliotsteene
**Git Commit**: 704a022e79448524bb95829f62c5afda799d2dee
**Branch**: (main)
**Repository**: polypy

## Research Question

Investigate UI options for PolyPy that would visualize market-level data with continuous updates to state flowing through. The UI should require minimal user interaction - selecting a market from available markets and viewing orderbook state.

## Summary

PolyPy currently has a multiprocessing architecture where orderbook data lives in isolated worker processes, making it challenging to access for UI visualization. However, the system has existing HTTP endpoints (`/health`, `/stats`, `/metrics`) that demonstrate patterns for exposing data. The market discovery system tracks comprehensive market metadata, and the system processes real-time WebSocket updates. Several architectural approaches are available for building a UI:

1. **HTTP REST API + Polling** - Extend existing HTTP server with orderbook query endpoints, poll for updates
2. **WebSocket Server + Event Streaming** - Add WebSocket server to stream orderbook updates in real-time
3. **Server-Sent Events (SSE)** - Use SSE for one-way streaming of orderbook updates
4. **Shared Memory Access** - Direct read access to worker orderbook state (requires Phase 4 shared memory implementation)

## Detailed Findings

### Current HTTP Server Infrastructure

**File**: `src/server/server.py`

The HTTP server exists at `src/server/server.py:19` and provides three endpoints:
- `/health` (line 110): Returns application health status and component breakdown
- `/stats` (line 150): Returns comprehensive statistics from all subsystems
- `/metrics` (line 161): Returns Prometheus-format metrics

The server uses `aiohttp` web framework and is started as part of the main application orchestrator in `src/app.py:173-184`. It runs in the same async event loop as the main application.

**Current data exposure pattern**:
The `/stats` endpoint (src/app.py:292) aggregates data from all components:
- Registry statistics (total markets, status counts)
- Connection pool stats (connection count, capacity)
- Router stats (messages routed, latency, queue depths)
- Worker stats (messages processed, orderbook count, memory usage)
- Lifecycle controller stats (known market count)
- Recycler stats (recycle operations, success rate)

This demonstrates the existing pattern for querying subsystem state from the HTTP server context.

### Market Discovery and Metadata

**Files**:
- `src/lifecycle/controller.py` - Market discovery loop
- `src/lifecycle/types.py` - Market data structures
- `src/registry/asset_registry.py` - Market/asset tracking

The LifecycleController discovers markets from the Gamma API every 60 seconds (DISCOVERY_INTERVAL at `src/lifecycle/types.py:26`). Each market is represented as a `MarketInfo` structure (`src/lifecycle/types.py:8-18`) containing:

```python
@dataclass(slots=True)
class MarketInfo:
    condition_id: str          # Market identifier
    question: str              # Human-readable question
    outcomes: list[str]        # Possible outcomes
    tokens: list[dict]         # Token IDs and outcomes
    end_date_iso: str          # ISO format end date
    end_timestamp: int         # Unix milliseconds
    active: bool               # Is market active?
    closed: bool               # Is market closed?
```

The AssetRegistry (`src/registry/asset_registry.py:10`) tracks all markets and their status transitions through states defined at `src/registry/asset_entry.py`:
- **PENDING**: Market discovered but not yet subscribed
- **SUBSCRIBED**: Actively receiving WebSocket updates
- **EXPIRED**: Market has ended

**Market querying capabilities**:
The registry provides methods for listing markets (all at `src/registry/asset_registry.py`):
- `get_by_status(status)` (line 82): Get all markets by status
- `get_by_condition(condition_id)` (line 47): Get assets for a specific market
- `get_pending_count()` (line 64): Count of pending subscriptions

For a market selection UI, you could:
1. Query `get_by_status(AssetStatus.SUBSCRIBED)` to list active markets
2. Use the condition_id to look up full MarketInfo from lifecycle controller
3. Display market question, outcomes, and expiration time

### Orderbook Data Structure and Access

**Files**:
- `src/orderbook/orderbook.py` - Individual orderbook state
- `src/orderbook/orderbook_store.py` - Per-worker orderbook registry
- `src/worker/worker.py` - Worker process containing orderbooks

Each orderbook is an `OrderbookState` instance (`src/orderbook/orderbook.py:10`) with the structure:

```python
@dataclass(slots=True)
class OrderbookState:
    asset_id: str
    market: str
    _bids: SortedDict     # Negated prices for descending sort
    _asks: SortedDict     # Natural ascending order
    last_hash: str
    last_update_ts: int
    local_update_ts: float
```

**Available orderbook data access methods** (all in `src/orderbook/orderbook.py`):
- `best_bid` property (line 105): Cached best bid price
- `best_ask` property (line 111): Cached best ask price
- `spread` property (line 117): Bid-ask spread
- `mid_price` property (line 125): Mid-market price
- `get_bids(depth=10)` (line 132): Top N bid levels
- `get_asks(depth=10)` (line 142): Top N ask levels

All prices/sizes use scaled integers:
- Prices: multiply by 1000 (PRICE_SCALE)
- Sizes: multiply by 100 (SIZE_SCALE)
- Example: "0.50" → 500, "10.25" → 1025

### Multiprocessing Architecture Constraints

**Files**:
- `src/worker/worker.py` - Worker process definition
- `src/worker/manager.py` - Worker manager
- `src/app.py` - Main orchestrator

PolyPy uses a multiprocessing architecture where orderbook data lives in isolated worker processes (`src/worker/worker.py:81`). Each worker process:
- Owns a subset of orderbooks via consistent hashing
- Processes messages from a dedicated multiprocessing queue
- Runs in a separate OS process (bypassing Python GIL)
- Reports stats back via multiprocessing queues every 5s (heartbeat) and 30s (stats)

**Current inter-process communication**:
1. **Message routing** (src/app.py:127-138): Main process → Workers via `mp.Queue`
2. **Stats reporting** (src/worker/worker.py:210-224): Workers → Main via stats queues
3. **No direct orderbook access**: HTTP server runs in main process, orderbooks live in worker processes

The WorkerManager (`src/worker/manager.py:19`) collects worker stats in `get_stats()` (line 145), which includes:
- `messages_processed`
- `orderbook_count`
- `memory_usage_bytes`
- `avg_processing_time_us`

**Current limitation**: No mechanism exists to query specific orderbook data from workers. The stats are aggregate counts, not actual orderbook state.

### Real-Time Data Flow Architecture

**Files**:
- `src/connection/websocket.py` - WebSocket connection to Polymarket
- `src/connection/pool.py` - Connection pool management
- `src/messages/parser.py` - Message parsing
- `src/router.py` - Message routing to workers

The real-time data flow follows this path:

1. **WebSocket connections** (`src/connection/websocket.py:21`):
   - Each connection subscribes to up to 500 asset IDs
   - Receives raw bytes from Polymarket CLOB API
   - Automatic reconnection with exponential backoff (1s → 30s)

2. **Message parsing** (`src/messages/parser.py:11`):
   - Zero-copy parsing using msgspec
   - Yields `ParsedMessage` instances with three event types:
     - `BookSnapshot`: Full orderbook state
     - `PriceChange`: Single price level update
     - `LastTradePrice`: Trade execution info

3. **Message routing** (`src/router.py:17`):
   - Batches messages (up to 100 msgs or 10ms timeout)
   - Routes to workers via multiprocessing queues
   - Uses consistent hashing by asset_id

4. **Worker processing** (`src/worker/worker.py:144`):
   - Applies updates to OrderbookState
   - Updates happen in worker process memory only

**Key insight**: Updates flow one-way from WebSocket → Parser → Router → Workers. There's no reverse path for querying orderbook state back to the HTTP server.

### Async Patterns Used Throughout

The codebase extensively uses asyncio patterns:

- **Component lifecycle** (`src/server/server.py:56-108`): All components implement `async def start()` and `async def stop()`
- **Async locks** (`src/registry/asset_registry.py:23`): `asyncio.Lock()` for concurrent access
- **Event loops** (`src/app.py:261-290`): Main application runs in asyncio event loop with signal handlers
- **Async HTTP server** (`src/server/server.py:69-83`): aiohttp web application
- **Connection pool** (`src/connection/pool.py`): Async management of multiple WebSocket connections

All async components run in the main process event loop, while CPU-bound orderbook updates happen in worker processes.

## Architecture Options for UI

Based on the current architecture, here are the available approaches:

### Option 1: HTTP REST API with Polling

**Approach**: Extend the existing HTTP server with orderbook query endpoints.

**Implementation pattern**:
- Add new endpoint: `GET /markets` - List all subscribed markets with metadata
- Add new endpoint: `GET /orderbook/{asset_id}` - Get current orderbook state
- Workers respond to query requests via new response queues
- HTTP handler sends query to worker via mp.Queue, waits for response
- UI polls these endpoints at intervals (e.g., every 500ms-1s)

**Pros**:
- Extends existing HTTP server infrastructure
- Simple client implementation (standard HTTP)
- Works with any frontend framework
- No new server components needed

**Cons**:
- Polling introduces latency and wastes bandwidth
- Requires worker query mechanism (not currently implemented)
- Each poll requires round-trip to worker process

### Option 2: WebSocket Server with Event Streaming

**Approach**: Add a WebSocket server alongside HTTP server, stream orderbook updates.

**Implementation pattern**:
- Add WebSocket endpoint: `ws://localhost:8080/ws/orderbook/{asset_id}`
- Workers push orderbook updates to a pub/sub system (new component)
- WebSocket server subscribes to updates and forwards to connected clients
- UI connects via WebSocket, receives real-time updates

**Pros**:
- True real-time updates (no polling)
- Efficient - only sends updates when they occur
- Bidirectional communication if needed

**Cons**:
- Requires pub/sub system between workers and WS server
- More complex client implementation
- New server component to maintain

### Option 3: Server-Sent Events (SSE)

**Approach**: Use SSE for one-way streaming from server to client.

**Implementation pattern**:
- Add SSE endpoint: `GET /stream/orderbook/{asset_id}`
- Similar pub/sub as WebSocket option
- UI uses EventSource API (built into browsers)

**Pros**:
- Simpler than WebSocket (one-way only)
- Built-in browser support
- Automatic reconnection handling

**Cons**:
- One-way only (server → client)
- Still requires pub/sub between workers and HTTP server
- Less efficient than WebSocket for high-frequency updates

### Option 4: Shared Memory Access (Phase 4)

**Approach**: Implement shared memory for orderbooks (mentioned in learning path).

**Implementation pattern**:
- Workers write orderbook state to shared memory segments
- HTTP server reads directly from shared memory (no IPC needed)
- Requires implementing Phase 4 shared memory patterns

**Pros**:
- Zero-copy access to orderbook data
- No message passing overhead
- Efficient for high-frequency queries

**Cons**:
- Requires significant refactoring (Phase 4 implementation)
- Synchronization complexity (locks, atomic operations)
- Not currently available

## Code References

- HTTP Server: `src/server/server.py:19`
- App Stats Collection: `src/app.py:292`
- Asset Registry: `src/registry/asset_registry.py:10`
- Market Discovery: `src/lifecycle/controller.py`
- Market Metadata: `src/lifecycle/types.py:8`
- Orderbook State: `src/orderbook/orderbook.py:10`
- Orderbook Methods: `src/orderbook/orderbook.py:105-150`
- Worker Process: `src/worker/worker.py:81`
- Worker Stats: `src/worker/manager.py:145`
- Message Router: `src/router.py:17`
- WebSocket Connection: `src/connection/websocket.py:21`

## Current Capabilities Summary

**What exists today**:
1. HTTP server with health/stats/metrics endpoints
2. Market discovery with full metadata (question, outcomes, expiration)
3. AssetRegistry with market status tracking and querying
4. Orderbook data structure with bid/ask access methods
5. Real-time WebSocket data ingestion and parsing
6. Worker stats reporting (aggregate counts only)

**What's missing for UI**:
1. Mechanism to query specific orderbook state from workers
2. Market listing endpoint with human-readable metadata
3. Real-time update streaming to UI clients
4. Pub/sub or query/response system between main process and workers

## Recommended Approach

For minimal implementation effort with the current architecture:

**Phase 1 - Market Selection UI**:
1. Add `GET /markets` endpoint to list subscribed markets
2. Query AssetRegistry for SUBSCRIBED status
3. Include condition_id, question, outcomes, expiration from MarketInfo
4. Simple REST endpoint, no worker communication needed

**Phase 2 - Orderbook Query API**:
1. Add query queue system: main process → workers
2. Add response queue system: workers → main process
3. Implement `GET /orderbook/{asset_id}` endpoint
4. Worker responds with serialized orderbook state (best bid/ask, depth)
5. UI polls this endpoint for selected market

**Phase 3 - Real-Time Streaming** (optional):
1. Implement pub/sub between workers and main process
2. Workers publish orderbook updates after applying changes
3. Add WebSocket or SSE server for streaming
4. UI subscribes to live updates instead of polling

This approach builds incrementally on existing infrastructure without requiring the Phase 4 shared memory refactor.

## Related Research

No prior research documents found in `thoughts/shared/research/`.

## Open Questions

1. What update frequency is acceptable for the UI? (determines polling vs streaming priority) 500ms
2. How many concurrent users need to view orderbooks? (affects scaling approach) 1
3. Should the UI show full orderbook depth or just best bid/ask? Show orderbook depth and any calculated metrics
4. Is historical data needed, or only current state? Historical data may also be fetched by the client while fetching orderbook data from server.
5. Would the UI benefit from bidirectional communication (user placing orders)? Bidirectional is needed for market selection.
