---
date: 2025-12-29T00:00:00Z
researcher: Elliot Steene
git_commit: 8c5b695c26ed305ad05a9a585c6c2c1eb63a2e13
branch: main
repository: cc-analyst
topic: "3-Month Rolling Average Not Reflecting Calculated Indicator Values"
tags: [research, codebase, calculated-indicator, rolling-average, visualization]
status: complete
last_updated: 2025-12-29
last_updated_by: Elliot Steene
---

# Research: 3-Month Rolling Average Not Reflecting Calculated Indicator Values

**Date**: 2025-12-29
**Researcher**: Elliot Steene
**Git Commit**: 8c5b695c26ed305ad05a9a585c6c2c1eb63a2e13
**Branch**: main
**Repository**: cc-analyst

## Research Question

When using a calculated indicator (like "Core PCE 3m Annualized"), if the "Show 3-Month Rolling Average" toggle is activated, the 3m avg displayed is NOT the average for the calculated field - it is the 3m avg for the original/source indicator. What is causing this behavior?

## Summary

The root cause is the **timing of when the `rolling_avg_3m` column is computed relative to when the calculation is applied**. The rolling average is computed by `FredIndicator.get_series()` on the raw source data BEFORE the `CalculatedIndicator` applies its transformation. The calculation function only transforms the `value` column, leaving `rolling_avg_3m` unchanged (or in some cases, dropping it entirely).

## Detailed Findings

### Rolling Average Computation Location

The rolling average is computed in `FredIndicator.get_series()`:

**File**: `src/indicators/fred_indicator.py:32-36`
```python
series = series.with_columns(
    rolling_avg_3m=pl.col("value").rolling_mean(
        window_size=self.frequency_window_size(3)
    )
)
```

This happens BEFORE data is returned to any caller, including `CalculatedIndicator`.

### CalculatedIndicator Data Flow

When `CalculatedIndicator.get_series()` is called for a single-source calculation:

**File**: `src/indicators/calculations/calculated_indicator.py:88-96`
```python
async def _get_single_series(
    self,
    observation_start: str | None,
) -> pl.DataFrame:
    source = cast(Indicator, self._source)
    calculation = cast(SingleSourceCalculation, self._calculation)

    data = await source.get_series(observation_start=observation_start)  # Step 1
    return calculation(data)  # Step 2
```

**Step 1**: Calls `source.get_series()` (e.g., `FredIndicator.get_series()`)
- Returns DataFrame with columns: `date`, `value`, `rolling_avg_3m`
- `rolling_avg_3m` is computed on the **raw source data** at this point

**Step 2**: Applies the calculation function to transform the data
- The calculation only modifies the `value` column
- `rolling_avg_3m` remains unchanged

### Calculation Function Behavior

**File**: `src/indicators/inflation/calculations.py:4-14`
```python
def calculate_3m_annualized_rate(data: pl.DataFrame) -> pl.DataFrame:
    return data.with_columns(
        (((pl.col("value") / pl.col("value").shift(3)).pow(4) - 1) * 100).alias("value")
    ).filter(pl.col("value").is_not_null())
```

This function:
- Replaces the `value` column with the annualized rate
- Does NOT update `rolling_avg_3m`
- The returned DataFrame still contains the original `rolling_avg_3m` from the source

**File**: `src/indicators/inflation/calculations.py:17-45`
```python
def calculate_cpi_momentum(data: pl.DataFrame) -> pl.DataFrame:
    return (
        data.with_columns([...])
        .with_columns((pl.col("rate_3m") - pl.col("rate_6m")).alias("value"))
        .select(["date", "value"])  # <-- Drops rolling_avg_3m entirely
        .filter(pl.col("value").is_not_null())
    )
```

This function actually drops the `rolling_avg_3m` column entirely via `.select(["date", "value"])`.

### Concrete Example: CORE_PCE_3M_ANNUALIZED

**File**: `src/indicators/inflation/__init__.py:81-87`
```python
.register_indicator(
    indicator_name=InflationIndicatorName.CORE_PCE_3M_ANNUALIZED,
    indicator=CalculatedIndicator(
        source=pcepilfe_indicator,
        calculation=calculate_3m_annualized_rate,
    ),
)
```

Data flow when this indicator is fetched with `show_3m_avg=True`:

1. `CalculatedIndicator.get_series()` called
2. Calls `pcepilfe_indicator.get_series()` (FredIndicator)
3. FredIndicator fetches PCE index data from FRED
4. FredIndicator adds `rolling_avg_3m` = rolling mean of **PCE index values**
5. Returns DataFrame: `{date, value: PCE_index, rolling_avg_3m: rolling_mean(PCE_index)}`
6. `calculate_3m_annualized_rate()` transforms `value` to annualized rate
7. Returns DataFrame: `{date, value: annualized_rate, rolling_avg_3m: rolling_mean(PCE_index)}`
8. Chart plots both columns, but `rolling_avg_3m` shows the **source index** rolling average, not the **annualized rate** rolling average

### Visualization Path

**File**: `src/components/indicator_chart.py:59-63`
```python
viz_config = (
    VisualizationConfig.create_with_original_and_3m_avg()
    if show_3m_avg
    else VisualizationConfig().add_series("value", "Original")
)
```

**File**: `src/indicators/visualization_config.py:33-39`
```python
@classmethod
def create_with_original_and_3m_avg(cls) -> "VisualizationConfig":
    return (
        cls()
        .add_series("value", "Original")
        .add_series("rolling_avg_3m", "3-Month Avg")
    )
```

The visualization config blindly looks for `rolling_avg_3m` column in the data, unaware of whether it represents a rolling average of the calculated value or the source value.

## Code References

- `src/indicators/fred_indicator.py:32-36` - Where rolling average is computed (on raw data)
- `src/indicators/calculations/calculated_indicator.py:95` - Where source data (with rolling_avg_3m) is fetched
- `src/indicators/calculations/calculated_indicator.py:96` - Where calculation is applied (doesn't update rolling_avg_3m)
- `src/indicators/inflation/calculations.py:12-14` - Calculation function that only transforms `value`
- `src/indicators/inflation/calculations.py:43` - CPI momentum drops rolling_avg_3m entirely
- `src/indicators/inflation/__init__.py:81-87` - Registration of calculated indicator
- `src/components/indicator_chart.py:59-63` - Visualization config selection
- `src/indicators/visualization_config.py:33-39` - Factory method that assumes rolling_avg_3m exists

## Architecture Documentation

### Current Pattern

The codebase uses an **eager computation pattern** for rolling averages:
- `FredIndicator` always computes `rolling_avg_3m` at data fetch time
- The column is added before data is returned to any consumer
- This works correctly for direct FRED indicators

### Protocol Design

The `Indicator` protocol (`src/indicators/protocol.py:16-24`) defines:
- `get_series()` - returns `pl.DataFrame`
- `visualise_series()` - returns `go.Figure`

Neither method specifies expected DataFrame columns, allowing flexibility but also enabling this mismatch.

### CalculatedIndicator Design

`CalculatedIndicator` wraps source indicators and applies transformations:
- Single-source: One indicator + transformation function
- Multi-source: Multiple indicators + combination function

The transformation functions receive the full DataFrame from the source, including any pre-computed columns like `rolling_avg_3m`.

## Solution Approach

All indicators should support the 3-month rolling average, with frequency-aware window sizing.

### Recommended Fix Location

The rolling average should be recomputed in `CalculatedIndicator.get_series()` **after** applying the calculation. This centralizes the logic and ensures consistency across all calculated indicators.

### Frequency Handling

`FredIndicator` already has the `frequency_window_size()` method that converts month count to data points:

**File**: `src/indicators/fred_indicator.py:65-74`
```python
def frequency_window_size(self, month_window: int) -> int:
    match self.frequency:
        case IndicatorFrequency.MONTH:
            return month_window
        case IndicatorFrequency.WEEK:
            return month_window * 4
        case IndicatorFrequency.DAY:
            return month_window * 30
```

This logic should be reused or extracted for `CalculatedIndicator` to compute the correct window size.

### Implementation Approach

**Infer frequency from source indicators with validation**

- Add `frequency` property to `Indicator` protocol
- `CalculatedIndicator` accesses `source.frequency` for single-source
- For multi-source, validate all sources share the same frequency (raise error if mismatch)
- This validation is important because calculations between indicators with different frequencies would produce inconsistent/invalid results

### Files Requiring Changes

1. `src/indicators/protocol.py`
   - Add `frequency: IndicatorFrequency` property to the `Indicator` protocol

2. `src/indicators/calculations/calculated_indicator.py`
   - Add `frequency` property that reads from source (single) or validates and returns shared frequency (multi)
   - Recompute `rolling_avg_3m` after calculation in `_get_single_series()` and `_get_multi_series()`
   - Add frequency validation for multi-source indicators

3. `src/indicators/inflation/calculations.py`
   - `CalculatedIndicator` will add `rolling_avg_3m` regardless of what calculation returns, so `.select(["date", "value"])` in `calculate_cpi_momentum()` is fine

### Calculation Function Considerations

Calculation functions return DataFrame with `date` and `value` columns. `CalculatedIndicator.get_series()` will:
1. Fetch source data
2. Apply calculation
3. Add/overwrite `rolling_avg_3m` column using inferred frequency

This centralizes rolling average logic in `CalculatedIndicator` and keeps calculation functions simple.
