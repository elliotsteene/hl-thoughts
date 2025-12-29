# Calculated Indicator Rolling Average Fix - Implementation Plan

## Overview

Fix the `rolling_avg_3m` column in `CalculatedIndicator` to compute rolling averages on **calculated values** rather than source data. Currently, when a `CalculatedIndicator` returns data, the `rolling_avg_3m` column contains the rolling average of the source data (e.g., PCE index values), not the rolling average of the calculated output (e.g., annualized rates).

## Current State Analysis

### The Problem

1. `FredIndicator.get_series()` computes `rolling_avg_3m` on raw FRED data (`src/indicators/fred_indicator.py:32-36`)
2. `CalculatedIndicator._get_single_series()` fetches this data with pre-computed rolling avg (`src/indicators/calculations/calculated_indicator.py:95`)
3. Calculation functions only transform the `value` column (`src/indicators/inflation/calculations.py:12-14`)
4. The `rolling_avg_3m` column either remains unchanged (wrong values) or is dropped by calculations that filter columns

### Key Discoveries

- `FredIndicator` has a `frequency` property (`src/indicators/fred_indicator.py:17`)
- `CalculatedIndicator` does NOT have a `frequency` property (`src/indicators/calculations/calculated_indicator.py:43-52`)
- The `Indicator` protocol does NOT define `frequency` (`src/indicators/protocol.py:16-24`)
- `frequency_window_size()` method exists only in `FredIndicator` (`src/indicators/fred_indicator.py:65-74`)

## Desired End State

After implementation:
1. All indicators expose a `frequency` property via the `Indicator` protocol
2. `CalculatedIndicator` derives its frequency from source indicator(s)
3. Multi-source indicators with mismatched frequencies raise a clear error
4. `CalculatedIndicator.get_series()` recomputes `rolling_avg_3m` on the **calculated output**
5. The rolling average computation is extensible for future derived calculations

### Verification

- `CORE_PCE_3M_ANNUALIZED.get_series()` returns a DataFrame where `rolling_avg_3m` is the 3-month rolling average of the annualized rate values, not the raw PCE index
- Multi-source indicators with different frequencies raise `FrequencyMismatchError`

## What We're NOT Doing

- Adding test coverage (explicitly out of scope)
- Maintaining backward compatibility for custom `Indicator` implementations
- Supporting mixed-frequency multi-source indicators
- Changing the behavior of `FredIndicator` (it will continue to compute rolling avg on raw data)

## Implementation Approach

The solution follows a layered approach:
1. Extend the protocol to require `frequency`
2. Create proper error handling for frequency mismatches
3. Add frequency derivation to `CalculatedIndicator`
4. Create an extensible post-calculation processing system for derived metrics like rolling averages

---

## Phase 1: Add Frequency to Indicator Protocol

### Overview
Extend the `Indicator` protocol to require a `frequency` property, ensuring all indicators expose their data frequency.

### Changes Required:

#### 1.1 Update Indicator Protocol

**File**: `src/indicators/protocol.py`
**Changes**: Add `frequency` property to the protocol

```python
class Indicator(Protocol):
    @property
    def frequency(self) -> IndicatorFrequency: ...

    async def get_series(
        self,
        observation_start: str | None = None,
    ) -> pl.DataFrame: ...

    def visualise_series(
        self, data: pl.DataFrame, config: VisualizationConfig | None = None
    ) -> go.Figure: ...
```

#### 1.2 Update FredIndicator

**File**: `src/indicators/fred_indicator.py`
**Changes**: Convert `frequency` instance attribute to a property (for protocol compliance)

The current implementation already stores `frequency` as an instance attribute on line 17. This is compatible with the protocol since Python protocols accept both attributes and properties. No change needed.

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] Application runs without errors: `just dashboard`

#### Manual Verification:
- [ ] Existing indicators continue to work in the UI

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Create Frequency Mismatch Error

### Overview
Add a custom exception for handling multi-source indicator frequency mismatches.

### Changes Required:

#### 2.1 Create Exceptions Module

**File**: `src/indicators/exceptions.py` (new file)
**Changes**: Create custom exception for frequency mismatch

```python
from src.indicators.protocol import IndicatorFrequency


class FrequencyMismatchError(Exception):
    """Raised when multi-source indicators have different frequencies."""

    def __init__(
        self,
        frequencies: dict[str, IndicatorFrequency],
    ) -> None:
        self.frequencies = frequencies
        freq_details = ", ".join(
            f"{name}: {freq.value}" for name, freq in frequencies.items()
        )
        super().__init__(
            f"Multi-source indicator has mismatched frequencies: {freq_details}. "
            "All source indicators must have the same frequency."
        )
```

#### 2.2 Export from Indicators Package

**File**: `src/indicators/__init__.py`
**Changes**: Export the new exception

Add to exports:
```python
from src.indicators.exceptions import FrequencyMismatchError
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `just check`

#### Manual Verification:
- [ ] N/A (no user-facing changes yet)

**Implementation Note**: After completing this phase and all automated verification passes, proceed to Phase 3.

---

## Phase 3: Implement Frequency in CalculatedIndicator

### Overview
Add frequency property to `CalculatedIndicator` that derives from source indicator(s), with validation for multi-source scenarios.

### Changes Required:

#### 3.1 Add Frequency Property to CalculatedIndicator

**File**: `src/indicators/calculations/calculated_indicator.py`
**Changes**: Add frequency property with single/multi-source handling

Add import at top:
```python
from src.indicators.exceptions import FrequencyMismatchError
from src.indicators.protocol import Indicator, IndicatorFrequency
```

Add property after `__init__`:
```python
@property
def frequency(self) -> IndicatorFrequency:
    """
    Derive frequency from source indicator(s).

    For single-source: returns the source's frequency.
    For multi-source: validates all sources have the same frequency and returns it.

    Raises:
        FrequencyMismatchError: If multi-source indicators have different frequencies.
    """
    if self._is_multi_source:
        return self._get_multi_source_frequency()
    else:
        source = cast(Indicator, self._source)
        return source.frequency

def _get_multi_source_frequency(self) -> IndicatorFrequency:
    """Validate and return frequency for multi-source indicators."""
    sources = cast(dict[str, Indicator], self._source)

    frequencies: dict[str, IndicatorFrequency] = {
        name: indicator.frequency for name, indicator in sources.items()
    }

    unique_frequencies = set(frequencies.values())
    if len(unique_frequencies) > 1:
        raise FrequencyMismatchError(frequencies)

    return next(iter(unique_frequencies))
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] Application runs without errors: `just dashboard`

#### Manual Verification:
- [ ] Accessing `.frequency` on a `CalculatedIndicator` returns the correct value

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 4.

---

## Phase 4: Recompute Rolling Average After Calculation

### Overview
Add post-calculation processing to `CalculatedIndicator` that recomputes `rolling_avg_3m` on the calculated output values. Design this as an extensible system for future derived calculations.

### Changes Required:

#### 4.1 Add Window Size Calculation Utility

**File**: `src/indicators/calculations/calculated_indicator.py`
**Changes**: Add helper method for frequency-based window size calculation

```python
def _frequency_window_size(self, month_window: int) -> int:
    """
    Convert a month-based window to frequency-specific data points.

    Args:
        month_window: Number of months for the window

    Returns:
        Number of data points for the rolling window
    """
    match self.frequency:
        case IndicatorFrequency.MONTH:
            return month_window
        case IndicatorFrequency.WEEK:
            return month_window * 4
        case IndicatorFrequency.DAY:
            return month_window * 30
        case _:
            assert_never(self.frequency)
```

Add import for `assert_never`:
```python
from typing import assert_never, cast, overload
```

#### 4.2 Add Post-Calculation Processing

**File**: `src/indicators/calculations/calculated_indicator.py`
**Changes**: Add extensible post-processing method that computes derived columns

```python
def _apply_post_calculation_columns(self, data: pl.DataFrame) -> pl.DataFrame:
    """
    Apply post-calculation derived columns to the result.

    This method computes derived metrics on the calculated output values,
    not the source data. Currently computes:
    - rolling_avg_3m: 3-month rolling average of the 'value' column

    This is extensible for future derived calculations.

    Args:
        data: DataFrame with calculated 'value' column

    Returns:
        DataFrame with additional derived columns
    """
    # Drop any pre-existing rolling_avg_3m from source data
    if "rolling_avg_3m" in data.columns:
        data = data.drop("rolling_avg_3m")

    # Compute rolling average on calculated values
    data = data.with_columns(
        rolling_avg_3m=pl.col("value").rolling_mean(
            window_size=self._frequency_window_size(3)
        )
    )

    return data
```

#### 4.3 Integrate Post-Processing into get_series Flow

**File**: `src/indicators/calculations/calculated_indicator.py`
**Changes**: Update `get_series` to apply post-calculation processing

Update `get_series` method:
```python
async def get_series(
    self,
    observation_start: str | None = None,
) -> pl.DataFrame:
    if self._is_multi_source:
        result = await self._get_multi_series(observation_start)
    else:
        result = await self._get_single_series(observation_start)

    return self._apply_post_calculation_columns(result)
```

### Success Criteria:

#### Automated Verification:
- [ ] Type checking passes: `just check`
- [ ] Application runs without errors: `just dashboard`

#### Manual Verification:
- [ ] Load `CORE_PCE_3M_ANNUALIZED` indicator in the UI
- [ ] Verify `rolling_avg_3m` values are the rolling average of the annualized rate (not raw PCE index)
- [ ] Compare values manually: `rolling_avg_3m` at a given date should be approximately the mean of the 3 preceding `value` entries
- [ ] Existing `FredIndicator` visualizations still work correctly

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the rolling average values are correct.

---

## Testing Strategy

### Manual Testing Steps:
1. Run the application: `just dashboard`
2. Navigate to the Indicator Explorer
3. Select `CORE_PCE_3M_ANNUALIZED`
4. Export or inspect the data
5. Verify that `rolling_avg_3m` values match the expected 3-point rolling mean of the `value` column
6. Test a multi-source indicator (if one exists) to confirm frequency validation works
7. Test daily-frequency indicator (e.g., `T5YIFR`) to confirm window size calculation is correct (should use 90 points for 3-month window)

### Edge Cases to Verify:
- First few rows should have `null` for `rolling_avg_3m` (insufficient data for window)
- Weekly indicators should use 12-point windows
- Daily indicators should use 90-point windows

---

## Future Extensibility

The `_apply_post_calculation_columns` method is designed to be extensible. Future derived calculations can be added by:

1. Adding new column computations to the method
2. Optionally making the calculations configurable via constructor parameters
3. Example future additions:
   - `rolling_avg_6m`: 6-month rolling average
   - `rolling_std_3m`: Rolling standard deviation
   - `yoy_change`: Year-over-year percentage change

Example extension pattern:
```python
def _apply_post_calculation_columns(self, data: pl.DataFrame) -> pl.DataFrame:
    # Existing: 3-month rolling average
    data = data.with_columns(
        rolling_avg_3m=pl.col("value").rolling_mean(
            window_size=self._frequency_window_size(3)
        )
    )

    # Future: 6-month rolling average
    # data = data.with_columns(
    #     rolling_avg_6m=pl.col("value").rolling_mean(
    #         window_size=self._frequency_window_size(6)
    #     )
    # )

    return data
```

---

## References

- Original research: `thoughts/searchable/shared/research/2025-12-29-calculated-indicator-3m-avg-issue.md`
- Related implementation plan: `thoughts/searchable/shared/plans/2024-12-24-inflation-calculated-fields.md`
- Key files:
  - `src/indicators/protocol.py:16-24` (Indicator protocol)
  - `src/indicators/fred_indicator.py:65-74` (frequency_window_size reference)
  - `src/indicators/calculations/calculated_indicator.py` (main implementation target)
  - `src/indicators/inflation/__init__.py:81-87` (CORE_PCE_3M_ANNUALIZED registration)
