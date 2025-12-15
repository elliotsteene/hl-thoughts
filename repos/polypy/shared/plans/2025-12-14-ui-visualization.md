# PolyPy UI Visualization Implementation Plan

## Overview

Implement a real-time market data visualization UI for PolyPy that displays orderbook state with continuous updates. The UI requires minimal user interaction - selecting a market from available markets and viewing live orderbook data with metrics (spread, mid_price, imbalance).

## Current State Analysis

### Existing Infrastructure
- **HTTP Server** (`src/server/server.py`): aiohttp-based server with `/health`, `/stats`, `/metrics` endpoints
- **LifecycleController** (`src/lifecycle/controller.py`): Discovers markets from Gamma API, but discards `MarketInfo` (question, outcomes) after registration
- **AssetRegistry** (`src/registry/asset_registry.py`): Stores only runtime tracking data (asset_id, condition_id, status, expiration_ts)
- **Workers** (`src/worker/worker.py`): Process orderbook updates in separate OS processes via multiprocessing queues
- **OrderbookState** (`src/orderbook/orderbook.py`): Maintains bid/ask levels with `get_bids(depth)`, `get_asks(depth)`, `spread`, `mid_price` properties

### Key Gaps
1. **No market metadata storage**: MarketInfo is discarded after discovery - question text, outcomes not available
2. **No orderbook query mechanism**: Workers only accept messages, no request/response pattern
3. **No WebSocket server**: Only HTTP endpoints exist
4. **No frontend**: Application is backend-only

### Key Discoveries
- `OrderbookState` already has `get_bids(depth=10)` and `get_asks(depth=10)` methods (`orderbook.py:132-150`)
- `OrderbookState` already has `spread` and `mid_price` properties (`orderbook.py:116-130`)
- aiohttp supports WebSocket endpoints via `web.WebSocketResponse`
- Workers use `multiprocessing.Queue` for input and a shared stats queue for output (`manager.py:63-82`)

## Desired End State

1. **`GET /markets`** endpoint returns list of subscribed markets with metadata (condition_id, question, outcomes, token mappings)
2. **`GET /orderbook/{asset_id}`** endpoint returns current orderbook state with top 10 levels and metrics
3. **`WS /ws/orderbook`** endpoint streams orderbook updates at 500ms intervals for selected asset
4. **React + Vite frontend** displays market list and live orderbook visualization
5. **Historical snapshots** captured in workers for potential time-series display

### Verification
- Manual: Navigate to UI, select a market, observe orderbook updating in real-time
- Automated: Unit tests for new endpoints and worker query mechanism

## What We're NOT Doing

- Complex connection management (single client optimization)
- Authentication/authorization
- Persistent storage of historical data
- Complex charting libraries (initial version)
- Mobile-responsive design (initial version)

## Implementation Approach

The implementation follows a bottom-up approach:
1. Add metadata storage to LifecycleController (data source)
2. Add request/response mechanism to workers (enables orderbook queries)
3. Add HTTP endpoints for markets and orderbooks
4. Add WebSocket endpoint for streaming
5. Build React frontend to consume APIs

## Phase 1: Market Metadata Storage

### Overview
Store `MarketInfo` during discovery so it can be served via API. Add a simple dict lookup keyed by condition_id.

### Changes Required

#### 1.1 Extend LifecycleController

**File**: `src/lifecycle/controller.py`
**Changes**: Add market metadata storage and lookup method

```python
# Add to __slots__ (line 33-42):
"_market_metadata",

# Add to __init__ (after line 64):
self._market_metadata: dict[str, MarketInfo] = {}

# Add new method:
def get_market_metadata(self, condition_id: str) -> MarketInfo | None:
    """Get market metadata by condition ID."""
    return self._market_metadata.get(condition_id)

def get_all_markets(self) -> list[MarketInfo]:
    """Get all known market metadata."""
    return list(self._market_metadata.values())

# Modify _discover_markets() (around line 220):
# After: self._known_conditions.add(market.condition_id)
# Add: self._market_metadata[market.condition_id] = market
```

#### 1.2 Add GET /markets Endpoint

**File**: `src/server/server.py`
**Changes**: Add new endpoint handler and route

```python
# Add route in start() (after line 72):
web_app.router.add_get("/markets", self._handle_markets)

# Add handler:
async def _handle_markets(self, request: web.Request) -> web.Response:
    """Handle GET /markets endpoint.

    Returns list of subscribed markets with metadata.
    """
    logger.debug("GET /markets")

    if not self._app._lifecycle:
        return web.json_response({"error": "Lifecycle controller not available"}, status=503)

    markets = self._app._lifecycle.get_all_markets()
    subscribed_assets = self._app._registry.get_by_status(AssetStatus.SUBSCRIBED) if self._app._registry else frozenset()

    result = []
    for market in markets:
        # Check if any token from this market is subscribed
        token_ids = [t.get("token_id", "") for t in market.tokens]
        if any(tid in subscribed_assets for tid in token_ids):
            result.append({
                "condition_id": market.condition_id,
                "question": market.question,
                "outcomes": market.outcomes,
                "tokens": market.tokens,
                "end_date_iso": market.end_date_iso,
                "end_timestamp": market.end_timestamp,
            })

    return web.json_response({"markets": result}, status=200)
```

### Success Criteria

#### Automated Verification:
- [ ] Tests pass: `just test tests/test_lifecycle_controller.py`
- [ ] Tests pass: `just test tests/server/test_http_server.py`
- [ ] Lint passes: `just check`

#### Manual Verification:
- [ ] Start application, wait for discovery
- [ ] `curl http://localhost:8080/markets` returns market list with question text and outcomes
- [ ] Market list only includes subscribed markets

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

**PLAN QUESTION** Does this endpoint block the event loop while querying? Is there an impact on websocket latency?

---

## Phase 2: Worker Orderbook Query Infrastructure

### Overview
Add a request/response queue mechanism to query orderbook state from workers. The router will determine which worker owns an asset and forward requests accordingly.

### Changes Required

#### 2.1 Define Request/Response Protocol

**File**: `src/worker/protocol.py` (new file)
**Changes**: Create message types for orderbook queries

```python
"""Worker request/response protocol for orderbook queries."""

from dataclasses import dataclass, field
from enum import IntEnum


class RequestType(IntEnum):
    """Request type enumeration."""
    GET_ORDERBOOK = 1


@dataclass(slots=True)
class OrderbookRequest:
    """Request for orderbook state."""
    request_id: str
    asset_id: str
    depth: int = 10


@dataclass(slots=True)
class OrderbookMetrics:
    """Calculated orderbook metrics."""
    spread: int | None
    mid_price: int | None
    imbalance: float  # bid_volume / (bid_volume + ask_volume), 0-1


@dataclass(slots=True)
class OrderbookResponse:
    """Response containing orderbook state."""
    request_id: str
    asset_id: str
    found: bool
    bids: list[tuple[int, int]] = field(default_factory=list)  # [(price, size), ...]
    asks: list[tuple[int, int]] = field(default_factory=list)
    metrics: OrderbookMetrics | None = None
    last_update_ts: int = 0
    error: str | None = None
```

#### 2.2 Add Response Queue to WorkerManager

**File**: `src/worker/manager.py`
**Changes**: Add response queue and request routing

```python
# Add to __slots__ (line 30-37):
"_response_queue",
"_pending_requests",

# Add to __init__ (after line 63):
self._response_queue: MPQueue = MPQueue()
self._pending_requests: dict[str, asyncio.Future] = {}

# Add new methods:
def get_response_queue(self) -> MPQueue:
    """Get the shared response queue for worker responses."""
    return self._response_queue

async def query_orderbook(
    self,
    asset_id: str,
    worker_idx: int,
    depth: int = 10,
    timeout: float = 1.0,
) -> OrderbookResponse:
    """
    Query orderbook state from a worker.

    Args:
        asset_id: Asset to query
        worker_idx: Worker index that owns this asset
        depth: Number of price levels to return
        timeout: Max wait time for response

    Returns:
        OrderbookResponse with orderbook state or error
    """
    # Add import at top:
    import uuid
    from src.worker.protocol import OrderbookRequest, OrderbookResponse

    request_id = str(uuid.uuid4())
    request = OrderbookRequest(
        request_id=request_id,
        asset_id=asset_id,
        depth=depth,
    )

    # Send request to worker
    queue = self._input_queues[worker_idx]
    try:
        queue.put((request, time.monotonic()), timeout=0.1)
    except Full:
        return OrderbookResponse(
            request_id=request_id,
            asset_id=asset_id,
            found=False,
            error="Worker queue full",
        )

    # Wait for response (poll response queue)
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        try:
            response = self._response_queue.get(timeout=0.05)
            if response.request_id == request_id:
                return response
            # Wrong response - put back? For now, discard (single client)
        except Empty:
            continue

    return OrderbookResponse(
        request_id=request_id,
        asset_id=asset_id,
        found=False,
        error="Request timeout",
    )
```
**PLAN QUESTION ** The operations on the queue are sync, but this function is async. Is there any impact on event loop? Will this function block?

#### 2.3 Handle Requests in Worker

**File**: `src/worker/worker.py`
**Changes**: Handle OrderbookRequest alongside ParsedMessage

```python
# Add import at top:
from src.worker.protocol import OrderbookRequest, OrderbookResponse, OrderbookMetrics

# Modify Worker.__init__ to accept response_queue:
def __init__(
    self,
    worker_id: int,
    input_queue: MPQueue,
    stats_queue: MPQueue,
    response_queue: MPQueue,  # NEW
    orderbook_store: OrderbookStore,
    stats: WorkerStats,
) -> None:
    # ... existing ...
    self._response_queue = response_queue

# Modify _process_message_loop() (line 94-113):
def _process_message_loop(self) -> None:
    item = self._input_queue.get(timeout=QUEUE_TIMEOUT)

    if item is None:
        raise ShutdownReceivedError("Shutdown sentinel received")

    message, receive_ts = item

    # Handle orderbook query request
    if isinstance(message, OrderbookRequest):
        self._handle_orderbook_request(message)
        return

    # Handle normal message processing
    process_start = time.monotonic()
    self._process_message(message)
    # ... rest of existing code ...

# Add new method:
def _handle_orderbook_request(self, request: OrderbookRequest) -> None:
    """Handle orderbook query request."""
    orderbook = self._store.get_state(request.asset_id)

    if not orderbook:
        response = OrderbookResponse(
            request_id=request.request_id,
            asset_id=request.asset_id,
            found=False,
            error="Asset not found",
        )
    else:
        bids = [(level.price, level.size) for level in orderbook.get_bids(request.depth)]
        asks = [(level.price, level.size) for level in orderbook.get_asks(request.depth)]

        # Calculate imbalance
        bid_vol = sum(size for _, size in bids)
        ask_vol = sum(size for _, size in asks)
        total_vol = bid_vol + ask_vol
        imbalance = bid_vol / total_vol if total_vol > 0 else 0.5

        metrics = OrderbookMetrics(
            spread=orderbook.spread,
            mid_price=orderbook.mid_price,
            imbalance=imbalance,
        )

        response = OrderbookResponse(
            request_id=request.request_id,
            asset_id=request.asset_id,
            found=True,
            bids=bids,
            asks=asks,
            metrics=metrics,
            last_update_ts=orderbook.last_update_ts,
        )

    try:
        self._response_queue.put_nowait(response)
    except Full:
        logger.warning(f"Response queue full, dropping response for {request.asset_id}")

# Update _worker_process to pass response_queue:
def _worker_process(
    worker_id: int,
    input_queue: MPQueue,
    stats_queue: MPQueue,
    response_queue: MPQueue,  # NEW
    shutdown_event: Event,
) -> None:
    # ... existing setup ...
    worker = Worker(
        worker_id=worker_id,
        input_queue=input_queue,
        stats_queue=stats_queue,
        response_queue=response_queue,  # NEW
        orderbook_store=store,
        stats=stats,
    )
    worker.run(shutdown_event=shutdown_event)
```

#### 2.4 Update WorkerManager Process Spawning

**File**: `src/worker/manager.py`
**Changes**: Pass response_queue to worker processes

```python
# Modify start() (lines 98-111):
for worker_id in range(self._num_workers):
    process = Process(
        target=_worker_process,
        args=(
            worker_id,
            queues[worker_id],
            self._stats_queue,
            self._response_queue,  # NEW
            self._shutdown_event,
        ),
        name=f"orderbook-worker-{worker_id}",
        daemon=False,
    )
    process.start()
    self._processes.append(process)
```

#### 2.5 Add GET /orderbook/{asset_id} Endpoint

**File**: `src/server/server.py`
**Changes**: Add orderbook query endpoint

```python
# Add route in start():
web_app.router.add_get("/orderbook/{asset_id}", self._handle_orderbook)

# Add handler:
async def _handle_orderbook(self, request: web.Request) -> web.Response:
    """Handle GET /orderbook/{asset_id} endpoint."""
    asset_id = request.match_info["asset_id"]
    depth = int(request.query.get("depth", "10"))

    logger.debug(f"GET /orderbook/{asset_id}")

    if not self._app._workers or not self._app._router:
        return web.json_response({"error": "Workers not available"}, status=503)

    # Determine which worker owns this asset
    worker_idx = self._app._router.get_worker_for_asset(asset_id)

    # Query worker
    response = await self._app._workers.query_orderbook(
        asset_id=asset_id,
        worker_idx=worker_idx,
        depth=depth,
    )

    if not response.found:
        return web.json_response(
            {"error": response.error or "Asset not found"},
            status=404,
        )

    return web.json_response({
        "asset_id": response.asset_id,
        "bids": response.bids,
        "asks": response.asks,
        "metrics": {
            "spread": response.metrics.spread,
            "mid_price": response.metrics.mid_price,
            "imbalance": response.metrics.imbalance,
        } if response.metrics else None,
        "last_update_ts": response.last_update_ts,
    }, status=200)
```

#### 2.6 Add Worker Lookup to Router

**File**: `src/router.py`
**Changes**: Expose worker assignment lookup

```python
# Add new method (public wrapper around existing _hash_to_worker):
def get_worker_for_asset(self, asset_id: str) -> int:
    """Get worker index that owns a given asset."""
    if asset_id in self._asset_worker_cache:
        return self._asset_worker_cache[asset_id]

    worker_idx = _hash_to_worker(asset_id, self._num_workers)
    self._asset_worker_cache[asset_id] = worker_idx
    return worker_idx
```

### Success Criteria

#### Automated Verification:
- [ ] New protocol file exists: `src/worker/protocol.py`
- [ ] Tests pass: `just test tests/test_worker.py`
- [ ] Tests pass: `just test tests/server/test_http_server.py`
- [ ] Lint passes: `just check`

#### Manual Verification:
- [ ] Start application, wait for markets to be subscribed
- [ ] `curl http://localhost:8080/orderbook/{asset_id}` returns orderbook with bids/asks/metrics
- [ ] Response includes spread, mid_price, imbalance metrics

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Phase 3: Historical Snapshot Capture

### Overview
Add a ring buffer to OrderbookState to capture recent snapshots for historical display. This is lightweight and doesn't require external storage.

### Changes Required

#### 3.1 Add Snapshot Ring Buffer to OrderbookState

**File**: `src/orderbook/orderbook.py`
**Changes**: Add history capture

```python
from collections import deque
from dataclasses import dataclass, field

# Add new dataclass for snapshot:
@dataclass(slots=True, frozen=True)
class OrderbookSnapshot:
    """Point-in-time orderbook snapshot."""
    timestamp: int  # Unix ms
    best_bid: int | None
    best_ask: int | None
    spread: int | None
    mid_price: int | None
    bid_depth: int  # Total bid volume
    ask_depth: int  # Total ask volume

# Modify OrderbookState (add to fields around line 30):
_history: deque = field(default_factory=lambda: deque(maxlen=100))
_history_interval_ms: int = 1000  # Capture every 1s

# Add method to capture snapshot:
def _maybe_capture_snapshot(self) -> None:
    """Capture snapshot if enough time has passed."""
    if not self._history or (self.last_update_ts - self._history[-1].timestamp >= self._history_interval_ms):
        bid_depth = sum(self._bids.values())
        ask_depth = sum(self._asks.values())

        snapshot = OrderbookSnapshot(
            timestamp=self.last_update_ts,
            best_bid=self.best_bid,
            best_ask=self.best_ask,
            spread=self.spread,
            mid_price=self.mid_price,
            bid_depth=bid_depth,
            ask_depth=ask_depth,
        )
        self._history.append(snapshot)

# Call from apply_snapshot and apply_price_change:
# At end of apply_snapshot (after line 55):
self._maybe_capture_snapshot()

# At end of apply_price_change (after line 84):
self._maybe_capture_snapshot()

# Add getter:
def get_history(self, limit: int = 100) -> list[OrderbookSnapshot]:
    """Get recent snapshots, newest first."""
    return list(reversed(list(self._history)))[:limit]
```

#### 3.2 Extend OrderbookResponse to Include History

**File**: `src/worker/protocol.py`
**Changes**: Add history field

```python
@dataclass(slots=True)
class HistoryPoint:
    """Single history data point."""
    timestamp: int
    best_bid: int | None
    best_ask: int | None
    spread: int | None
    mid_price: int | None

@dataclass(slots=True)
class OrderbookResponse:
    """Response containing orderbook state."""
    request_id: str
    asset_id: str
    found: bool
    bids: list[tuple[int, int]] = field(default_factory=list)
    asks: list[tuple[int, int]] = field(default_factory=list)
    metrics: OrderbookMetrics | None = None
    last_update_ts: int = 0
    history: list[HistoryPoint] = field(default_factory=list)  # NEW
    error: str | None = None
```

#### 3.3 Include History in Worker Response

**File**: `src/worker/worker.py`
**Changes**: Add history to response

```python
# In _handle_orderbook_request, after creating metrics:
history = [
    HistoryPoint(
        timestamp=snap.timestamp,
        best_bid=snap.best_bid,
        best_ask=snap.best_ask,
        spread=snap.spread,
        mid_price=snap.mid_price,
    )
    for snap in orderbook.get_history(limit=50)
]

response = OrderbookResponse(
    # ... existing fields ...
    history=history,
)
```

### Success Criteria

#### Automated Verification:
- [ ] Tests pass: `just test tests/test_orderbook.py`
- [ ] Tests pass: `just test tests/test_worker.py`
- [ ] Lint passes: `just check`

#### Manual Verification:
- [ ] Query orderbook endpoint, verify `history` array is populated
- [ ] History contains timestamp, spread, mid_price data points

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Phase 4: WebSocket Streaming

### Overview
Add WebSocket endpoint for real-time orderbook streaming. Client sends market selection, server streams updates at configurable interval.

### Changes Required

#### 4.1 Add WebSocket Handler

**File**: `src/server/server.py`
**Changes**: Add WebSocket endpoint

```python
import asyncio

# Add to __slots__:
"_ws_client",
"_ws_task",

# Add to __init__:
self._ws_client: web.WebSocketResponse | None = None
self._ws_task: asyncio.Task | None = None

# Add route in start():
web_app.router.add_get("/ws/orderbook", self._handle_ws_orderbook)

# Add handler:
async def _handle_ws_orderbook(self, request: web.Request) -> web.WebSocketResponse:
    """Handle WebSocket connection for orderbook streaming."""
    ws = web.WebSocketResponse()
    await ws.prepare(request)

    logger.info("WebSocket client connected")

    # Single client - replace existing if any
    if self._ws_client:
        await self._ws_client.close()
    self._ws_client = ws

    current_asset: str | None = None
    stream_task: asyncio.Task | None = None

    try:
        async for msg in ws:
            if msg.type == web.WSMsgType.TEXT:
                try:
                    data = msg.json()

                    if "subscribe" in data:
                        # Cancel existing stream
                        if stream_task:
                            stream_task.cancel()
                            try:
                                await stream_task
                            except asyncio.CancelledError:
                                pass

                        current_asset = data["subscribe"]
                        interval = data.get("interval_ms", 500) / 1000.0

                        # Start streaming
                        stream_task = asyncio.create_task(
                            self._stream_orderbook(ws, current_asset, interval)
                        )

                    elif "unsubscribe" in data:
                        if stream_task:
                            stream_task.cancel()
                            stream_task = None
                        current_asset = None

                except Exception as e:
                    await ws.send_json({"error": str(e)})

            elif msg.type == web.WSMsgType.ERROR:
                logger.error(f"WebSocket error: {ws.exception()}")
                break

    finally:
        if stream_task:
            stream_task.cancel()
        if self._ws_client == ws:
            self._ws_client = None
        logger.info("WebSocket client disconnected")

    return ws

async def _stream_orderbook(
    self,
    ws: web.WebSocketResponse,
    asset_id: str,
    interval: float,
) -> None:
    """Stream orderbook updates to WebSocket client."""
    if not self._app._workers or not self._app._router:
        await ws.send_json({"error": "Workers not available"})
        return

    worker_idx = self._app._router.get_worker_for_asset(asset_id)

    try:
        while True:
            response = await self._app._workers.query_orderbook(
                asset_id=asset_id,
                worker_idx=worker_idx,
                depth=10,
            )

            if response.found:
                await ws.send_json({
                    "type": "orderbook",
                    "asset_id": response.asset_id,
                    "bids": response.bids,
                    "asks": response.asks,
                    "metrics": {
                        "spread": response.metrics.spread,
                        "mid_price": response.metrics.mid_price,
                        "imbalance": response.metrics.imbalance,
                    } if response.metrics else None,
                    "last_update_ts": response.last_update_ts,
                })
            else:
                await ws.send_json({
                    "type": "error",
                    "error": response.error or "Asset not found",
                })

            await asyncio.sleep(interval)

    except asyncio.CancelledError:
        pass
    except Exception as e:
        logger.error(f"Stream error: {e}")
        await ws.send_json({"type": "error", "error": str(e)})
```

### Success Criteria

#### Automated Verification:
- [ ] Tests pass: `just test tests/server/test_http_server.py`
- [ ] Lint passes: `just check`

#### Manual Verification:
- [ ] Connect via websocat or browser: `websocat ws://localhost:8080/ws/orderbook`
- [ ] Send `{"subscribe": "asset_id"}` and receive orderbook updates
- [ ] Updates arrive at ~500ms intervals
- [ ] Send `{"unsubscribe": true}` to stop updates

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation.

---

## Phase 5: React Frontend

### Overview
Create a React + Vite frontend that displays market list and live orderbook. Uses TanStack Query for data fetching and a simple component structure for extensibility.

### Changes Required

#### 5.1 Initialize Frontend Project

**Directory**: `ui/` (new directory at project root)
**Commands**:
```bash
cd /Users/elliotsteene/Documents/prediction_markets/polypy
npm create vite@latest ui -- --template react-ts
cd ui
npm install
npm install @tanstack/react-query axios
```

#### 5.2 Project Structure

```
ui/
├── src/
│   ├── api/
│   │   ├── client.ts          # Axios instance
│   │   ├── markets.ts         # Market API hooks
│   │   └── orderbook.ts       # Orderbook API/WS hooks
│   ├── components/
│   │   ├── MarketList.tsx     # Market selector dropdown
│   │   ├── OrderbookView.tsx  # Main orderbook display
│   │   ├── PriceLevels.tsx    # Bid/ask level rows
│   │   └── Metrics.tsx        # Spread, mid_price, imbalance
│   ├── hooks/
│   │   └── useOrderbookStream.ts  # WebSocket hook
│   ├── types/
│   │   └── index.ts           # TypeScript interfaces
│   ├── App.tsx
│   └── main.tsx
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

#### 5.3 Type Definitions

**File**: `ui/src/types/index.ts`

```typescript
export interface Token {
  token_id: string;
  outcome: string;
}

export interface Market {
  condition_id: string;
  question: string;
  outcomes: string[];
  tokens: Token[];
  end_date_iso: string;
  end_timestamp: number;
}

export interface MarketsResponse {
  markets: Market[];
}

export interface OrderbookMetrics {
  spread: number | null;
  mid_price: number | null;
  imbalance: number;
}

export interface OrderbookData {
  asset_id: string;
  bids: [number, number][];  // [price, size]
  asks: [number, number][];
  metrics: OrderbookMetrics | null;
  last_update_ts: number;
}

export interface OrderbookMessage {
  type: 'orderbook' | 'error';
  asset_id?: string;
  bids?: [number, number][];
  asks?: [number, number][];
  metrics?: OrderbookMetrics;
  last_update_ts?: number;
  error?: string;
}
```

#### 5.4 API Client

**File**: `ui/src/api/client.ts`

```typescript
import axios from 'axios';

export const api = axios.create({
  baseURL: 'http://localhost:8080',
});
```

**File**: `ui/src/api/markets.ts`

```typescript
import { useQuery } from '@tanstack/react-query';
import { api } from './client';
import type { MarketsResponse } from '../types';

export function useMarkets() {
  return useQuery({
    queryKey: ['markets'],
    queryFn: async () => {
      const { data } = await api.get<MarketsResponse>('/markets');
      return data.markets;
    },
    refetchInterval: 60000,  // Refresh every minute
  });
}
```

#### 5.5 WebSocket Hook

**File**: `ui/src/hooks/useOrderbookStream.ts`

```typescript
import { useEffect, useRef, useState, useCallback } from 'react';
import type { OrderbookData, OrderbookMessage } from '../types';

export function useOrderbookStream(assetId: string | null) {
  const [data, setData] = useState<OrderbookData | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [connected, setConnected] = useState(false);
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const ws = new WebSocket('ws://localhost:8080/ws/orderbook');
    wsRef.current = ws;

    ws.onopen = () => {
      setConnected(true);
      if (assetId) {
        ws.send(JSON.stringify({ subscribe: assetId }));
      }
    };

    ws.onmessage = (event) => {
      const msg: OrderbookMessage = JSON.parse(event.data);
      if (msg.type === 'orderbook' && msg.bids && msg.asks) {
        setData({
          asset_id: msg.asset_id!,
          bids: msg.bids,
          asks: msg.asks,
          metrics: msg.metrics || null,
          last_update_ts: msg.last_update_ts!,
        });
        setError(null);
      } else if (msg.type === 'error') {
        setError(msg.error || 'Unknown error');
      }
    };

    ws.onerror = () => setError('WebSocket error');
    ws.onclose = () => setConnected(false);

    return () => {
      ws.close();
    };
  }, []);

  // Subscribe to new asset when assetId changes
  useEffect(() => {
    if (connected && wsRef.current && assetId) {
      wsRef.current.send(JSON.stringify({ subscribe: assetId }));
    }
  }, [assetId, connected]);

  return { data, error, connected };
}
```

#### 5.6 Components

**File**: `ui/src/components/MarketList.tsx`

```typescript
import { useMarkets } from '../api/markets';
import type { Market, Token } from '../types';

interface Props {
  selectedAssetId: string | null;
  onSelectAsset: (assetId: string, market: Market) => void;
}

export function MarketList({ selectedAssetId, onSelectAsset }: Props) {
  const { data: markets, isLoading, error } = useMarkets();

  if (isLoading) return <div>Loading markets...</div>;
  if (error) return <div>Error loading markets</div>;

  return (
    <div className="market-list">
      <h2>Markets</h2>
      <select
        value={selectedAssetId || ''}
        onChange={(e) => {
          const assetId = e.target.value;
          const market = markets?.find(m =>
            m.tokens.some(t => t.token_id === assetId)
          );
          if (market) onSelectAsset(assetId, market);
        }}
      >
        <option value="">Select a market...</option>
        {markets?.map((market) => (
          <optgroup key={market.condition_id} label={market.question.slice(0, 50)}>
            {market.tokens.map((token: Token) => (
              <option key={token.token_id} value={token.token_id}>
                {token.outcome}
              </option>
            ))}
          </optgroup>
        ))}
      </select>
    </div>
  );
}
```

**File**: `ui/src/components/Metrics.tsx`

```typescript
import type { OrderbookMetrics } from '../types';

interface Props {
  metrics: OrderbookMetrics | null;
}

// Price scale factor (from protocol.py)
const PRICE_SCALE = 1000;

export function Metrics({ metrics }: Props) {
  if (!metrics) return null;

  const formatPrice = (price: number | null) =>
    price !== null ? (price / PRICE_SCALE).toFixed(3) : '-';

  const imbalancePercent = (metrics.imbalance * 100).toFixed(1);
  const imbalanceLabel = metrics.imbalance > 0.5 ? 'Bid heavy' : 'Ask heavy';

  return (
    <div className="metrics">
      <div className="metric">
        <span className="label">Spread</span>
        <span className="value">{formatPrice(metrics.spread)}</span>
      </div>
      <div className="metric">
        <span className="label">Mid Price</span>
        <span className="value">{formatPrice(metrics.mid_price)}</span>
      </div>
      <div className="metric">
        <span className="label">Imbalance</span>
        <span className="value">{imbalancePercent}% ({imbalanceLabel})</span>
      </div>
    </div>
  );
}
```

**File**: `ui/src/components/PriceLevels.tsx`

```typescript
const PRICE_SCALE = 1000;
const SIZE_SCALE = 100;

interface Props {
  bids: [number, number][];
  asks: [number, number][];
}

export function PriceLevels({ bids, asks }: Props) {
  const formatPrice = (price: number) => (price / PRICE_SCALE).toFixed(3);
  const formatSize = (size: number) => (size / SIZE_SCALE).toFixed(2);

  // Find max size for bar width calculation
  const maxSize = Math.max(
    ...bids.map(([, s]) => s),
    ...asks.map(([, s]) => s),
    1
  );

  return (
    <div className="price-levels">
      <div className="asks">
        <h3>Asks</h3>
        {asks.slice().reverse().map(([price, size], i) => (
          <div key={i} className="level ask">
            <div
              className="bar"
              style={{ width: `${(size / maxSize) * 100}%` }}
            />
            <span className="price">{formatPrice(price)}</span>
            <span className="size">{formatSize(size)}</span>
          </div>
        ))}
      </div>
      <div className="bids">
        <h3>Bids</h3>
        {bids.map(([price, size], i) => (
          <div key={i} className="level bid">
            <div
              className="bar"
              style={{ width: `${(size / maxSize) * 100}%` }}
            />
            <span className="price">{formatPrice(price)}</span>
            <span className="size">{formatSize(size)}</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**File**: `ui/src/components/OrderbookView.tsx`

```typescript
import { useOrderbookStream } from '../hooks/useOrderbookStream';
import { Metrics } from './Metrics';
import { PriceLevels } from './PriceLevels';
import type { Market } from '../types';

interface Props {
  assetId: string;
  market: Market;
}

export function OrderbookView({ assetId, market }: Props) {
  const { data, error, connected } = useOrderbookStream(assetId);

  const token = market.tokens.find(t => t.token_id === assetId);

  return (
    <div className="orderbook-view">
      <div className="header">
        <h2>{market.question}</h2>
        <span className="outcome">{token?.outcome}</span>
        <span className={`status ${connected ? 'connected' : 'disconnected'}`}>
          {connected ? 'Live' : 'Disconnected'}
        </span>
      </div>

      {error && <div className="error">{error}</div>}

      {data && (
        <>
          <Metrics metrics={data.metrics} />
          <PriceLevels bids={data.bids} asks={data.asks} />
        </>
      )}
    </div>
  );
}
```

#### 5.7 Main App

**File**: `ui/src/App.tsx`

```typescript
import { useState } from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { MarketList } from './components/MarketList';
import { OrderbookView } from './components/OrderbookView';
import type { Market } from './types';
import './App.css';

const queryClient = new QueryClient();

function AppContent() {
  const [selectedAssetId, setSelectedAssetId] = useState<string | null>(null);
  const [selectedMarket, setSelectedMarket] = useState<Market | null>(null);

  const handleSelectAsset = (assetId: string, market: Market) => {
    setSelectedAssetId(assetId);
    setSelectedMarket(market);
  };

  return (
    <div className="app">
      <h1>PolyPy Market Viewer</h1>
      <MarketList
        selectedAssetId={selectedAssetId}
        onSelectAsset={handleSelectAsset}
      />
      {selectedAssetId && selectedMarket && (
        <OrderbookView
          assetId={selectedAssetId}
          market={selectedMarket}
        />
      )}
    </div>
  );
}

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <AppContent />
    </QueryClientProvider>
  );
}
```

#### 5.8 Styles

**File**: `ui/src/App.css`

```css
.app {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: system-ui, -apple-system, sans-serif;
}

.market-list select {
  width: 100%;
  padding: 8px;
  font-size: 14px;
  margin-bottom: 20px;
}

.orderbook-view .header {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 16px;
}

.orderbook-view .header h2 {
  margin: 0;
  font-size: 18px;
}

.orderbook-view .outcome {
  background: #e0e0e0;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
}

.orderbook-view .status {
  margin-left: auto;
  font-size: 12px;
}

.status.connected { color: green; }
.status.disconnected { color: red; }

.metrics {
  display: flex;
  gap: 24px;
  margin-bottom: 20px;
  padding: 12px;
  background: #f5f5f5;
  border-radius: 8px;
}

.metric {
  display: flex;
  flex-direction: column;
}

.metric .label {
  font-size: 12px;
  color: #666;
}

.metric .value {
  font-size: 18px;
  font-weight: 600;
}

.price-levels {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}

.price-levels h3 {
  margin: 0 0 8px 0;
  font-size: 14px;
  color: #666;
}

.level {
  display: flex;
  align-items: center;
  padding: 4px 8px;
  margin-bottom: 2px;
  position: relative;
  font-family: monospace;
}

.level .bar {
  position: absolute;
  left: 0;
  top: 0;
  height: 100%;
  opacity: 0.2;
}

.bid .bar { background: #22c55e; }
.ask .bar { background: #ef4444; }

.level .price {
  flex: 1;
  z-index: 1;
}

.level .size {
  z-index: 1;
  color: #666;
}

.error {
  color: red;
  padding: 12px;
  background: #fee;
  border-radius: 4px;
  margin-bottom: 16px;
}
```

#### 5.9 Vite Config for CORS

**File**: `ui/vite.config.ts`

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
})
```

#### 5.10 Update API Client for Proxy

**File**: `ui/src/api/client.ts`

```typescript
import axios from 'axios';

// Use relative path - Vite proxy handles the rest
export const api = axios.create({
  baseURL: '/api',
});
```

#### 5.11 Update WebSocket Hook for Proxy

**File**: `ui/src/hooks/useOrderbookStream.ts`

```typescript
// Change WebSocket URL to use relative path
const ws = new WebSocket(`ws://${window.location.host}/ws/orderbook`);
```

#### 5.12 Add npm Scripts to justfile

**File**: `justfile`
**Changes**: Add frontend commands

```makefile
# Frontend development
ui-install:
    cd ui && npm install

ui-dev:
    cd ui && npm run dev

ui-build:
    cd ui && npm run build
```

### Success Criteria

#### Automated Verification:
- [ ] Frontend builds: `cd ui && npm run build`
- [ ] TypeScript compiles: `cd ui && npm run typecheck`
- [ ] No lint errors: `cd ui && npm run lint`

#### Manual Verification:
- [ ] Start backend: `just run`
- [ ] Start frontend: `cd ui && npm run dev`
- [ ] Navigate to http://localhost:3000
- [ ] Market dropdown loads with subscribed markets
- [ ] Selecting a market shows live orderbook updates
- [ ] Metrics (spread, mid_price, imbalance) update in real-time
- [ ] Bid/ask levels display with size bars

**Implementation Note**: After completing this phase and all verification passes, the UI visualization feature is complete.

---

## Testing Strategy

### Unit Tests

**Phase 1:**
- Test `LifecycleController.get_market_metadata()` returns stored info
- Test `LifecycleController.get_all_markets()` returns list
- Test `/markets` endpoint returns correct structure

**Phase 2:**
- Test `OrderbookRequest/OrderbookResponse` serialization
- Test worker handles `OrderbookRequest` and responds
- Test `WorkerManager.query_orderbook()` returns data
- Test `/orderbook/{asset_id}` endpoint

**Phase 3:**
- Test `OrderbookState.get_history()` returns snapshots
- Test ring buffer limits to 100 entries
- Test snapshot interval respects timing

**Phase 4:**
- Test WebSocket connection lifecycle
- Test subscribe/unsubscribe messages
- Test streaming interval

**Phase 5:**
- Component render tests with React Testing Library
- Mock WebSocket for stream testing

### Integration Tests

- Start full application, connect frontend, verify end-to-end flow

### Manual Testing Steps

1. Start application with `just run`
2. Wait for market discovery (check logs)
3. Start frontend with `cd ui && npm run dev`
4. Open browser to http://localhost:3000
5. Select a market from dropdown
6. Verify orderbook displays and updates
7. Verify metrics display correctly
8. Test switching between markets

## Performance Considerations

- **Worker queries**: Request/response adds latency (~10-50ms). For streaming, this is acceptable at 500ms intervals.
- **History buffer**: 100 snapshots at ~100 bytes each = ~10KB per orderbook. With 1000 orderbooks = ~10MB total.
- **WebSocket streaming**: Single client means no fan-out complexity. Worker query per interval is acceptable.

## Migration Notes

- No database migrations required
- No breaking changes to existing APIs
- Frontend is new directory, doesn't affect backend

## References

- Research document: `thoughts/shared/research/2025-12-14-ui-visualization-options.md`
- HTTP Server: `src/server/server.py`
- Worker implementation: `src/worker/worker.py`
- Orderbook state: `src/orderbook/orderbook.py`
- aiohttp WebSocket docs: https://docs.aiohttp.org/en/stable/web_quickstart.html#websockets
