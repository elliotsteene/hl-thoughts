# ENG-007: Refactor ConnectionRecycler to proper dependency injection pattern

**Status**: Open
**Priority**: High
**Estimated Effort**: Medium (2-3 hours including testing)

## Context

The current implementation has ConnectionPool creating ConnectionRecycler in its `__init__`, which creates a circular dependency (Pool → Recycler → Pool) and violates the intended architecture from Component 10 of the system spec.

PR comment on phase-6-system-integration identified this as a structural code smell: "The ConnectionRecycler is instantiated within the init of the ConnectionPool and also itself requires a ConnectionPool. This feels like a structural code smell - there is likely something compositionally wrong between the ConnectionPool, the Recycler and how lifecycle of the connections are managed."

## Problem

1. ConnectionPool creates ConnectionRecycler in its `__init__` (`src/connection/pool.py:91`)
2. ConnectionRecycler requires ConnectionPool as parameter, creating circular dependency
3. Required TYPE_CHECKING workaround to avoid circular imports
4. Violates Component 10 spec where Application is the composition root
5. Pool cannot be tested without Recycler being created

## Root Cause

According to the system spec (Component 10, `lessons/polymarket_websocket_system_spec.md:3145-3202`), the Application Orchestrator should be the composition root that creates and wires all components together. ConnectionPool should NOT create ConnectionRecycler - the Application should create both and wire them.

## Reference Architecture

LifecycleController follows the correct pattern:
- Created by Application (not by MarketRegistry)
- Dependencies injected: `LifecycleController(registry, ...)`
- Application manages lifecycle

ConnectionRecycler should follow the same pattern.

## Proposed Solution

Refactor to proper dependency injection where Application (or a minimal orchestrator) creates both ConnectionPool and ConnectionRecycler and wires them together.

## Implementation Steps

1. Remove ConnectionRecycler creation from `ConnectionPool.__init__`
2. Remove `_recycler` attribute from `ConnectionPool.__slots__`
3. Remove recycler lifecycle management from `ConnectionPool.start()` and `stop()`
4. Remove `recycler_stats` property from ConnectionPool (or make it take recycler as parameter)
5. Create minimal Application/Orchestrator class that:
   - Creates ConnectionPool
   - Creates ConnectionRecycler
   - Wires them together (passes pool to recycler)
   - Manages both lifecycles
6. Update all tests to reflect new structure
7. Update Phase 6 integration tests
8. Remove TYPE_CHECKING workaround (no longer needed)

## Success Criteria

- [ ] No circular dependency between Pool and Recycler
- [ ] ConnectionPool does not import ConnectionRecycler
- [ ] Application/Orchestrator is composition root
- [ ] All tests pass (252 tests)
- [ ] Linting passes
- [ ] Recycler functionality unchanged (zero-downtime recycling still works)
- [ ] Follows Component 10 specification pattern

## Acceptance Criteria

- ConnectionPool can be instantiated and tested without ConnectionRecycler
- ConnectionRecycler receives ConnectionPool via dependency injection
- Application/Orchestrator manages both components' lifecycles
- No TYPE_CHECKING import workarounds needed
- Architecture ready for full Component 10 implementation

## Dependencies

- **Depends on**: Phase 6 completion (ENG-005)
- **Blocks**: None (but prepares for ENG-006 Application Orchestrator)

## References

- System spec Component 10: `lessons/polymarket_websocket_system_spec.md:3041-3449`
- System spec Component 9: `lessons/polymarket_websocket_system_spec.md:2750-3038`
- Research document: `thoughts/shared/research/2025-12-03-remaining-system-spec-components.md:394-463`
- PR comment: https://github.com/elliotsteene/polypy/pull/23#discussion_r...
- Current implementation: `src/connection/pool.py:91`, `src/lifecycle/recycler.py`

## Notes

This is architectural debt that makes future work harder. The circular dependency pattern should be eliminated before proceeding with full Application Orchestrator implementation.
