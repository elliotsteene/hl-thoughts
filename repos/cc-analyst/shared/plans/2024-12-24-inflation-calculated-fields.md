# Inflation Calculated Fields Implementation Plan

## Overview

Implement an extensible system for calculated/derived indicators that can compute metrics from one or more source indicators. The initial implementation includes two inflation calculations: Core PCE 3-Month Annualized Rate and CPI Momentum.

## Current State Analysis

- `FredIndicator` fetches raw FRED data and adds `rolling_avg_3m` column inline
- `IndicatorRegistry` manages indicators via the `Indicator` protocol
- No abstraction exists for calculated metrics - rolling average is hardcoded in `FredIndicator`
- Dashboard displays indicators by category via `RegistryManager`

### Key Files:
- `src/indicators/protocol.py:16-24` - `Indicator` protocol definition
- `src/indicators/fred_indicator.py` - FRED data fetching
- `src/indicators/inflation/__init__.py` - Inflation registry setup
- `src/indicators/registry.py` - Generic indicator registry

## Desired End State

1. A `CalculatedIndicator` class that computes derived metrics from source indicator data
2. Two initial calculated indicators registered in the inflation category:
   - "Core PCE 3m Annualized" - computed from PCEPILFE
   - "CPI Momentum (3m vs 6m)" - computed from CPIAUCSL
3. These appear as separate selectable indicators in the Inflation dropdown
4. Architecture supports both single-source and multi-source calculations for future extensibility

### Verification:
- Run dashboard with `streamlit run src/dashboard.py`
- Select "Inflation" category
- Verify "Core PCE 3m Annualized" and "CPI Momentum (3m vs 6m)" appear in dropdown
- Select each and verify chart displays correctly with calculated values

## What We're NOT Doing

- No Fed target reference lines on charts
- No interpretation zone visualization (colored bands or legends)
- No changes to the existing `FredIndicator` or raw indicator display
- No new dashboard category - calculated indicators live within "Inflation"

## Implementation Approach

Create a `CalculatedIndicator` class that:
1. Accepts one or more source `Indicator` instances
2. Accepts a calculation function that transforms the source DataFrame(s)
3. Implements the `Indicator` protocol so it integrates seamlessly with the registry

This approach keeps the calculation logic decoupled and testable while reusing existing visualization infrastructure.

---

## Phase 1: Create CalculatedIndicator Infrastructure

### Overview
Build the base `CalculatedIndicator` class and calculation function types.

### Changes Required:

#### 1.1 Create Calculation Types Module

**File**: `src/indicators/calculations/__init__.py`
**Changes**: Create new module for calculation infrastructure

```python
from src.indicators.calculations.calculated_indicator import CalculatedIndicator
from src.indicators.calculations.types import (
    SingleSourceCalculation,
    MultiSourceCalculation,
)

__all__ = [
    "CalculatedIndicator",
    "SingleSourceCalculation",
    "MultiSourceCalculation",
]
```

#### 1.2 Create Type Definitions

**File**: `src/indicators/calculations/types.py`
**Changes**: Define calculation function signatures

```python
from typing import Callable

import polars as pl

# Single source: takes one DataFrame, returns DataFrame with calculated columns
SingleSourceCalculation = Callable[[pl.DataFrame], pl.DataFrame]

# Multi source: takes dict of named DataFrames, returns DataFrame with calculated columns
MultiSourceCalculation = Callable[[dict[str, pl.DataFrame]], pl.DataFrame]
```

#### 1.3 Create CalculatedIndicator Class

**File**: `src/indicators/calculations/calculated_indicator.py`
**Changes**: Implement the calculated indicator class

```python
import asyncio
from typing import overload

import plotly.graph_objs as go
import polars as pl

from src.indicators.multi_series_visualizer import MultiSeriesVisualizer
from src.indicators.protocol import Indicator
from src.indicators.visualization_config import VisualizationConfig
from src.indicators.calculations.types import (
    SingleSourceCalculation,
    MultiSourceCalculation,
)


class CalculatedIndicator:
    """
    An indicator that computes derived metrics from one or more source indicators.

    Supports both single-source calculations (e.g., annualized rate from one series)
    and multi-source calculations (e.g., spread between two series).

    For multi-source calculations, all source indicators are fetched concurrently
    using asyncio.TaskGroup for optimal performance.
    """

    @overload
    def __init__(
        self,
        source: Indicator,
        calculation: SingleSourceCalculation,
        output_column: str = "value",
    ) -> None: ...

    @overload
    def __init__(
        self,
        source: dict[str, Indicator],
        calculation: MultiSourceCalculation,
        output_column: str = "value",
    ) -> None: ...

    def __init__(
        self,
        source: Indicator | dict[str, Indicator],
        calculation: SingleSourceCalculation | MultiSourceCalculation,
        output_column: str = "value",
    ) -> None:
        self._source = source
        self._calculation = calculation
        self._output_column = output_column
        self._is_multi_source = isinstance(source, dict)

    async def _fetch_source(
        self,
        name: str,
        indicator: Indicator,
        observation_start: str | None,
        results: dict[str, pl.DataFrame],
    ) -> None:
        """Fetch a single source indicator and store result in the results dict."""
        results[name] = await indicator.get_series(observation_start=observation_start)

    async def get_series(
        self,
        observation_start: str | None = None,
    ) -> pl.DataFrame:
        if self._is_multi_source:
            # Multi-source: fetch all sources concurrently using TaskGroup
            sources = self._source  # type: dict[str, Indicator]
            source_data: dict[str, pl.DataFrame] = {}

            async with asyncio.TaskGroup() as tg:
                for name, indicator in sources.items():
                    tg.create_task(
                        self._fetch_source(name, indicator, observation_start, source_data)
                    )

            calculation = self._calculation  # type: MultiSourceCalculation
            return calculation(source_data)
        else:
            # Single-source: fetch and calculate
            source = self._source  # type: Indicator
            data = await source.get_series(observation_start=observation_start)
            calculation = self._calculation  # type: SingleSourceCalculation
            return calculation(data)

    def visualise_series(
        self,
        data: pl.DataFrame,
        config: VisualizationConfig | None = None,
    ) -> go.Figure:
        if config is None:
            config = VisualizationConfig().add_series(self._output_column, "Value")

        visualizer = MultiSeriesVisualizer(config)
        return visualizer.visualize(data)
```

### Success Criteria:

#### Automated Verification:
- [ ] Module imports correctly: `python -c "from src.indicators.calculations import CalculatedIndicator"`
- [ ] Type checking passes: `just check` (if mypy is configured)

#### Manual Verification:
- [ ] N/A for this phase - infrastructure only

---

## Phase 2: Implement Inflation Calculation Functions

### Overview
Create the specific calculation functions for Core PCE 3m Annualized and CPI Momentum.

### Changes Required:

#### 2.1 Create Inflation Calculations Module

**File**: `src/indicators/inflation/calculations.py`
**Changes**: Implement the two calculation functions

```python
import polars as pl


def calculate_3m_annualized_rate(data: pl.DataFrame) -> pl.DataFrame:
    """
    Calculate the 3-month annualized rate of change.

    Formula: (((current / 3m_ago) ^ 4) - 1) * 100

    The exponent of 4 annualizes a 3-month rate (4 quarters per year).
    """
    return data.with_columns(
        (
            ((pl.col("value") / pl.col("value").shift(3)).pow(4) - 1) * 100
        ).alias("value")
    ).filter(pl.col("value").is_not_null())


def calculate_cpi_momentum(data: pl.DataFrame) -> pl.DataFrame:
    """
    Calculate CPI momentum as the difference between 3m and 6m annualized rates.

    Formulas:
        3m_rate = (((current / 3m_ago) ^ 4) - 1) * 100
        6m_rate = (((current / 6m_ago) ^ 2) - 1) * 100
        momentum = 3m_rate - 6m_rate

    Interpretation:
        Positive: Inflation accelerating
        Near zero: Stable inflation
        Negative: Inflation decelerating
    """
    return data.with_columns(
        [
            (
                ((pl.col("value") / pl.col("value").shift(3)).pow(4) - 1) * 100
            ).alias("rate_3m"),
            (
                ((pl.col("value") / pl.col("value").shift(6)).pow(2) - 1) * 100
            ).alias("rate_6m"),
        ]
    ).with_columns(
        (pl.col("rate_3m") - pl.col("rate_6m")).alias("value")
    ).select(
        ["date", "value"]
    ).filter(pl.col("value").is_not_null())
```

### Success Criteria:

#### Automated Verification:
- [ ] Module imports correctly: `python -c "from src.indicators.inflation.calculations import calculate_3m_annualized_rate, calculate_cpi_momentum"`

#### Manual Verification:
- [ ] N/A for this phase - functions only

---

## Phase 3: Register Calculated Indicators

### Overview
Add the calculated indicators to the inflation registry and update the indicator enum.

### Changes Required:

#### 3.1 Update Inflation Indicator Names

**File**: `src/indicators/inflation/indicators.py`
**Changes**: Add new enum values for calculated indicators

```python
from enum import StrEnum


class InflationIndicatorName(StrEnum):
    PCEPILFE = "Core PCE (US)"
    CPILFESL = "Core CPI (US)"
    CPIAUCSL = "CPI All Items (US)"
    PPIACO = "Producer Price Index (PPI)"
    T5YIFR = "5Y5Y Breakeven Inflation"
    T10YIE = "10Y Breakeven Inflation"
    # Calculated indicators
    CORE_PCE_3M_ANNUALIZED = "Core PCE 3m Annualized"
    CPI_MOMENTUM = "CPI Momentum (3m vs 6m)"


INFLATION_INDICATOR_NAME_CODE_MAPPING = {
    InflationIndicatorName.PCEPILFE: "PCEPILFE",
    InflationIndicatorName.CPILFESL: "CPILFESL",
    InflationIndicatorName.CPIAUCSL: "CPIAUCSL",
    InflationIndicatorName.PPIACO: "PPIACO",
    InflationIndicatorName.T5YIFR: "T5YIFR",
    InflationIndicatorName.T10YIE: "T10YIE",
}
```

Note: Calculated indicators don't have FRED codes, so they're not added to the mapping.

#### 3.2 Update Inflation Registry

**File**: `src/indicators/inflation/__init__.py`
**Changes**: Register the calculated indicators

```python
import streamlit as st

from src.indicators.calculations import CalculatedIndicator
from src.indicators.fred_indicator import FredIndicator
from src.indicators.inflation.calculations import (
    calculate_3m_annualized_rate,
    calculate_cpi_momentum,
)
from src.indicators.inflation.indicators import (
    INFLATION_INDICATOR_NAME_CODE_MAPPING,
    InflationIndicatorName,
)
from src.indicators.protocol import IndicatorFrequency
from src.indicators.registry import IndicatorRegistry


@st.cache_resource
def get_inflation_indicator_registry() -> IndicatorRegistry:
    # Create base FRED indicators
    pcepilfe_indicator = FredIndicator(
        fred_code=INFLATION_INDICATOR_NAME_CODE_MAPPING[InflationIndicatorName.PCEPILFE],
        frequency=IndicatorFrequency.MONTH,
    )
    cpiaucsl_indicator = FredIndicator(
        fred_code=INFLATION_INDICATOR_NAME_CODE_MAPPING[InflationIndicatorName.CPIAUCSL],
        frequency=IndicatorFrequency.MONTH,
    )

    return (
        IndicatorRegistry[InflationIndicatorName]()
        # Raw FRED indicators
        .register_indicator(
            indicator_name=InflationIndicatorName.PCEPILFE,
            indicator=pcepilfe_indicator,
        )
        .register_indicator(
            indicator_name=InflationIndicatorName.CPILFESL,
            indicator=FredIndicator(
                fred_code=INFLATION_INDICATOR_NAME_CODE_MAPPING[
                    InflationIndicatorName.CPILFESL
                ],
                frequency=IndicatorFrequency.MONTH,
            ),
        )
        .register_indicator(
            indicator_name=InflationIndicatorName.CPIAUCSL,
            indicator=cpiaucsl_indicator,
        )
        .register_indicator(
            indicator_name=InflationIndicatorName.PPIACO,
            indicator=FredIndicator(
                fred_code=INFLATION_INDICATOR_NAME_CODE_MAPPING[
                    InflationIndicatorName.PPIACO
                ],
                frequency=IndicatorFrequency.MONTH,
            ),
        )
        .register_indicator(
            indicator_name=InflationIndicatorName.T5YIFR,
            indicator=FredIndicator(
                fred_code=INFLATION_INDICATOR_NAME_CODE_MAPPING[
                    InflationIndicatorName.T5YIFR
                ],
                frequency=IndicatorFrequency.DAY,
            ),
        )
        .register_indicator(
            indicator_name=InflationIndicatorName.T10YIE,
            indicator=FredIndicator(
                fred_code=INFLATION_INDICATOR_NAME_CODE_MAPPING[
                    InflationIndicatorName.T10YIE
                ],
                frequency=IndicatorFrequency.DAY,
            ),
        )
        # Calculated indicators
        .register_indicator(
            indicator_name=InflationIndicatorName.CORE_PCE_3M_ANNUALIZED,
            indicator=CalculatedIndicator(
                source=pcepilfe_indicator,
                calculation=calculate_3m_annualized_rate,
            ),
        )
        .register_indicator(
            indicator_name=InflationIndicatorName.CPI_MOMENTUM,
            indicator=CalculatedIndicator(
                source=cpiaucsl_indicator,
                calculation=calculate_cpi_momentum,
            ),
        )
    )
```

### Success Criteria:

#### Automated Verification:
- [ ] Registry loads without errors: `python -c "from src.indicators.inflation import get_inflation_indicator_registry; get_inflation_indicator_registry()"`

#### Manual Verification:
- [ ] Run `streamlit run src/dashboard.py`
- [ ] Select "Inflation" category
- [ ] Verify "Core PCE 3m Annualized" appears in dropdown
- [ ] Verify "CPI Momentum (3m vs 6m)" appears in dropdown
- [ ] Select each calculated indicator and verify chart renders with data

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation from the human that the dashboard testing was successful before proceeding.

---

## Testing Strategy

### Unit Tests:
- Test `calculate_3m_annualized_rate` with known input/output values
- Test `calculate_cpi_momentum` with known input/output values
- Test `CalculatedIndicator` with mock source indicators

### Integration Tests:
- Test that calculated indicators return valid DataFrames with date and value columns
- Test that visualizations render without errors

### Manual Testing Steps:
1. Run dashboard and select "Core PCE 3m Annualized"
2. Verify values are reasonable (typically in the 1-5% range for recent data)
3. Select "CPI Momentum" and verify values oscillate around zero
4. Test date range changes work correctly
5. Verify 3-month rolling average toggle works on calculated indicators

---

## Future Extensibility

The `CalculatedIndicator` class supports both patterns you mentioned:

### Single-Source Example (already implemented):
```python
CalculatedIndicator(
    source=pcepilfe_indicator,
    calculation=calculate_3m_annualized_rate,
)
```

### Multi-Source Example (for future use):
```python
def calculate_real_rate(sources: dict[str, pl.DataFrame]) -> pl.DataFrame:
    nominal = sources["nominal"]
    inflation = sources["inflation"]
    # Join and calculate spread
    ...

CalculatedIndicator(
    source={
        "nominal": treasury_10y_indicator,
        "inflation": breakeven_10y_indicator,
    },
    calculation=calculate_real_rate,
)
```

---

## References

- Existing indicator pattern: `src/indicators/inflation/__init__.py`
- Protocol definition: `src/indicators/protocol.py:16-24`
- Visualization infrastructure: `src/indicators/multi_series_visualizer.py`
