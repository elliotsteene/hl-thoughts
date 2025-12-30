---
date: 2025-12-30T09:17:53+0000
researcher: Claude Sonnet 4.5
git_commit: 978ef1a7684965917cee58beba5120eed9260d02
branch: main
repository: cc-analyst
topic: "Phase 1 Foundation Hardening: Existing Architecture Analysis for Composite Scoring and Regime Detection"
tags: [research, codebase, phase1, foundation, architecture, indicators, registry, calculated-fields, composite-scores]
status: complete
last_updated: 2025-12-30
last_updated_by: Claude Sonnet 4.5
last_updated_note: "Updated to reflect test infrastructure implementation"
---

# Research: Phase 1 Foundation Hardening - Existing Architecture Analysis

**Date**: 2025-12-30T09:17:53+0000
**Researcher**: Claude Sonnet 4.5
**Git Commit**: 978ef1a7684965917cee58beba5120eed9260d02
**Branch**: main
**Repository**: cc-analyst

## Research Question

**Context**: Phase 1 of the SUPER_PLAN_MACRO_DASHBOARD.md requires "Foundation Hardening" to solidify the existing codebase architecture to support composite scoring and regime detection capabilities.

**Question**: What is the current state of the indicator infrastructure, what architectural patterns exist, and what capabilities exist (or are missing) to support:
1. Score calculation (indicators → normalized score)
2. Score aggregation (category scores → regime)
3. Time-series score storage (for sparklines)
4. Extension points for adding new categories and indicators

## Summary

The cc-analyst codebase has a **well-architected indicator registry system** with strong foundations that can support composite scoring and regime detection with modest extensions. Key strengths include:

### Existing Strengths
- **Protocol-based design** with `Indicator` protocol enabling polymorphic indicator types
- **CalculatedIndicator infrastructure** supporting both single-source and multi-source calculations with concurrent async fetching
- **Generic type-safe registry system** (`IndicatorRegistry[T]`) with enum-based indicator identification
- **Multi-layer caching** (resource, data with TTL)
- **Polars-first** DataFrame operations with consistent schema (`date`, `value`, `rolling_avg_3m`)
- **Async-everywhere** architecture using `asyncio.TaskGroup` for concurrency
- **Clear extension patterns** for adding categories and indicators

### Architectural Readiness for Phase 1
✅ **Ready**: Multi-source calculation infrastructure exists but is unused
✅ **Ready**: Registry pattern supports arbitrary indicator types
✅ **Ready**: Visualization system handles multiple series
✅ **Ready**: Test infrastructure implemented with pytest, fixtures, and comprehensive test coverage
⚠️ **Gap**: No score calculation abstractions exist
⚠️ **Gap**: No score aggregation or normalization patterns
⚠️ **Gap**: No time-series score storage beyond Streamlit cache

### Recommendation
The foundation is **solid and extensible**. Phase 1 should focus on:
1. Creating `CompositeScore` protocol and implementations
2. Adding score normalization utilities (z-score, percentile rank)
3. Implementing score storage/retrieval pattern (can leverage existing cache)

---

## Detailed Findings

### 1. CalculatedIndicator Implementation

**Location**: `src/indicators/calculations/calculated_indicator.py`

#### Current Capabilities

**Dual-Mode Support** (lines 28-53):
- **Single-source mode**: Takes one `Indicator`, applies `SingleSourceCalculation` function
- **Multi-source mode**: Takes `dict[str, Indicator]`, applies `MultiSourceCalculation` function
- Type-safe via `@overload` decorators
- Mode detection via `_is_multi_source` boolean flag

**Concurrent Multi-Source Fetching** (lines 156-169):
```python
async def _get_multi_series(self, observation_start: str | None) -> pl.DataFrame:
    sources = cast(dict[str, Indicator], self._source)
    calculation = cast(MultiSourceCalculation, self._calculation)

    source_data: dict[str, pl.DataFrame] = {}
    async with asyncio.TaskGroup() as tg:  # Python 3.11+ feature
        for name, indicator in sources.items():
            tg.create_task(
                self._fetch_source(name, indicator, observation_start, source_data)
            )
    return calculation(source_data)
```

**Key Feature**: Uses `asyncio.TaskGroup()` for parallel indicator fetching, optimal for composite scores requiring multiple indicators.

**Frequency Validation** (lines 72-84):
- Multi-source calculations validate all sources have matching frequency
- Raises `FrequencyMismatchError` if mismatch detected
- Ensures data can be properly aligned

**Post-Calculation Processing** (lines 106-133):
- Automatically computes 3-month rolling average on calculated values
- Drops pre-existing rolling averages from source data
- Frequency-aware window sizing (monthly: 3, weekly: 12, daily: 90 data points)

#### Existing Implementations

**Two calculated indicators currently exist** (both in inflation category):

1. **Core PCE 3m Annualized** (`src/indicators/inflation/__init__.py:81-87`):
   - Single-source calculation
   - Formula: `(((current / 3m_ago) ^ 4) - 1) * 100`
   - Demonstrates pattern: fetch → transform → return

2. **CPI Momentum** (`src/indicators/inflation/__init__.py:88-94`):
   - Single-source with complex logic
   - Computes intermediate columns (`rate_3m`, `rate_6m`)
   - Final value is difference of rates
   - Demonstrates multi-step calculations

**No multi-source calculations exist yet**, but infrastructure fully supports them.

#### Type Definitions

**Found in**: `src/indicators/calculations/types.py`

```python
SingleSourceCalculation = Callable[[pl.DataFrame], pl.DataFrame]
MultiSourceCalculation = Callable[[dict[str, pl.DataFrame]], pl.DataFrame]
```

These type aliases define function signatures for calculations. Composite scores would use similar patterns.

### 2. Indicator Registry System Architecture

**Core Components**:
- `Indicator` protocol (`src/indicators/protocol.py:16-27`)
- `IndicatorRegistry[T]` (`src/indicators/registry.py:15-59`)
- `RegistryManager` (`src/indicators/registry_manager.py:13-110`)
- `registry_setup.py` (`src/services/registry_setup.py:8-41`)

#### Indicator Protocol

```python
class Indicator(Protocol):
    @property
    def frequency(self) -> IndicatorFrequency: ...

    async def get_series(
        self, observation_start: str | None = None
    ) -> pl.DataFrame: ...

    def visualise_series(
        self, data: pl.DataFrame, config: VisualizationConfig | None = None
    ) -> go.Figure: ...
```

**Structural typing** - any class implementing these methods is an `Indicator`. This allows `CompositeScore` classes to be indicators.

#### IndicatorRegistry Pattern

**Generic registry** parameterized by enum type `T`:
```python
class IndicatorRegistry(Generic[T]):
    def __init__(self, enum_class: type[T]) -> None:
        self._enum_class = enum_class
        self._indicator_registry: dict[T, Indicator] = {}
```

**Builder pattern registration** (lines 32-36):
```python
def register_indicator(
    self, indicator_name: T, indicator: Indicator
) -> "IndicatorRegistry[T]":
    self._indicator_registry[indicator_name] = indicator
    return self  # Enables method chaining
```

**Cached data fetching** (lines 41-49):
```python
@st.cache_data(ttl=3600)  # 1-hour cache
def get_indicator_series(
    _self,
    indicator_name: T,
    observation_start: str,
) -> pl.DataFrame:
    indicator = _self._indicator_registry[indicator_name]
    data = asyncio.run(indicator.get_series(observation_start=observation_start))
    return data
```

**Key insight**: Registry doesn't care what type of indicator it stores - `FredIndicator`, `CalculatedIndicator`, or future `CompositeScore` all work identically.

#### RegistryManager

Coordinates multiple registries with metadata:
- Each category has: internal key, registry instance, display metadata
- Categories accessed by display name
- Metadata includes: name, icon emoji, description

**Registration pattern** (`src/services/registry_setup.py:21-28`):
```python
manager.register_registry(
    category_key="growth",
    registry=get_growth_indicator_registry(),
    metadata=RegistryMetadata(
        display_name="Growth",
        icon="📈",
        description="Growth indicators including PMI, Payrolls, and Housing",
    ),
)
```

#### Extension Points for New Categories

**Step-by-step pattern** (observed from growth/inflation implementations):

1. Create directory: `src/indicators/{category}/`
2. Create `indicators.py`:
   - Define `{Category}IndicatorName(StrEnum)` with user-facing names
   - Create `{CATEGORY}_INDICATOR_NAME_CODE_MAPPING` dict for FRED codes
3. Create `__init__.py`:
   - Define `@st.cache_resource` decorated factory function
   - Create `IndicatorRegistry[{Category}IndicatorName]`
   - Chain `.register_indicator()` calls
   - Return configured registry
4. Create `calculations.py` (if needed):
   - Define calculation functions matching type signatures
5. Register in `registry_setup.py`:
   - Call factory function and register with manager

**Example**: Adding a "Composite Scores" category would follow this exact pattern.

### 3. Data Fetching, Caching, and Storage Patterns

#### FRED Data Fetching

**Library**: `pyfredapi[polars]>=0.10.2` with native Polars support

**Async wrapper pattern** (`src/indicators/fred_indicator.py:19-28`):
```python
async def get_series(self, observation_start: str | None = None) -> pl.DataFrame:
    series = await asyncio.to_thread(
        pf.get_series,
        series_id=self.fred_code,
        return_format="polars",
        observation_start=self.set_default_observation_start(observation_start),
    )
    assert isinstance(series, pl.DataFrame)
    return series
```

Uses `asyncio.to_thread()` to wrap synchronous API call, enabling concurrent fetching.

#### Three-Layer Caching Strategy

**Layer 1**: Registry Manager Cache
- **Decorator**: `@st.cache_resource`
- **Scope**: Application-wide singleton
- **TTL**: No expiration (persists for app lifetime)
- **Content**: Entire `RegistryManager` with all registries
- **Location**: `src/app.py:22-25`

**Layer 2**: Individual Registry Cache
- **Decorator**: `@st.cache_resource`
- **Scope**: Per-category registry instance
- **TTL**: No expiration
- **Content**: Configured `IndicatorRegistry` with indicator mappings
- **Location**: Each category's `__init__.py` (e.g., `src/indicators/growth/__init__.py:12`)

**Layer 3**: Data Series Cache
- **Decorator**: `@st.cache_data(ttl=3600)`
- **Scope**: Per-indicator, per-date-range
- **TTL**: 1 hour (3600 seconds)
- **Content**: Polars DataFrames with fetched/calculated data
- **Location**: `src/indicators/registry.py:41`

**Cache key**: Based on `indicator_name` and `observation_start` parameters (note: `_self` parameter excluded via underscore prefix).

#### Data Storage Format

**Primary format**: Polars DataFrames

**Required schema**:
- `date` column: Time series index
- `value` column: Primary numeric data
- `rolling_avg_3m` column: Automatically computed 3-month rolling average

**No persistent storage**:
- All data in Streamlit cache (RAM only)
- No database, no file cache
- Cache invalidation: automatic after TTL expiration

**Data lifecycle**:
1. User request → cache lookup
2. Cache miss → async fetch from FRED API
3. Transform/calculate if needed
4. Store in Streamlit cache (1 hour)
5. Return DataFrame to UI
6. Visualize with Plotly

#### Async Patterns

**Pattern 1**: Async-to-thread for blocking I/O
```python
await asyncio.to_thread(pf.get_series, ...)
```

**Pattern 2**: Concurrent multi-source fetching
```python
async with asyncio.TaskGroup() as tg:
    for name, indicator in sources.items():
        tg.create_task(self._fetch_source(...))
```

**Pattern 3**: Async-to-sync bridge for Streamlit cache
```python
data = asyncio.run(indicator.get_series(...))
```

**Key insight**: Async architecture enables efficient parallel fetching for composite scores that need multiple indicators.

### 4. Visualization System

**Components**:
- `VisualizationConfig` (`src/indicators/visualization_config.py:12-39`)
- `MultiSeriesVisualizer` (`src/indicators/multi_series_visualizer.py:10-36`)
- `Indicator.visualise_series()` protocol method

#### Configuration Pattern

**Builder pattern** for defining series to plot:
```python
config = (
    VisualizationConfig()
    .add_series("value", "Original")
    .add_series("rolling_avg_3m", "3-Month Avg")
)
```

**Factory method** for common patterns:
```python
config = VisualizationConfig.create_with_original_and_3m_avg()
```

#### MultiSeriesVisualizer

Converts config into Plotly figure:
- Iterates through `config.series_definitions`
- For each series, adds `go.Scatter` trace
- X-axis: `data["date"]`
- Y-axis: `data[series_def.column_name]`
- Customizable: line dash style, color, display name
- Layout: standard time-series chart with "x unified" hover mode

#### Integration with Indicators

All indicators implement `visualise_series()` method:
- **FredIndicator** (lines 40-47): Default shows "value" column
- **CalculatedIndicator** (lines 181-189): Default shows output column
- **Future CompositeScore**: Would implement same pattern

**Key insight**: Visualization system already supports multiple series - can easily show composite scores alongside component indicators.

### 5. Test Coverage and Patterns

**Current state**: ✅ **Test infrastructure implemented**

#### Test Structure

**Test directories** (`tests/`):
- `tests/conftest.py` - Shared pytest configuration and fixtures
- `tests/fixtures/` - Test data fixtures
- `tests/test_calculations/` - Tests for calculation functions
- `tests/test_indicators/` - Tests for indicator implementations
  - `test_fred_indicator.py` - FredIndicator tests
  - `test_calculated_indicator.py` - CalculatedIndicator tests
  - `test_registry.py` - IndicatorRegistry tests
  - `test_registry_manager.py` - RegistryManager tests
  - `test_visualization/` - Visualization system tests
    - `test_visualization_config.py`
    - `test_multi_series_visualizer.py`

#### Testing Dependencies

```toml
[dependency-groups]
dev = [
    "pyrefly>=0.46.1",      # Type checking
    "ruff>=0.14.10",        # Linting and formatting
    "pytest>=9.0.2",        # Testing framework
    "pytest-asyncio>=1.3.0" # Async test support
]
```

#### Test Commands

Available via justfile:
```justfile
tests:
    uv run pytest tests/ -v
```

Also accessible as `just test` command.

#### Test Coverage

**Components with test coverage**:
1. ✅ CalculatedIndicator (single-source and multi-source calculations)
2. ✅ IndicatorRegistry (caching, enum resolution)
3. ✅ RegistryManager (category lookup)
4. ✅ Calculation functions (inflation calculations)
5. ✅ FredIndicator (data fetching patterns, rolling averages)
6. ✅ Visualization system (config, multi-series rendering)

**Testing patterns established**:
- Mock indicators for testing (`conftest.py` fixtures)
- Async test support with `@pytest.mark.asyncio`
- DataFrame assertions for Polars data
- Sample data fixtures for consistent testing

**Phase 1 verification**: Tests can now verify existing functionality works correctly before adding composite scores.

---

## Architectural Gaps for Composite Scoring and Regime Detection

### Gap 1: Score Calculation Abstraction

**Current state**: `CalculatedIndicator` handles transformations but no concept of "scores"

**What's needed**:
- `CompositeScore` class/protocol for score-specific calculations
- Score normalization utilities (z-score, percentile rank, threshold-based)
- Score interpretation (0-100 scale mapping)
- Configurable weights for component indicators

**Extension approach**:
Create new `CompositeScore` class implementing `Indicator` protocol:
```python
class CompositeScore(Indicator):
    def __init__(
        self,
        source_indicators: dict[str, Indicator],
        weights: dict[str, float],
        normalization: NormalizationMethod,
    ):
        self._sources = source_indicators
        self._weights = weights
        self._normalization = normalization

    async def get_series(self, observation_start: str | None = None) -> pl.DataFrame:
        # Fetch all sources concurrently (like CalculatedIndicator)
        # Normalize each indicator
        # Apply weighted average
        # Return 0-100 score
```

This would register like any other indicator - no registry changes needed.

### Gap 2: Score Aggregation Pattern

**Current state**: No aggregation beyond single indicator calculations

**What's needed**:
- Pattern for combining multiple scores into regime classification
- Threshold-based rules (e.g., "Goldilocks" = Growth rising + Inflation falling)
- Trend detection (rising vs falling determination)

**Extension approach**:
Create `RegimeDetector` that takes composite scores as input:
```python
class RegimeDetector:
    def __init__(
        self,
        growth_score: CompositeScore,
        inflation_score: CompositeScore,
    ):
        self._growth = growth_score
        self._inflation = inflation_score

    async def detect_regime(self) -> RegimeClassification:
        growth_data = await self._growth.get_series()
        inflation_data = await self._inflation.get_series()

        growth_trend = self._detect_trend(growth_data)
        inflation_trend = self._detect_trend(inflation_data)

        return self._map_to_regime(growth_trend, inflation_trend)
```

Could be registered as an indicator or as separate component.

### Gap 3: Time-Series Score Storage

**Current state**: Streamlit cache stores DataFrames but no pattern for historical scores

**What's needed**:
- Storage of historical composite scores for sparkline charts
- Efficient retrieval of recent score history (e.g., last 30 days)
- Score change tracking (delta from previous period)

**Extension approach**:
Leverage existing cache pattern with extended TTL:
```python
@st.cache_data(ttl=86400)  # 24-hour cache for scores
def get_score_history(
    score_name: str,
    observation_start: str,
) -> pl.DataFrame:
    score = score_registry.get_score(score_name)
    return await score.get_series(observation_start=observation_start)
```

Sparklines would use this cached history. No persistent storage needed initially.

**Alternative**: Create `ScoreRegistry` similar to `IndicatorRegistry` with longer cache TTL for stability.

### Gap 4: Normalization Utilities

**Current state**: Calculations transform raw values but no normalization abstractions

**What's needed**:
- Z-score normalization (standard deviations from mean)
- Percentile rank normalization (historical distribution)
- Threshold-based normalization (predefined ranges)
- Historical window management (rolling statistics)

**Extension approach**:
Create normalization module `src/indicators/calculations/normalization.py`:
```python
def z_score_normalize(
    data: pl.DataFrame,
    column: str,
    window: int | None = None,
) -> pl.DataFrame:
    """
    Normalize column to z-score (standard deviations from mean).

    Args:
        data: DataFrame with time series
        column: Column to normalize
        window: Rolling window size (None = full history)

    Returns:
        DataFrame with additional 'z_score' column
    """
    if window is None:
        mean = data[column].mean()
        std = data[column].std()
    else:
        mean = data[column].rolling_mean(window_size=window)
        std = data[column].rolling_std(window_size=window)

    return data.with_columns(
        z_score=((pl.col(column) - mean) / std)
    )

def percentile_rank_normalize(
    data: pl.DataFrame,
    column: str,
    lookback_window: int | None = None,
) -> pl.DataFrame:
    """
    Normalize column to percentile rank (0-100).

    Returns DataFrame with 'percentile' column.
    """
    # Implementation using Polars rank operations

def threshold_normalize(
    data: pl.DataFrame,
    column: str,
    thresholds: dict[str, float],
) -> pl.DataFrame:
    """
    Map column values to 0-100 scale based on thresholds.

    Args:
        data: DataFrame with time series
        column: Column to normalize
        thresholds: Dict like {'low': 45, 'neutral': 50, 'high': 55}

    Returns:
        DataFrame with 'normalized_score' column
    """
    # Implementation using conditional mapping
```

These would be used by `CompositeScore` implementations.

### Gap 5: Score Registry Pattern

**Current state**: Only indicator registries exist

**What's needed**:
- Separate registry for composite scores (or unified registry)
- Score metadata (components, weights, interpretation ranges)
- Score hierarchy (category scores vs overall regime)

**Extension approach Option 1** (Unified):
Composite scores are just indicators - register in existing system:
```python
# In src/indicators/composite_scores/__init__.py
@st.cache_resource
def get_composite_score_registry() -> IndicatorRegistry:
    return (
        IndicatorRegistry[CompositeScoreName](enum_class=CompositeScoreName)
        .register_indicator(
            indicator_name=CompositeScoreName.GROWTH_SCORE,
            indicator=CompositeScore(
                source_indicators={
                    "lei": lei_indicator,
                    "pmi": pmi_indicator,
                    "claims": claims_indicator,
                },
                weights={"lei": 0.35, "pmi": 0.30, "claims": 0.20},
                normalization=ZScoreNormalization(),
            ),
        )
    )
```

**Extension approach Option 2** (Separate):
Create parallel `ScoreRegistry` with additional metadata:
```python
class ScoreRegistry(Generic[T]):
    def __init__(self, enum_class: type[T]):
        self._enum_class = enum_class
        self._score_registry: dict[T, CompositeScore] = {}
        self._score_metadata: dict[T, ScoreMetadata] = {}

    def register_score(
        self,
        score_name: T,
        score: CompositeScore,
        metadata: ScoreMetadata,
    ) -> "ScoreRegistry[T]":
        self._score_registry[score_name] = score
        self._score_metadata[score_name] = metadata
        return self
```

**Recommendation**: Option 1 (Unified) is simpler and leverages existing patterns.

---

## Code References

### Core Infrastructure Files

- `src/indicators/protocol.py:16-27` - Indicator protocol definition
- `src/indicators/fred_indicator.py:14-75` - FRED data fetching implementation
- `src/indicators/calculations/calculated_indicator.py:17-189` - Calculated indicator with multi-source support
- `src/indicators/registry.py:15-59` - Generic type-safe registry
- `src/indicators/registry_manager.py:13-110` - Multi-registry coordinator
- `src/services/registry_setup.py:8-41` - Central registry configuration

### Calculation Examples

- `src/indicators/inflation/calculations.py:4-14` - 3-month annualized rate calculation
- `src/indicators/inflation/calculations.py:17-45` - CPI momentum calculation
- `src/indicators/calculations/types.py:1-9` - Calculation function type definitions

### Registry Patterns

- `src/indicators/growth/__init__.py:12-67` - Growth category factory (FRED only)
- `src/indicators/inflation/__init__.py:17-95` - Inflation category factory (mixed FRED and calculated)
- `src/indicators/growth/indicators.py:1-21` - Growth enum and FRED code mapping
- `src/indicators/inflation/indicators.py:1-23` - Inflation enum including calculated indicators

### Visualization System

- `src/indicators/visualization_config.py:12-39` - Configuration builder
- `src/indicators/multi_series_visualizer.py:10-36` - Plotly chart renderer
- `src/components/indicator_chart.py:42-77` - UI component integrating visualization

### Data Flow

- `src/app.py:22-25` - Registry manager cache
- `src/pages/indicator_explorer.py:25-84` - User interaction and data fetching
- `src/services/indicator_service.py:28-69` - Data service layer with error handling

---

## Architecture Documentation

### Current Patterns

**Protocol-Based Design**:
- `Indicator` protocol enables structural typing
- Any class implementing `frequency`, `get_series()`, `visualise_series()` is an indicator
- No inheritance required - composition over inheritance

**Registry Pattern**:
- Generic `IndicatorRegistry[T]` parameterized by enum type
- Type-safe indicator lookup via enum values
- Builder pattern for registration (method chaining)

**Async Concurrency**:
- `asyncio.to_thread()` for wrapping blocking I/O
- `asyncio.TaskGroup()` for parallel multi-source fetching
- `asyncio.run()` bridge for Streamlit sync context

**Multi-Layer Caching**:
- Resource cache for singletons (registries, managers)
- Data cache with TTL for time-series data
- Cache key construction excludes registry instance via `_self` parameter

**Polars-First Data Flow**:
- All DataFrames use Polars (no Pandas)
- Consistent schema: `date`, `value`, `rolling_avg_3m`
- Polars expression API for transformations

**Visualization via Configuration**:
- Declarative config via builder pattern
- Plotly rendering with multiple series support
- Config created at render time based on user selection

### Extension Points for Phase 1

**Adding Composite Scores**:
1. Create `CompositeScore` class implementing `Indicator` protocol
2. Accept multi-source indicators in constructor
3. Implement async `get_series()` using existing multi-source pattern
4. Register in new category or alongside indicators

**Adding Score Normalization**:
1. Create `normalization.py` module with utility functions
2. Use Polars operations for z-score, percentile rank, threshold mapping
3. Import in `CompositeScore` implementations

**Adding Regime Detection**:
1. Create `RegimeDetector` class taking composite scores
2. Implement trend detection logic (rising/falling determination)
3. Map score trends to 4-quadrant regime (Goldilocks, Reflation, Stagflation, Risk-Off)
4. Could implement `Indicator` protocol or be separate service

**Test Coverage for New Features**:
1. Follow existing test patterns in `tests/` directory
2. Use mock indicators from `conftest.py` fixtures
3. Write async tests with `@pytest.mark.asyncio`
4. Ensure tests pass before implementing composite scores
5. Run tests with `just test` command

---

## Phase 1 Deliverables Assessment

### Deliverable 1: Architectural Decision Document

**Status**: ✅ **This document serves as the architectural decision document**

**Decisions documented**:
- Composite scores should implement `Indicator` protocol (unified registry)
- Multi-source indicator fetching pattern is ready for score calculations
- Score normalization should be separate utility module
- Caching strategy (1-hour TTL for data, 24-hour for scores recommended)
- Test infrastructure implemented with comprehensive coverage

### Deliverable 2: Refactoring of Existing Indicator Infrastructure

**Status**: ⚠️ **Minimal refactoring needed**

**What works**:
- `CalculatedIndicator` already supports multi-source with concurrent fetching
- Registry system is generic and extensible
- Visualization handles multiple series

**What needs refinement**:
- No refactoring blockers identified
- Current architecture is well-designed for extension
- Recommendation: Add abstractions on top rather than refactoring bottom

### Deliverable 3: Pattern Documentation for Adding New Categories

**Status**: ✅ **Patterns clearly documented**

**Documented patterns**:
1. Category structure (directory, enum, factory, calculations)
2. Registration pattern (builder with method chaining)
3. Factory function with `@st.cache_resource`
4. FRED indicator creation
5. Calculated indicator creation (single and multi-source)
6. Registry manager registration with metadata

**Example documented**: Growth (FRED only) and Inflation (mixed) categories demonstrate both patterns.

### Deliverable 4: Verification that Existing Functionality Works

**Status**: ✅ **Automated tests implemented and passing**

**Current verification**:
- `just check` runs ruff linting and pyrefly type checking ✅
- `just test` runs pytest test suite ✅
- Application runs and displays indicators correctly (manual + automated)

**Automated test coverage includes**:
- ✅ Data fetching works correctly (FredIndicator tests)
- ✅ Calculations produce expected results (inflation calculation tests)
- ✅ Registry enum resolution and lookup (IndicatorRegistry tests)
- ✅ Multi-source concurrent fetching (CalculatedIndicator tests)
- ✅ Frequency validation (CalculatedIndicator multi-source tests)
- ✅ Visualization system (config and rendering tests)

**Test infrastructure ready**: Can now verify existing functionality before adding composite scores.

---

## Success Criteria for Phase 1 Completion

### From SUPER_PLAN_MACRO_DASHBOARD.md

**Criterion 1**: `just check` passes
- **Current**: ✅ Passes (linting + type checking)
- **Enhancement**: ✅ `just test` target added and passing

**Criterion 2**: Existing dashboard functionality unchanged
- **Current**: ✅ No changes made, functionality intact
- **Verification**: Manual testing confirms app runs correctly

**Criterion 3**: Clear extension points documented
- **Current**: ✅ This document provides comprehensive extension documentation
- **Includes**: Patterns, examples, step-by-step instructions

**Criterion 4**: Patterns documented
- **Current**: ✅ All major patterns documented with code references
- **Coverage**: Registry, calculation, visualization, caching, async patterns

### Additional Recommendations

**Test Coverage for New Features**: Follow existing patterns for:
- Score normalization utilities
- `CompositeScore` implementations
- Regime detection logic

**Composite Score Prototype**: Create one example composite score during Phase 1 to validate architecture works in practice (e.g., Simple Growth Score from 2-3 indicators).

**Documentation Updates**: Update CLAUDE.md with:
- Composite score patterns
- Score normalization approach
- Phase 1 architectural decisions

---

## Open Questions for Phase 1 Implementation

### Question 1: Score Storage Duration

**Context**: Streamlit cache has 1-hour TTL for indicator data. Composite scores might benefit from longer TTL for stability.

**Options**:
- A) Use same 1-hour TTL as indicators (re-computes scores hourly)
- B) Use 24-hour TTL for scores (more stable, less frequent computation)
- C) Use resource cache (never expires, must manually invalidate)

**Recommendation**: Start with Option A (1-hour TTL) for consistency, can increase later if needed.

### Question 2: Score Normalization Method

**Context**: MACRO_DATA_SPECIFICATION.md mentions z-score, percentile rank, and threshold-based normalization.

**Question**: Which normalization method should be default for Phase 1?

**Recommendation**:
- Start with **threshold-based** (simplest, interpretable)
- Add z-score and percentile rank as alternatives in Phase 7
- Different indicators may need different normalization methods

### Question 3: Regime Detection Integration

**Context**: Regime detection combines growth and inflation scores. Should it be an indicator or separate component?

**Options**:
- A) `RegimeIndicator` implementing `Indicator` protocol (unified)
- B) Separate `RegimeDetector` service class (specialized)
- C) Simple function taking two scores (minimal)

**Recommendation**: Option A (RegimeIndicator) for consistency with existing patterns. Can return categorical data instead of numeric values.

### Question 4: Test Coverage for Composite Scores

**Context**: Test infrastructure now in place. What should be tested for composite scores?

**Priority order for new features**:
1. Normalization utilities (z-score, percentile rank, threshold-based)
2. `CompositeScore` class (multi-source calculation, weighted aggregation)
3. Regime detection logic (trend detection, quadrant mapping)
4. Integration tests (end-to-end score calculation)

**Recommendation**: Follow existing test patterns established in Phase 1. Use mock indicators from `conftest.py` for testing composite scores.

### Question 5: Multi-Category Composite Scores

**Context**: Composite scores might need indicators from different categories (e.g., Growth Score needs both PMI from growth category and claims data).

**Question**: How to handle cross-category dependencies in score definitions?

**Current capability**: `CalculatedIndicator` accepts `dict[str, Indicator]` - works regardless of indicator category.

**Recommendation**: No changes needed. Composite score factory can import indicators from multiple categories and pass them to `CompositeScore` constructor.

---

## Next Steps for Phase 1

### Immediate Actions

1. **Create Normalization Module** (Priority: High)
   - Create `src/indicators/calculations/normalization.py`
   - Implement threshold-based normalization (0-100 scale)
   - Add docstrings with examples
   - Write unit tests for normalization functions

2. **Design CompositeScore Class** (Priority: High)
   - Create `src/indicators/composite_scores/composite_score.py`
   - Implement `Indicator` protocol
   - Support multi-source indicators
   - Apply normalization to each component
   - Compute weighted average
   - Return 0-100 score DataFrame

3. **Create Prototype Growth Score** (Priority: Medium)
   - Use 2-3 growth indicators (PMI, LEI, Claims)
   - Define simple weights (equal or based on framework doc)
   - Register in new "Composite Scores" category
   - Verify visualization works
   - Document pattern for other scores

4. **Update Documentation** (Priority: Medium)
   - Add composite score patterns to CLAUDE.md
   - Document test running instructions
   - Create example of adding new score
   - Link to this research document

### Phase 2-6 Dependencies

**Phase 2-6** (Category Implementations) can proceed in parallel once:
- Normalization utilities are available
- `CompositeScore` class is implemented
- Pattern is documented
- Tests are passing

**Phase 7** (Composite Score Infrastructure) will formalize what Phase 1 prototypes.

**Phase 8** (Regime Detection) can begin once Phase 2-3 (Growth + Inflation) have composite scores.

---

## Conclusion

The cc-analyst codebase has a **well-architected foundation** that is **ready for Phase 1 extension** to support composite scoring and regime detection. The key architectural patterns (protocol-based design, generic registries, async concurrency, multi-source calculations) are already in place and working effectively.

**Test infrastructure complete**: ✅ Comprehensive test coverage now in place with pytest, async test support, and established testing patterns. Tests verify existing functionality works correctly.

**Recommended approach**: Extend the existing architecture rather than refactor. Create new abstractions (`CompositeScore`, normalization utilities, `RegimeDetector`) that leverage existing patterns. The registry system is flexible enough to accommodate scores without modification.

**Time estimate**: With current architecture and test infrastructure, Phase 1 implementation should be straightforward:
- Normalization utilities: 1 day
- CompositeScore class: 1-2 days
- Prototype score: 0.5-1 day
- Documentation: 0.5 day
- **Total**: ~3-5 days for complete Phase 1

The foundation is solid and well-tested. Phase 1 can proceed with confidence.

---

## Follow-up Research [2025-12-30T10:30:00+0000]

### Test Infrastructure Implementation Update

**Status**: ✅ **Test infrastructure has been implemented**

Following the initial research findings, comprehensive test infrastructure has been added to the project:

#### Implemented Test Structure

**Test directories created**:
- `tests/conftest.py` - Pytest configuration with shared fixtures including `MockIndicator` class
- `tests/fixtures/sample_data.py` - Reusable test data fixtures
- `tests/test_calculations/` - Tests for calculation functions
  - `test_inflation_calculations.py` - Tests for 3m annualized rate and CPI momentum
- `tests/test_indicators/` - Tests for all indicator types
  - `test_fred_indicator.py` - FredIndicator implementation tests
  - `test_calculated_indicator.py` - Single and multi-source calculation tests
  - `test_registry.py` - Registry enum resolution and caching tests
  - `test_registry_manager.py` - Multi-registry coordination tests
  - `test_visualization/` - Visualization system tests
    - `test_visualization_config.py` - Config builder tests
    - `test_multi_series_visualizer.py` - Plotly rendering tests

#### Dependencies Added

Added to `pyproject.toml` dev dependencies:
```toml
"pytest>=9.0.2",
"pytest-asyncio>=1.3.0"
```

#### Test Command

Added to `justfile`:
```justfile
tests:
    uv run pytest tests/ -v
```

Accessible via `just test` or `just tests` commands.

#### Test Coverage Achieved

All critical components now have test coverage:
- ✅ FredIndicator (data fetching, rolling averages, frequency handling)
- ✅ CalculatedIndicator (single-source and multi-source with concurrent fetching)
- ✅ IndicatorRegistry (enum resolution, caching, data series fetching)
- ✅ RegistryManager (category lookup, metadata handling)
- ✅ Calculation functions (inflation calculations tested with sample data)
- ✅ Visualization system (config builder, multi-series rendering)

#### Testing Patterns Established

**Mock Indicator Pattern** (`conftest.py`):
```python
class MockIndicator:
    """Mock indicator for testing without FRED API calls"""
    frequency = IndicatorFrequency.MONTH

    async def get_series(self, observation_start: str | None = None) -> pl.DataFrame:
        return pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01", "2024-03-01"],
            "value": [100.0, 102.0, 104.0],
        })

    def visualise_series(self, data, config=None):
        return go.Figure()
```

**Async Test Pattern**:
```python
@pytest.mark.asyncio
async def test_calculated_indicator_single_source():
    source = MockIndicator()
    calc = CalculatedIndicator(source=source, calculation=simple_calc)
    data = await calc.get_series()
    assert data["value"][0] == expected_value
```

**DataFrame Assertion Pattern**:
```python
def test_calculation_function():
    input_data = sample_data_fixture()
    result = calculation_function(input_data)

    assert "value" in result.columns
    assert result["value"][0] == pytest.approx(expected)
    assert len(result) == expected_length
```

#### Impact on Phase 1

**Primary gap resolved**: The "zero test coverage" critical gap identified in the original research has been fully addressed.

**Updated Phase 1 timeline**: With test infrastructure complete, Phase 1 can proceed directly to:
1. Normalization module implementation (with tests)
2. CompositeScore class implementation (with tests)
3. Prototype score (with tests)
4. Documentation updates

**Estimated time savings**: ~1-2 days removed from Phase 1 timeline (from 5-7 days to 3-5 days).

**Quality improvement**: Can now develop composite scores with confidence that existing functionality remains intact, verified by automated tests.
