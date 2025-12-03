---
date: 2025-12-03T21:00:46+00:00
researcher: Elliot Steene
git_commit: 1759ee69c612a851d7b5e2307740cfc73feea17c
branch: main
repository: polypy
topic: "Remaining System Spec Components (Components 5-10)"
tags: [research, codebase, polymarket, websocket, system-spec, architecture]
status: complete
last_updated: 2025-12-03
last_updated_by: Elliot Steene
---

# Research: Remaining System Spec Components After Connection Pool

**Date**: 2025-12-03T21:00:46+00:00
**Researcher**: Elliot Steene
**Git Commit**: 1759ee69c612a851d7b5e2307740cfc73feea17c
**Branch**: main
**Repository**: polypy

## Research Question

Review the remaining steps in `/lessons/polymarket_websocket_system_spec.md` after the Connection Pool (Component 5) to document what needs to be built to complete the full
system.

## Summary

The Polymarket WebSocket system spec defines 10 components across 5 phases. The current codebase has implemented **Components 1-4** (Protocol, Orderbook, Registry,
Connection) but is missing **Components 5-10**. The system currently operates as a Phase 1 single-connection baseline. The remaining components introduce connection pooling,
multiprocessing for CPU parallelism, async message routing, market lifecycle management, and connection recycling for production deployment.

**Implementation Status:**
- ✅ **Implemented (4/10)**: Protocol Messages, Orderbook State, Market Registry, WebSocket Connection
- ❌ **Missing (6/10)**: Connection Pool, Message Router, Worker Manager, Lifecycle Controller, Connection Recycler, Full Application Orchestrator

## Detailed Findings

### Current Architecture (Phase 1 Baseline)

The existing system (`src/main.py`) implements a simplified single-connection architecture:

**Core Components:**
- **MessageParser** (`src/messages/parser.py:22`) - Parses raw WebSocket bytes into typed structures
- **OrderbookStore** (`src/orderbook/orderbook_store.py:11`) - Registry managing multiple OrderbookState instances
- **WebsocketConnection** (`src/connection/websocket.py:24`) - Single managed WebSocket connection with auto-reconnection
- **App.message_callback** (`src/main.py:28`) - Synchronously applies updates to orderbook state

**Limitations:**
- Hardcoded 2 asset IDs (lines 21-26 of src/main.py)
- Single WebSocket connection (no pooling)
- Single-threaded processing (no worker processes)
- No market discovery or lifecycle management
- No connection recycling
- Manual asset ID specification

### Component 1: Protocol Messages & Types ✅

**Status**: IMPLEMENTED as `src/messages/protocol.py` and `src/messages/parser.py`

**Implementation**:
- Uses msgspec.Struct for zero-copy parsing
- Integer price representation (PRICE_SCALE=1000, SIZE_SCALE=100)
- EventType enum: BOOK, PRICE_CHANGE, LAST_TRADE_PRICE, TICK_SIZE_CHANGE, UNKNOWN
- Generator-based parsing via `MessageParser.parse_messages()`

**Differences from Spec**:
- Spec uses 6 decimal places (10^6), implementation uses 3 (10^3)
- PriceChange has additional fields (hash, best_bid, best_ask)
- BookSnapshot missing asset_id, market, timestamp fields (commented out)
- Parse function returns generator instead of single ParsedMessage | None

### Component 2: Orderbook State Container ✅

**Status**: IMPLEMENTED as `src/orderbook/orderbook.py` and `src/orderbook/orderbook_store.py`

**Implementation**:
- SortedDict for O(log n) price level operations
- Negated bid keys for descending order
- Cache for best bid/ask with invalidation
- Memory tracking via `__sizeof__()`

**Missing from Spec**:
- `bid_depth`, `ask_depth`, `total_bid_size`, `total_ask_size`, `age_seconds` properties
- `update_count` field
- Some fields in spec but commented out in implementation

### Component 3: Market Registry ✅

**Status**: IMPLEMENTED as `src/registry/asset_registry.py` and `src/registry/asset_entry.py`

**Implementation**:
- Multi-index storage: by asset_id, condition_id, connection_id, status, expiration
- AssetStatus enum: PENDING, SUBSCRIBED, EXPIRED
- asyncio.Lock for write operations, lock-free reads
- SortedDict for expiration time index

**Missing from Spec**:
- `get_by_status()` method
- `get_pending_count()` method
- `get_expiring_before()` method for time-range queries
- `connection_stats()` method for pollution ratio
- `iter_active()` and `iter_all()` iterators

**Bug Found**: Line 120 of asset_registry.py sets `entry.condition_id = connection_id` instead of `entry.connection_id = connection_id`

### Component 4: WebSocket Connection Wrapper ✅

**Status**: IMPLEMENTED as `src/connection/websocket.py`, `src/connection/stats.py`, `src/connection/types.py`

**Implementation**:
- websockets.asyncio.client for connection management
- Automatic reconnection with exponential backoff (1s → 30s)
- ConnectionStatus enum: DISCONNECTED, CONNECTING, CONNECTED, DRAINING, CLOSED
- Health monitoring via `is_healthy` property (60s silence detection)
- Stats tracking: messages, bytes, parse errors, reconnect count

**Differences from Spec**:
- Spec uses manual PING/PONG loop, implementation delegates to websockets library
- No ping/pong latency tracking in stats (spec has `last_ping_ts`, `last_pong_ts`)
- Uses library's `ping_interval=10.0` parameter instead of manual ping task

---

## Remaining Components to Build

### Component 5: Connection Pool Manager ❌

**File**: `src/pool.py` (does not exist)

**Purpose**: Manages multiple WebSocket connections with capacity tracking and recycling

**Spec Location**: `lessons/polymarket_websocket_system_spec.md:1417-1754`

**Key Requirements**:

1. **Capacity Management**:
  - Track 400 active markets per connection (500 hard limit)
  - Create new connections when capacity exhausted
  - Assign markets to connections with available capacity

2. **Connection Tracking**:
  - `ConnectionInfo` dataclass wrapping WebSocketConnection
  - Track created_at timestamp and is_draining flag
  - Maintain dict of connection_id → ConnectionInfo

3. **Subscription Management** (lines 1566-1611):
  - Periodic subscription loop checking for pending markets
  - `BATCH_SUBSCRIPTION_INTERVAL = 30.0` seconds
  - `MIN_PENDING_FOR_NEW_CONNECTION = 50` threshold
  - `_process_pending_markets()` creates connections for batches

4. **Recycling Detection** (lines 1613-1635):
  - Check pollution ratio (30% expired triggers recycling)
  - Check connection age (don't recycle if < 5 minutes old)
  - Initiate recycling via `_initiate_recycling()` task

5. **Key Methods**:
  - `start()` - Start subscription management loop
  - `stop()` - Stop all connections concurrently
  - `force_subscribe(asset_ids)` - Immediate connection for priority assets
  - `get_connection_stats()` - Aggregate stats from all connections
  - `_remove_connection()` - Cleanup connection and registry

**Integration Points**:
- Depends on: MarketRegistry, WebSocketConnection, MessageCallback
- Used by: ConnectionRecycler, Application

---

### Component 6: Async Message Router ❌

**File**: `src/router.py` (does not exist)

**Purpose**: Routes messages from connections to worker processes via multiprocessing queues

**Spec Location**: `lessons/polymarket_websocket_system_spec.md:1758-2057`

**Key Requirements**:

1. **Queue Architecture**:
  - Bounded asyncio.Queue for backpressure (ASYNC_QUEUE_SIZE = 10,000)
  - Multiprocessing.Queue per worker (WORKER_QUEUE_SIZE = 5,000)
  - Two-stage routing: async domain → multiprocessing domain

2. **Consistent Hashing** (lines 1827-1835):
  - Hash asset_id to worker index using MD5
  - Cache mapping in `_asset_worker_cache` dict
  - Same asset always routes to same worker

3. **Batching** (lines 1948-1982):
  - BATCH_SIZE = 100 messages
  - BATCH_TIMEOUT = 10ms
  - Groups by worker before sending

4. **Backpressure Handling** (lines 1939-1946, 2031-2033):
  - Non-blocking puts with timeout (PUT_TIMEOUT = 1ms)
  - Drops messages on queue full
  - Tracks dropped count in stats

5. **Special Routing** (lines 2002-2008):
  - PRICE_CHANGE events may contain multiple assets
  - Fan out to all relevant workers (one message per affected worker)

6. **Key Methods**:
  - `route_message(connection_id, message)` - Entry point from connection callback
  - `_routing_loop()` - Batches messages and routes to workers
  - `_route_batch()` - Groups by worker and sends
  - `get_queue_depths()` - Monitoring for all queues
  - `stop()` - Send None sentinel to all workers

**Stats Tracked** (lines 1810-1824):
- messages_routed, messages_dropped
- batches_sent, queue_full_events
- routing_errors, total_latency_ms
- avg_latency_ms (computed property)

**Integration Points**:
- Depends on: ParsedMessage protocol
- Used by: ConnectionPool (provides message callback), WorkerManager (provides queues)

---

### Component 7: Worker Process Manager ❌

**File**: `src/worker.py` (does not exist)

**Purpose**: Manages worker processes that maintain orderbook state

**Spec Location**: `lessons/polymarket_websocket_system_spec.md:2060-2387`

**Note**: Reference implementation exists at `src/exercises/worker.py`

**Key Requirements**:

1. **Process Management**:
  - Spawn separate OS processes (bypasses GIL)
  - Each worker owns subset of orderbooks (via consistent hash)
  - Uses multiprocessing.Process, Queue, Event

2. **Worker Process Function** (lines 2129-2200):
  - Entry point: `_worker_process(worker_id, input_queue, stats_queue, shutdown_event, handler_factory)`
  - Must be module-level function (pickleable)
  - Initializes OrderbookStore per worker
  - Processes messages with QUEUE_TIMEOUT = 100ms

3. **Message Processing** (lines 2203-2240):
  - `_process_message()` applies updates to OrderbookStore
  - Handles BOOK (snapshot), PRICE_CHANGE (delta), LAST_TRADE_PRICE
  - Invokes optional handler callbacks (on_snapshot, on_price_change, on_trade)

4. **Health Monitoring**:
  - Heartbeat every HEARTBEAT_INTERVAL = 5.0 seconds
  - Stats report every STATS_INTERVAL = 30.0 seconds
  - Stats sent via stats_queue (non-blocking put)

5. **WorkerManager Class** (lines 2243-2375):
  - Creates input queues and stats queue
  - Starts worker processes with proper naming
  - Graceful shutdown with timeout (default 10s)
  - Force terminates if not stopped within timeout

6. **Key Methods**:
  - `start()` - Spawn all worker processes
  - `stop(timeout)` - Graceful shutdown with force terminate fallback
  - `get_stats()` - Collect latest stats from stats_queue
  - `is_healthy()` - Check all workers are alive
  - `get_alive_count()` - Count living processes

**Stats Tracked** (lines 2110-2126):
- messages_processed, updates_applied, snapshots_received
- processing_time_ms, avg_processing_time_us
- last_message_ts, orderbook_count, memory_usage_bytes

**Integration Points**:
- Depends on: ParsedMessage, OrderbookStore, OrderbookState
- Used by: MessageRouter (provides queues), Application

---

### Component 8: Market Lifecycle Controller ❌

**File**: `src/lifecycle.py` (does not exist)

**Purpose**: Manages market discovery, expiration detection, and lifecycle events

**Spec Location**: `lessons/polymarket_websocket_system_spec.md:2390-2747`

**Key Requirements**:

1. **Market Discovery** (lines 2451-2508):
  - Poll Gamma API: `https://gamma-api.polymarket.com/markets`
  - DISCOVERY_INTERVAL = 60.0 seconds
  - Fetch active, non-closed markets
  - Parse endDate ISO format to unix timestamp

2. **Market Registration** (lines 2600-2650):
  - Track known condition_ids in set to avoid duplicates
  - Register all tokens (Yes/No outcomes) for each market
  - Add to MarketRegistry with condition_id and expiration_ts
  - Invoke on_new_market callback

3. **Expiration Checking** (lines 2652-2688):
  - Check every EXPIRATION_CHECK_INTERVAL = 30.0 seconds
  - Query registry.get_expiring_before(current_time)
  - Filter to only SUBSCRIBED markets
  - Mark as EXPIRED via registry.mark_expired()
  - Invoke on_market_expired callback

4. **Cleanup** (lines 2690-2718):
  - Run every CLEANUP_DELAY = 3600.0 seconds (1 hour)
  - Remove EXPIRED markets that have aged beyond grace period
  - Calls registry.remove_market() to fully delete

5. **LifecycleController Class** (lines 2511-2735):
  - Uses aiohttp.ClientSession for API calls
  - Three background tasks: discovery, expiration, cleanup
  - Stores known_conditions set to avoid re-processing
  - Optional callbacks: on_new_market, on_market_expired

6. **Key Methods**:
  - `start()` - Initial discovery + start background tasks
  - `stop()` - Cancel tasks + close HTTP session
  - `add_market_manually()` - Bypass discovery for specific markets
  - `_discover_markets()` - Fetch from Gamma API and register
  - `_check_expirations()` - Mark expired markets
  - `_cleanup_expired()` - Remove old expired markets

**Integration Points**:
- Depends on: MarketRegistry, MarketStatus
- Used by: Application

---

### Component 9: Connection Recycler ❌

**File**: `src/recycler.py` (does not exist)

**Purpose**: Handles zero-downtime connection recycling

**Spec Location**: `lessons/polymarket_websocket_system_spec.md:2750-3038`

**Key Requirements**:

1. **Recycling Triggers** (lines 2908-2920):
  - Pollution ratio ≥ 30% (POLLUTION_THRESHOLD)
  - Age ≥ 24 hours (AGE_THRESHOLD = 86400.0 seconds)
  - Unhealthy connection (is_healthy = False)

2. **Recycling Flow** (lines 2766-2779):
  ```
  1. Detect trigger (pollution/age/health)
  2. Mark old connection as draining
  3. Create new connection with active markets
  4. Wait for new connection to stabilize (STABILIZATION_DELAY = 3.0s)
  5. Atomically update registry mappings
  6. Close old connection
  ```

3. **Zero-Message-Loss** (lines 2777-2779):
  - Both connections receive messages during transition
  - Only swap after new connection is stable and healthy
  - Registry update is atomic via reassign_connection()

4. **Concurrency Control** (lines 2798, 2936):
  - MAX_CONCURRENT_RECYCLES = 2 via asyncio.Semaphore
  - Track active_recycles set to prevent duplicate recycling
  - Skip connections already being recycled

5. **ConnectionRecycler Class** (lines 2817-3025):
  - Monitors all connections every HEALTH_CHECK_INTERVAL = 60.0 seconds
  - Uses pool.get_connection_stats() to check triggers
  - Creates recycling task per connection
  - Verifies new connection health before swap

6. **Key Methods**:
  - `start()` - Start monitoring loop
  - `stop()` - Stop monitoring
  - `_check_all_connections()` - Iterate connections and check triggers
  - `_recycle_connection()` - Perform full recycling workflow
  - `force_recycle(connection_id)` - Manual intervention

**Stats Tracked** (lines 2801-2814):
- recycles_initiated, recycles_completed, recycles_failed
- markets_migrated, total_downtime_ms
- success_rate (computed property)

**Integration Points**:
- Depends on: MarketRegistry, ConnectionPool
- Used by: Application

---

### Component 10: Main Application Orchestrator ❌

**File**: `src/app.py` (does not exist, current `src/main.py` is simplified)

**Purpose**: Ties all components together into running application

**Spec Location**: `lessons/polymarket_websocket_system_spec.md:3041-3449`

**Key Requirements**:

1. **Startup Sequence** (lines 3145-3202):
  ```
  1. Initialize MarketRegistry
  2. Start WorkerManager (spawn processes)
  3. Create MessageRouter (wire to worker queues)
  4. Start ConnectionPool (with message callback to router)
  5. Start LifecycleController (market discovery)
  6. Start ConnectionRecycler (monitoring)
  ```

2. **Shutdown Sequence** (lines 3204-3228):
  ```
  Reverse order:
  1. Stop ConnectionRecycler
  2. Stop LifecycleController
  3. Stop ConnectionPool
  4. Stop MessageRouter
  5. Stop WorkerManager
  6. Cleanup
  ```

3. **Application Class** (lines 3094-3339):
  - __slots__ for all components
  - `num_workers` parameter (default 4)
  - Optional `handler_factory` for per-worker callbacks
  - `_shutdown_event` for signal coordination

4. **Signal Handling** (lines 3236-3257):
  - Register SIGINT and SIGTERM handlers
  - Set shutdown_event to trigger graceful stop
  - Remove handlers after shutdown

5. **Stats Endpoint** (lines 3268-3324):
  - `get_stats()` returns comprehensive dict with:
    - registry: total_markets, pending, subscribed, expired
    - pool: connection_count, active_connections, per-connection stats
    - router: messages_routed, dropped, avg_latency, queue_depths
    - workers: alive_count, is_healthy, per-worker stats
    - recycler: recycles_completed, failed, markets_migrated, active_recycles

6. **Health Check** (lines 3326-3338):
  - `is_healthy()` checks:
    - Application is running
    - All workers are healthy
    - Has active connections (allows grace during startup)

7. **Key Methods**:
  - `start()` - Initialize and start all components in order
  - `stop()` - Stop all components in reverse order
  - `run()` - Convenience method with signal handling
  - `get_stats()` - Comprehensive statistics
  - `is_healthy()` - Overall health check
  - `_on_new_market()` - Lifecycle callback handler
  - `_on_market_expired()` - Lifecycle callback handler

**Integration Points**:
- Wires all 9 previous components together
- Provides unified interface for starting/stopping system
- Main entry point via `asyncio.run(main())` at lines 3342-3354

---

## Code References

### Current Implementation
- `src/messages/protocol.py` - Message type definitions
- `src/messages/parser.py:22` - MessageParser.parse_messages()
- `src/orderbook/orderbook.py:9` - OrderbookState class
- `src/orderbook/orderbook_store.py:11` - OrderbookStore class
- `src/registry/asset_registry.py:15` - AssetRegistry class
- `src/registry/asset_entry.py:6` - AssetStatus, AssetEntry
- `src/connection/websocket.py:24` - WebsocketConnection class
- `src/connection/stats.py:6` - ConnectionStats dataclass
- `src/connection/types.py:7` - ConnectionStatus enum
- `src/main.py:21` - App class (simplified orchestrator)
- `src/main.py:60` - main() entry point

### Spec Document
- `lessons/polymarket_websocket_system_spec.md:164-372` - Component 1 spec
- `lessons/polymarket_websocket_system_spec.md:375-650` - Component 2 spec
- `lessons/polymarket_websocket_system_spec.md:653-1036` - Component 3 spec
- `lessons/polymarket_websocket_system_spec.md:1039-1413` - Component 4 spec
- `lessons/polymarket_websocket_system_spec.md:1417-1754` - Component 5 spec (Connection Pool)
- `lessons/polymarket_websocket_system_spec.md:1758-2057` - Component 6 spec (Message Router)
- `lessons/polymarket_websocket_system_spec.md:2060-2387` - Component 7 spec (Worker Manager)
- `lessons/polymarket_websocket_system_spec.md:2390-2747` - Component 8 spec (Lifecycle Controller)
- `lessons/polymarket_websocket_system_spec.md:2750-3038` - Component 9 spec (Connection Recycler)
- `lessons/polymarket_websocket_system_spec.md:3041-3449` - Component 10 spec (Application Orchestrator)

### Reference Implementations
- `src/exercises/message_router.py` - Reference router implementation
- `src/exercises/worker.py` - Reference worker implementation
- `src/exercises/main.py` - Reference main application

---

## Architecture Documentation

### Current System (Phase 1 - Single Connection Baseline)

```
┌─────────────────────────────────────────────────────────┐
│                    Main Event Loop                       │
│                      (src/main.py)                       │
│                                                          │
│  ┌────────────┐    ┌──────────────┐    ┌─────────────┐ │
│  │  Message   │───►│  Websocket   │───►│   Message   │ │
│  │   Parser   │    │  Connection  │    │  Callback   │ │
│  └────────────┘    └──────────────┘    └──────┬──────┘ │
│                                                │         │
│                                                ▼         │
│                                         ┌──────────────┐ │
│                                         │  Orderbook   │ │
│                                         │    Store     │ │
│                                         └──────────────┘ │
└─────────────────────────────────────────────────────────┘

Features:
- Single WebSocket connection to 2 hardcoded assets
- Synchronous message processing in main event loop
- Direct callback from connection to orderbook updates
- Auto-reconnection with exponential backoff
```

### Target System (Phases 2-5 - Production Architecture)

```
┌──────────────────────────────────────────────────────────────┐
│                        ASYNC DOMAIN                           │
│                      (Single Event Loop)                      │
│                                                               │
│  ┌─────────────┐    ┌──────────────────┐    ┌──────────────┐│
│  │  Lifecycle  │    │  Connection Pool │    │   Message    ││
│  │ Controller  │───►│     Manager      │───►│   Router     ││
│  └─────────────┘    └────────┬─────────┘    └───────┬──────┘│
│        │                     │                       │        │
│        │          ┌──────────┼──────────┐            │        │
│        │          ▼          ▼          ▼            │        │
│        │       ┌────┐     ┌────┐     ┌────┐         │        │
│        │       │Conn│     │Conn│     │Conn│         │        │
│        │       │ #1 │     │ #2 │     │ #N │         │        │
│        │       └────┘     └────┘     └────┘         │        │
│        │                                             │        │
│        │       ┌─────────────┐                       │        │
│        └──────►│   Market    │                       │        │
│                │  Registry   │                       │        │
│                └─────────────┘                       │        │
└────────────────────────────────────────────────────────┼──────┘
                                                        │
                                multiprocessing.Queue   │
                                    (per worker)       │
                                                        ▼
┌──────────────────────────────────────────────────────────────┐
│                       PROCESS DOMAIN                          │
│                  (Separate OS Processes)                      │
│                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │
│  │  Worker #1      │  │  Worker #2      │  │  Worker #N   │ │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │ ┌──────────┐│ │
│  │  │ Orderbook │  │  │  │ Orderbook │  │  │ │Orderbook ││ │
│  │  │  States   │  │  │  │  States   │  │  │ │  States  ││ │
│  │  │ (assets   │  │  │  │ (assets   │  │  │ │ (assets  ││ │
│  │  │  A,B,C)   │  │  │  │  D,E,F)   │  │  │ │  G,H,I)  ││ │
│  │  └───────────┘  │  │  └───────────┘  │  │ └──────────┘│ │
│  └─────────────────┘  └─────────────────┘  └──────────────┘ │
└──────────────────────────────────────────────────────────────┘

Features:
- Multiple WebSocket connections (500 assets each)
- Automatic market discovery via Gamma API
- Connection pooling with capacity tracking
- Connection recycling when pollution > 30%
- Multiprocessing workers for CPU-intensive orderbook updates
- Async message router with consistent hashing
- Market lifecycle management (discovery, expiration, cleanup)
- Zero-downtime connection recycling
```

---

## Component Dependencies

### Build Order (from spec)

| Order | Component | File | Dependencies |
|-------|-----------|------|--------------|
| 1 | Protocol Messages | protocol.py | None |
| 2 | Orderbook State | orderbook.py | protocol.py |
| 3 | Market Registry | registry.py | None |
| 4 | WebSocket Connection | connection.py | protocol.py |
| 5 | Connection Pool | pool.py | connection.py, registry.py |
| 6 | Message Router | router.py | protocol.py |
| 7 | Worker Manager | worker.py | protocol.py, orderbook.py |
| 8 | Lifecycle Controller | lifecycle.py | registry.py |
| 9 | Connection Recycler | recycler.py | registry.py, pool.py |
| 10 | Application | app.py | All above |

### Implementation Progress

- ✅ **Phase 1 Complete**: Components 1-4 (Protocol, Orderbook, Registry, Connection)
- ❌ **Phase 2 Needed**: Component 5 (Connection Pool)
- ❌ **Phase 3 Needed**: Components 6-7 (Message Router, Worker Manager)
- ❌ **Phase 4 Needed**: Components 8-9 (Lifecycle Controller, Connection Recycler)
- ❌ **Phase 5 Needed**: Component 10 (Full Application Orchestrator)

---

## Performance Targets

From spec (lines 62-71):

| Metric | Target |
|--------|--------|
| Message routing latency | < 1ms |
| Orderbook update processing | < 5ms per update |
| Memory per orderbook | < 50KB typical |
| Connection recycling | Zero message loss |
| Reconnection time | < 5 seconds |

---

## Next Steps

To complete the system, implement components in order:

### 1. Component 5: Connection Pool Manager
- Create `src/pool.py`
- Implement ConnectionPool class with capacity tracking
- Add periodic subscription loop
- Integrate with MarketRegistry for market assignment
- Test with multiple connections

### 2. Component 6: Async Message Router
- Create `src/router.py`
- Implement consistent hashing for worker selection
- Add bounded queues with backpressure handling
- Implement batching logic
- Test routing latency

### 3. Component 7: Worker Process Manager
- Create `src/worker.py`
- Implement WorkerManager for process spawning
- Add worker process function with OrderbookStore
- Implement stats collection via multiprocessing.Queue
- Test with multiple workers

### 4. Component 8: Market Lifecycle Controller
- Create `src/lifecycle.py`
- Implement Gamma API client
- Add market discovery loop
- Add expiration checking loop
- Add cleanup loop
- Test with live Gamma API

### 5. Component 9: Connection Recycler
- Create `src/recycler.py`
- Implement ConnectionRecycler with trigger detection
- Add recycling workflow with zero-downtime
- Test recycling with active connections

### 6. Component 10: Full Application Orchestrator
- Create `src/app.py`
- Wire all components together
- Implement startup/shutdown sequences
- Add stats and health check endpoints
- Migrate current `src/main.py` logic
- Test full system integration

---

## Open Questions

1. **Registry Bug**: Line 120 of `src/registry/asset_registry.py` appears to set `entry.condition_id = connection_id` instead of `entry.connection_id`. Should this be fixed
before building on top of it?

2. **Price Scaling**: Spec uses 6 decimal places (PRICE_SCALE=1_000_000) but implementation uses 3 (PRICE_SCALE=1000). Is this intentional or should it be updated?

3. **Missing Registry Methods**: Several methods specified in Component 3 are missing (`get_by_status`, `get_expiring_before`, `connection_stats`). Should these be added
before implementing components 5-10 that depend on them?

4. **Reference Implementations**: The `src/exercises/` directory contains reference implementations for message_router and worker. Should these be migrated to `src/` or
rewritten from scratch following the spec?

5. **Testing Strategy**: Should each component be tested independently before integration, or build all components first then test the full system?
