# Dashboard Refactor: SOLID Structure Implementation Plan

## Overview

Refactor the monolithic `src/dashboard.py` into a well-structured multi-page Streamlit application following SOLID principles. The refactored code will eliminate pyrefly errors, enable dynamic category handling without code changes, and separate Streamlit dependencies from pure business logic for testability.

## Current State Analysis

### Problems Identified

1. **Pyrefly Errors (3 unbound-name errors)**:
   - `data` variable at lines 146, 150, 160 may be uninitialized
   - Caused by `st.stop()` not being recognized as a control flow terminator

2. **Hardcoded Category Dispatch** (lines 109-116):
   ```python
   if category_key == "growth":
       indicator_enum = _string_to_enum(GrowthIndicatorName, selected_indicator)
   elif category_key == "inflation":
       indicator_enum = _string_to_enum(InflationIndicatorName, selected_indicator)
   ```
   Adding new categories requires modifying the dashboard.

3. **Monolithic Structure**:
   - 167 lines of global-scope execution
   - UI logic, data fetching, and configuration all mixed together
   - No separation of concerns

4. **Tight Coupling**:
   - Dashboard imports every category's enum class directly
   - Business logic intertwined with Streamlit components

### Key Discoveries

- **Existing Registry Pattern** (`src/indicators/registry.py:15-46`): Already uses Generic[T] but doesn't store enum class reference
- **RegistryManager** (`src/indicators/registry_manager.py`): Manages categories but lacks enum lookup capability
- **Factory Pattern** (`src/indicators/growth/__init__.py`): Uses `@st.cache_resource` for singleton registries

## Desired End State

After implementation:

1. **Zero pyrefly errors**: `uv run pyrefly check src/` passes cleanly
2. **Dynamic categories**: New indicator categories work without modifying dashboard code
3. **Multi-page structure**: Entry point at `src/app.py` with pages in `src/pages/`
4. **Testable services**: Pure Python business logic separated from Streamlit UI
5. **Component-based UI**: Reusable sidebar and chart components

### Verification

```bash
# All checks pass
uv run pyrefly check src/
uv run ruff check src/
uv run streamlit run src/app.py  # App loads and functions correctly
```

## What We're NOT Doing

- Not adding new indicator categories (out of scope)
- Not changing the visual design of the dashboard
- Not modifying the FRED data fetching logic in `FredIndicator`
- Not adding unit tests (can be follow-up work)
- Not changing the `VisualizationConfig` or `MultiSeriesVisualizer` classes

## Implementation Approach

The refactoring follows an incremental approach:

1. **Phase 1**: Modify registry infrastructure to support dynamic enum lookup
2. **Phase 2**: Extract services layer (pure Python, testable)
3. **Phase 3**: Create component layer (reusable Streamlit UI pieces)
4. **Phase 4**: Create multi-page app structure
5. **Phase 5**: Clean up and verify

---

## Phase 1: Registry Infrastructure Enhancement

### Overview

Extend `IndicatorRegistry` to store its enum class and provide string-to-enum conversion. This eliminates the need for hardcoded `if/elif` category dispatch.

### Changes Required

#### 1.1 Modify IndicatorRegistry

**File**: `src/indicators/registry.py`
**Changes**: Add `enum_class` parameter to constructor and `get_indicator_enum()` method

```python
import asyncio
from enum import StrEnum
from typing import Generic, TypeVar

import plotly.graph_objs as go
import polars as pl
import streamlit as st

from src.indicators.protocol import Indicator
from src.indicators.visualization_config import VisualizationConfig

T = TypeVar("T", bound=StrEnum)


class IndicatorRegistry(Generic[T]):
    def __init__(self, enum_class: type[T]) -> None:
        self._enum_class = enum_class
        self._indicator_registry: dict[T, Indicator] = {}

    @property
    def enum_class(self) -> type[T]:
        """Return the enum class for this registry"""
        return self._enum_class

    def get_indicator_enum(self, indicator_name: str) -> T | None:
        """Convert string indicator name to enum member"""
        for member in self._enum_class:
            if member.value == indicator_name:
                return member
        return None

    def register_indicator(
        self, indicator_name: T, indicator: Indicator
    ) -> "IndicatorRegistry[T]":
        self._indicator_registry[indicator_name] = indicator
        return self

    def get_registered_indicators(self) -> list[str]:
        return [key.value for key in self._indicator_registry.keys()]

    @st.cache_data(ttl=3600)
    def get_indicator_series(
        _self,
        indicator_name: T,
        observation_start: str,
    ) -> pl.DataFrame:
        indicator = _self._indicator_registry[indicator_name]
        data = asyncio.run(indicator.get_series(observation_start=observation_start))
        return data

    def get_series_visualisation(
        self,
        indicator_name: T,
        data: pl.DataFrame,
        config: VisualizationConfig | None = None,
    ) -> go.Figure:
        indicator = self._indicator_registry[indicator_name]
        return indicator.visualise_series(data=data, config=config)
```

#### 1.2 Update Growth Registry Factory

**File**: `src/indicators/growth/__init__.py`
**Changes**: Pass `enum_class` to `IndicatorRegistry` constructor

```python
import streamlit as st

from src.indicators.fred_indicator import FredIndicator
from src.indicators.growth.indicators import (
    GROWTH_INDICATOR_NAME_CODE_MAPPING,
    GrowthIndicatorName,
)
from src.indicators.protocol import IndicatorFrequency
from src.indicators.registry import IndicatorRegistry


@st.cache_resource
def get_growth_indicator_registry() -> IndicatorRegistry[GrowthIndicatorName]:
    """Initialize and cache the indicator registry"""
    registry = (
        IndicatorRegistry[GrowthIndicatorName](enum_class=GrowthIndicatorName)
        .register_indicator(
            indicator_name=GrowthIndicatorName.PMI_US,
            indicator=FredIndicator(
                fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[
                    GrowthIndicatorName.PMI_US
                ],
                frequency=IndicatorFrequency.MONTH,
            ),
        )
        .register_indicator(
            indicator_name=GrowthIndicatorName.PAYEMS,
            indicator=FredIndicator(
                fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[
                    GrowthIndicatorName.PAYEMS
                ],
                frequency=IndicatorFrequency.MONTH,
            ),
        )
        .register_indicator(
            indicator_name=GrowthIndicatorName.ICSA,
            indicator=FredIndicator(
                fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[GrowthIndicatorName.ICSA],
                frequency=IndicatorFrequency.WEEK,
            ),
        )
        .register_indicator(
            indicator_name=GrowthIndicatorName.CCSA,
            indicator=FredIndicator(
                fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[GrowthIndicatorName.CCSA],
                frequency=IndicatorFrequency.WEEK,
            ),
        )
        .register_indicator(
            indicator_name=GrowthIndicatorName.UNRATE,
            indicator=FredIndicator(
                fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[
                    GrowthIndicatorName.UNRATE
                ],
                frequency=IndicatorFrequency.MONTH,
            ),
        )
        .register_indicator(
            indicator_name=GrowthIndicatorName.HOUST,
            indicator=FredIndicator(
                fred_code=GROWTH_INDICATOR_NAME_CODE_MAPPING[GrowthIndicatorName.HOUST],
                frequency=IndicatorFrequency.MONTH,
            ),
        )
    )

    return registry
```

#### 1.3 Update Inflation Registry Factory

**File**: `src/indicators/inflation/__init__.py`
**Changes**: Pass `enum_class` to `IndicatorRegistry` constructor

```python
import streamlit as st

from src.indicators.fred_indicator import FredIndicator
from src.indicators.inflation.indicators import (
    INFLATION_INDICATOR_NAME_CODE_MAPPING,
    InflationIndicatorName,
)
from src.indicators.protocol import IndicatorFrequency
from src.indicators.registry import IndicatorRegistry


@st.cache_resource
def get_inflation_indicator_registry() -> IndicatorRegistry[InflationIndicatorName]:
    return (
        IndicatorRegistry[InflationIndicatorName](enum_class=InflationIndicatorName)
        .register_indicator(
            indicator_name=InflationIndicatorName.PCEPILFE,
            indicator=FredIndicator(
                fred_code=INFLATION_INDICATOR_NAME_CODE_MAPPING[
                    InflationIndicatorName.PCEPILFE
                ],
                frequency=IndicatorFrequency.MONTH,
            ),
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
            indicator=FredIndicator(
                fred_code=INFLATION_INDICATOR_NAME_CODE_MAPPING[
                    InflationIndicatorName.CPIAUCSL
                ],
                frequency=IndicatorFrequency.MONTH,
            ),
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
    )
```

### Success Criteria

#### Automated Verification:
- [ ] Pyrefly check passes: `uv run pyrefly check src/indicators/`
- [ ] Ruff check passes: `uv run ruff check src/indicators/`
- [ ] Existing dashboard still runs: `uv run streamlit run src/dashboard.py`

#### Manual Verification:
- [ ] Dashboard loads without errors
- [ ] Both Growth and Inflation categories work correctly
- [ ] Data fetching and visualization still function

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Services Layer

### Overview

Create a services layer with pure Python business logic separated from Streamlit. This enables unit testing and clarifies the separation of concerns.

### Changes Required

#### 2.1 Create Registry Setup Service

**File**: `src/services/__init__.py`
**Changes**: Create empty package init

```python
```

**File**: `src/services/registry_setup.py`
**Changes**: Pure Python registry configuration (no Streamlit imports)

```python
"""Registry configuration service - pure Python, no Streamlit dependencies."""

from src.indicators.growth import get_growth_indicator_registry
from src.indicators.inflation import get_inflation_indicator_registry
from src.indicators.registry_manager import RegistryManager, RegistryMetadata


def create_registry_manager() -> RegistryManager:
    """
    Create and configure the registry manager with all indicator categories.

    This function is pure configuration - it defines what categories exist
    and their metadata. The actual registry instances are created lazily
    by the cached factory functions.

    Returns:
        Configured RegistryManager with all categories registered.
    """
    manager = RegistryManager()

    manager.register_registry(
        category_key="growth",
        registry=get_growth_indicator_registry(),
        metadata=RegistryMetadata(
            display_name="Growth",
            icon="📈",
            description="Growth indicators including PMI, Payrolls, and Housing",
        ),
    )

    manager.register_registry(
        category_key="inflation",
        registry=get_inflation_indicator_registry(),
        metadata=RegistryMetadata(
            display_name="Inflation",
            icon="📊",
            description="Inflation indicators including CPI, PPI, and Breakeven Rates",
        ),
    )

    return manager
```

#### 2.2 Create Indicator Data Service

**File**: `src/services/indicator_service.py`
**Changes**: Data fetching logic with clear success/failure handling

```python
"""Indicator data service - handles data fetching with clear error handling."""

from dataclasses import dataclass
from enum import StrEnum

import polars as pl

from src.indicators.registry import IndicatorRegistry


@dataclass(frozen=True)
class IndicatorData:
    """Successfully fetched indicator data."""
    data: pl.DataFrame
    indicator_name: str
    observation_start: str


@dataclass(frozen=True)
class FetchError:
    """Error during data fetch."""
    message: str
    details: str | None = None


def fetch_indicator_data(
    registry: IndicatorRegistry,
    indicator_name: str,
    observation_start: str,
) -> IndicatorData | FetchError:
    """
    Fetch indicator data from the registry.

    Args:
        registry: The indicator registry to fetch from
        indicator_name: Display name of the indicator
        observation_start: Start date in YYYY-MM-DD format

    Returns:
        IndicatorData on success, FetchError on failure
    """
    # Convert string to enum
    indicator_enum = registry.get_indicator_enum(indicator_name)
    if indicator_enum is None:
        return FetchError(
            message=f"Invalid indicator: {indicator_name}",
            details="Indicator not found in registry",
        )

    # Fetch data
    try:
        data = registry.get_indicator_series(indicator_enum, observation_start)
    except Exception as e:
        return FetchError(
            message=f"Error fetching data: {str(e)}",
            details="Please check your FRED_API_KEY in .env file",
        )

    # Check for empty data
    if data.is_empty():
        return FetchError(
            message="No data available",
            details="No data available for the selected date range",
        )

    return IndicatorData(
        data=data,
        indicator_name=indicator_name,
        observation_start=observation_start,
    )


def get_indicator_enum(registry: IndicatorRegistry, indicator_name: str) -> StrEnum | None:
    """
    Get the enum value for an indicator name.

    Args:
        registry: The indicator registry
        indicator_name: Display name of the indicator

    Returns:
        The enum member or None if not found
    """
    return registry.get_indicator_enum(indicator_name)
```

### Success Criteria

#### Automated Verification:
- [ ] Pyrefly check passes: `uv run pyrefly check src/services/`
- [ ] Ruff check passes: `uv run ruff check src/services/`

#### Manual Verification:
- [ ] Services module can be imported without Streamlit running

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 3.

---

## Phase 3: Components Layer

### Overview

Create reusable Streamlit UI components that encapsulate presentation logic. Components receive data and return user selections - they don't fetch data themselves.

### Changes Required

#### 3.1 Create Components Package

**File**: `src/components/__init__.py`
**Changes**: Create empty package init

```python
```

#### 3.2 Create Sidebar Selection Dataclass

**File**: `src/components/sidebar.py`
**Changes**: Sidebar component with clear return type

```python
"""Sidebar component for indicator selection."""

from dataclasses import dataclass
from datetime import datetime, timedelta

import streamlit as st

from src.indicators.registry_manager import RegistryManager


@dataclass(frozen=True)
class SidebarSelection:
    """User selections from the sidebar."""
    category: str
    indicator: str
    start_date: str
    show_3m_avg: bool


def render_sidebar(registry_manager: RegistryManager) -> SidebarSelection:
    """
    Render the sidebar and return user selections.

    Args:
        registry_manager: The registry manager with available categories

    Returns:
        SidebarSelection with all user choices
    """
    st.sidebar.header("Indicator Selection")

    # Category selection
    available_categories = registry_manager.get_categories()
    selected_category = st.sidebar.selectbox(
        "Indicator Category",
        options=available_categories,
        index=0,
        help="Select a category of macro indicators",
    )

    # Category description
    category_metadata = registry_manager.get_metadata_by_display_name(selected_category)
    if category_metadata.description:
        st.sidebar.caption(f"ℹ️ {category_metadata.description}")

    # Indicator selection
    available_indicators = registry_manager.get_indicators_for_category(selected_category)
    selected_indicator = st.sidebar.selectbox(
        "Choose Indicator",
        options=available_indicators,
        index=0,
        help=f"Select a {selected_category} indicator to visualize",
    )

    # Date range
    st.sidebar.markdown("---")
    st.sidebar.subheader("Date Range")

    default_start = datetime.now() - timedelta(weeks=52 * 5)
    start_date = st.sidebar.date_input(
        "Start Date",
        value=default_start,
        max_value=datetime.now(),
        help="Select the start date for the time series",
    )

    # Display options
    st.sidebar.markdown("---")
    st.sidebar.subheader("Display Options")

    show_3m_avg = st.sidebar.checkbox(
        "Show 3-Month Rolling Average",
        value=False,
        help="Display the 3-month rolling average alongside original values",
    )

    st.sidebar.markdown("---")

    return SidebarSelection(
        category=selected_category,
        indicator=selected_indicator,
        start_date=start_date.strftime("%Y-%m-%d"),
        show_3m_avg=show_3m_avg,
    )
```

#### 3.3 Create Indicator Chart Component

**File**: `src/components/indicator_chart.py`
**Changes**: Chart rendering component

```python
"""Indicator chart component for data visualization."""

import polars as pl
import streamlit as st

from src.indicators.registry import IndicatorRegistry
from src.indicators.visualization_config import VisualizationConfig
from src.services.indicator_service import get_indicator_enum


def render_indicator_header(
    indicator_name: str,
    observation_start: str,
    data: pl.DataFrame,
) -> None:
    """
    Render the indicator header with metrics.

    Args:
        indicator_name: Display name of the indicator
        observation_start: Start date string
        data: The indicator data
    """
    st.title(f"📊 {indicator_name}")

    col1, col2, col3 = st.columns(3)

    with col1:
        st.metric("Start Date", observation_start)

    with col2:
        latest_date = data.select(pl.col("date").max()).item()
        st.metric(
            "Latest Date",
            latest_date.strftime("%Y-%m-%d") if latest_date else "N/A",
        )

    with col3:
        st.metric("Data Points", f"{len(data):,}")


def render_indicator_chart(
    registry: IndicatorRegistry,
    indicator_name: str,
    data: pl.DataFrame,
    show_3m_avg: bool,
) -> None:
    """
    Render the indicator time series chart.

    Args:
        registry: The indicator registry
        indicator_name: Display name of the indicator
        data: The indicator data
        show_3m_avg: Whether to show 3-month rolling average
    """
    st.markdown("### Time Series Chart")

    # Build visualization config
    viz_config = (
        VisualizationConfig.create_with_original_and_3m_avg()
        if show_3m_avg
        else VisualizationConfig().add_series("value", "Original")
    )

    # Get enum for visualization
    indicator_enum = get_indicator_enum(registry, indicator_name)
    if indicator_enum is None:
        st.error(f"Could not resolve indicator: {indicator_name}")
        return

    # Create and display chart
    fig = registry.get_series_visualisation(indicator_enum, data, config=viz_config)
    fig.update_layout(
        title=f"{indicator_name} Over Time",
        height=500,
        hovermode="x unified",
    )

    st.plotly_chart(fig, use_container_width=True)
```

### Success Criteria

#### Automated Verification:
- [ ] Pyrefly check passes: `uv run pyrefly check src/components/`
- [ ] Ruff check passes: `uv run ruff check src/components/`

#### Manual Verification:
- [ ] Components are syntactically correct

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 4.

---

## Phase 4: Multi-Page App Structure

### Overview

Create the multi-page app structure with `st.navigation` for flexible navigation. The entry point (`app.py`) handles page configuration and routing.

### Changes Required

#### 4.1 Create Pages Package

**File**: `src/pages/__init__.py`
**Changes**: Create empty package init

```python
```

#### 4.2 Create Home Page

**File**: `src/pages/home.py`
**Changes**: Simple welcome page with navigation

```python
"""Home page - welcome and navigation."""

import streamlit as st

from src.indicators.registry_manager import RegistryManager


def render_home_page(registry_manager: RegistryManager) -> None:
    """
    Render the home page with welcome message and navigation.

    Args:
        registry_manager: The registry manager with available categories
    """
    st.title("📊 Macro Indicators Dashboard")

    st.markdown("""
    Welcome to the Macro Indicators Dashboard. This application provides
    visualization and analysis of key macroeconomic indicators sourced from
    the Federal Reserve Economic Data (FRED) API.
    """)

    st.markdown("---")

    st.subheader("Available Categories")

    # Display available categories
    categories = registry_manager.get_categories()

    cols = st.columns(len(categories))
    for col, category in zip(cols, categories):
        with col:
            metadata = registry_manager.get_metadata_by_display_name(category)
            st.markdown(f"### {metadata.icon} {category}")
            st.caption(metadata.description)

            indicators = registry_manager.get_indicators_for_category(category)
            st.markdown(f"**{len(indicators)} indicators**")

    st.markdown("---")

    st.info("👈 Use the sidebar navigation to explore indicators.")
```

#### 4.3 Create Indicator Explorer Page

**File**: `src/pages/indicator_explorer.py`
**Changes**: Main indicator visualization page with guard clauses

```python
"""Indicator explorer page - main visualization interface."""

import streamlit as st

from src.components.indicator_chart import render_indicator_chart, render_indicator_header
from src.components.sidebar import SidebarSelection, render_sidebar
from src.indicators.registry_manager import RegistryManager
from src.services.indicator_service import FetchError, IndicatorData, fetch_indicator_data


def _handle_fetch_error(error: FetchError) -> None:
    """Display fetch error and stop execution."""
    st.error(error.message)
    if error.details:
        st.error(error.details)


def _fetch_data(
    registry_manager: RegistryManager,
    selection: SidebarSelection,
) -> IndicatorData | None:
    """
    Fetch indicator data with loading spinner.

    Returns IndicatorData on success, None on failure (error already displayed).
    """
    registry = registry_manager.get_registry_by_display_name(selection.category)

    with st.spinner(f"Fetching {selection.indicator} data from FRED..."):
        result = fetch_indicator_data(
            registry=registry,
            indicator_name=selection.indicator,
            observation_start=selection.start_date,
        )

    # Guard clause: handle errors
    if isinstance(result, FetchError):
        _handle_fetch_error(result)
        return None

    return result


def render_indicator_explorer(registry_manager: RegistryManager) -> None:
    """
    Render the indicator explorer page.

    Args:
        registry_manager: The registry manager with available categories
    """
    # Get user selections from sidebar
    selection = render_sidebar(registry_manager)

    # Fetch data (returns None on error)
    indicator_data = _fetch_data(registry_manager, selection)

    # Guard clause: stop if fetch failed
    if indicator_data is None:
        return

    # Get registry for visualization
    registry = registry_manager.get_registry_by_display_name(selection.category)

    # Render header with metrics
    render_indicator_header(
        indicator_name=indicator_data.indicator_name,
        observation_start=indicator_data.observation_start,
        data=indicator_data.data,
    )

    # Render chart
    render_indicator_chart(
        registry=registry,
        indicator_name=indicator_data.indicator_name,
        data=indicator_data.data,
        show_3m_avg=selection.show_3m_avg,
    )
```

#### 4.4 Create App Entry Point

**File**: `src/app.py`
**Changes**: Main entry point with st.navigation

```python
"""Macro Indicators Dashboard - Entry Point."""

import streamlit as st
from dotenv import load_dotenv

from src.pages.home import render_home_page
from src.pages.indicator_explorer import render_indicator_explorer
from src.services.registry_setup import create_registry_manager

# Load environment variables
load_dotenv()

# Page configuration
st.set_page_config(
    page_title="Macro Indicators Dashboard",
    page_icon="📊",
    layout="wide",
    initial_sidebar_state="expanded",
)


@st.cache_resource
def get_registry_manager():
    """Get cached registry manager instance."""
    return create_registry_manager()


def main() -> None:
    """Main application entry point."""
    registry_manager = get_registry_manager()

    # Define pages
    home_page = st.Page(
        lambda: render_home_page(registry_manager),
        title="Home",
        icon="🏠",
        default=True,
    )

    explorer_page = st.Page(
        lambda: render_indicator_explorer(registry_manager),
        title="Indicator Explorer",
        icon="📈",
    )

    # Navigation
    pg = st.navigation([home_page, explorer_page])
    pg.run()


if __name__ == "__main__":
    main()
```

### Success Criteria

#### Automated Verification:
- [ ] Pyrefly check passes on all new files: `uv run pyrefly check src/`
- [ ] Ruff check passes: `uv run ruff check src/`
- [ ] App runs without errors: `uv run streamlit run src/app.py`

#### Manual Verification:
- [ ] Home page loads and displays category information
- [ ] Navigation between pages works correctly
- [ ] Indicator Explorer shows sidebar with all categories
- [ ] Selecting indicators loads and displays data correctly
- [ ] 3-month rolling average toggle works
- [ ] Date range selection works
- [ ] Error states display correctly (e.g., invalid API key)

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 5.

---

## Phase 5: Cleanup and Final Verification

### Overview

Remove the old monolithic dashboard file and perform final verification that all functionality works correctly.

### Changes Required

#### 5.1 Remove Old Dashboard

**File**: `src/dashboard.py`
**Changes**: Delete file

This file is no longer needed as all functionality has been moved to the new multi-page structure.

#### 5.2 Update Any Documentation

If there are any references to `src/dashboard.py` in documentation or scripts, update them to point to `src/app.py`.

### Success Criteria

#### Automated Verification:
- [ ] Full pyrefly check passes: `uv run pyrefly check src/`
- [ ] Full ruff check passes: `uv run ruff check src/`
- [ ] App starts successfully: `uv run streamlit run src/app.py`

#### Manual Verification:
- [ ] Complete user flow works:
  1. App loads to home page
  2. Navigate to Indicator Explorer
  3. Select Growth category
  4. Select any indicator
  5. Verify chart displays
  6. Toggle 3-month average
  7. Change date range
  8. Switch to Inflation category
  9. Select any indicator
  10. Verify chart displays
- [ ] No console errors or warnings
- [ ] Old `dashboard.py` no longer exists

**Implementation Note**: After completing this phase and all verification passes, the refactoring is complete.

---

## Testing Strategy

### Unit Tests (Future Enhancement)

The refactored structure enables unit testing:

**Services Layer Tests:**
- `test_registry_setup.py`: Verify registry manager configuration
- `test_indicator_service.py`: Test fetch logic with mock registry

**Example Test Structure:**
```python
# tests/services/test_indicator_service.py
def test_fetch_indicator_data_success():
    """Test successful data fetch."""
    # Mock registry with test data
    # Call fetch_indicator_data
    # Assert IndicatorData returned

def test_fetch_indicator_data_invalid_indicator():
    """Test error handling for invalid indicator."""
    # Call with invalid indicator name
    # Assert FetchError returned

def test_fetch_indicator_data_empty_data():
    """Test error handling for empty result."""
    # Mock registry to return empty DataFrame
    # Assert FetchError returned
```

### Integration Tests (Future Enhancement)

- Test full page render with mock data
- Test navigation between pages
- Test session state preservation

### Manual Testing Steps

1. Start the app: `uv run streamlit run src/app.py`
2. Verify home page loads with category cards
3. Click "Indicator Explorer" in navigation
4. Test each category/indicator combination
5. Test date range changes
6. Test 3-month average toggle
7. Verify error handling by temporarily removing FRED_API_KEY

---

## Performance Considerations

1. **Registry Caching**: `@st.cache_resource` ensures registries are created once per session
2. **Data Caching**: `@st.cache_data(ttl=3600)` caches FRED API responses for 1 hour
3. **Lazy Loading**: Pages only render when navigated to
4. **No Additional Overhead**: The refactoring adds no new API calls or computations

---

## Migration Notes

### For Users

- The entry point changes from `src/dashboard.py` to `src/app.py`
- Update any run scripts: `uv run streamlit run src/app.py`
- No changes to environment variables or API keys required

### For Developers

- New categories should be added in `src/services/registry_setup.py`
- New pages should be added in `src/pages/` and registered in `src/app.py`
- Follow existing patterns for components and services

---

## Final Project Structure

```
src/
├── app.py                              # Entry point with st.navigation
├── pages/
│   ├── __init__.py
│   ├── home.py                         # Welcome page
│   └── indicator_explorer.py           # Main visualization page
├── components/
│   ├── __init__.py
│   ├── sidebar.py                      # Sidebar selection component
│   └── indicator_chart.py              # Chart rendering component
├── services/
│   ├── __init__.py
│   ├── registry_setup.py               # Registry configuration
│   └── indicator_service.py            # Data fetching logic
└── indicators/                         # Existing module (modified)
    ├── __init__.py
    ├── protocol.py                     # (unchanged)
    ├── registry.py                     # (modified: added enum_class)
    ├── registry_manager.py             # (unchanged)
    ├── fred_indicator.py               # (unchanged)
    ├── visualization_config.py         # (unchanged)
    ├── multi_series_visualizer.py      # (unchanged)
    ├── growth/
    │   ├── __init__.py                 # (modified: pass enum_class)
    │   └── indicators.py               # (unchanged)
    └── inflation/
        ├── __init__.py                 # (modified: pass enum_class)
        └── indicators.py               # (unchanged)
```

---

## References

- Original file: `src/dashboard.py`
- Registry pattern: `src/indicators/registry.py:15-46`
- Registry manager: `src/indicators/registry_manager.py`
- [Streamlit Multi-Page App Documentation](https://docs.streamlit.io/develop/concepts/multipage-apps/overview)
- [Streamlit Best Practices](https://medium.com/@johnpascualkumar077/best-practices-for-developing-streamlit-applications-a-guide-to-efficient-and-maintainable-code-4ae279b6ea4e)
