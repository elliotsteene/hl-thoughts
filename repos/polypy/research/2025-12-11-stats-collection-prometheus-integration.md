---
date: 2025-12-11T09:47:54+05:30
researcher: Elliot Steene
git_commit: cddc12261cd1bed454223f179daf3d9b92eb32f0
branch: main
repository: polypy
topic: "How stats are currently collected and how it might integrate with Prometheus"
tags: [research, codebase, stats, metrics, prometheus, monitoring, http-api]
status: complete
last_updated: 2025-12-11
last_updated_by: Elliot Steene
---

# Research: How Stats Are Currently Collected and Prometheus Integration

**Date**: 2025-12-11 09:47:54 IST
**Researcher**: Elliot Steene
**Git Commit**: cddc12261cd1bed454223f179daf3d9b92eb32f0
**Branch**: main
**Repository**: polypy

## Research Question

How are application statistics currently collected across the PolyPy codebase, and how might this architecture integrate with Prometheus metrics?

## Summary

The PolyPy application uses a comprehensive stats collection architecture with:

1. **Stats Dataclasses**: Five dataclass-based stats structures (`ConnectionStats`, `WorkerStats`, `RouterStats`, `RecycleStats`, `OrderbookStore` memory tracking)
2. **Component-Level Collection**: Each major component (connections, workers, router, recycler, registry) maintains its own statistics
3. **Centralized Aggregation**: `PolyPy.get_stats()` method aggregates all component stats into a single dict
4. **HTTP Exposure**: `/stats` endpoint exposes aggregated stats as JSON via aiohttp server
5. **Multiprocessing Queue Pattern**: Worker processes report stats to parent via non-blocking queue

The architecture is well-suited for Prometheus integration as:
- All metrics are already collected at component level
- Stats are numeric (counters, gauges, derived metrics)
- HTTP server infrastructure exists for `/metrics` endpoint
- Clear separation between stat collection and exposure

## Detailed Findings

### Stats Data Structures

#### ConnectionStats
**Location**: `src/connection/stats.py:1-28`

Fields tracked per WebSocket connection:
- `messages_received: int` - Total messages received
- `bytes_received: int` - Total bytes received
- `parse_errors: int` - Number of parse errors
- `reconnect_count: int` - Number of reconnections
- `last_message_ts: float` - Timestamp of last message (monotonic)
- `created_at: float` - Connection creation time (monotonic)

Computed properties:
- `uptime: float` - Seconds since creation
- `message_rate: float` - Messages per second (lifetime average)

**Prometheus mapping potential**:
- Counter: `messages_received`, `bytes_received`, `parse_errors`, `reconnect_count`
- Gauge: `uptime`
- Gauge (derived): `message_rate`

#### WorkerStats
**Location**: `src/worker.py:36-53`

Fields tracked per worker process:
- `messages_processed: int` - Total messages processed
- `updates_applied: int` - Price level updates applied
- `snapshots_received: int` - Full orderbook snapshots
- `processing_time_ms: float` - Accumulated processing time
- `last_message_ts: float` - Last activity timestamp
- `orderbook_count: int` - Number of orderbooks managed
- `memory_usage_bytes: int` - Worker memory usage

Computed properties:
- `avg_processing_time_us: float` - Average per-message processing time in microseconds

**Prometheus mapping potential**:
- Counter: `messages_processed`, `updates_applied`, `snapshots_received`
- Summary/Histogram: `processing_time_ms` (can track distribution)
- Gauge: `orderbook_count`, `memory_usage_bytes`
- Gauge (derived): `avg_processing_time_us`

#### RouterStats
**Location**: `src/router.py:42-58`

Fields tracked for message routing:
- `messages_routed: int` - Messages successfully routed
- `messages_dropped: int` - Messages dropped (worker queue full)
- `batches_sent: int` - Batches sent to workers
- `queue_full_events: int` - Backpressure events
- `routing_errors: int` - Routing failures
- `total_latency_ms: float` - Accumulated routing latency

Computed properties:
- `avg_latency_ms: float` - Average routing latency

**Prometheus mapping potential**:
- Counter: `messages_routed`, `messages_dropped`, `batches_sent`, `queue_full_events`, `routing_errors`
- Summary/Histogram: `total_latency_ms` (routing latency distribution)
- Gauge (derived): `avg_latency_ms`

#### RecycleStats
**Location**: `src/lifecycle/recycler.py:24-54`

Fields tracked for connection recycling:
- `recycles_initiated: int` - Recycle operations started
- `recycles_completed: int` - Recycle operations completed
- `recycles_failed: int` - Recycle operation failures
- `markets_migrated: int` - Markets migrated during recycles
- `total_downtime_ms: float` - Accumulated downtime during recycles

Computed properties:
- `success_rate: float` - Percentage of successful recycles (1.0 default)
- `avg_downtime_ms: float` - Average downtime per completed recycle

**Prometheus mapping potential**:
- Counter: `recycles_initiated`, `recycles_completed`, `recycles_failed`, `markets_migrated`
- Summary/Histogram: `total_downtime_ms` (downtime distribution)
- Gauge (derived): `success_rate`, `avg_downtime_ms`

#### OrderbookStore Memory Tracking
**Location**: `src/orderbook/orderbook_store.py` (has `memory_usage()` method)

Provides memory footprint of all managed orderbooks.

**Prometheus mapping potential**:
- Gauge: Total orderbook memory usage

### Component-Level Stats Collection

#### WebSocket Connections
**Location**: `src/connection/websocket.py:49-81, 233-237`

Pattern:
- Creates `ConnectionStats` instance in `__init__`
- Exposes via read-only `stats` property
- Updates inline in `_track_stats()` method during message receipt
- Simple counter increments for high-frequency updates

```python
self._stats: ConnectionStats = ConnectionStats()

@property
def stats(self) -> ConnectionStats:
    return self._stats

def _track_stats(self, message: bytes) -> None:
    self._stats.last_message_ts = time.monotonic()
    self._stats.messages_received += 1
    self._stats.bytes_received += len(message)
```

#### Worker Processes (Multiprocessing)
**Location**: `src/worker.py:144-239, 392-412`

Pattern:
- Each worker process maintains its own `WorkerStats` instance
- Workers report stats periodically (every `STATS_INTERVAL` seconds) via multiprocessing queue
- Non-blocking `put_nowait()` to avoid stalling worker on full queue
- Parent process drains queue with `get_nowait()` in `WorkerManager.get_stats()`
- Latest stats from each worker kept (newer overwrites older)
- Final stats sent on shutdown

```python
# In worker process
if now - last_stats_report >= STATS_INTERVAL:
    try:
        stats_queue.put_nowait((worker_id, stats))
    except Full:
        pass  # Drop stats if queue full

# In parent process
def get_stats(self) -> dict[int, WorkerStats]:
    stats: dict[int, WorkerStats] = {}
    while True:
        try:
            worker_id, worker_stats = self._stats_queue.get_nowait()
            stats[worker_id] = worker_stats
        except Empty:
            break
    return stats
```

**Key insight for Prometheus**: Worker stats collection is asynchronous and non-blocking. Prometheus scrapes would need to work with potentially stale worker stats (updated every STATS_INTERVAL).

#### Router
**Location**: `src/router.py:139`

Pattern:
- Maintains `RouterStats` instance
- Updates inline during routing operations
- Exposes via read-only `stats` property

#### Recycler
**Location**: `src/lifecycle/recycler.py:100`

Pattern:
- Maintains `RecycleStats` instance
- Updates during recycle operations
- Exposes via read-only `stats` property

### Centralized Stats Aggregation

**Location**: `src/app.py:292-382`

The `PolyPy.get_stats()` method serves as the single aggregation point:

```python
def get_stats(self) -> dict[str, Any]:
    """Get comprehensive application statistics."""

    stats: dict[str, Any] = {
        "running": self._running,
        "registry": {},
        "pool": {},
        "router": {},
        "workers": {},
        "recycler": {},
    }

    # Collects from each component with null-safety
    if self._registry:
        stats["registry"] = {
            "total_markets": len(self._registry._assets),
            "pending": self._registry.get_pending_count(),
            "subscribed": len(self._registry.get_by_status(AssetStatus.SUBSCRIBED)),
            "expired": len(self._registry.get_by_status(AssetStatus.EXPIRED)),
        }

    if self._pool:
        stats["pool"] = {
            "connection_count": self._pool.connection_count,
            "active_connections": self._pool.active_connection_count,
            "total_capacity": self._pool.get_total_capacity(),
            "stats": self._pool.get_connection_stats(),
        }

    # ... similar for router, workers, lifecycle, recycler

    return stats
```

**Structure of returned dict**:
- `running: bool` - Application running state
- `registry: dict` - Market/asset registry stats
- `pool: dict` - Connection pool stats (includes per-connection breakdown)
- `router: dict` - Message routing stats
- `workers: dict` - Per-worker stats with alive count and health
- `lifecycle: dict` - Lifecycle controller stats
- `recycler: dict` - Connection recycler stats

### HTTP Endpoints

**Location**: `src/server/server.py`

#### HTTPServer Class
**Lines**: 17-157

The HTTP server uses aiohttp and follows the async component lifecycle pattern:

- **Host/Port**: Default `0.0.0.0:8080` (container-compatible)
- **Lifecycle**: `start()` and `stop()` methods for graceful management
- **Routes**: Two endpoints registered in `start()`

#### GET /health Endpoint
**Lines**: 107-145

Returns component health breakdown:

```json
{
  "healthy": true,
  "timestamp": "2025-12-11T09:47:54.123456+00:00",
  "components": {
    "registry": true,
    "pool": true,
    "router": true,
    "workers": true,
    "lifecycle": true,
    "recycler": true
  }
}
```

- **Status codes**: 200 (healthy) or 503 (unhealthy)
- **Health logic**: Checks if each component is initialized and running
- **Timestamp**: ISO format with UTC timezone

#### GET /stats Endpoint
**Lines**: 147-157

Returns comprehensive application statistics:

```python
async def _handle_stats(self, request: web.Request) -> web.Response:
    """Handle GET /stats endpoint."""
    logger.debug("GET /stats")

    stats = self._app.get_stats()
    return web.json_response(stats, status=200)
```

- **Status code**: Always 200
- **Response**: JSON from `app.get_stats()`
- **No authentication**: Currently open endpoint

### Connection Pool Stats

**Location**: `src/connection/pool.py:313-339`

The connection pool provides detailed per-connection statistics:

```python
def get_connection_stats(self) -> list[dict]:
    """Get stats for all connections."""
    result = []

    for cid, info in self._connections.items():
        conn_stats = info.connection.stats
        registry_stats = self._registry.connection_stats(cid)

        result.append({
            "connection_id": cid,
            "status": info.connection.status.name,
            "is_draining": info.is_draining,
            "age_seconds": info.age,
            "messages_received": conn_stats.messages_received,
            "bytes_received": conn_stats.bytes_received,
            "parse_errors": conn_stats.parse_errors,
            "reconnect_count": conn_stats.reconnect_count,
            "is_healthy": info.connection.is_healthy,
            "total_markets": registry_stats["total"],
            "subscribed_markets": registry_stats["subscribed"],
            "expired_markets": registry_stats["expired"],
            "pollution_ratio": registry_stats["pollution_ratio"],
        })

    return result
```

This merges:
- Connection-level stats (`ConnectionStats`)
- Registry stats (per-connection market subscriptions)
- Connection metadata (age, status, health, draining state)

### Asset Registry Stats

**Location**: `src/registry/asset_registry.py:86`

Provides per-connection subscription statistics:

```python
def connection_stats(self, connection_id: str) -> dict[str, int | float]:
    """Get connection-specific statistics."""
    return {
        "total": total_assets,
        "subscribed": subscribed_count,
        "expired": expired_count,
        "pollution_ratio": expired_count / total_assets if total_assets > 0 else 0.0,
    }
```

**Pollution ratio**: Percentage of expired markets on a connection (indicates need for recycling)

## Prometheus Integration Analysis

### Current Stats → Prometheus Metric Types

The existing stats map naturally to Prometheus metric types:

#### Counters (monotonically increasing)
- All `*_received`, `*_processed`, `*_applied` fields
- All error counts (`parse_errors`, `routing_errors`, `recycles_failed`)
- Event counts (`recycles_initiated`, `queue_full_events`)
- Byte counters (`bytes_received`)

#### Gauges (point-in-time values)
- Connection counts (`connection_count`, `active_connections`)
- Queue depths (`queue_depths` from router)
- Worker counts (`alive_count`)
- Orderbook counts (`orderbook_count`)
- Memory usage (`memory_usage_bytes`)
- Current state flags (`running`, `is_healthy`, `is_draining`)

#### Summaries/Histograms (distributions)
- Processing times (`processing_time_ms`, `avg_processing_time_us`)
- Latencies (`total_latency_ms`, `avg_latency_ms`)
- Downtime (`total_downtime_ms`, `avg_downtime_ms`)

#### Derived Metrics (computed from counters)
- Rates: `message_rate`, `success_rate`
- Averages: `avg_processing_time_us`, `avg_latency_ms`, `avg_downtime_ms`
- Ratios: `pollution_ratio`

**Note**: Prometheus can compute rates and averages from raw counters using PromQL, so derived metrics may be redundant.

### Architectural Advantages for Prometheus

1. **HTTP Server Infrastructure Exists**: Already has aiohttp server with endpoint routing
2. **Centralized Aggregation**: Single `get_stats()` method collects from all components
3. **Component Isolation**: Each component manages its own stats independently
4. **Numeric Metrics**: All stats are numeric (int/float), no string aggregation
5. **Consistent Patterns**: All stats use dataclass pattern with slots
6. **Non-Blocking Collection**: Worker stats use non-blocking queue pattern
7. **Health Checks**: Existing `/health` endpoint complements Prometheus monitoring

### Implementation Considerations

#### Label Strategy

The current per-connection and per-worker stats suggest these label dimensions:

- `connection_id`: For connection-level metrics
- `worker_id`: For worker-level metrics
- `status`: For connection status (CONNECTED, DISCONNECTED, etc.)
- `component`: For component-level aggregation (registry, pool, router, workers, recycler)

Example Prometheus metrics:
```
polypy_messages_received_total{connection_id="conn-0"} 1234
polypy_messages_processed_total{worker_id="0"} 5678
polypy_worker_orderbook_count{worker_id="0"} 150
```

#### Cardinality Management

Current architecture has bounded cardinality:
- Connections: Limited by pool size (configurable, typically 10-20)
- Workers: Fixed at startup (typically 2-8 based on CPU cores)
- Markets: Thousands, but tracked in aggregate (not per-market labels)

**Concern**: Per-connection and per-market labels could create high cardinality if connections churn frequently or if per-market metrics are exposed. Should aggregate at component level for high-level metrics.

#### Metric Naming Convention

Suggested Prometheus naming following best practices:

```
polypy_connection_messages_received_total
polypy_connection_bytes_received_total
polypy_connection_parse_errors_total
polypy_connection_reconnect_count_total
polypy_connection_uptime_seconds

polypy_worker_messages_processed_total
polypy_worker_updates_applied_total
polypy_worker_processing_time_seconds
polypy_worker_orderbook_count
polypy_worker_memory_bytes

polypy_router_messages_routed_total
polypy_router_messages_dropped_total
polypy_router_queue_full_events_total
polypy_router_latency_seconds

polypy_recycler_operations_total{result="completed"}
polypy_recycler_operations_total{result="failed"}
polypy_recycler_markets_migrated_total
polypy_recycler_downtime_seconds

polypy_registry_markets_total{status="pending"}
polypy_registry_markets_total{status="subscribed"}
polypy_registry_markets_total{status="expired"}

polypy_pool_connections_active
polypy_pool_connection_capacity
```

#### Multiprocessing Queue Pattern

The worker stats collection pattern (non-blocking queue with periodic updates) means:
- Worker metrics may be slightly stale (up to STATS_INTERVAL seconds old)
- Prometheus scrapes would see the most recently reported worker stats
- This is acceptable for most monitoring use cases
- No changes needed to existing pattern

#### Integration Points

Two approaches for Prometheus integration:

**Option A: Replace /stats with /metrics**
- Remove existing `/stats` JSON endpoint
- Add `/metrics` endpoint returning Prometheus text format
- Transform `get_stats()` output to Prometheus format

**Option B: Add /metrics alongside /stats**
- Keep `/stats` for debugging and direct inspection
- Add `/metrics` for Prometheus scraping
- Both endpoints share same `get_stats()` data source

**Recommendation**: Option B provides backward compatibility and debugging flexibility.

### Extensibility

The current architecture supports easy metric additions:

1. **New Component Stats**: Add new dataclass in component, expose via property
2. **New Fields**: Add fields to existing stats dataclasses
3. **Aggregation**: Update `app.get_stats()` to include new component
4. **Exposure**: Prometheus exporter auto-discovers new fields via inspection

The dataclass pattern with slots provides:
- Clear schema for each component's metrics
- Type safety (int/float enforcement)
- Memory efficiency (slots reduce overhead)
- Easy serialization (dataclass fields are introspectable)

## Code References

- `src/server/server.py:147-157` - Current `/stats` endpoint implementation
- `src/app.py:292-382` - Central stats aggregation in `get_stats()`
- `src/connection/stats.py:1-28` - ConnectionStats dataclass
- `src/worker.py:36-53` - WorkerStats dataclass
- `src/router.py:42-58` - RouterStats dataclass
- `src/lifecycle/recycler.py:24-54` - RecycleStats dataclass
- `src/connection/websocket.py:233-237` - Stats tracking in connections
- `src/worker.py:144-239` - Worker stats reporting via queue
- `src/worker.py:392-412` - Worker stats collection in parent
- `src/connection/pool.py:313-339` - Per-connection stats aggregation
- `src/registry/asset_registry.py:86` - Registry stats per connection

## Historical Context (from thoughts/)

No prior research documents found related to metrics, Prometheus, or monitoring architecture.

## Related Research

No related research documents currently exist in the thoughts directory.

## Open Questions

1. **Scrape Interval**: What should be the Prometheus scrape interval relative to worker STATS_INTERVAL?
2. **Authentication**: Should `/metrics` endpoint require authentication?
3. **Per-Market Metrics**: Should individual market metrics be exposed (high cardinality concern)?
4. **Histogram Buckets**: What latency/timing buckets are appropriate for processing time and routing latency?
5. **Metric Retention**: How long should Prometheus retain metrics (affects storage)?
6. **Alerting Rules**: What thresholds should trigger alerts (error rates, latencies, queue depths)?
7. **Dashboard Design**: What Grafana dashboards would be most useful for operators?

## Recommendations for Implementation

While this research focuses on documenting the current state, the following observations may inform future work:

1. **Direct Metric Mapping**: Most stats can map 1:1 to Prometheus metrics with appropriate types
2. **Label Cardinality**: Use connection_id and worker_id labels, avoid per-market labels
3. **Backward Compatibility**: Keep `/stats` endpoint while adding `/metrics`
4. **Library Choice**: Use `prometheus_client` Python library for Prometheus exposition
5. **Worker Stats Staleness**: Current non-blocking queue pattern is acceptable for Prometheus
6. **Health Integration**: Use existing `/health` endpoint for liveness/readiness probes
7. **Metric Naming**: Follow Prometheus naming conventions (units in suffix, `_total` for counters)
