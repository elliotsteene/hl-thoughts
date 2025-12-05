# ENG-007: Add HTTP API for Stats and Health Endpoints

**Status**: Open
**Priority**: Medium
**Component**: Application Orchestrator Enhancement
**Depends on**: ENG-006 (Application Orchestrator must be complete first)

## Problem to Solve

The Application Orchestrator (ENG-006) exposes `get_stats()` and `is_healthy()` methods, but these are only accessible via direct method calls within the application. For production monitoring, observability, and integration with external monitoring systems (Prometheus, Grafana, health checks), we need HTTP endpoints that expose this information over the network.

Without HTTP endpoints, operators cannot:
- Monitor application health from external systems
- Integrate with load balancers for health checks
- View real-time stats without code access
- Debug production issues remotely

## Solution

Add an aiohttp-based HTTP server to the Application class that exposes health and stats information through REST endpoints.

### Core Requirements

1. **HTTP Server Integration**
   - Add aiohttp server to Application class
   - Add `enable_http` parameter to `Application.__init__` (default: False)
   - Add `http_port` parameter to `Application.__init__` (default: 8080)
   - Server lifecycle: start after all components running, stop before other components during shutdown

2. **GET /health Endpoint**
   ```json
   {
     "healthy": true,
     "timestamp": "2025-12-05T10:30:00Z",
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
   - Return 200 OK when healthy
   - Return 503 Service Unavailable when unhealthy
   - Response time target: < 10ms

3. **GET /stats Endpoint**
   - Return comprehensive stats in JSON format (same structure as `get_stats()`)
   - Include all component statistics
   - Response time target: < 10ms

4. **Error Handling**
   - Handle server startup failures gracefully
   - Log binding errors (port already in use)
   - Continue application startup if HTTP server disabled

### Optional Enhancements

- **GET /metrics Endpoint**: Prometheus-compatible metrics format
- **GET /ready Endpoint**: Separate readiness check for k8s
- **Rate limiting**: Prevent health check spam

## Implementation Suggestions

- Use `aiohttp.web` for lightweight HTTP server
- Endpoints should be fast read-only operations (no heavy computation)
- Add structured logging for HTTP requests
- Consider adding CORS headers if needed for web dashboards
- Server should bind to 0.0.0.0 for container compatibility

## Acceptance Criteria

- [ ] HTTP server starts on configured port when `enable_http=True`
- [ ] `/health` endpoint returns 200 OK with correct JSON when healthy
- [ ] `/health` endpoint returns 503 with correct JSON when unhealthy
- [ ] `/stats` endpoint returns comprehensive stats JSON
- [ ] Server shuts down gracefully during application shutdown
- [ ] Integration tests verify HTTP endpoints
- [ ] API documented in README or `docs/http-api.md`
- [ ] Error handling tested (port conflicts, startup failures)
- [ ] Both endpoints respond in < 10ms (measured in tests)

## Technical Notes

### Dependency
```bash
# Add to pyproject.toml
aiohttp = "^3.9.0"
```

### Example Usage
```python
app = Application(
    enable_http=True,
    http_port=8080,
    # ... other params
)
await app.start()

# External monitoring can now:
# curl http://localhost:8080/health
# curl http://localhost:8080/stats
```

### Integration Points
- `Application.is_healthy()` → `/health` endpoint logic
- `Application.get_stats()` → `/stats` endpoint data
- Shutdown sequence: stop HTTP server before stopping components

## References

- Parent: ENG-006 Application Orchestrator implementation
- Related: Phase 8 (Production observability and recovery)
- Code: `src/orchestrator/application.py`

## Questions to Resolve

1. Should we add authentication for these endpoints?
2. Do we need request logging middleware?
3. Should metrics endpoint be in this ticket or separate?
4. Do we want to expose more granular component stats (per-worker, per-connection)?

---

**Created**: 2025-12-05
**Last Updated**: 2025-12-05
