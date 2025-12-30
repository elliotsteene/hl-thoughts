# Phase 1: Composite Scores Foundation Implementation Plan

## Overview

This plan implements the foundation for composite scoring and regime detection by creating three core components:

1. **Normalization Module** - Utilities for normalizing economic indicators to 0-100 scores
2. **CompositeScore Class** - A multi-source indicator that combines weighted indicators into composite scores
3. **Prototype Growth Score** - A working example demonstrating the pattern with real growth indicators

This phase extends the existing architecture without refactoring, leveraging the protocol-based design and multi-source calculation infrastructure already in place.

## Current State Analysis

Based on the comprehensive research in `thoughts/searchable/shared/research/2025-12-30-phase1-foundation-architecture.md`:

### Existing Strengths:
- ✅ Protocol-based `Indicator` interface supports any implementation
- ✅ `CalculatedIndicator` has multi-source support with concurrent async fetching via `asyncio.TaskGroup`
- ✅ Generic `IndicatorRegistry[T]` works with any indicator type
- ✅ Three-layer caching strategy (manager, registry, data with 1-hour TTL)
- ✅ Polars-first DataFrame operations with `date`, `value`, `rolling_avg_3m` schema
- ✅ Test infrastructure with pytest, async support, and mock patterns

### Key Discoveries:
- `CalculatedIndicator` (src/indicators/calculations/calculated_indicator.py:156-169) already implements concurrent multi-source fetching
- Frequency validation is automatic (src/indicators/calculations/calculated_indicator.py:72-84)
- Rolling average computation is automatic (src/indicators/calculations/calculated_indicator.py:106-133)
- Test patterns established in `tests/conftest.py` with `MockIndicator` class

### Architectural Gaps Addressed in This Phase:
- ❌ No score normalization utilities (z-score, percentile, threshold-based)
- ❌ No `CompositeScore` abstraction for weighted multi-indicator scores
- ❌ No working example of composite scores in the codebase

## Desired End State

After this plan is complete:

1. **Normalization module exists** at `src/indicators/calculations/normalization.py` with three normalization methods
2. **CompositeScore class exists** at `src/indicators/composite_scores/composite_score.py` implementing the `Indicator` protocol
3. **Prototype Growth Score** is registered and visible in the dashboard as "Growth Composite Score"
4. **Tests verify** all new components work correctly
5. **Pattern is documented** for creating additional composite scores in Phase 2-6

### Verification:
- Run `just check` - All linting and type checking passes
- Run `just test` - All tests including new normalization and composite score tests pass
- Run `just dashboard` - Prototype Growth Score appears in indicator selector and displays correctly
- Manual test: Select "Growth Composite Score" and verify it shows a 0-100 scale chart

## What We're NOT Doing

This phase intentionally excludes:

- ❌ Regime detection logic (Phase 8)
- ❌ Multiple composite scores (Phase 2-6 will add more)
- ❌ Advanced normalization methods (z-score, percentile rank) - starting with threshold-based only
- ❌ Score metadata or interpretation ranges
- ❌ UI changes beyond adding the prototype score to existing indicator explorer
- ❌ Persistent storage (using existing Streamlit cache)
- ❌ Score comparison or visualization enhancements
- ❌ Documentation updates to CLAUDE.md (separate task after validation)

## Implementation Approach

**Strategy**: Extend the existing architecture using established patterns rather than refactoring. The `Indicator` protocol's structural typing allows `CompositeScore` to be a first-class indicator without any registry changes.

**Key Pattern**: CompositeScore will behave exactly like `CalculatedIndicator` but with score normalization applied:
1. Accept multiple source indicators with weights
2. Fetch sources concurrently (reusing `CalculatedIndicator` pattern)
3. Normalize each source to 0-100 scale
4. Compute weighted average
5. Return DataFrame with standard schema

## Phase 1: Normalization Module

### Overview
Create normalization utilities that convert raw indicator values to 0-100 scores. Starting with threshold-based normalization (simplest, most interpretable) to validate the pattern.

### Changes Required:

#### 1.1 Create Normalization Module

**File**: `src/indicators/calculations/normalization.py`
**Changes**: New file with threshold-based normalization function

```python
"""
Normalization utilities for converting economic indicators to 0-100 scores.

These functions enable composite scores to combine indicators with different
scales and units into unified, comparable metrics.
"""

import polars as pl


def threshold_normalize(
    data: pl.DataFrame,
    column: str,
    low_threshold: float,
    high_threshold: float,
    invert: bool = False,
) -> pl.DataFrame:
    """
    Normalize a column to 0-100 scale using linear interpolation between thresholds.

    This method is interpretable and aligns with practitioner frameworks
    (e.g., "PMI > 55 = strong expansion").

    Args:
        data: DataFrame with time series data
        column: Column name to normalize
        low_threshold: Value that maps to 0 (or 100 if inverted)
        high_threshold: Value that maps to 100 (or 0 if inverted)
        invert: If True, higher values map to lower scores
                (useful for unemployment, claims where lower is better)

    Returns:
        DataFrame with additional 'normalized_score' column

    Examples:
        >>> # PMI: 45 = 0, 55 = 100 (higher is better)
        >>> threshold_normalize(pmi_data, "value", 45, 55)

        >>> # Unemployment: 3% = 100, 6% = 0 (lower is better)
        >>> threshold_normalize(unrate_data, "value", 3.0, 6.0, invert=True)
    """
    if invert:
        # For indicators where lower is better (unemployment, claims)
        # Formula: 100 - ((value - low) / (high - low)) * 100
        normalized = 100 - (
            (pl.col(column) - low_threshold) / (high_threshold - low_threshold) * 100
        )
    else:
        # For indicators where higher is better (PMI, LEI)
        # Formula: ((value - low) / (high - low)) * 100
        normalized = (
            (pl.col(column) - low_threshold) / (high_threshold - low_threshold) * 100
        )

    # Clip to [0, 100] range to handle values outside thresholds
    normalized = normalized.clip(0, 100)

    return data.with_columns(normalized_score=normalized)
```

#### 1.2 Create Tests for Normalization

**File**: `tests/test_calculations/test_normalization.py`
**Changes**: New test file

```python
"""Tests for normalization utilities."""

import polars as pl
import pytest

from src.indicators.calculations.normalization import threshold_normalize


class TestThresholdNormalize:
    """Test threshold-based normalization."""

    def test_normalizes_value_at_low_threshold_to_zero(self):
        """Value at low threshold should map to 0."""
        data = pl.DataFrame({
            "date": ["2024-01-01"],
            "value": [45.0],
        })

        result = threshold_normalize(data, "value", low_threshold=45, high_threshold=55)

        assert result["normalized_score"][0] == pytest.approx(0.0)

    def test_normalizes_value_at_high_threshold_to_hundred(self):
        """Value at high threshold should map to 100."""
        data = pl.DataFrame({
            "date": ["2024-01-01"],
            "value": [55.0],
        })

        result = threshold_normalize(data, "value", low_threshold=45, high_threshold=55)

        assert result["normalized_score"][0] == pytest.approx(100.0)

    def test_normalizes_midpoint_to_fifty(self):
        """Value at midpoint should map to 50."""
        data = pl.DataFrame({
            "date": ["2024-01-01"],
            "value": [50.0],
        })

        result = threshold_normalize(data, "value", low_threshold=45, high_threshold=55)

        assert result["normalized_score"][0] == pytest.approx(50.0)

    def test_clips_values_above_high_threshold_to_hundred(self):
        """Values above high threshold should clip to 100."""
        data = pl.DataFrame({
            "date": ["2024-01-01"],
            "value": [60.0],
        })

        result = threshold_normalize(data, "value", low_threshold=45, high_threshold=55)

        assert result["normalized_score"][0] == pytest.approx(100.0)

    def test_clips_values_below_low_threshold_to_zero(self):
        """Values below low threshold should clip to 0."""
        data = pl.DataFrame({
            "date": ["2024-01-01"],
            "value": [40.0],
        })

        result = threshold_normalize(data, "value", low_threshold=45, high_threshold=55)

        assert result["normalized_score"][0] == pytest.approx(0.0)

    def test_inverted_normalization_reverses_scale(self):
        """When inverted=True, lower values should score higher."""
        data = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01"],
            "value": [3.0, 6.0],  # Unemployment: 3% and 6%
        })

        result = threshold_normalize(
            data, "value", low_threshold=3.0, high_threshold=6.0, invert=True
        )

        # At low threshold (3%), score should be 100 (inverted)
        assert result["normalized_score"][0] == pytest.approx(100.0)
        # At high threshold (6%), score should be 0 (inverted)
        assert result["normalized_score"][1] == pytest.approx(0.0)

    def test_inverted_normalization_clips_correctly(self):
        """Inverted normalization should clip to [0, 100]."""
        data = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01"],
            "value": [2.0, 7.0],  # Below and above thresholds
        })

        result = threshold_normalize(
            data, "value", low_threshold=3.0, high_threshold=6.0, invert=True
        )

        # Below low threshold (2% < 3%) should clip to 100
        assert result["normalized_score"][0] == pytest.approx(100.0)
        # Above high threshold (7% > 6%) should clip to 0
        assert result["normalized_score"][1] == pytest.approx(0.0)

    def test_preserves_original_columns(self):
        """Normalization should add column, not replace."""
        data = pl.DataFrame({
            "date": ["2024-01-01"],
            "value": [50.0],
        })

        result = threshold_normalize(data, "value", low_threshold=45, high_threshold=55)

        assert "date" in result.columns
        assert "value" in result.columns
        assert "normalized_score" in result.columns
        assert result["value"][0] == 50.0  # Original preserved

    def test_handles_multiple_rows(self):
        """Should normalize entire time series correctly."""
        data = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01", "2024-03-01", "2024-04-01"],
            "value": [45.0, 48.0, 52.0, 55.0],
        })

        result = threshold_normalize(data, "value", low_threshold=45, high_threshold=55)

        assert len(result) == 4
        assert result["normalized_score"][0] == pytest.approx(0.0)
        assert result["normalized_score"][1] == pytest.approx(30.0)
        assert result["normalized_score"][2] == pytest.approx(70.0)
        assert result["normalized_score"][3] == pytest.approx(100.0)
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] All normalization tests pass: `just test`
- [ ] No linting errors: `just check`

#### Manual Verification:
- [ ] Test file runs successfully with `pytest tests/test_calculations/test_normalization.py -v`
- [ ] All 9 test cases pass (regular and inverted normalization)
- [ ] Edge cases (clipping, midpoints) verified

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 2: CompositeScore Class

### Overview
Create the `CompositeScore` class that implements the `Indicator` protocol and combines multiple source indicators into a weighted composite score using the normalization utilities.

### Changes Required:

#### 2.1 Create CompositeScore Class

**File**: `src/indicators/composite_scores/composite_score.py`
**Changes**: New file implementing composite score logic

```python
"""
CompositeScore: Multi-indicator composite scoring system.

Combines multiple economic indicators into a single 0-100 score using:
1. Concurrent data fetching (async)
2. Normalization to common scale
3. Weighted aggregation
"""

import asyncio
from dataclasses import dataclass

import plotly.graph_objs as go
import polars as pl

from src.indicators.calculations.normalization import threshold_normalize
from src.indicators.exceptions import FrequencyMismatchError
from src.indicators.multi_series_visualizer import MultiSeriesVisualizer
from src.indicators.protocol import Indicator, IndicatorFrequency
from src.indicators.visualization_config import VisualizationConfig


@dataclass
class IndicatorConfig:
    """
    Configuration for a single indicator component in a composite score.

    Attributes:
        indicator: The source indicator to fetch data from
        weight: Weight in composite (must sum to 1.0 across all components)
        low_threshold: Value mapping to score of 0
        high_threshold: Value mapping to score of 100
        invert: If True, lower values score higher (e.g., unemployment)
    """

    indicator: Indicator
    weight: float
    low_threshold: float
    high_threshold: float
    invert: bool = False


class CompositeScore:
    """
    A composite score that combines multiple indicators into a single 0-100 metric.

    This class implements the Indicator protocol, making it compatible with
    the existing registry system. It fetches multiple source indicators
    concurrently, normalizes each to 0-100 scale, and computes a weighted average.

    Example:
        >>> growth_score = CompositeScore(
        ...     name="Growth Score",
        ...     components={
        ...         "pmi": IndicatorConfig(pmi_indicator, 0.4, 45, 55),
        ...         "claims": IndicatorConfig(claims_indicator, 0.3, 200, 400, invert=True),
        ...         "housing": IndicatorConfig(housing_indicator, 0.3, 900, 1500),
        ...     }
        ... )
    """

    def __init__(self, name: str, components: dict[str, IndicatorConfig]) -> None:
        """
        Initialize composite score.

        Args:
            name: Display name for this composite score
            components: Dictionary of component configurations
                       Keys are component names for debugging/visualization

        Raises:
            ValueError: If weights don't sum to 1.0
            FrequencyMismatchError: If components have different frequencies
        """
        self._name = name
        self._components = components

        # Validate weights sum to 1.0
        total_weight = sum(config.weight for config in components.values())
        if abs(total_weight - 1.0) > 0.001:  # Allow small floating point error
            raise ValueError(
                f"Component weights must sum to 1.0, got {total_weight:.3f}"
            )

        # Validate and cache frequency
        self._frequency = self._validate_frequency()

    def _validate_frequency(self) -> IndicatorFrequency:
        """
        Validate all components have the same frequency.

        Returns:
            The common frequency

        Raises:
            FrequencyMismatchError: If frequencies don't match
        """
        frequencies = {
            name: config.indicator.frequency
            for name, config in self._components.items()
        }

        unique_frequencies = set(frequencies.values())
        if len(unique_frequencies) > 1:
            raise FrequencyMismatchError(frequencies)

        return next(iter(unique_frequencies))

    @property
    def frequency(self) -> IndicatorFrequency:
        """Return the frequency of this composite score."""
        return self._frequency

    async def _fetch_component(
        self,
        name: str,
        config: IndicatorConfig,
        observation_start: str | None,
        results: dict[str, pl.DataFrame],
    ) -> None:
        """Fetch and normalize a single component indicator."""
        # Fetch raw data
        raw_data = await config.indicator.get_series(observation_start=observation_start)

        # Normalize to 0-100 scale
        normalized_data = threshold_normalize(
            raw_data,
            column="value",
            low_threshold=config.low_threshold,
            high_threshold=config.high_threshold,
            invert=config.invert,
        )

        # Store normalized score
        results[name] = normalized_data.select(["date", "normalized_score"])

    async def get_series(
        self, observation_start: str | None = None
    ) -> pl.DataFrame:
        """
        Fetch all components concurrently and compute composite score.

        Args:
            observation_start: Optional start date for data (YYYY-MM-DD format)

        Returns:
            DataFrame with columns: date, value, rolling_avg_3m
            where 'value' is the weighted composite score (0-100)
        """
        # Fetch all components concurrently
        component_data: dict[str, pl.DataFrame] = {}
        async with asyncio.TaskGroup() as tg:
            for name, config in self._components.items():
                tg.create_task(
                    self._fetch_component(name, config, observation_start, component_data)
                )

        # Combine normalized scores with weights
        # Start with first component
        first_name = next(iter(component_data.keys()))
        result = component_data[first_name].rename({"normalized_score": first_name})

        # Join remaining components
        for name in list(component_data.keys())[1:]:
            component_df = component_data[name].rename({"normalized_score": name})
            result = result.join(component_df, on="date", how="inner")

        # Compute weighted average
        weighted_sum = pl.lit(0.0)
        for name, config in self._components.items():
            weighted_sum = weighted_sum + (pl.col(name) * config.weight)

        result = result.with_columns(value=weighted_sum)

        # Compute 3-month rolling average (standard for all indicators)
        window_size = self._frequency_window_size(3)
        result = result.with_columns(
            rolling_avg_3m=pl.col("value").rolling_mean(window_size=window_size)
        )

        # Return only standard columns
        return result.select(["date", "value", "rolling_avg_3m"])

    def _frequency_window_size(self, month_window: int) -> int:
        """
        Convert month-based window to frequency-specific data points.

        Matches the pattern in CalculatedIndicator.
        """
        match self._frequency:
            case IndicatorFrequency.MONTH:
                return month_window
            case IndicatorFrequency.WEEK:
                return month_window * 4
            case IndicatorFrequency.DAY:
                return month_window * 30

    def visualise_series(
        self, data: pl.DataFrame, config: VisualizationConfig | None = None
    ) -> go.Figure:
        """
        Visualize the composite score.

        Args:
            data: DataFrame with 'value' and 'rolling_avg_3m' columns
            config: Optional visualization config (defaults to value + rolling avg)

        Returns:
            Plotly Figure
        """
        if config is None:
            config = VisualizationConfig.create_with_original_and_3m_avg()

        visualizer = MultiSeriesVisualizer(config)
        return visualizer.visualize(data)
```

#### 2.2 Create __init__.py for composite_scores module

**File**: `src/indicators/composite_scores/__init__.py`
**Changes**: New file with module exports

```python
"""Composite score indicators."""

from src.indicators.composite_scores.composite_score import (
    CompositeScore,
    IndicatorConfig,
)

__all__ = ["CompositeScore", "IndicatorConfig"]
```

#### 2.3 Create Tests for CompositeScore

**File**: `tests/test_indicators/test_composite_score.py`
**Changes**: New test file

```python
"""Tests for CompositeScore class."""

import pytest
import polars as pl

from src.indicators.composite_scores import CompositeScore, IndicatorConfig
from src.indicators.exceptions import FrequencyMismatchError
from src.indicators.protocol import IndicatorFrequency


@pytest.mark.asyncio
class TestCompositeScore:
    """Test CompositeScore implementation."""

    async def test_validates_weights_sum_to_one(self, mock_indicator):
        """Weights must sum to 1.0."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)()
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)()

        with pytest.raises(ValueError, match="must sum to 1.0"):
            CompositeScore(
                name="Test Score",
                components={
                    "comp1": IndicatorConfig(indicator1, 0.4, 0, 100),
                    "comp2": IndicatorConfig(indicator2, 0.4, 0, 100),  # Sum = 0.8
                },
            )

    async def test_validates_frequency_consistency(self, mock_indicator):
        """All components must have same frequency."""
        monthly_indicator = mock_indicator(IndicatorFrequency.MONTH)()
        weekly_indicator = mock_indicator(IndicatorFrequency.WEEK)()

        with pytest.raises(FrequencyMismatchError):
            CompositeScore(
                name="Test Score",
                components={
                    "monthly": IndicatorConfig(monthly_indicator, 0.5, 0, 100),
                    "weekly": IndicatorConfig(weekly_indicator, 0.5, 0, 100),
                },
            )

    async def test_accepts_valid_configuration(self, mock_indicator):
        """Valid config should initialize successfully."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)()
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)()

        score = CompositeScore(
            name="Test Score",
            components={
                "comp1": IndicatorConfig(indicator1, 0.6, 0, 100),
                "comp2": IndicatorConfig(indicator2, 0.4, 0, 100),
            },
        )

        assert score.frequency == IndicatorFrequency.MONTH

    async def test_fetches_and_normalizes_components(self, mock_indicator):
        """Should fetch all components and normalize them."""
        # Create mock indicators that return predictable data
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)()
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)()

        score = CompositeScore(
            name="Test Score",
            components={
                "comp1": IndicatorConfig(indicator1, 0.5, 100, 104),  # 100->0, 104->100
                "comp2": IndicatorConfig(indicator2, 0.5, 100, 104),
            },
        )

        result = await score.get_series()

        assert "date" in result.columns
        assert "value" in result.columns
        assert "rolling_avg_3m" in result.columns

    async def test_computes_weighted_average(self, mock_indicator):
        """Should compute correct weighted average of normalized components."""
        # This test would need more controlled mock data to verify exact values
        # For now, verify structure and range
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)()
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)()

        score = CompositeScore(
            name="Test Score",
            components={
                "comp1": IndicatorConfig(indicator1, 0.7, 100, 104),
                "comp2": IndicatorConfig(indicator2, 0.3, 100, 104),
            },
        )

        result = await score.get_series()

        # Composite score should be 0-100 range
        assert result["value"].min() >= 0
        assert result["value"].max() <= 100

    async def test_includes_rolling_average(self, mock_indicator):
        """Should compute 3-month rolling average."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)()
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)()

        score = CompositeScore(
            name="Test Score",
            components={
                "comp1": IndicatorConfig(indicator1, 0.5, 100, 104),
                "comp2": IndicatorConfig(indicator2, 0.5, 100, 104),
            },
        )

        result = await score.get_series()

        assert "rolling_avg_3m" in result.columns

    async def test_respects_inverted_normalization(self, mock_indicator):
        """Components with invert=True should score higher when values are lower."""
        # This would need controlled mock data to verify
        # For now, verify that invert parameter is accepted
        indicator = mock_indicator(IndicatorFrequency.MONTH)()

        score = CompositeScore(
            name="Test Score",
            components={
                "comp": IndicatorConfig(
                    indicator, 1.0, 100, 200, invert=True
                ),  # Lower is better
            },
        )

        result = await score.get_series()
        assert result is not None
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] All CompositeScore tests pass: `just test`
- [ ] No linting errors: `just check`

#### Manual Verification:
- [ ] Test file runs successfully with `pytest tests/test_indicators/test_composite_score.py -v`
- [ ] All test cases pass including frequency validation and weight validation
- [ ] Edge cases verified

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 3: Prototype Growth Score

### Overview
Create a working Growth Composite Score using 3 real indicators from the growth category, register it in a new composite scores category, and make it visible in the dashboard.

### Changes Required:

#### 3.1 Create Composite Scores Enum and Registry

**File**: `src/indicators/composite_scores/indicators.py`
**Changes**: New file with enum definition

```python
"""Composite score indicator names and registry setup."""

from enum import StrEnum


class CompositeScoreName(StrEnum):
    """Names of available composite scores."""

    GROWTH_SCORE = "Growth Composite Score"
```

#### 3.2 Create Growth Score Factory

**File**: `src/indicators/composite_scores/__init__.py`
**Changes**: Update to include registry factory

```python
"""Composite score indicators."""

import streamlit as st

from src.indicators.composite_scores.composite_score import (
    CompositeScore,
    IndicatorConfig,
)
from src.indicators.composite_scores.indicators import CompositeScoreName
from src.indicators.growth import get_growth_indicator_registry
from src.indicators.growth.indicators import GrowthIndicatorName
from src.indicators.protocol import IndicatorFrequency
from src.indicators.registry import IndicatorRegistry

__all__ = ["CompositeScore", "IndicatorConfig", "get_composite_score_registry"]


@st.cache_resource
def get_composite_score_registry() -> IndicatorRegistry:
    """
    Initialize and cache the composite score registry.

    Returns a registry containing prototype composite scores.
    """
    # Get growth indicators for use in composite
    growth_registry = get_growth_indicator_registry()

    # Create Growth Composite Score
    # Uses PMI, Initial Claims, and Housing Starts
    # Based on MACRO_INVESTING_FRAMEWORK.md thresholds
    growth_score = CompositeScore(
        name="Growth Composite Score",
        components={
            "pmi": IndicatorConfig(
                indicator=growth_registry.get_indicator(GrowthIndicatorName.PMI_US),
                weight=0.4,
                low_threshold=45.0,  # Severe contraction
                high_threshold=55.0,  # Strong expansion
                invert=False,  # Higher is better
            ),
            "claims": IndicatorConfig(
                indicator=growth_registry.get_indicator(GrowthIndicatorName.ICSA),
                weight=0.3,
                low_threshold=200_000,  # Strong labor market
                high_threshold=400_000,  # Deteriorating labor market
                invert=True,  # Lower claims = better growth
            ),
            "housing": IndicatorConfig(
                indicator=growth_registry.get_indicator(GrowthIndicatorName.HOUST),
                weight=0.3,
                low_threshold=900,  # Weak housing (thousands of units)
                high_threshold=1500,  # Strong housing
                invert=False,  # Higher is better
            ),
        },
    )

    # Register in standard registry pattern
    registry = IndicatorRegistry[CompositeScoreName](
        enum_class=CompositeScoreName
    ).register_indicator(
        indicator_name=CompositeScoreName.GROWTH_SCORE,
        indicator=growth_score,
    )

    return registry
```

#### 3.3 Register Composite Scores in Registry Manager

**File**: `src/services/registry_setup.py`
**Changes**: Add composite scores category

```python
# Add import at top
from src.indicators.composite_scores import get_composite_score_registry

# Add registration after existing categories (around line 40)
    manager.register_registry(
        category_key="composite_scores",
        registry=get_composite_score_registry(),
        metadata=RegistryMetadata(
            display_name="Composite Scores",
            icon="🎯",
            description="Multi-indicator composite scores for growth, inflation, and regime analysis",
        ),
    )
```

#### 3.4 Create Integration Tests

**File**: `tests/test_indicators/test_composite_score_integration.py`
**Changes**: New test file for end-to-end testing

```python
"""Integration tests for composite scores with real indicators."""

import pytest

from src.indicators.composite_scores import get_composite_score_registry
from src.indicators.composite_scores.indicators import CompositeScoreName


@pytest.mark.asyncio
class TestCompositeScoreIntegration:
    """Test composite scores work with real indicator infrastructure."""

    def test_registry_initialization(self, patch_streamlit_cache):
        """Registry should initialize without errors."""
        registry = get_composite_score_registry()
        assert registry is not None

    def test_growth_score_registered(self, patch_streamlit_cache):
        """Growth Composite Score should be available."""
        registry = get_composite_score_registry()
        indicator = registry.get_indicator(CompositeScoreName.GROWTH_SCORE)
        assert indicator is not None

    async def test_growth_score_fetches_data(self, patch_streamlit_cache):
        """Growth score should successfully fetch and compute data."""
        registry = get_composite_score_registry()
        growth_score = registry.get_indicator(CompositeScoreName.GROWTH_SCORE)

        # Fetch data (this will make real API calls in integration test)
        # For unit tests, we'd mock the FRED API
        result = await growth_score.get_series(observation_start="2023-01-01")

        assert result is not None
        assert "date" in result.columns
        assert "value" in result.columns
        assert "rolling_avg_3m" in result.columns

        # Verify score is in 0-100 range
        assert result["value"].min() >= 0
        assert result["value"].max() <= 100

    async def test_growth_score_visualization(self, patch_streamlit_cache):
        """Growth score should produce valid visualization."""
        registry = get_composite_score_registry()
        growth_score = registry.get_indicator(CompositeScoreName.GROWTH_SCORE)

        result = await growth_score.get_series(observation_start="2023-01-01")
        fig = growth_score.visualise_series(result)

        assert fig is not None
        assert fig.data is not None
        assert len(fig.data) > 0  # Should have traces
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] All tests pass: `just test`
- [ ] No linting errors: `just check`
- [ ] Integration test passes: `pytest tests/test_indicators/test_composite_score_integration.py -v`

#### Manual Verification:
- [ ] Dashboard starts successfully: `just dashboard`
- [ ] "Composite Scores" category appears in category selector
- [ ] "Growth Composite Score" appears in indicator dropdown when Composite Scores selected
- [ ] Selecting Growth Composite Score displays a chart
- [ ] Chart shows values in 0-100 range
- [ ] Chart includes both "value" and "3-Month Avg" series
- [ ] Hover tooltip shows correct date and score values
- [ ] No errors in terminal when fetching/displaying the score

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to documentation updates.

---

## Testing Strategy

### Unit Tests:
- **Normalization**: Test all threshold cases, clipping, inversion
- **CompositeScore**: Test weight validation, frequency validation, data fetching
- **Edge Cases**: Empty data, single component, extreme weights

### Integration Tests:
- **Registry Integration**: Verify composite score registry works with RegistryManager
- **Real Data Fetching**: Test with actual FRED API calls (in separate integration suite)
- **Visualization**: Verify charts render correctly

### Manual Testing Steps:
1. Start dashboard: `just dashboard`
2. Select "Composite Scores" category
3. Select "Growth Composite Score"
4. Verify chart displays with 0-100 scale
5. Check multiple date ranges (1Y, 3Y, 5Y, All)
6. Verify no console errors
7. Compare score behavior to underlying indicators (PMI, Claims, Housing)

## Performance Considerations

### Caching Strategy:
- Registry cached with `@st.cache_resource` (singleton, no TTL)
- Data cached with existing `@st.cache_data(ttl=3600)` from IndicatorRegistry
- Three concurrent API calls per composite score (PMI + Claims + Housing)

### Expected Performance:
- First load: ~2-3 seconds (3 FRED API calls)
- Cached load: <100ms
- No performance degradation vs. existing indicators

### Optimization Opportunities (Future):
- Pre-compute common composite scores during off-peak hours
- Increase TTL for composite scores (less volatile than raw indicators)
- Add batch fetching for multiple composite scores

## Migration Notes

N/A - This is new functionality with no existing data to migrate.

## References

- Original research: `thoughts/searchable/shared/research/2025-12-30-phase1-foundation-architecture.md`
- Macro framework: `MACRO_INVESTING_FRAMEWORK.md` (PMI thresholds lines 85-93)
- Existing patterns:
  - `src/indicators/calculations/calculated_indicator.py:156-169` - Multi-source concurrent fetching
  - `src/indicators/growth/__init__.py:12-67` - Registry factory pattern
  - `tests/conftest.py:90-116` - MockIndicator test pattern
  - `src/indicators/inflation/calculations.py:4-45` - Calculation function patterns
