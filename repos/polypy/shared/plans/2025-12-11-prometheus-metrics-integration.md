# Prometheus Metrics Integration Implementation Plan

## Overview

Add Prometheus metrics exposition to the PolyPy application by creating a `/metrics` HTTP endpoint that transforms existing application statistics into Prometheus text format. This enables standard Prometheus scraping for monitoring, alerting, and observability without changing the existing stats collection architecture.

## Current State Analysis

The application already has comprehensive stats collection infrastructure:

### Existing Stats Architecture

1. **Stats Dataclasses** (`src/connection/stats.py`, `src/worker.py`, `src/router.py`, `src/lifecycle/recycler.py`):
   - `ConnectionStats`: messages_received, bytes_received, parse_errors, reconnect_count
   - `WorkerStats`: messages_processed, updates_applied, snapshots_received, processing_time_ms, orderbook_count, memory_usage_bytes
   - `RouterStats`: messages_routed, messages_dropped, batches_sent, queue_full_events, routing_errors, total_latency_ms
   - `RecycleStats`: recycles_initiated, recycles_completed, recycles_failed, markets_migrated, total_downtime_ms

2. **Centralized Aggregation** (`src/app.py:292-382`):
   - `PolyPy.get_stats()` method aggregates all component stats into a single dictionary
   - Null-safe pattern: checks each component before accessing
   - Returns structured dict with registry, pool, router, workers, lifecycle, recycler sections

3. **HTTP Server** (`src/server/server.py`):
   - aiohttp-based server with `/health` and `/stats` endpoints
   - Routes registered during `start()` method
   - Already has `prometheus-client>=0.23.1` as a dependency (line 11 in `pyproject.toml`)

### Current Limitations

- Stats are only available as JSON via `/stats` endpoint
- No Prometheus-compatible metrics exposition
- Cannot use Prometheus scraping, querying (PromQL), or alerting
- Manual correlation required between different stat types

## Desired End State

### Functionality

A fully functional `/metrics` endpoint that:
- Exposes all existing stats as Prometheus metrics in text exposition format
- Uses prometheus_client's built-in registry for metric generation
- Generates fresh metrics on each scrape (stateless, follows Prometheus best practices)
- Supports standard Prometheus scraping and PromQL queries
- Enables alerting on key metrics (error rates, latencies, queue depths)

### Metric Types Mapping

**Counters** (monotonically increasing):
- `polypy_connection_messages_received_total{connection_id}`
- `polypy_connection_bytes_received_total{connection_id}`
- `polypy_connection_parse_errors_total{connection_id}`
- `polypy_connection_reconnects_total{connection_id}`
- `polypy_worker_messages_processed_total{worker_id}`
- `polypy_worker_updates_applied_total{worker_id}`
- `polypy_worker_snapshots_received_total{worker_id}`
- `polypy_router_messages_routed_total`
- `polypy_router_messages_dropped_total`
- `polypy_router_batches_sent_total`
- `polypy_router_queue_full_events_total`
- `polypy_router_routing_errors_total`
- `polypy_recycler_recycles_total{result="completed"|"failed"}`
- `polypy_recycler_markets_migrated_total`

**Gauges** (point-in-time values):
- `polypy_connection_message_rate{connection_id}` - messages/second
- `polypy_worker_orderbook_count{worker_id}`
- `polypy_worker_memory_bytes{worker_id}`
- `polypy_router_queue_depth{queue_name}`
- `polypy_recycler_success_rate` - ratio 0.0-1.0
- `polypy_recycler_active_recycles` - current count
- `polypy_registry_markets_total{status}` - with status labels: pending, subscribed, expired
- `polypy_pool_connections{type}` - with type labels: total, active
- `polypy_pool_capacity` - total capacity
- `polypy_workers_alive` - alive worker count

**Summaries** (latency distributions):
- `polypy_worker_processing_seconds{worker_id}` - from processing_time_ms
- `polypy_router_routing_latency_seconds` - from total_latency_ms
- `polypy_recycler_downtime_seconds` - from total_downtime_ms

### Verification

**Automated**:
- `just check` passes (linting and formatting)
- `uv run pytest` passes (all tests including new metrics tests)
- Endpoint returns valid Prometheus text format
- Metrics values match those from `/stats` endpoint

**Manual**:
- `/metrics` endpoint returns HTTP 200
- Prometheus can scrape the endpoint successfully
- Metrics appear in Prometheus UI with correct labels
- PromQL queries work as expected (rate(), sum(), histogram_quantile())
- Grafana dashboards can visualize the metrics

## What We're NOT Doing

1. **Not changing existing stats collection** - All current dataclasses, update patterns, and aggregation logic remain unchanged
2. **Not adding authentication** - `/metrics` endpoint will be open like `/health` and `/stats`
3. **Not adding per-market metrics** - Would create excessive cardinality (thousands of markets)
4. **Not implementing push gateway** - Using standard pull-based scraping model
5. **Not adding alerting rules** - Those belong in Prometheus server configuration
6. **Not configuring retention policies** - Handled by Prometheus server, not the application
7. **Not replacing `/stats` endpoint** - Keep existing JSON endpoint for debugging and backward compatibility
8. **Not tracking worker stats staleness** - 30-second reporting interval is acceptable
9. **Not using histograms** - User specified summaries for latency metrics
10. **Not implementing distributed metrics** - Single-instance metrics only (no aggregation across multiple PolyPy instances)

## Implementation Approach

### Strategy

Use a **"transformation layer"** pattern:
1. Keep existing stats collection completely unchanged
2. Create a new `MetricsCollector` class that transforms `get_stats()` output to Prometheus format
3. Generate metrics fresh on each `/metrics` request (stateless, no persistent metric objects)
4. Use prometheus_client's `CollectorRegistry` and `generate_latest()` for standard exposition

### Why This Approach

- **Minimal changes**: No modifications to core stats collection logic
- **Follows existing patterns**: Matches how `/stats` works (calls `get_stats()` per request)
- **Testable**: Easy to unit test the transformation logic
- **Follows Prometheus best practices**: Stateless metric generation, fresh data on each scrape
- **Low risk**: Transformation layer is isolated, failures don't affect core application
- **Maintainable**: New stats automatically exposed if added to `get_stats()`

## Stacked PR Strategy

This plan is designed for implementation using stacked PRs, where each phase becomes one branch/PR in a stack:

- **Phase Sequencing**: 4 phases total, each builds on the previous, showing the implementation journey
  - Phase 0: Docker infrastructure (Dockerfile, docker-compose.yml, Prometheus config)
  - Phase 1: Core metrics module and `/metrics` endpoint
  - Phase 2: Per-connection and per-worker labeled metrics
  - Phase 3: Worker processing time summaries
- **Independent Review**: Each PR is independently testable and deployable
- **Pause Points**: Manual verification after each phase before proceeding
- **Estimated Scope**: Each phase is roughly <300 lines for optimal reviewability
- **Stack Management**: Use the `stacked-pr` skill after completing each phase

---

## Phase 0: Docker Infrastructure Setup

### Overview
Create Docker infrastructure to run the PolyPy application alongside Prometheus for local development and testing. This enables easy testing of the Prometheus integration without requiring separate installation of services.

### PR Context
**Stack Position**: First PR in the stack (base: main)
**Purpose**: Establish containerized development environment for Prometheus integration testing
**Enables**: Phase 1 will add the `/metrics` endpoint that Prometheus will scrape
**Review Focus**: Dockerfile best practices, docker-compose service configuration, Prometheus scrape config

### Changes Required

#### 1. Create Dockerfile
**File**: `Dockerfile`
**Changes**: Add new file with multi-stage build for optimal image size

```dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM python:3.14-slim AS builder

# Install uv
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Set working directory
WORKDIR /app

# Copy dependency files
COPY pyproject.toml uv.lock ./

# Install dependencies to virtual environment
RUN uv sync --frozen --no-dev

# Runtime stage
FROM python:3.14-slim

# Set working directory
WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /app/.venv /app/.venv

# Copy application source
COPY src ./src

# Set PATH to use virtual environment
ENV PATH="/app/.venv/bin:$PATH"

# Expose HTTP server port
EXPOSE 8080

# Run the application
CMD ["python", "src/main.py"]
```

#### 2. Create Docker Compose Configuration
**File**: `docker-compose.yml`
**Changes**: Add new file with services for polypy and prometheus

```yaml
version: '3.8'

services:
  polypy:
    build: .
    container_name: polypy
    ports:
      - "8080:8080"
    environment:
      - LOG_LEVEL=INFO
      - HTTP_PORT=8080
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=1h'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    networks:
      - monitoring
    depends_on:
      polypy:
        condition: service_healthy

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
```

#### 3. Create Prometheus Configuration
**File**: `prometheus.yml`
**Changes**: Add new file with scrape configuration for polypy

```yaml
# Prometheus configuration for PolyPy monitoring

global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'polypy-dev'

scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # PolyPy application metrics
  - job_name: 'polypy'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['polypy:8080']
        labels:
          service: 'polypy'
          environment: 'development'
    scrape_interval: 15s
    scrape_timeout: 10s

  # PolyPy health check (optional, for monitoring)
  - job_name: 'polypy-health'
    metrics_path: '/health'
    metric_relabel_configs:
      # Drop this since /health returns JSON, not metrics
      - action: drop
        regex: '.*'
    static_configs:
      - targets: ['polypy:8080']
```

#### 4. Update .dockerignore
**File**: `.dockerignore`
**Changes**: Add new file to exclude unnecessary files from Docker build

```
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
*.egg-info/
dist/
build/

# Virtual environments
.venv/
venv/
env/

# IDE
.vscode/
.idea/
*.swp
*.swo

# Git
.git/
.gitignore

# Tests
tests/
pytest.ini
.pytest_cache/

# Documentation
docs/
lessons/
*.md
!README.md

# Thoughts
thoughts/

# Development
.pre-commit-config.yaml
.ruff.toml
justfile

# Docker
Dockerfile
docker-compose.yml
prometheus.yml
.dockerignore
```

#### 5. Add Docker Commands to Justfile
**File**: `justfile`
**Changes**: Add docker-related commands after existing commands

**Add at the end of the file**:
```makefile
# Docker commands
docker-build:
    docker build -t polypy:latest .

docker-up:
    docker-compose up -d

docker-down:
    docker-compose down

docker-logs:
    docker-compose logs -f

docker-restart:
    docker-compose restart

docker-clean:
    docker-compose down -v
    docker rmi polypy:latest || true

# Start full stack (polypy + prometheus)
stack-up: docker-build docker-up
    @echo "✓ Stack is up!"
    @echo "  - PolyPy: http://localhost:8080"
    @echo "  - PolyPy Stats: http://localhost:8080/stats"
    @echo "  - PolyPy Health: http://localhost:8080/health"
    @echo "  - Prometheus: http://localhost:9090"

stack-down: docker-down
    @echo "✓ Stack is down"
```

#### 6. Update README Documentation
**File**: `README.md`
**Changes**: Add Docker section to README

**Add after the "Development Commands" section**:
```markdown
## Docker Development

### Quick Start with Docker Compose

Run the application with Prometheus monitoring:

```bash
# Build and start the stack
just stack-up

# View logs
just docker-logs

# Stop the stack
just stack-down

# Clean up everything (including volumes)
just docker-clean
```

### Services

- **PolyPy**: http://localhost:8080
  - Stats endpoint: http://localhost:8080/stats
  - Health endpoint: http://localhost:8080/health
  - Metrics endpoint: http://localhost:8080/metrics (added in Phase 1)
- **Prometheus**: http://localhost:9090
  - Scrapes PolyPy metrics every 15 seconds
  - 1 hour data retention

### Docker Commands

```bash
# Build image only
just docker-build

# Start services
just docker-up

# Stop services
just docker-down

# View logs
just docker-logs

# Restart services
just docker-restart
```
```

### Success Criteria

#### Automated Verification:
- [ ] Dockerfile builds successfully: `docker build -t polypy:latest .`
- [ ] Image size is reasonable (<500MB)
- [ ] docker-compose validates: `docker-compose config`
- [ ] Code passes linting: `just check`

#### Manual Verification:
- [ ] Start stack: `just stack-up`
- [ ] Verify polypy container is running: `docker ps | grep polypy`
- [ ] Verify prometheus container is running: `docker ps | grep prometheus`
- [ ] Access PolyPy health endpoint: `curl http://localhost:8080/health`
- [ ] Access PolyPy stats endpoint: `curl http://localhost:8080/stats`
- [ ] Access Prometheus UI at http://localhost:9090
- [ ] Verify Prometheus shows polypy target (Status → Targets)
- [ ] Check polypy target is in "DOWN" state (expected - /metrics doesn't exist yet)
- [ ] View container logs: `just docker-logs`
- [ ] Stop stack cleanly: `just stack-down`
- [ ] Clean up: `just docker-clean`

**Implementation Note**: After completing this phase and all verification passes, pause here for manual confirmation that Docker infrastructure works correctly before proceeding to Phase 1. The polypy target in Prometheus will show as DOWN until Phase 1 adds the `/metrics` endpoint - this is expected.

---

## Phase 1: Core Metrics Module and Basic Endpoint

### Overview
Create the metrics collection infrastructure and add a basic `/metrics` endpoint that exposes high-level application metrics (running status, component counts). This establishes the foundation for more detailed metrics in later phases.

### PR Context
**Stack Position**: Second PR in the stack (base: phase-0-branch)
**Purpose**: Establish Prometheus integration foundation with basic metrics
**Builds On**: Phase 0's Docker infrastructure for testing
**Enables**: Phase 2 will add detailed per-component metrics with labels
**Review Focus**: Metric naming conventions, prometheus_client usage patterns, test coverage

### Changes Required

#### 1. Create Metrics Module
**File**: `src/metrics/__init__.py`
**Changes**: Create empty init file for module

```python
"""Prometheus metrics exposition."""
```

#### 2. Create MetricsCollector Class
**File**: `src/metrics/prometheus.py`
**Changes**: Add new file with MetricsCollector class

```python
"""Prometheus metrics collector for PolyPy application stats."""

from typing import TYPE_CHECKING, Any

from prometheus_client import CollectorRegistry, Counter, Gauge, Summary, generate_latest

if TYPE_CHECKING:
    from src.app import PolyPy


class MetricsCollector:
    """
    Collects application statistics and exposes them as Prometheus metrics.

    Generates fresh metrics on each collection by calling app.get_stats()
    and transforming the results into Prometheus format.
    """

    def __init__(self, app: "PolyPy") -> None:
        """Initialize metrics collector.

        Args:
            app: PolyPy application instance to collect stats from
        """
        self._app = app

    def collect_metrics(self) -> bytes:
        """
        Collect current stats and return Prometheus text format.

        Creates a fresh registry on each call and populates it with
        current application state.

        Returns:
            Prometheus text exposition format bytes
        """
        # Create fresh registry for this scrape
        registry = CollectorRegistry()

        # Get current stats
        stats = self._app.get_stats()

        # Create and populate basic metrics
        self._collect_application_metrics(registry, stats)
        self._collect_registry_metrics(registry, stats)
        self._collect_pool_metrics(registry, stats)
        self._collect_router_metrics(registry, stats)
        self._collect_worker_metrics(registry, stats)
        self._collect_lifecycle_metrics(registry, stats)
        self._collect_recycler_metrics(registry, stats)

        # Generate Prometheus text format
        return generate_latest(registry)

    def _collect_application_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
        """Collect application-level metrics."""
        running = Gauge(
            "polypy_application_running",
            "Whether the application is running (1) or stopped (0)",
            registry=registry,
        )
        running.set(1 if stats.get("running") else 0)

    def _collect_registry_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
        """Collect asset registry metrics."""
        registry_stats = stats.get("registry", {})
        if not registry_stats:
            return

        # Market counts by status
        markets = Gauge(
            "polypy_registry_markets_total",
            "Number of markets by status",
            ["status"],
            registry=registry,
        )
        markets.labels(status="total").set(registry_stats.get("total_markets", 0))
        markets.labels(status="pending").set(registry_stats.get("pending", 0))
        markets.labels(status="subscribed").set(registry_stats.get("subscribed", 0))
        markets.labels(status="expired").set(registry_stats.get("expired", 0))

    def _collect_pool_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
        """Collect connection pool metrics."""
        pool_stats = stats.get("pool", {})
        if not pool_stats:
            return

        # Connection counts
        connections = Gauge(
            "polypy_pool_connections",
            "Number of connections by type",
            ["type"],
            registry=registry,
        )
        connections.labels(type="total").set(pool_stats.get("connection_count", 0))
        connections.labels(type="active").set(pool_stats.get("active_connections", 0))

        # Pool capacity
        capacity = Gauge(
            "polypy_pool_capacity",
            "Total subscription capacity across all connections",
            registry=registry,
        )
        capacity.set(pool_stats.get("total_capacity", 0))

    def _collect_router_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
        """Collect message router metrics."""
        router_stats = stats.get("router", {})
        if not router_stats:
            return

        # Counters
        messages_routed = Counter(
            "polypy_router_messages_routed_total",
            "Total messages successfully routed to workers",
            registry=registry,
        )
        messages_routed._value.set(router_stats.get("messages_routed", 0))

        messages_dropped = Counter(
            "polypy_router_messages_dropped_total",
            "Total messages dropped due to backpressure",
            registry=registry,
        )
        messages_dropped._value.set(router_stats.get("messages_dropped", 0))

        batches_sent = Counter(
            "polypy_router_batches_sent_total",
            "Total batches sent to workers",
            registry=registry,
        )
        batches_sent._value.set(router_stats.get("batches_sent", 0))

        queue_full_events = Counter(
            "polypy_router_queue_full_events_total",
            "Total worker queue full events",
            registry=registry,
        )
        queue_full_events._value.set(router_stats.get("queue_full_events", 0))

        routing_errors = Counter(
            "polypy_router_routing_errors_total",
            "Total routing errors",
            registry=registry,
        )
        routing_errors._value.set(router_stats.get("routing_errors", 0))

        # Queue depths (gauges)
        queue_depths = router_stats.get("queue_depths", {})
        if queue_depths:
            queue_depth = Gauge(
                "polypy_router_queue_depth",
                "Current queue depth by queue name",
                ["queue_name"],
                registry=registry,
            )
            for queue_name, depth in queue_depths.items():
                if depth >= 0:  # Skip -1 values (NotImplementedError on macOS)
                    queue_depth.labels(queue_name=queue_name).set(depth)

        # Routing latency summary
        avg_latency_ms = router_stats.get("avg_latency_ms", 0)
        if avg_latency_ms > 0:
            # Note: Summary requires observations, but we only have aggregate data
            # We'll use Counter for total and compute rate in PromQL
            latency_total = Counter(
                "polypy_router_routing_latency_seconds_total",
                "Total routing latency in seconds",
                registry=registry,
            )
            # Convert avg_latency_ms to total seconds
            messages_routed_count = router_stats.get("messages_routed", 0)
            total_latency_seconds = (avg_latency_ms * messages_routed_count) / 1000.0
            latency_total._value.set(total_latency_seconds)

    def _collect_worker_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
        """Collect worker process metrics."""
        worker_stats = stats.get("workers", {})
        if not worker_stats:
            return

        # Alive worker count
        alive_count = Gauge(
            "polypy_workers_alive",
            "Number of alive worker processes",
            registry=registry,
        )
        alive_count.set(worker_stats.get("alive_count", 0))

        # Healthy status
        healthy = Gauge(
            "polypy_workers_healthy",
            "Whether all workers are healthy (1) or not (0)",
            registry=registry,
        )
        healthy.set(1 if worker_stats.get("is_healthy") else 0)

    def _collect_lifecycle_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
        """Collect lifecycle controller metrics."""
        lifecycle_stats = stats.get("lifecycle", {})
        if not lifecycle_stats:
            return

        # Running status
        running = Gauge(
            "polypy_lifecycle_running",
            "Whether lifecycle controller is running (1) or stopped (0)",
            registry=registry,
        )
        running.set(1 if lifecycle_stats.get("is_running") else 0)

        # Known market count
        known_markets = Gauge(
            "polypy_lifecycle_known_markets",
            "Number of known market condition IDs",
            registry=registry,
        )
        known_markets.set(lifecycle_stats.get("known_market_count", 0))

    def _collect_recycler_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
        """Collect connection recycler metrics."""
        recycler_stats = stats.get("recycler", {})
        if not recycler_stats:
            return

        # Recycle operation counters by result
        recycles = Counter(
            "polypy_recycler_recycles_total",
            "Total recycle operations by result",
            ["result"],
            registry=registry,
        )
        recycles.labels(result="completed")._value.set(
            recycler_stats.get("recycles_completed", 0)
        )
        recycles.labels(result="failed")._value.set(
            recycler_stats.get("recycles_failed", 0)
        )

        # Markets migrated
        markets_migrated = Counter(
            "polypy_recycler_markets_migrated_total",
            "Total markets migrated during recycles",
            registry=registry,
        )
        markets_migrated._value.set(recycler_stats.get("markets_migrated", 0))

        # Success rate gauge
        success_rate = Gauge(
            "polypy_recycler_success_rate",
            "Recycle success rate (0.0 to 1.0)",
            registry=registry,
        )
        success_rate.set(recycler_stats.get("success_rate", 1.0))

        # Active recycles gauge
        active_recycles = Gauge(
            "polypy_recycler_active_recycles",
            "Number of currently active recycle operations",
            registry=registry,
        )
        active_list = recycler_stats.get("active_recycles", [])
        active_recycles.set(len(active_list))

        # Average downtime (convert ms to seconds)
        avg_downtime_ms = recycler_stats.get("avg_downtime_ms", 0)
        if avg_downtime_ms > 0:
            avg_downtime = Gauge(
                "polypy_recycler_avg_downtime_seconds",
                "Average downtime per recycle in seconds",
                registry=registry,
            )
            avg_downtime.set(avg_downtime_ms / 1000.0)
```

#### 3. Add /metrics Endpoint
**File**: `src/server/server.py`
**Changes**: Add metrics endpoint handler and route registration

**At line 69** (after `/stats` route):
```python
web_app.router.add_get("/metrics", self._handle_metrics)
```

**After line 156** (after `_handle_stats` method):
```python
async def _handle_metrics(self, request: web.Request) -> web.Response:
    """Handle GET /metrics endpoint.

    Returns:
        200 OK with Prometheus text exposition format
    """
    logger.debug("GET /metrics")

    # Import here to avoid circular dependency
    from src.metrics.prometheus import MetricsCollector

    # Collect and generate metrics
    collector = MetricsCollector(self._app)
    metrics_bytes = collector.collect_metrics()

    # Return Prometheus text format
    return web.Response(
        body=metrics_bytes,
        status=200,
        content_type="text/plain; version=0.0.4; charset=utf-8",
    )
```

**At line 25** (add to `__slots__`):
No changes needed - we don't store MetricsCollector as it's created fresh per request

#### 4. Add Unit Tests
**File**: `tests/test_metrics_prometheus.py`
**Changes**: Add new test file

```python
"""Tests for Prometheus metrics collection."""

import pytest
from prometheus_client.parser import text_string_to_metric_families

from src.app import PolyPy
from src.metrics.prometheus import MetricsCollector


@pytest.fixture
def mock_stats() -> dict:
    """Mock stats dictionary matching PolyPy.get_stats() structure."""
    return {
        "running": True,
        "registry": {
            "total_markets": 150,
            "pending": 10,
            "subscribed": 120,
            "expired": 20,
        },
        "pool": {
            "connection_count": 5,
            "active_connections": 4,
            "total_capacity": 2500,
            "stats": [],
        },
        "router": {
            "messages_routed": 10000,
            "messages_dropped": 50,
            "batches_sent": 500,
            "queue_full_events": 5,
            "routing_errors": 2,
            "avg_latency_ms": 1.5,
            "queue_depths": {"async_queue": 10, "worker_0": 5},
        },
        "workers": {
            "alive_count": 2,
            "is_healthy": True,
            "worker_stats": {},
        },
        "lifecycle": {
            "is_running": True,
            "known_market_count": 100,
        },
        "recycler": {
            "recycles_initiated": 10,
            "recycles_completed": 8,
            "recycles_failed": 2,
            "success_rate": 0.8,
            "markets_migrated": 500,
            "avg_downtime_ms": 150.0,
            "active_recycles": [],
        },
    }


class TestMetricsCollector:
    """Test MetricsCollector class."""

    def test_collect_metrics_returns_bytes(self, mock_stats):
        """Test that collect_metrics returns bytes."""
        # Create mock app
        class MockApp:
            def get_stats(self):
                return mock_stats

        collector = MetricsCollector(MockApp())
        result = collector.collect_metrics()

        assert isinstance(result, bytes)
        assert len(result) > 0

    def test_metrics_are_valid_prometheus_format(self, mock_stats):
        """Test that output is valid Prometheus text format."""
        class MockApp:
            def get_stats(self):
                return mock_stats

        collector = MetricsCollector(MockApp())
        result = collector.collect_metrics()

        # Parse using prometheus_client parser
        metrics_text = result.decode("utf-8")
        families = list(text_string_to_metric_families(metrics_text))

        # Should have multiple metric families
        assert len(families) > 0

        # Check for expected metric names
        metric_names = {family.name for family in families}
        assert "polypy_application_running" in metric_names
        assert "polypy_registry_markets_total" in metric_names
        assert "polypy_router_messages_routed_total" in metric_names

    def test_application_running_metric(self, mock_stats):
        """Test application running metric."""
        class MockApp:
            def get_stats(self):
                return mock_stats

        collector = MetricsCollector(MockApp())
        result = collector.collect_metrics()
        metrics_text = result.decode("utf-8")

        # Check for running=1
        assert "polypy_application_running 1.0" in metrics_text

    def test_registry_metrics(self, mock_stats):
        """Test registry metrics with status labels."""
        class MockApp:
            def get_stats(self):
                return mock_stats

        collector = MetricsCollector(MockApp())
        result = collector.collect_metrics()
        metrics_text = result.decode("utf-8")

        # Check for market counts by status
        assert 'polypy_registry_markets_total{status="total"} 150.0' in metrics_text
        assert 'polypy_registry_markets_total{status="pending"} 10.0' in metrics_text
        assert 'polypy_registry_markets_total{status="subscribed"} 120.0' in metrics_text
        assert 'polypy_registry_markets_total{status="expired"} 20.0' in metrics_text

    def test_router_counter_metrics(self, mock_stats):
        """Test router counter metrics."""
        class MockApp:
            def get_stats(self):
                return mock_stats

        collector = MetricsCollector(MockApp())
        result = collector.collect_metrics()
        metrics_text = result.decode("utf-8")

        # Check for counters
        assert "polypy_router_messages_routed_total 10000.0" in metrics_text
        assert "polypy_router_messages_dropped_total 50.0" in metrics_text
        assert "polypy_router_batches_sent_total 500.0" in metrics_text

    def test_recycler_metrics_with_labels(self, mock_stats):
        """Test recycler metrics with result labels."""
        class MockApp:
            def get_stats(self):
                return mock_stats

        collector = MetricsCollector(MockApp())
        result = collector.collect_metrics()
        metrics_text = result.decode("utf-8")

        # Check for labeled counters
        assert 'polypy_recycler_recycles_total{result="completed"} 8.0' in metrics_text
        assert 'polypy_recycler_recycles_total{result="failed"} 2.0' in metrics_text
        assert "polypy_recycler_success_rate 0.8" in metrics_text

    def test_empty_stats_handling(self):
        """Test that collector handles empty stats gracefully."""
        class MockApp:
            def get_stats(self):
                return {
                    "running": False,
                    "registry": {},
                    "pool": {},
                    "router": {},
                    "workers": {},
                    "lifecycle": {},
                    "recycler": {},
                }

        collector = MetricsCollector(MockApp())
        result = collector.collect_metrics()

        # Should still return valid Prometheus format
        assert isinstance(result, bytes)
        metrics_text = result.decode("utf-8")
        families = list(text_string_to_metric_families(metrics_text))

        # Should at least have application_running metric
        metric_names = {family.name for family in families}
        assert "polypy_application_running" in metric_names


@pytest.mark.asyncio
async def test_metrics_endpoint_integration():
    """Integration test for /metrics endpoint."""
    # This would require a running PolyPy instance
    # Skipping for now, to be implemented with integration test suite
    pass
```

### Success Criteria

#### Automated Verification:
- [ ] Code passes linting: `just check`
- [ ] All tests pass: `uv run pytest tests/test_metrics_prometheus.py -v`
- [ ] All existing tests still pass: `uv run pytest`

#### Manual Verification:
- [ ] Start application with `just run`
- [ ] Access http://localhost:8080/metrics in browser
- [ ] Verify HTTP 200 response with `text/plain` content-type
- [ ] Verify output contains Prometheus text format with metrics like:
  - `polypy_application_running 1.0`
  - `polypy_registry_markets_total{status="subscribed"} <number>`
  - `polypy_router_messages_routed_total <number>`
- [ ] Verify metrics values match those from http://localhost:8080/stats
- [ ] Run `curl -s http://localhost:8080/metrics | promtool check metrics` to validate format (if promtool installed)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the metrics endpoint works correctly before proceeding to Phase 2.

---

## Phase 2: Per-Connection and Per-Worker Labeled Metrics

### Overview
Add detailed per-connection and per-worker metrics with appropriate labels (connection_id, worker_id). This enables granular observability and troubleshooting of individual connections and workers.

### PR Context
**Stack Position**: Third PR in the stack (base: phase-1-branch)
**Purpose**: Add granular per-component metrics with labels for detailed observability
**Builds On**: Phase 1's MetricsCollector infrastructure
**Enables**: Phase 3 will add worker processing time summaries
**Review Focus**: Label cardinality (bounded by pool/worker counts), metric completeness

### Changes Required

#### 1. Add Per-Connection Metrics
**File**: `src/metrics/prometheus.py`
**Changes**: Add method to collect per-connection metrics and call it from `collect_metrics()`

**Add method after `_collect_pool_metrics`**:
```python
def _collect_connection_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
    """Collect per-connection metrics with connection_id labels."""
    pool_stats = stats.get("pool", {})
    if not pool_stats:
        return

    connection_stats = pool_stats.get("stats", [])
    if not connection_stats:
        return

    # Define metrics with connection_id label
    messages_received = Counter(
        "polypy_connection_messages_received_total",
        "Total messages received per connection",
        ["connection_id"],
        registry=registry,
    )

    bytes_received = Counter(
        "polypy_connection_bytes_received_total",
        "Total bytes received per connection",
        ["connection_id"],
        registry=registry,
    )

    parse_errors = Counter(
        "polypy_connection_parse_errors_total",
        "Total parse errors per connection",
        ["connection_id"],
        registry=registry,
    )

    reconnects = Counter(
        "polypy_connection_reconnects_total",
        "Total reconnection attempts per connection",
        ["connection_id"],
        registry=registry,
    )

    message_rate = Gauge(
        "polypy_connection_message_rate",
        "Message rate (messages/second) per connection",
        ["connection_id"],
        registry=registry,
    )

    healthy = Gauge(
        "polypy_connection_healthy",
        "Connection health status (1=healthy, 0=unhealthy)",
        ["connection_id", "status"],
        registry=registry,
    )

    markets_total = Gauge(
        "polypy_connection_markets_total",
        "Number of markets per connection by type",
        ["connection_id", "type"],
        registry=registry,
    )

    pollution_ratio = Gauge(
        "polypy_connection_pollution_ratio",
        "Ratio of expired to total markets per connection",
        ["connection_id"],
        registry=registry,
    )

    # Populate metrics for each connection
    for conn in connection_stats:
        conn_id = conn.get("connection_id", "unknown")

        # Counters
        messages_received.labels(connection_id=conn_id)._value.set(
            conn.get("messages_received", 0)
        )
        bytes_received.labels(connection_id=conn_id)._value.set(
            conn.get("bytes_received", 0)
        )
        parse_errors.labels(connection_id=conn_id)._value.set(
            conn.get("parse_errors", 0)
        )
        reconnects.labels(connection_id=conn_id)._value.set(
            conn.get("reconnect_count", 0)
        )

        # Gauges
        # Note: message_rate not available in stats dict, would need to add
        # For now, skip or compute from messages_received / age_seconds
        age_seconds = conn.get("age_seconds", 0)
        if age_seconds > 0:
            messages = conn.get("messages_received", 0)
            message_rate.labels(connection_id=conn_id).set(messages / age_seconds)

        # Health status
        status_name = conn.get("status", "UNKNOWN")
        is_healthy = conn.get("is_healthy", False)
        healthy.labels(connection_id=conn_id, status=status_name).set(
            1 if is_healthy else 0
        )

        # Market counts
        markets_total.labels(connection_id=conn_id, type="total").set(
            conn.get("total_markets", 0)
        )
        markets_total.labels(connection_id=conn_id, type="subscribed").set(
            conn.get("subscribed_markets", 0)
        )
        markets_total.labels(connection_id=conn_id, type="expired").set(
            conn.get("expired_markets", 0)
        )

        # Pollution ratio
        pollution_ratio.labels(connection_id=conn_id).set(
            conn.get("pollution_ratio", 0.0)
        )
```

**Update `collect_metrics` method** to call the new method:
```python
def collect_metrics(self) -> bytes:
    """
    Collect current stats and return Prometheus text format.

    Creates a fresh registry on each call and populates it with
    current application state.

    Returns:
        Prometheus text exposition format bytes
    """
    # Create fresh registry for this scrape
    registry = CollectorRegistry()

    # Get current stats
    stats = self._app.get_stats()

    # Create and populate metrics
    self._collect_application_metrics(registry, stats)
    self._collect_registry_metrics(registry, stats)
    self._collect_pool_metrics(registry, stats)
    self._collect_connection_metrics(registry, stats)  # NEW
    self._collect_router_metrics(registry, stats)
    self._collect_worker_metrics(registry, stats)
    self._collect_worker_detail_metrics(registry, stats)  # NEW (to be added below)
    self._collect_lifecycle_metrics(registry, stats)
    self._collect_recycler_metrics(registry, stats)

    # Generate Prometheus text format
    return generate_latest(registry)
```

#### 2. Add Per-Worker Detail Metrics
**File**: `src/metrics/prometheus.py`
**Changes**: Add method to collect per-worker detailed metrics

**Add method after `_collect_worker_metrics`**:
```python
def _collect_worker_detail_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
    """Collect detailed per-worker metrics with worker_id labels."""
    worker_stats = stats.get("workers", {})
    if not worker_stats:
        return

    worker_details = worker_stats.get("worker_stats", {})
    if not worker_details:
        return

    # Define metrics with worker_id label
    messages_processed = Counter(
        "polypy_worker_messages_processed_total",
        "Total messages processed per worker",
        ["worker_id"],
        registry=registry,
    )

    updates_applied = Counter(
        "polypy_worker_updates_applied_total",
        "Total orderbook updates applied per worker",
        ["worker_id"],
        registry=registry,
    )

    snapshots_received = Counter(
        "polypy_worker_snapshots_received_total",
        "Total orderbook snapshots received per worker",
        ["worker_id"],
        registry=registry,
    )

    orderbook_count = Gauge(
        "polypy_worker_orderbook_count",
        "Number of orderbooks managed per worker",
        ["worker_id"],
        registry=registry,
    )

    memory_bytes = Gauge(
        "polypy_worker_memory_bytes",
        "Memory usage in bytes per worker",
        ["worker_id"],
        registry=registry,
    )

    avg_processing_time = Gauge(
        "polypy_worker_avg_processing_time_seconds",
        "Average message processing time per worker in seconds",
        ["worker_id"],
        registry=registry,
    )

    # Populate metrics for each worker
    for worker_id_str, worker_data in worker_details.items():
        worker_id = str(worker_id_str)

        # Counters
        messages_processed.labels(worker_id=worker_id)._value.set(
            worker_data.get("messages_processed", 0)
        )
        updates_applied.labels(worker_id=worker_id)._value.set(
            worker_data.get("updates_applied", 0)
        )
        snapshots_received.labels(worker_id=worker_id)._value.set(
            worker_data.get("snapshots_received", 0)
        )

        # Gauges
        orderbook_count.labels(worker_id=worker_id).set(
            worker_data.get("orderbook_count", 0)
        )

        # Memory: convert MB back to bytes for consistency
        memory_mb = worker_data.get("memory_usage_mb", 0.0)
        memory_bytes.labels(worker_id=worker_id).set(memory_mb * 1024 * 1024)

        # Processing time: convert microseconds to seconds
        avg_processing_us = worker_data.get("avg_processing_time_us", 0.0)
        avg_processing_time.labels(worker_id=worker_id).set(
            avg_processing_us / 1_000_000.0
        )
```

#### 3. Update Unit Tests
**File**: `tests/test_metrics_prometheus.py`
**Changes**: Add tests for per-connection and per-worker metrics

**Update `mock_stats` fixture** to include per-connection and per-worker details:
```python
@pytest.fixture
def mock_stats() -> dict:
    """Mock stats dictionary matching PolyPy.get_stats() structure."""
    return {
        "running": True,
        "registry": {
            "total_markets": 150,
            "pending": 10,
            "subscribed": 120,
            "expired": 20,
        },
        "pool": {
            "connection_count": 2,
            "active_connections": 2,
            "total_capacity": 1000,
            "stats": [
                {
                    "connection_id": "conn-0",
                    "status": "CONNECTED",
                    "is_draining": False,
                    "age_seconds": 100.0,
                    "messages_received": 1000,
                    "bytes_received": 50000,
                    "parse_errors": 2,
                    "reconnect_count": 1,
                    "is_healthy": True,
                    "total_markets": 75,
                    "subscribed_markets": 60,
                    "expired_markets": 15,
                    "pollution_ratio": 0.2,
                },
                {
                    "connection_id": "conn-1",
                    "status": "CONNECTED",
                    "is_draining": False,
                    "age_seconds": 200.0,
                    "messages_received": 2000,
                    "bytes_received": 100000,
                    "parse_errors": 0,
                    "reconnect_count": 0,
                    "is_healthy": True,
                    "total_markets": 75,
                    "subscribed_markets": 60,
                    "expired_markets": 15,
                    "pollution_ratio": 0.2,
                },
            ],
        },
        "router": {
            "messages_routed": 3000,
            "messages_dropped": 10,
            "batches_sent": 150,
            "queue_full_events": 2,
            "routing_errors": 1,
            "avg_latency_ms": 1.5,
            "queue_depths": {"async_queue": 10, "worker_0": 5, "worker_1": 3},
        },
        "workers": {
            "alive_count": 2,
            "is_healthy": True,
            "worker_stats": {
                "0": {
                    "messages_processed": 1500,
                    "updates_applied": 4500,
                    "snapshots_received": 75,
                    "avg_processing_time_us": 250.5,
                    "orderbook_count": 75,
                    "memory_usage_mb": 128.5,
                },
                "1": {
                    "messages_processed": 1500,
                    "updates_applied": 4500,
                    "snapshots_received": 75,
                    "avg_processing_time_us": 230.2,
                    "orderbook_count": 75,
                    "memory_usage_mb": 125.3,
                },
            },
        },
        "lifecycle": {
            "is_running": True,
            "known_market_count": 150,
        },
        "recycler": {
            "recycles_initiated": 5,
            "recycles_completed": 4,
            "recycles_failed": 1,
            "success_rate": 0.8,
            "markets_migrated": 300,
            "avg_downtime_ms": 150.0,
            "active_recycles": [],
        },
    }
```

**Add new test cases**:
```python
def test_per_connection_metrics(mock_stats):
    """Test per-connection metrics with connection_id labels."""
    class MockApp:
        def get_stats(self):
            return mock_stats

    collector = MetricsCollector(MockApp())
    result = collector.collect_metrics()
    metrics_text = result.decode("utf-8")

    # Check for conn-0 metrics
    assert 'polypy_connection_messages_received_total{connection_id="conn-0"} 1000.0' in metrics_text
    assert 'polypy_connection_bytes_received_total{connection_id="conn-0"} 50000.0' in metrics_text
    assert 'polypy_connection_parse_errors_total{connection_id="conn-0"} 2.0' in metrics_text

    # Check for conn-1 metrics
    assert 'polypy_connection_messages_received_total{connection_id="conn-1"} 2000.0' in metrics_text


def test_per_worker_metrics(mock_stats):
    """Test per-worker metrics with worker_id labels."""
    class MockApp:
        def get_stats(self):
            return mock_stats

    collector = MetricsCollector(MockApp())
    result = collector.collect_metrics()
    metrics_text = result.decode("utf-8")

    # Check for worker 0 metrics
    assert 'polypy_worker_messages_processed_total{worker_id="0"} 1500.0' in metrics_text
    assert 'polypy_worker_updates_applied_total{worker_id="0"} 4500.0' in metrics_text
    assert 'polypy_worker_orderbook_count{worker_id="0"} 75.0' in metrics_text

    # Check for worker 1 metrics
    assert 'polypy_worker_messages_processed_total{worker_id="1"} 1500.0' in metrics_text


def test_connection_pollution_ratio(mock_stats):
    """Test connection pollution ratio metric."""
    class MockApp:
        def get_stats(self):
            return mock_stats

    collector = MetricsCollector(MockApp())
    result = collector.collect_metrics()
    metrics_text = result.decode("utf-8")

    # Check pollution ratio for both connections
    assert 'polypy_connection_pollution_ratio{connection_id="conn-0"} 0.2' in metrics_text
    assert 'polypy_connection_pollution_ratio{connection_id="conn-1"} 0.2' in metrics_text


def test_worker_memory_bytes_conversion(mock_stats):
    """Test that worker memory is correctly converted from MB to bytes."""
    class MockApp:
        def get_stats(self):
            return mock_stats

    collector = MetricsCollector(MockApp())
    result = collector.collect_metrics()
    metrics_text = result.decode("utf-8")

    # 128.5 MB = 134742016 bytes
    expected_bytes = 128.5 * 1024 * 1024
    # Check approximately (floating point)
    assert f'polypy_worker_memory_bytes{{worker_id="0"}} {expected_bytes}' in metrics_text or \
           'polypy_worker_memory_bytes{worker_id="0"} 1.34' in metrics_text  # Scientific notation
```

### Success Criteria

#### Automated Verification:
- [ ] Code passes linting: `just check`
- [ ] All tests pass: `uv run pytest tests/test_metrics_prometheus.py -v`
- [ ] All existing tests still pass: `uv run pytest`

#### Manual Verification:
- [ ] Start application with `just run`
- [ ] Access http://localhost:8080/metrics
- [ ] Verify per-connection metrics appear with connection_id labels:
  - `polypy_connection_messages_received_total{connection_id="conn-X"}`
  - `polypy_connection_pollution_ratio{connection_id="conn-X"}`
- [ ] Verify per-worker metrics appear with worker_id labels:
  - `polypy_worker_messages_processed_total{worker_id="X"}`
  - `polypy_worker_memory_bytes{worker_id="X"}`
- [ ] Verify metric cardinality is bounded (number of labels = number of connections/workers)
- [ ] Compare values with /stats endpoint to ensure accuracy
- [ ] Test PromQL queries if Prometheus is available:
  - `sum(rate(polypy_worker_messages_processed_total[1m]))` - total message rate
  - `polypy_connection_pollution_ratio > 0.3` - unhealthy connections

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that labeled metrics work correctly before proceeding to Phase 3.

---

## Phase 3: Worker Processing Time Summaries

### Overview
Add Summary metrics for worker processing times, router routing latency, and recycler downtime. Summaries track distributions and enable percentile queries in Prometheus.

### PR Context
**Stack Position**: Fourth PR in the stack (base: phase-2-branch)
**Purpose**: Add latency distribution metrics using Summaries for percentile analysis
**Builds On**: Phase 2's per-worker infrastructure
**Enables**: Complete Prometheus integration with all metric types
**Review Focus**: Summary implementation, latency tracking accuracy

### Changes Required

#### 1. Add Worker Processing Time Summary
**File**: `src/metrics/prometheus.py`
**Changes**: Update `_collect_worker_detail_metrics` to add processing time summary

**Replace the avg_processing_time gauge section with**:
```python
def _collect_worker_detail_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
    """Collect detailed per-worker metrics with worker_id labels."""
    worker_stats = stats.get("workers", {})
    if not worker_stats:
        return

    worker_details = worker_stats.get("worker_stats", {})
    if not worker_details:
        return

    # ... (previous counter and gauge definitions remain) ...

    # Processing time summary (replaces avg_processing_time gauge)
    processing_time_summary = Summary(
        "polypy_worker_processing_seconds",
        "Worker message processing time distribution in seconds",
        ["worker_id"],
        registry=registry,
    )

    # Populate metrics for each worker
    for worker_id_str, worker_data in worker_details.items():
        worker_id = str(worker_id_str)

        # ... (previous counter/gauge population remains) ...

        # Processing time summary
        # Note: Since we only have avg_processing_time_us, we can't create
        # a true distribution. We'll track it as a counter+count pair.
        # In a future enhancement, we could store individual observations.
        avg_processing_us = worker_data.get("avg_processing_time_us", 0.0)
        messages_processed_count = worker_data.get("messages_processed", 0)

        if messages_processed_count > 0 and avg_processing_us > 0:
            # Set summary internal metrics directly
            avg_processing_seconds = avg_processing_us / 1_000_000.0
            total_processing_seconds = avg_processing_seconds * messages_processed_count

            # Access summary internal metrics
            summary_metric = processing_time_summary.labels(worker_id=worker_id)
            summary_metric._sum.set(total_processing_seconds)
            summary_metric._count.set(messages_processed_count)
```

#### 2. Add Router Latency Summary
**File**: `src/metrics/prometheus.py`
**Changes**: Update `_collect_router_metrics` to add routing latency summary

**Replace the routing latency section with**:
```python
def _collect_router_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
    """Collect message router metrics."""
    router_stats = stats.get("router", {})
    if not router_stats:
        return

    # ... (previous counter and gauge definitions remain) ...

    # Routing latency summary (replaces latency counter)
    routing_latency_summary = Summary(
        "polypy_router_routing_latency_seconds",
        "Message routing latency distribution in seconds",
        registry=registry,
    )

    # Populate summary from aggregate stats
    messages_routed_count = router_stats.get("messages_routed", 0)
    avg_latency_ms = router_stats.get("avg_latency_ms", 0.0)

    if messages_routed_count > 0 and avg_latency_ms > 0:
        avg_latency_seconds = avg_latency_ms / 1000.0
        total_latency_seconds = avg_latency_seconds * messages_routed_count

        routing_latency_summary._sum.set(total_latency_seconds)
        routing_latency_summary._count.set(messages_routed_count)
```

#### 3. Add Recycler Downtime Summary
**File**: `src/metrics/prometheus.py`
**Changes**: Update `_collect_recycler_metrics` to add downtime summary

**Add after the active_recycles gauge**:
```python
def _collect_recycler_metrics(self, registry: CollectorRegistry, stats: dict[str, Any]) -> None:
    """Collect connection recycler metrics."""
    recycler_stats = stats.get("recycler", {})
    if not recycler_stats:
        return

    # ... (previous counters and gauges remain) ...

    # Downtime summary (complements avg_downtime gauge)
    downtime_summary = Summary(
        "polypy_recycler_downtime_seconds",
        "Recycle operation downtime distribution in seconds",
        registry=registry,
    )

    # Populate summary from aggregate stats
    recycles_completed = recycler_stats.get("recycles_completed", 0)
    avg_downtime_ms = recycler_stats.get("avg_downtime_ms", 0.0)

    if recycles_completed > 0 and avg_downtime_ms > 0:
        avg_downtime_seconds = avg_downtime_ms / 1000.0
        total_downtime_seconds = avg_downtime_seconds * recycles_completed

        downtime_summary._sum.set(total_downtime_seconds)
        downtime_summary._count.set(recycles_completed)
```

#### 4. Update Unit Tests
**File**: `tests/test_metrics_prometheus.py`
**Changes**: Add tests for summary metrics

**Add new test cases**:
```python
def test_worker_processing_summary(mock_stats):
    """Test worker processing time summary metric."""
    class MockApp:
        def get_stats(self):
            return mock_stats

    collector = MetricsCollector(MockApp())
    result = collector.collect_metrics()
    metrics_text = result.decode("utf-8")

    # Check for summary metrics (sum and count)
    assert 'polypy_worker_processing_seconds_sum{worker_id="0"}' in metrics_text
    assert 'polypy_worker_processing_seconds_count{worker_id="0"} 1500.0' in metrics_text


def test_router_latency_summary(mock_stats):
    """Test router routing latency summary metric."""
    class MockApp:
        def get_stats(self):
            return mock_stats

    collector = MetricsCollector(MockApp())
    result = collector.collect_metrics()
    metrics_text = result.decode("utf-8")

    # Check for summary metrics
    assert "polypy_router_routing_latency_seconds_sum" in metrics_text
    assert "polypy_router_routing_latency_seconds_count 3000.0" in metrics_text


def test_recycler_downtime_summary(mock_stats):
    """Test recycler downtime summary metric."""
    class MockApp:
        def get_stats(self):
            return mock_stats

    collector = MetricsCollector(MockApp())
    result = collector.collect_metrics()
    metrics_text = result.decode("utf-8")

    # Check for summary metrics
    assert "polypy_recycler_downtime_seconds_sum" in metrics_text
    assert "polypy_recycler_downtime_seconds_count 4.0" in metrics_text  # recycles_completed


def test_summary_calculations_accuracy(mock_stats):
    """Test that summary sum values are calculated correctly."""
    class MockApp:
        def get_stats(self):
            return mock_stats

    collector = MetricsCollector(MockApp())
    result = collector.collect_metrics()
    metrics_text = result.decode("utf-8")

    # Worker 0: avg_processing_time_us=250.5, messages_processed=1500
    # Total = 250.5 * 1500 / 1_000_000 = 0.37575 seconds
    # Check that sum is approximately correct
    assert 'polypy_worker_processing_seconds_sum{worker_id="0"}' in metrics_text

    # Router: avg_latency_ms=1.5, messages_routed=3000
    # Total = 1.5 * 3000 / 1000 = 4.5 seconds
    assert "polypy_router_routing_latency_seconds_sum" in metrics_text
```

#### 5. Add Documentation Comment
**File**: `src/metrics/prometheus.py`
**Changes**: Add docstring note about summary limitations

**Add to module docstring**:
```python
"""Prometheus metrics collector for PolyPy application stats.

Note on Summaries:
------------------
The Summary metrics (processing_seconds, routing_latency_seconds, downtime_seconds)
are populated from aggregate statistics (average * count) rather than individual
observations. This means:

1. Quantiles are not available (only _sum and _count)
2. The distribution is reconstructed from aggregates
3. For true percentile queries, use rate(_sum) / rate(_count) in PromQL

Example PromQL queries:
- Average processing time: rate(polypy_worker_processing_seconds_sum[1m]) / rate(polypy_worker_processing_seconds_count[1m])
- Total throughput: rate(polypy_worker_processing_seconds_count[1m])

To get true quantiles in the future, would need to track individual observations
in the stats collection layer, not just aggregate averages.
"""
```

### Success Criteria

#### Automated Verification:
- [ ] Code passes linting: `just check`
- [ ] All tests pass: `uv run pytest tests/test_metrics_prometheus.py -v`
- [ ] All existing tests still pass: `uv run pytest`

#### Manual Verification:
- [ ] Start application with `just run`
- [ ] Access http://localhost:8080/metrics
- [ ] Verify summary metrics appear with _sum and _count suffixes:
  - `polypy_worker_processing_seconds_sum{worker_id="X"}`
  - `polypy_worker_processing_seconds_count{worker_id="X"}`
  - `polypy_router_routing_latency_seconds_sum`
  - `polypy_router_routing_latency_seconds_count`
  - `polypy_recycler_downtime_seconds_sum`
  - `polypy_recycler_downtime_seconds_count`
- [ ] Verify _sum values are in seconds (not milliseconds or microseconds)
- [ ] Test PromQL queries if Prometheus is available:
  - `rate(polypy_worker_processing_seconds_sum[1m]) / rate(polypy_worker_processing_seconds_count[1m])` - average processing time
  - `rate(polypy_router_routing_latency_seconds_count[1m])` - routing throughput

**Implementation Note**: After completing this phase and all verification passes, the Prometheus integration is complete. The `/metrics` endpoint exposes all application statistics in Prometheus format.

---

## Testing Strategy

### Unit Tests

**Coverage targets**:
- `src/metrics/prometheus.py`: 100% coverage
  - Test each metric collection method independently
  - Test with empty stats (graceful handling)
  - Test with populated stats (correct values)
  - Test metric naming and labels
  - Test Prometheus format validity

**Key test cases** (already included in phases):
- Metrics return valid Prometheus text format
- Counter values match source stats
- Gauge values match source stats
- Summary _sum and _count are calculated correctly
- Labels are applied correctly (connection_id, worker_id, status, type, result)
- Empty components don't cause errors
- Unit conversions are correct (ms→s, us→s, MB→bytes)

### Integration Tests

**HTTP endpoint tests** (`tests/integration/test_metrics_endpoint.py`):
```python
import pytest
from aiohttp import ClientSession


@pytest.mark.asyncio
async def test_metrics_endpoint_responds():
    """Test that /metrics endpoint responds with 200."""
    async with ClientSession() as session:
        async with session.get("http://localhost:8080/metrics") as resp:
            assert resp.status == 200
            assert "text/plain" in resp.content_type


@pytest.mark.asyncio
async def test_metrics_format_is_valid():
    """Test that /metrics returns valid Prometheus format."""
    async with ClientSession() as session:
        async with session.get("http://localhost:8080/metrics") as resp:
            text = await resp.text()

            # Should contain TYPE and HELP lines
            assert "# TYPE" in text
            assert "# HELP" in text

            # Should contain our metrics
            assert "polypy_application_running" in text
            assert "polypy_router_messages_routed_total" in text


@pytest.mark.asyncio
async def test_metrics_values_match_stats():
    """Test that /metrics values match /stats values."""
    async with ClientSession() as session:
        # Get stats
        async with session.get("http://localhost:8080/stats") as resp:
            stats = await resp.json()

        # Get metrics
        async with session.get("http://localhost:8080/metrics") as resp:
            metrics_text = await resp.text()

        # Compare values
        messages_routed = stats["router"]["messages_routed"]
        assert f"polypy_router_messages_routed_total {float(messages_routed)}" in metrics_text
```

### Manual Testing Steps

**Basic functionality**:
1. Start application: `just run`
2. Wait for application to be healthy: `curl http://localhost:8080/health`
3. Access metrics endpoint: `curl http://localhost:8080/metrics`
4. Verify output contains Prometheus text format
5. Verify metrics have sensible values (non-zero where expected)

**Prometheus integration** (if Prometheus is running):
1. Configure Prometheus to scrape the endpoint:
   ```yaml
   scrape_configs:
     - job_name: 'polypy'
       static_configs:
         - targets: ['localhost:8080']
       metrics_path: '/metrics'
       scrape_interval: 15s
   ```
2. Start Prometheus and verify target is UP
3. Query metrics in Prometheus UI:
   - `polypy_application_running` should be 1
   - `rate(polypy_router_messages_routed_total[1m])` for message rate
   - `polypy_connection_pollution_ratio > 0.3` for unhealthy connections
4. Create sample Grafana dashboard with:
   - Message throughput graph
   - Connection health status
   - Worker memory usage
   - Processing latency

**Validate format** (if promtool is installed):
```bash
curl -s http://localhost:8080/metrics | promtool check metrics
```

## Performance Considerations

### Scrape Performance

**Expected overhead per scrape**:
- Calls `app.get_stats()` once (existing operation)
- Creates ~50 metric objects
- Generates text format (typically <50KB)
- **Estimated total time**: <10ms per scrape

**Prometheus scrape interval**: Recommend 15-30 seconds
- Too frequent: unnecessary overhead
- Too infrequent: miss short-lived issues

### Memory Impact

**Per-scrape memory**:
- Fresh `CollectorRegistry` created per request
- Metrics objects are temporary (garbage collected after response)
- No persistent state maintained
- **Estimated peak memory**: <1MB per scrape

**No memory leaks**:
- No global metric objects
- No persistent registries
- Stateless design

### Label Cardinality

**Current cardinality**:
- `connection_id`: Bounded by pool size (~5-20 connections)
- `worker_id`: Bounded by worker count (~2-8 workers)
- `status`: Fixed set (CONNECTED, DISCONNECTED, etc.)
- `type`: Fixed set (total, active, pending, etc.)
- `result`: Fixed set (completed, failed)
- `queue_name`: Bounded by worker count + 1 async queue

**Total unique metric series**: ~200-500 (acceptable for Prometheus)

**Not creating high cardinality**:
- No `market_id` labels (would be thousands)
- No `message_id` labels (would be millions)
- No timestamp labels

## Migration Notes

### Backward Compatibility

**No breaking changes**:
- `/stats` endpoint remains unchanged
- No modifications to stats collection
- New `/metrics` endpoint is additive
- Existing monitoring can continue using `/stats`

**Gradual migration path**:
1. Deploy with `/metrics` endpoint
2. Configure Prometheus scraping
3. Create Grafana dashboards
4. Migrate alerts from `/stats` polling to Prometheus
5. Eventually deprecate custom monitoring if desired

### Deployment Considerations

**Configuration**:
- No new configuration needed
- HTTP server port remains 8080
- Metrics exposed automatically when HTTP server enabled

**Docker/Kubernetes**:
- Expose port 8080 in container
- Add Prometheus annotations:
  ```yaml
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
  ```

**Monitoring**:
- Set up Prometheus alerts on key metrics:
  - `polypy_application_running == 0` - application down
  - `rate(polypy_router_messages_dropped_total[1m]) > 10` - backpressure
  - `polypy_connection_pollution_ratio > 0.3` - needs recycling
  - `polypy_recycler_success_rate < 0.9` - recycling issues

## References

- Original research: `thoughts/searchable/research/2025-12-11-stats-collection-prometheus-integration.md`
- Prometheus best practices: https://prometheus.io/docs/instrumenting/writing_exporters/
- prometheus_client documentation: https://github.com/prometheus/client_python
- Phase 8 lesson file: `lessons/008_production.md:118-161`
- Existing stats implementation: `src/app.py:292-382`
- HTTP server implementation: `src/server/server.py:17-156`

## Open Questions and Resolutions

All questions from the research document have been answered:

1. ✅ **Scrape Interval**: Recommend 15-30 seconds (Prometheus configuration)
2. ✅ **Authentication**: No authentication (specified by user)
3. ✅ **Per-Market Metrics**: No per-market labels (specified in research)
4. ✅ **Histogram Buckets**: Using Summaries instead (specified by user)
5. ✅ **Metric Retention**: 1hr period (Prometheus server config, specified in research)
6. ✅ **Alerting Rules**: None for now (specified in research)
7. ⚠️ **Dashboard Design**: Deferred to operations team (out of scope)

## Success Metrics

**Completion criteria**:
- ✅ All three phases implemented
- ✅ All tests passing (unit + integration)
- ✅ `/metrics` endpoint returns valid Prometheus format
- ✅ Metrics values match `/stats` endpoint
- ✅ Prometheus can scrape successfully
- ✅ PromQL queries work as expected
- ✅ Documentation complete

**Post-deployment validation**:
- Monitor Prometheus target status (should be UP)
- Verify metrics are being collected
- Test sample PromQL queries
- Create basic Grafana dashboard
- Set up initial alerts
