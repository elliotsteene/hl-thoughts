# Phase 1: Composite Scoring Foundation Implementation Plan

## Overview

This plan implements the foundational infrastructure for composite scoring in the cc-analyst macroeconomic dashboard. Phase 1 creates the core abstractions (normalization utilities, CompositeScore class) and validates them with a prototype Growth Score. This establishes patterns for all future composite scores and regime detection capabilities.

**Based on**: `thoughts/searchable/shared/research/2025-12-30-phase1-foundation-architecture.md`

## Current State Analysis

The research document confirms the codebase has excellent foundations:

### Existing Strengths
- **Protocol-based design** (`src/indicators/protocol.py:16-27`): `Indicator` protocol with structural typing allows any class implementing `frequency`, `get_series()`, and `visualise_series()` to work as an indicator
- **CalculatedIndicator** (`src/indicators/calculations/calculated_indicator.py`): Supports both single-source and multi-source calculations with concurrent async fetching via `asyncio.TaskGroup`
- **Generic registry system** (`src/indicators/registry.py`): Type-safe `IndicatorRegistry[T]` works with any enum type
- **Comprehensive test infrastructure**: pytest with async support, mock fixtures, and established patterns in `tests/conftest.py`
- **Polars-first data flow**: Consistent schema (`date`, `value`, `rolling_avg_3m`)

### Architectural Gaps (What We're Building)
- **No score normalization utilities**: Need functions to convert raw indicator values to 0-100 scores
- **No CompositeScore abstraction**: Need a class that combines multiple indicators into a single score
- **No composite score examples**: Need a working prototype to validate the pattern

### Key Discoveries
- `CalculatedIndicator` already has multi-source infrastructure (lines 156-169) but it's currently unused - we can model `CompositeScore` on this pattern
- Existing inflation calculations (`src/indicators/inflation/calculations.py`) demonstrate clean calculation function patterns we should follow
- Test patterns in `tests/test_calculations/test_inflation_calculations.py` provide excellent templates for testing our normalization functions
- Registry pattern (`src/indicators/inflation/__init__.py:17-95`) shows how to mix FRED and calculated indicators - composite scores will follow the same approach

## Desired End State

After this plan is complete:

1. **Normalization module exists** at `src/indicators/calculations/normalization.py` with threshold-based normalization utilities
2. **CompositeScore class exists** at `src/indicators/composite_scores/composite_score.py` implementing the `Indicator` protocol
3. **Prototype Growth Score exists** demonstrating the end-to-end pattern
4. **Comprehensive test coverage** for all new components following established patterns
5. **All existing tests still pass** - no regressions introduced
6. **Documentation updated** in CLAUDE.md with composite score patterns

### Verification Criteria
- `just check` passes (linting + type checking)
- `just tests` passes (all existing + new tests)
- Dashboard runs without errors
- Prototype Growth Score visible in dashboard
- Pattern documented for future score implementations

## What We're NOT Doing

To prevent scope creep, Phase 1 explicitly excludes:

- **Advanced normalization methods**: No z-score or percentile rank normalization (only threshold-based)
- **Regime detection logic**: That's Phase 8 - we're only building composite scores
- **Multiple composite scores**: Only one prototype Growth Score to validate the pattern
- **Score metadata infrastructure**: No special score registry or metadata beyond what `IndicatorRegistry` provides
- **Sparklines or score trends**: UI enhancements come in later phases
- **Persistent storage**: Leveraging existing Streamlit cache is sufficient
- **Category-specific score implementations**: Growth/Inflation/Financial scores come in Phases 2-6

## Implementation Approach

### Strategy
We'll extend the existing architecture rather than refactor it. The `Indicator` protocol's structural typing means `CompositeScore` can integrate seamlessly into the existing registry system. We'll follow the established patterns:

1. **Build utilities first** (normalization functions) - these are pure functions easy to test
2. **Build CompositeScore class** - implements `Indicator` protocol, uses utilities
3. **Create prototype** - validates the entire pattern works end-to-end
4. **Document patterns** - ensures future implementations follow the same approach

### Technical Decisions
- **Unified registry approach**: Composite scores register as indicators (no separate `ScoreRegistry`)
- **Threshold-based normalization**: Simplest approach, sufficient for MVP, extensible later
- **0-100 score scale**: Industry standard, intuitive interpretation
- **Equal weighting for prototype**: Simplifies initial implementation, weights configurable per score
- **Leverage existing async patterns**: Use `asyncio.TaskGroup` like `CalculatedIndicator` does

---

## Phase 1.1: Normalization Utilities

### Overview
Create the normalization module with threshold-based functions to convert raw indicator values to 0-100 scores. These utilities will be used by all composite scores.

### Changes Required

#### 1.1.1 Create Normalization Module

**File**: `src/indicators/calculations/normalization.py`
**Changes**: Create new file with threshold-based normalization functions

```python
"""
Score normalization utilities for converting raw indicator values to 0-100 scale.

Normalization functions take Polars DataFrames with indicator data and return
DataFrames with normalized score columns.
"""

import polars as pl


def threshold_normalize(
    data: pl.DataFrame,
    column: str = "value",
    thresholds: dict[str, float] | None = None,
    reverse: bool = False,
) -> pl.DataFrame:
    """
    Normalize indicator values to 0-100 scale using threshold mapping.

    Maps indicator values to scores based on threshold breakpoints:
    - Values below 'low' threshold → scores 0-40
    - Values between 'low' and 'neutral' → scores 40-50
    - Values between 'neutral' and 'high' → scores 50-60
    - Values above 'high' threshold → scores 60-100

    Args:
        data: DataFrame with time series data
        column: Name of column to normalize (default: "value")
        thresholds: Dict with 'low', 'neutral', 'high' threshold values
                   If None, uses percentiles: 25th, 50th, 75th
        reverse: If True, higher indicator values produce lower scores
                (useful for unemployment, claims, etc.)

    Returns:
        DataFrame with additional 'score' column (0-100)

    Example:
        >>> data = pl.DataFrame({"date": [...], "value": [45, 50, 55, 60]})
        >>> thresholds = {"low": 48, "neutral": 52, "high": 58}
        >>> result = threshold_normalize(data, thresholds=thresholds)
        >>> # result has 'score' column with normalized values
    """
    # Calculate thresholds from data if not provided
    if thresholds is None:
        thresholds = {
            "low": data[column].quantile(0.25),
            "neutral": data[column].quantile(0.50),
            "high": data[column].quantile(0.75),
        }

    low = thresholds["low"]
    neutral = thresholds["neutral"]
    high = thresholds["high"]

    # Create score using conditional logic
    # Below low: linear scale from 0-40
    # Low to neutral: linear scale from 40-50
    # Neutral to high: linear scale from 50-60
    # Above high: linear scale from 60-100

    score_expr = (
        pl.when(pl.col(column) <= low)
        .then(pl.col(column) / low * 40)
        .when(pl.col(column) <= neutral)
        .then(40 + (pl.col(column) - low) / (neutral - low) * 10)
        .when(pl.col(column) <= high)
        .then(50 + (pl.col(column) - neutral) / (high - neutral) * 10)
        .otherwise(60 + pl.min_horizontal((pl.col(column) - high) / (high - neutral) * 40, 40))
    )

    # Reverse if needed (for inverse indicators like unemployment)
    if reverse:
        score_expr = 100 - score_expr

    # Clip to 0-100 range
    score_expr = pl.max_horizontal(0, pl.min_horizontal(100, score_expr))

    return data.with_columns(score=score_expr)


def weighted_average_scores(
    scores: dict[str, pl.DataFrame],
    weights: dict[str, float],
    score_column: str = "score",
) -> pl.DataFrame:
    """
    Compute weighted average of multiple score DataFrames.

    All DataFrames must have the same date index and be pre-aligned.
    Weights should sum to 1.0 (will be normalized if they don't).

    Args:
        scores: Dict mapping indicator names to their score DataFrames
        weights: Dict mapping indicator names to their weights
        score_column: Name of column containing scores (default: "score")

    Returns:
        DataFrame with 'date' and 'value' columns containing weighted average

    Example:
        >>> scores = {
        ...     "pmi": df_pmi_scores,
        ...     "claims": df_claims_scores,
        ... }
        >>> weights = {"pmi": 0.6, "claims": 0.4}
        >>> result = weighted_average_scores(scores, weights)
    """
    # Normalize weights to sum to 1.0
    total_weight = sum(weights.values())
    normalized_weights = {k: v / total_weight for k, v in weights.items()}

    # Start with the first score DataFrame (for date alignment)
    first_name = next(iter(scores.keys()))
    result = scores[first_name].select(["date"])

    # Add weighted scores
    weighted_sum_expr = pl.lit(0.0)
    for name, score_df in scores.items():
        weight = normalized_weights[name]
        # Join each score DataFrame and multiply by weight
        result = result.join(
            score_df.select(["date", pl.col(score_column).alias(f"{name}_score")]),
            on="date",
            how="left",
        )
        weighted_sum_expr = weighted_sum_expr + pl.col(f"{name}_score") * weight

    # Compute final weighted average
    result = result.with_columns(value=weighted_sum_expr)

    # Return only date and value (drop intermediate score columns)
    score_columns = [f"{name}_score" for name in scores.keys()]
    return result.drop(score_columns)
```

#### 1.1.2 Create Normalization Tests

**File**: `tests/test_calculations/test_normalization.py`
**Changes**: Create comprehensive tests for normalization functions

```python
"""Tests for score normalization utilities."""

import polars as pl
import pytest

from src.indicators.calculations.normalization import (
    threshold_normalize,
    weighted_average_scores,
)


class TestThresholdNormalize:
    """Tests for threshold-based normalization."""

    def test_normalizes_values_to_0_100_scale(self):
        """Values should be mapped to 0-100 scale."""
        data = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01", "2024-03-01", "2024-04-01"],
            "value": [40.0, 50.0, 60.0, 70.0],
        })

        thresholds = {"low": 45.0, "neutral": 55.0, "high": 65.0}
        result = threshold_normalize(data, thresholds=thresholds)

        assert "score" in result.columns
        assert result["score"].min() >= 0
        assert result["score"].max() <= 100

    def test_uses_percentiles_when_no_thresholds_provided(self):
        """Should compute thresholds from data percentiles."""
        data = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01", "2024-03-01", "2024-04-01"],
            "value": [10.0, 20.0, 30.0, 40.0],
        })

        result = threshold_normalize(data)

        assert "score" in result.columns
        assert result["score"].is_not_null().all()

    def test_reverse_inverts_scores(self):
        """Reverse=True should invert scores (higher values = lower scores)."""
        data = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01"],
            "value": [40.0, 60.0],
        })

        thresholds = {"low": 45.0, "neutral": 50.0, "high": 55.0}

        normal = threshold_normalize(data, thresholds=thresholds, reverse=False)
        reversed_result = threshold_normalize(data, thresholds=thresholds, reverse=True)

        # Higher value should have higher score in normal mode
        assert normal["score"][1] > normal["score"][0]
        # Higher value should have lower score in reverse mode
        assert reversed_result["score"][1] < reversed_result["score"][0]

    def test_clips_scores_to_0_100_range(self):
        """Extreme values should be clipped to 0-100."""
        data = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01", "2024-03-01"],
            "value": [0.0, 50.0, 1000.0],
        })

        thresholds = {"low": 40.0, "neutral": 50.0, "high": 60.0}
        result = threshold_normalize(data, thresholds=thresholds)

        assert result["score"].min() >= 0
        assert result["score"].max() <= 100

    def test_neutral_value_maps_to_50(self):
        """Neutral threshold value should map to ~50 score."""
        data = pl.DataFrame({
            "date": ["2024-01-01"],
            "value": [50.0],
        })

        thresholds = {"low": 40.0, "neutral": 50.0, "high": 60.0}
        result = threshold_normalize(data, thresholds=thresholds)

        assert abs(result["score"][0] - 50.0) < 1.0


class TestWeightedAverageScores:
    """Tests for weighted average score computation."""

    def test_computes_weighted_average_of_scores(self):
        """Should correctly compute weighted average."""
        score1 = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01"],
            "score": [60.0, 70.0],
        })
        score2 = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01"],
            "score": [40.0, 50.0],
        })

        scores = {"pmi": score1, "claims": score2}
        weights = {"pmi": 0.6, "claims": 0.4}

        result = weighted_average_scores(scores, weights)

        # 0.6 * 60 + 0.4 * 40 = 36 + 16 = 52
        assert abs(result["value"][0] - 52.0) < 0.01
        # 0.6 * 70 + 0.4 * 50 = 42 + 20 = 62
        assert abs(result["value"][1] - 62.0) < 0.01

    def test_normalizes_weights_to_sum_to_one(self):
        """Weights should be normalized if they don't sum to 1."""
        score1 = pl.DataFrame({
            "date": ["2024-01-01"],
            "score": [60.0],
        })
        score2 = pl.DataFrame({
            "date": ["2024-01-01"],
            "score": [40.0],
        })

        scores = {"pmi": score1, "claims": score2}
        weights = {"pmi": 3.0, "claims": 2.0}  # Sum to 5, should normalize to 0.6/0.4

        result = weighted_average_scores(scores, weights)

        # Should be same as 0.6 * 60 + 0.4 * 40 = 52
        assert abs(result["value"][0] - 52.0) < 0.01

    def test_returns_date_and_value_columns_only(self):
        """Result should only have date and value columns."""
        score1 = pl.DataFrame({
            "date": ["2024-01-01"],
            "score": [60.0],
        })
        score2 = pl.DataFrame({
            "date": ["2024-01-01"],
            "score": [40.0],
        })

        scores = {"pmi": score1, "claims": score2}
        weights = {"pmi": 0.6, "claims": 0.4}

        result = weighted_average_scores(scores, weights)

        assert set(result.columns) == {"date", "value"}

    def test_handles_multiple_scores(self):
        """Should handle more than 2 scores."""
        score1 = pl.DataFrame({"date": ["2024-01-01"], "score": [60.0]})
        score2 = pl.DataFrame({"date": ["2024-01-01"], "score": [40.0]})
        score3 = pl.DataFrame({"date": ["2024-01-01"], "score": [80.0]})

        scores = {"pmi": score1, "claims": score2, "housing": score3}
        weights = {"pmi": 0.4, "claims": 0.3, "housing": 0.3}

        result = weighted_average_scores(scores, weights)

        # 0.4 * 60 + 0.3 * 40 + 0.3 * 80 = 24 + 12 + 24 = 60
        assert abs(result["value"][0] - 60.0) < 0.01
```

### Success Criteria

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] All normalization tests pass: `just tests`
- [ ] No linting errors: `just check`
- [ ] All existing tests still pass (no regressions)

#### Manual Verification:
- [ ] Threshold normalization produces scores in 0-100 range
- [ ] Reverse parameter correctly inverts scores
- [ ] Weighted average correctly combines multiple scores
- [ ] Edge cases (extreme values, nulls) handled gracefully

---

## Phase 1.2: CompositeScore Class

### Overview
Create the `CompositeScore` class that implements the `Indicator` protocol. This class will combine multiple indicators into a single score using normalization and weighted averaging.

### Changes Required

#### 1.2.1 Create CompositeScore Directory

**File**: `src/indicators/composite_scores/__init__.py`
**Changes**: Create new directory and init file

```python
"""Composite score indicators combining multiple data sources."""

from src.indicators.composite_scores.composite_score import CompositeScore

__all__ = ["CompositeScore"]
```

#### 1.2.2 Implement CompositeScore Class

**File**: `src/indicators/composite_scores/composite_score.py`
**Changes**: Create new file with CompositeScore implementation

```python
"""
CompositeScore: Combines multiple indicators into a single 0-100 score.

CompositeScore implements the Indicator protocol and can be registered in
the standard IndicatorRegistry system. It fetches multiple source indicators
concurrently, normalizes each to 0-100 scale, and computes a weighted average.
"""

import asyncio
from typing import cast

import plotly.graph_objs as go
import polars as pl

from src.indicators.calculations.normalization import (
    threshold_normalize,
    weighted_average_scores,
)
from src.indicators.exceptions import FrequencyMismatchError
from src.indicators.multi_series_visualizer import MultiSeriesVisualizer
from src.indicators.protocol import Indicator, IndicatorFrequency
from src.indicators.visualization_config import VisualizationConfig


class CompositeScore:
    """
    A composite indicator that combines multiple source indicators into a single score.

    Each source indicator is:
    1. Fetched concurrently (async)
    2. Normalized to 0-100 scale using threshold normalization
    3. Combined using weighted average

    The result is a single time-series score (0-100) that can be visualized
    and interpreted like any other indicator.

    Example:
        >>> growth_score = CompositeScore(
        ...     name="Growth Score",
        ...     source_indicators={
        ...         "pmi": pmi_indicator,
        ...         "claims": claims_indicator,
        ...         "housing": housing_indicator,
        ...     },
        ...     weights={"pmi": 0.4, "claims": 0.3, "housing": 0.3},
        ...     thresholds={
        ...         "pmi": {"low": 48, "neutral": 50, "high": 52},
        ...         "claims": {"low": 220000, "neutral": 250000, "high": 280000},
        ...         "housing": {"low": 1.2, "neutral": 1.4, "high": 1.6},
        ...     },
        ...     reverse={"claims": True},  # Higher claims = worse growth
        ... )
    """

    def __init__(
        self,
        name: str,
        source_indicators: dict[str, Indicator],
        weights: dict[str, float],
        thresholds: dict[str, dict[str, float]] | None = None,
        reverse: dict[str, bool] | None = None,
    ) -> None:
        """
        Initialize composite score.

        Args:
            name: Display name for this score
            source_indicators: Dict mapping indicator names to Indicator instances
            weights: Dict mapping indicator names to weights (will be normalized)
            thresholds: Optional dict mapping indicator names to threshold dicts
                       If None, thresholds computed from data percentiles
            reverse: Optional dict mapping indicator names to reverse flags
                    Use True for inverse indicators (higher value = worse)
        """
        self._name = name
        self._source_indicators = source_indicators
        self._weights = weights
        self._thresholds = thresholds or {}
        self._reverse = reverse or {}

        # Validate all source indicator names have weights
        if set(source_indicators.keys()) != set(weights.keys()):
            raise ValueError(
                f"Source indicators and weights must have same keys. "
                f"Sources: {set(source_indicators.keys())}, "
                f"Weights: {set(weights.keys())}"
            )

    @property
    def frequency(self) -> IndicatorFrequency:
        """
        Derive frequency from source indicators.

        All sources must have the same frequency.

        Raises:
            FrequencyMismatchError: If source indicators have different frequencies.
        """
        frequencies: dict[str, IndicatorFrequency] = {
            name: indicator.frequency
            for name, indicator in self._source_indicators.items()
        }

        unique_frequencies = set(frequencies.values())
        if len(unique_frequencies) > 1:
            raise FrequencyMismatchError(frequencies)

        return next(iter(unique_frequencies))

    async def _fetch_and_normalize_indicator(
        self,
        name: str,
        indicator: Indicator,
        observation_start: str | None,
        results: dict[str, pl.DataFrame],
    ) -> None:
        """
        Fetch a single indicator and normalize it to 0-100 score.

        Results are stored in the results dict for concurrent collection.
        """
        # Fetch indicator data
        data = await indicator.get_series(observation_start=observation_start)

        # Normalize to score
        thresholds = self._thresholds.get(name)
        reverse = self._reverse.get(name, False)

        normalized = threshold_normalize(
            data=data,
            column="value",
            thresholds=thresholds,
            reverse=reverse,
        )

        results[name] = normalized

    async def get_series(
        self,
        observation_start: str | None = None,
    ) -> pl.DataFrame:
        """
        Fetch all source indicators, normalize them, and compute weighted average.

        Returns:
            DataFrame with 'date' and 'value' columns (value is 0-100 score)
        """
        # Fetch and normalize all indicators concurrently
        normalized_scores: dict[str, pl.DataFrame] = {}
        async with asyncio.TaskGroup() as tg:
            for name, indicator in self._source_indicators.items():
                tg.create_task(
                    self._fetch_and_normalize_indicator(
                        name, indicator, observation_start, normalized_scores
                    )
                )

        # Compute weighted average
        result = weighted_average_scores(
            scores=normalized_scores,
            weights=self._weights,
            score_column="score",
        )

        # Add 3-month rolling average
        result = result.with_columns(
            rolling_avg_3m=pl.col("value").rolling_mean(window_size=3)
        )

        return result

    def visualise_series(
        self,
        data: pl.DataFrame,
        config: VisualizationConfig | None = None,
    ) -> go.Figure:
        """
        Visualize the composite score.

        Default visualization shows both the score value and 3-month rolling average.
        """
        if config is None:
            config = VisualizationConfig.create_with_original_and_3m_avg()

        visualizer = MultiSeriesVisualizer(config)
        return visualizer.visualize(data)
```

#### 1.2.3 Create CompositeScore Tests

**File**: `tests/test_indicators/test_composite_score.py`
**Changes**: Create comprehensive tests for CompositeScore class

```python
"""Tests for CompositeScore class."""

import pytest
import polars as pl

from src.indicators.composite_scores.composite_score import CompositeScore
from src.indicators.protocol import IndicatorFrequency
from src.indicators.exceptions import FrequencyMismatchError


@pytest.mark.asyncio
class TestCompositeScore:
    """Tests for CompositeScore indicator."""

    async def test_implements_indicator_protocol(self, mock_indicator):
        """CompositeScore should implement Indicator protocol."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)

        score = CompositeScore(
            name="Test Score",
            source_indicators={"ind1": indicator1, "ind2": indicator2},
            weights={"ind1": 0.5, "ind2": 0.5},
        )

        # Should have required protocol methods
        assert hasattr(score, "frequency")
        assert hasattr(score, "get_series")
        assert hasattr(score, "visualise_series")

    async def test_fetches_all_source_indicators_concurrently(self, mock_indicator):
        """Should fetch all source indicators."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)

        score = CompositeScore(
            name="Test Score",
            source_indicators={"ind1": indicator1, "ind2": indicator2},
            weights={"ind1": 0.6, "ind2": 0.4},
        )

        result = await score.get_series()

        assert result is not None
        assert "date" in result.columns
        assert "value" in result.columns

    async def test_returns_0_100_score(self, mock_indicator):
        """Result values should be in 0-100 range."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)

        score = CompositeScore(
            name="Test Score",
            source_indicators={"ind1": indicator1, "ind2": indicator2},
            weights={"ind1": 0.5, "ind2": 0.5},
        )

        result = await score.get_series()

        assert result["value"].min() >= 0
        assert result["value"].max() <= 100

    async def test_computes_weighted_average_of_normalized_scores(self, mock_indicator):
        """Should apply weights correctly."""
        # This is more of an integration test - verifies the full pipeline
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)
        indicator2 = mock_indicator(IndicatorFrequency.MONTH)

        score = CompositeScore(
            name="Test Score",
            source_indicators={"ind1": indicator1, "ind2": indicator2},
            weights={"ind1": 0.8, "ind2": 0.2},
        )

        result = await score.get_series()

        # Should have computed some value (weighted average of normalized scores)
        assert result["value"].is_not_null().any()

    async def test_raises_error_if_frequencies_mismatch(self, mock_indicator):
        """Should raise FrequencyMismatchError if sources have different frequencies."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)
        indicator2 = mock_indicator(IndicatorFrequency.DAY)

        score = CompositeScore(
            name="Test Score",
            source_indicators={"ind1": indicator1, "ind2": indicator2},
            weights={"ind1": 0.5, "ind2": 0.5},
        )

        with pytest.raises(FrequencyMismatchError):
            _ = score.frequency

    async def test_raises_error_if_weights_dont_match_sources(self, mock_indicator):
        """Should raise ValueError if weight keys don't match source keys."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)

        with pytest.raises(ValueError):
            CompositeScore(
                name="Test Score",
                source_indicators={"ind1": indicator1},
                weights={"wrong_key": 0.5},
            )

    async def test_applies_reverse_flag_correctly(self, mock_indicator):
        """Reverse flag should invert scores for specified indicators."""
        # Create custom mock that returns consistent data
        class CustomMockIndicator:
            frequency = IndicatorFrequency.MONTH

            async def get_series(self, observation_start=None):
                return pl.DataFrame({
                    "date": ["2024-01-01", "2024-02-01"],
                    "value": [100.0, 200.0],  # Higher value in second row
                })

            def visualise_series(self, data, config=None):
                from src.indicators.visualization_config import VisualizationConfig
                from src.indicators.multi_series_visualizer import MultiSeriesVisualizer

                if config is None:
                    config = VisualizationConfig().add_series("value", "Value")
                visualizer = MultiSeriesVisualizer(config)
                return visualizer.visualize(data)

        indicator1 = CustomMockIndicator()

        # Without reverse
        score_normal = CompositeScore(
            name="Normal Score",
            source_indicators={"ind1": indicator1},
            weights={"ind1": 1.0},
            reverse={},
        )

        # With reverse
        score_reversed = CompositeScore(
            name="Reversed Score",
            source_indicators={"ind1": indicator1},
            weights={"ind1": 1.0},
            reverse={"ind1": True},
        )

        result_normal = await score_normal.get_series()
        result_reversed = await score_reversed.get_series()

        # Normal: higher value should give higher score
        # Reversed: higher value should give lower score
        # So reversed scores should be inverted compared to normal
        assert result_normal["value"][1] > result_normal["value"][0]
        assert result_reversed["value"][1] < result_reversed["value"][0]

    async def test_includes_rolling_avg_3m_column(self, mock_indicator):
        """Result should include 3-month rolling average."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)

        score = CompositeScore(
            name="Test Score",
            source_indicators={"ind1": indicator1},
            weights={"ind1": 1.0},
        )

        result = await score.get_series()

        assert "rolling_avg_3m" in result.columns

    async def test_visualise_series_returns_figure(self, mock_indicator):
        """visualise_series should return Plotly figure."""
        indicator1 = mock_indicator(IndicatorFrequency.MONTH)

        score = CompositeScore(
            name="Test Score",
            source_indicators={"ind1": indicator1},
            weights={"ind1": 1.0},
        )

        data = await score.get_series()
        fig = score.visualise_series(data)

        assert fig is not None
        assert hasattr(fig, "data")  # Plotly figure attribute
```

### Success Criteria

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] All CompositeScore tests pass: `just tests`
- [ ] No linting errors: `just check`
- [ ] All existing tests still pass (no regressions)

#### Manual Verification:
- [ ] CompositeScore correctly implements Indicator protocol
- [ ] Concurrent fetching works as expected
- [ ] Weighted averaging produces sensible results
- [ ] Frequency validation prevents mismatched sources

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the manual testing was successful before proceeding to the next phase.

---

## Phase 1.3: Prototype Growth Score

### Overview
Create a working prototype Growth Score that combines real growth indicators. This validates the entire pattern works end-to-end and provides a template for future composite scores.

### Changes Required

#### 1.3.1 Create Composite Scores Registry Module

**File**: `src/indicators/composite_scores/indicators.py`
**Changes**: Create enum and factory for composite scores

```python
"""Composite score definitions and registry setup."""

from enum import StrEnum


class CompositeScoreName(StrEnum):
    """Enum of available composite scores."""

    GROWTH_SCORE = "Growth Score (Composite)"
```

#### 1.3.2 Create Prototype Growth Score Factory

**File**: `src/indicators/composite_scores/scores.py`
**Changes**: Create factory function that builds the Growth Score

```python
"""Factory functions for creating composite score instances."""

import streamlit as st

from src.indicators.composite_scores.composite_score import CompositeScore
from src.indicators.composite_scores.indicators import CompositeScoreName
from src.indicators.fred_indicator import FredIndicator
from src.indicators.growth.indicators import GROWTH_INDICATOR_NAME_CODE_MAPPING, GrowthIndicatorName
from src.indicators.protocol import IndicatorFrequency
from src.indicators.registry import IndicatorRegistry


@st.cache_resource
def get_composite_score_registry() -> IndicatorRegistry:
    """
    Create and configure the composite scores registry.

    Returns a registry with all composite score implementations.
    Currently includes:
    - Growth Score: Combines PMI, Initial Claims, and Housing Starts
    """
    # Create source indicators for Growth Score
    pmi_indicator = FredIndicator(
        fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[GrowthIndicatorName.PMI_US],
        frequency=IndicatorFrequency.MONTH,
    )
    claims_indicator = FredIndicator(
        fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[GrowthIndicatorName.ICSA],
        frequency=IndicatorFrequency.WEEK,
    )
    housing_indicator = FredIndicator(
        fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[GrowthIndicatorName.HOUST],
        frequency=IndicatorFrequency.MONTH,
    )

    # Create Growth Score
    # NOTE: Using monthly frequency indicators only for initial prototype
    # Will need to address frequency alignment in future iterations
    growth_score = CompositeScore(
        name="Growth Score",
        source_indicators={
            "pmi": pmi_indicator,
            "housing": housing_indicator,
        },
        weights={
            "pmi": 0.6,  # PMI weighted higher as leading indicator
            "housing": 0.4,
        },
        thresholds={
            "pmi": {
                "low": 48.0,  # Below 50 = contraction
                "neutral": 50.0,  # Expansion/contraction threshold
                "high": 52.0,  # Strong expansion
            },
            "housing": {
                "low": 1.2,  # Million units - weak
                "neutral": 1.4,  # Million units - moderate
                "high": 1.6,  # Million units - strong
            },
        },
        reverse={},  # Both are positive indicators (higher = better)
    )

    # Register in standard IndicatorRegistry
    return IndicatorRegistry[CompositeScoreName](
        enum_class=CompositeScoreName
    ).register_indicator(
        indicator_name=CompositeScoreName.GROWTH_SCORE,
        indicator=growth_score,
    )
```

#### 1.3.3 Register Composite Scores in Registry Manager

**File**: `src/services/registry_setup.py`
**Changes**: Add composite scores to registry manager

```python
# Add after existing imports
from src.indicators.composite_scores.scores import get_composite_score_registry

# Add after existing registry registrations (around line 40)
    manager.register_registry(
        category_key="composite_scores",
        registry=get_composite_score_registry(),
        metadata=RegistryMetadata(
            display_name="Composite Scores",
            icon="⭐",
            description="Multi-indicator composite scores for macro regime analysis",
        ),
    )
```

#### 1.3.4 Create Integration Test for Prototype

**File**: `tests/test_indicators/test_growth_score_integration.py`
**Changes**: Create integration test for Growth Score

```python
"""Integration test for Growth Score prototype."""

import pytest

from src.indicators.composite_scores.scores import get_composite_score_registry
from src.indicators.composite_scores.indicators import CompositeScoreName


@pytest.mark.asyncio
class TestGrowthScoreIntegration:
    """Integration tests for the Growth Score prototype."""

    def test_growth_score_registered(self, patch_streamlit_cache):
        """Growth Score should be registered in composite score registry."""
        registry = get_composite_score_registry()

        # Should be able to get the indicator
        growth_score = registry.get_indicator(CompositeScoreName.GROWTH_SCORE)
        assert growth_score is not None

    async def test_growth_score_fetches_data(self, patch_streamlit_cache, mock_fred_api):
        """Growth Score should successfully fetch and compute data."""
        import polars as pl

        # Mock FRED API to return sample data
        mock_fred_api.return_value = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01", "2024-03-01"],
            "value": [50.0, 51.0, 52.0],
        })

        registry = get_composite_score_registry()
        growth_score = registry.get_indicator(CompositeScoreName.GROWTH_SCORE)

        # Should be able to fetch series
        result = await growth_score.get_series(observation_start="2024-01-01")

        assert result is not None
        assert "date" in result.columns
        assert "value" in result.columns
        assert "rolling_avg_3m" in result.columns

    async def test_growth_score_returns_valid_scores(
        self, patch_streamlit_cache, mock_fred_api
    ):
        """Growth Score values should be in 0-100 range."""
        import polars as pl

        mock_fred_api.return_value = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01", "2024-03-01"],
            "value": [50.0, 51.0, 52.0],
        })

        registry = get_composite_score_registry()
        growth_score = registry.get_indicator(CompositeScoreName.GROWTH_SCORE)

        result = await growth_score.get_series(observation_start="2024-01-01")

        # Scores should be in 0-100 range
        assert result["value"].min() >= 0
        assert result["value"].max() <= 100

    def test_growth_score_visualizes(self, patch_streamlit_cache, mock_fred_api):
        """Growth Score should be visualizable."""
        import polars as pl

        registry = get_composite_score_registry()
        growth_score = registry.get_indicator(CompositeScoreName.GROWTH_SCORE)

        sample_data = pl.DataFrame({
            "date": ["2024-01-01", "2024-02-01", "2024-03-01"],
            "value": [50.0, 55.0, 60.0],
            "rolling_avg_3m": [50.0, 52.5, 55.0],
        })

        fig = growth_score.visualise_series(sample_data)

        assert fig is not None
        assert hasattr(fig, "data")
```

### Success Criteria

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] All tests pass including integration tests: `just tests`
- [ ] No linting errors: `just check`
- [ ] Dashboard starts without errors: `just dashboard`

#### Manual Verification:
- [ ] "Composite Scores" category appears in dashboard sidebar
- [ ] Growth Score can be selected and displays data
- [ ] Chart renders correctly with score values
- [ ] Score values are in reasonable 0-100 range
- [ ] No errors in Streamlit console
- [ ] Existing indicators still work (no regressions)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the dashboard displays the Growth Score correctly before proceeding to documentation.

---

## Phase 1.4: Documentation Updates

### Overview
Update CLAUDE.md with composite score patterns and Phase 1 architectural decisions to guide future implementations.

### Changes Required

#### 1.4.1 Update CLAUDE.md with Composite Score Patterns

**File**: `CLAUDE.md`
**Changes**: Add new section after "Adding New Indicators" section

```markdown
### Adding Composite Scores

Composite scores combine multiple indicators into a single 0-100 score. They implement the `Indicator` protocol and register in the standard registry system.

**Pattern established in Phase 1** (see `thoughts/searchable/shared/research/2025-12-30-phase1-foundation-architecture.md`):

1. **Create score instance** using `CompositeScore` class:
   ```python
   from src.indicators.composite_scores.composite_score import CompositeScore

   score = CompositeScore(
       name="My Score",
       source_indicators={
           "indicator1": indicator1_instance,
           "indicator2": indicator2_instance,
       },
       weights={"indicator1": 0.6, "indicator2": 0.4},
       thresholds={  # Optional - uses percentiles if not provided
           "indicator1": {"low": 45, "neutral": 50, "high": 55},
           "indicator2": {"low": 1.0, "neutral": 1.5, "high": 2.0},
       },
       reverse={"indicator2": True},  # Optional - for inverse indicators
   )
   ```

2. **Register in a registry** (typically in `src/indicators/composite_scores/scores.py`):
   ```python
   registry.register_indicator(
       indicator_name=CompositeScoreName.MY_SCORE,
       indicator=score,
   )
   ```

3. **Key concepts**:
   - **Threshold normalization**: Maps raw values to 0-100 scale based on low/neutral/high thresholds
   - **Reverse flag**: Use `True` for inverse indicators (e.g., unemployment - higher is worse)
   - **Weights**: Automatically normalized to sum to 1.0
   - **Concurrent fetching**: All source indicators fetched in parallel using `asyncio.TaskGroup`
   - **Frequency validation**: All sources must have same frequency (monthly, weekly, or daily)

4. **Testing composite scores**:
   - Follow patterns in `tests/test_indicators/test_composite_score.py`
   - Test normalization, weighting, frequency validation
   - Use mock indicators from `tests/conftest.py`

**Example**: See Growth Score prototype in `src/indicators/composite_scores/scores.py`
```

### Success Criteria

#### Automated Verification:
- [ ] CLAUDE.md file updated with composite score patterns
- [ ] Markdown formatting is valid (check with `just check` or preview)

#### Manual Verification:
- [ ] Documentation is clear and accurate
- [ ] Examples are complete and runnable
- [ ] Links to research document are correct
- [ ] Patterns match actual implementation

---

## Testing Strategy

### Unit Tests

**Normalization utilities** (`tests/test_calculations/test_normalization.py`):
- Threshold normalization edge cases (extreme values, nulls, clips)
- Reverse flag behavior
- Weighted average computation
- Weight normalization

**CompositeScore class** (`tests/test_indicators/test_composite_score.py`):
- Protocol implementation
- Concurrent fetching
- Frequency validation
- Normalization application
- Weighted averaging
- Rolling average computation

### Integration Tests

**Growth Score prototype** (`tests/test_indicators/test_growth_score_integration.py`):
- Registry registration
- End-to-end data fetching
- Score computation
- Visualization

### Manual Testing Steps

After Phase 1.3 automated tests pass:

1. **Start the dashboard**: `just dashboard`
2. **Verify sidebar**: "Composite Scores" category appears with ⭐ icon
3. **Select Growth Score**: Choose "Growth Score (Composite)" from dropdown
4. **Verify chart loads**: Chart should display with data
5. **Check score values**: Values should be in 0-100 range
6. **Test date range**: Change observation start date, verify chart updates
7. **Check existing indicators**: Verify Growth and Inflation categories still work
8. **Console check**: No errors in Streamlit console

## Performance Considerations

### Concurrent Fetching
- `CompositeScore` uses `asyncio.TaskGroup()` to fetch all source indicators in parallel
- For a 3-indicator score, this is ~3x faster than sequential fetching
- Pattern scales well: 10 indicators would still fetch concurrently

### Caching Strategy
- Composite scores leverage existing `@st.cache_data(ttl=3600)` from `IndicatorRegistry`
- Source indicators are cached individually (so PMI data shared across multiple scores)
- Computed scores are cached at the registry level
- No additional caching needed in Phase 1

### Data Volume
- Threshold normalization is O(n) where n = number of data points
- Weighted averaging is O(n * m) where m = number of indicators
- For typical use (3 indicators, 5 years monthly data = 180 points): negligible overhead
- Polars operations are highly optimized for this scale

## Migration Notes

### No Breaking Changes
Phase 1 is purely additive:
- No modifications to existing indicator classes
- No changes to registry system
- No changes to visualization system
- All existing tests continue to pass

### Backward Compatibility
- Existing indicators (FRED, Calculated) work identically
- Dashboard UI unchanged (only adds new category)
- No configuration changes required
- No database migrations (no persistent storage)

## References

- **Original research**: `thoughts/searchable/shared/research/2025-12-30-phase1-foundation-architecture.md`
- **Indicator protocol**: `src/indicators/protocol.py:16-27`
- **CalculatedIndicator pattern**: `src/indicators/calculations/calculated_indicator.py:17-189`
- **Existing calculation examples**: `src/indicators/inflation/calculations.py`
- **Test patterns**: `tests/test_calculations/test_inflation_calculations.py`
- **Registry pattern**: `src/indicators/inflation/__init__.py:17-95`

---

## Implementation Checklist

### Phase 1.1: Normalization Utilities
- [ ] Create `src/indicators/calculations/normalization.py`
- [ ] Implement `threshold_normalize()` function
- [ ] Implement `weighted_average_scores()` function
- [ ] Create `tests/test_calculations/test_normalization.py`
- [ ] Run `just tests` - all normalization tests pass
- [ ] Run `just check` - no linting/type errors

### Phase 1.2: CompositeScore Class
- [ ] Create `src/indicators/composite_scores/` directory
- [ ] Create `src/indicators/composite_scores/__init__.py`
- [ ] Create `src/indicators/composite_scores/composite_score.py`
- [ ] Implement `CompositeScore` class with Indicator protocol
- [ ] Create `tests/test_indicators/test_composite_score.py`
- [ ] Run `just tests` - all CompositeScore tests pass
- [ ] Run `just check` - no linting/type errors
- [ ] **MANUAL**: Verify CompositeScore behavior matches expectations

### Phase 1.3: Prototype Growth Score
- [ ] Create `src/indicators/composite_scores/indicators.py`
- [ ] Create `src/indicators/composite_scores/scores.py`
- [ ] Implement Growth Score factory function
- [ ] Update `src/services/registry_setup.py` to register composite scores
- [ ] Create `tests/test_indicators/test_growth_score_integration.py`
- [ ] Run `just tests` - all integration tests pass
- [ ] Run `just dashboard` - dashboard starts without errors
- [ ] **MANUAL**: Verify Growth Score appears in dashboard
- [ ] **MANUAL**: Verify Growth Score chart renders correctly
- [ ] **MANUAL**: Verify existing indicators still work

### Phase 1.4: Documentation
- [ ] Update `CLAUDE.md` with composite score patterns
- [ ] Add examples and references
- [ ] Link to research document
- [ ] **MANUAL**: Review documentation for clarity and accuracy

### Final Verification
- [ ] Run `just check` - passes
- [ ] Run `just tests` - all tests pass
- [ ] Run `just dashboard` - application works correctly
- [ ] **MANUAL**: Complete manual testing checklist
- [ ] **MANUAL**: Verify no regressions in existing functionality

---

## Success Metrics

Phase 1 is complete when:

1. ✅ All automated tests pass (`just tests`)
2. ✅ All linting and type checks pass (`just check`)
3. ✅ Dashboard runs without errors (`just dashboard`)
4. ✅ Growth Score appears in dashboard and displays data
5. ✅ Documentation updated and accurate
6. ✅ No regressions in existing indicators
7. ✅ Pattern documented for future composite score implementations

This provides the foundation for Phases 2-6 (category-specific composite scores) and Phase 8 (regime detection).
