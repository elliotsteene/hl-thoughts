# Streamlit Dashboard for Macro Indicators - Implementation Plan

## Overview

Implement a basic Streamlit web dashboard that allows users to select macro economic indicators from a dropdown and visualize the time series data with interactive charts. This MVP will leverage the existing `FredIndicator` and `GrowthIndicatorRegistry` infrastructure to display the two currently implemented growth indicators (Manufacturing PMI and Nonfarm Payrolls).

## Current State Analysis

The codebase has a solid foundation with:

- **Indicator Protocol** (`src/indicators/protocol.py:6`): Protocol-based design defining the `Indicator` contract
- **FredIndicator** (`src/indicators/fred_indicator.py:10`): Working FRED API integration with:
  - Async data fetching via `get_series()` method
  - Built-in Plotly visualization via `visualise_series()` method
  - Default 5-year lookback with configurable `observation_start` parameter
  - Returns Polars DataFrames with `date` and `value` columns
- **GrowthIndicatorRegistry** (`src/indicators/growth.py:19`): Registry pattern managing two indicators:
  - `GrowthIndicatorName.PMI_US` (FRED code: `MANEMP`)
  - `GrowthIndicatorName.PAYEMS` (FRED code: `PAYEMS`)
- **Dependencies**: Streamlit (≥1.52.2) already in `pyproject.toml` but not yet used
- **Environment**: `.env` file with FRED API key loaded via `python-dotenv`

**What's Missing**:
- No Streamlit application file exists
- No UI components for indicator selection or visualization
- Registry doesn't support date range parameters for visualization
- No error handling for API failures or missing environment variables

## Desired End State

After implementation, users will be able to:

1. Run the Streamlit dashboard via `streamlit run src/dashboard.py`
2. See a dropdown selector with all available growth indicators
3. Select a date range using date input widgets
4. Click a button to fetch and display the indicator data
5. View an interactive Plotly line chart
6. See summary statistics (min, max, mean, latest value) below the chart
7. Receive clear error messages if API calls fail or credentials are missing

### Verification:
- Navigate to `http://localhost:8501` in browser
- Select "FRED Manufacturing PMI (US)" from dropdown
- Set start date to 2020-01-01
- Click "Fetch Data" button
- Verify chart displays with correct date range
- Verify summary stats table shows accurate statistics
- Repeat for "Nonfarm Payrolls (US)" indicator
- Test with invalid date range (e.g., future date) to verify error handling

## What We're NOT Doing

- **NOT** implementing multi-indicator comparison charts (future enhancement)
- **NOT** adding data caching or local persistence (keeping it simple for MVP)
- **NOT** implementing other indicator categories (inflation, rates, etc.) yet
- **NOT** adding calculated fields (6-month growth, moving averages, etc.)
- **NOT** building composite scores (0-100 scoring system)
- **NOT** adding data export functionality (CSV downloads)
- **NOT** implementing authentication or user management
- **NOT** deploying to production (local development only)

## Implementation Approach

We'll build a single-file Streamlit application (`src/dashboard.py`) that:
1. Reuses the existing `GrowthIndicatorRegistry` and `FredIndicator` classes
2. Uses Streamlit's native async support (Streamlit handles async automatically)
3. Follows Streamlit's component model with sidebar for controls and main area for visualization
4. Implements proper error handling with user-friendly messages
5. Leverages the existing `FredIndicator.visualise_series()` method for charts

The implementation will be minimal and focused, avoiding over-engineering while providing a solid foundation for future enhancements.

---

## Phase 1: Core Dashboard Structure

### Overview
Create the basic Streamlit application structure with sidebar controls and main visualization area. This phase establishes the UI framework without backend integration.

### Changes Required:

#### 1.1 Create Dashboard Application File

**File**: `src/dashboard.py`
**Changes**: Create new file with Streamlit page configuration and layout structure

```python
import streamlit as st
from dotenv import load_dotenv

# Load environment variables for FRED API key
load_dotenv()

# Page configuration
st.set_page_config(
    page_title="Macro Indicators Dashboard",
    page_icon="📊",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Header
st.title("📊 Macro Economic Indicators Dashboard")
st.markdown("Select an indicator and date range to visualize FRED data")

# Sidebar for controls
st.sidebar.header("Indicator Selection")
st.sidebar.markdown("Choose an economic indicator and configure the date range")

# Main content area placeholder
st.info("Select an indicator from the sidebar and click 'Fetch Data' to begin")
```

**Rationale**:
- Uses `wide` layout to maximize chart visibility
- Loads `.env` early to ensure FRED API key is available
- Establishes clear separation between controls (sidebar) and content (main area)

#### 1.2 Update justfile for Dashboard Command

**File**: `justfile`
**Changes**: Add new command to run Streamlit dashboard

```makefile
run:
    @ PYTHONASYNCIODEBUG=1 && uv run src/main.py

dashboard:
    @ uv run streamlit run src/dashboard.py
```

**Rationale**: Provides convenient command for developers to launch dashboard

### Success Criteria:

#### Automated Verification:
- [ ] Dashboard file exists: `test -f src/dashboard.py`
- [ ] Streamlit imports successfully: `python -c "import streamlit; from src.dashboard import *"`
- [ ] No syntax errors: `python -m py_compile src/dashboard.py`

#### Manual Verification:
- [ ] Dashboard launches without errors: `just dashboard`
- [ ] Page loads at `http://localhost:8501`
- [ ] Title and header display correctly
- [ ] Sidebar is visible and expanded by default
- [ ] Info message appears in main area

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the dashboard loads correctly before proceeding to Phase 2.

---

## Phase 2: Indicator Selection and Date Controls

### Overview
Add UI controls for selecting indicators and configuring date ranges. Integrate with the existing `GrowthIndicatorRegistry` to populate the dropdown with available indicators.

### Changes Required:

#### 2.1 Add Indicator Registry Integration

**File**: `src/dashboard.py`
**Changes**: Import and initialize the growth indicator registry

```python
import streamlit as st
from datetime import datetime, timedelta
from dotenv import load_dotenv

from src.indicators.fred_indicator import FredIndicator
from src.indicators.growth import (
    INDICATOR_NAME_CODE_MAPPING,
    GrowthIndicatorName,
    GrowthIndicatorRegistry,
)

# Load environment variables
load_dotenv()

# Page configuration
st.set_page_config(
    page_title="Macro Indicators Dashboard",
    page_icon="📊",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Initialize registry
@st.cache_resource
def get_indicator_registry():
    """Initialize and cache the indicator registry"""
    registry = GrowthIndicatorRegistry()

    # Register all growth indicators
    for name, fred_code in INDICATOR_NAME_CODE_MAPPING.items():
        registry.register_indicator(
            indicator_name=name,
            indicator=FredIndicator(fred_code)
        )

    return registry

registry = get_indicator_registry()
```

**Rationale**:
- `@st.cache_resource` ensures registry is created once and reused across reruns
- Dynamically registers all indicators from the mapping, making it easy to add more

#### 2.2 Add Sidebar Controls

**File**: `src/dashboard.py`
**Changes**: Add indicator dropdown and date range pickers in sidebar

```python
# Header
st.title("📊 Macro Economic Indicators Dashboard")
st.markdown("Select an indicator and date range to visualize FRED data")

# Sidebar controls
st.sidebar.header("Indicator Selection")

# Get available indicators
available_indicators = registry.get_registered_indicators()

# Indicator dropdown
selected_indicator = st.sidebar.selectbox(
    "Choose Indicator",
    options=available_indicators,
    help="Select a macro economic indicator to visualize"
)

# Date range controls
st.sidebar.markdown("---")
st.sidebar.subheader("Date Range")

# Default to 5 years of data
default_start = datetime.now() - timedelta(weeks=52 * 5)

start_date = st.sidebar.date_input(
    "Start Date",
    value=default_start,
    max_value=datetime.now(),
    help="Select the start date for the time series"
)

st.sidebar.markdown("---")

# Fetch button
fetch_button = st.sidebar.button(
    "📈 Fetch Data",
    type="primary",
    use_container_width=True
)

# Main content area
if not fetch_button:
    st.info("Select an indicator from the sidebar and click 'Fetch Data' to begin")
```

**Rationale**:
- Dropdown populates from registry, ensuring UI stays in sync with available indicators
- Default 5-year lookback matches the existing `FredIndicator` behavior
- Max date validation prevents users from selecting future dates
- Primary button styling makes the action clear

### Success Criteria:

#### Automated Verification:
- [ ] Dashboard file imports successfully: `python -c "from src.dashboard import *"`
- [ ] No Python syntax errors: `python -m py_compile src/dashboard.py`
- [ ] Registry initialization works: `python -c "from src.indicators.growth import GrowthIndicatorRegistry; r = GrowthIndicatorRegistry()"`

#### Manual Verification:
- [ ] Dropdown shows exactly 2 indicators: "FRED Manufacturing PMI (US)" and "Nonfarm Payrolls (US)"
- [ ] Date picker defaults to 5 years ago
- [ ] Date picker prevents selecting future dates
- [ ] "Fetch Data" button appears as primary (blue) button
- [ ] Info message persists until button is clicked
- [ ] No errors in Streamlit logs when interacting with controls

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the controls work correctly before proceeding to Phase 3.

---

## Phase 3: Data Fetching and Visualization

### Overview
Implement the data fetching logic when the user clicks "Fetch Data" and display the results with an interactive Plotly chart and summary statistics table.

### Changes Required:

#### 3.1 Add Async Data Fetching Logic

**File**: `src/dashboard.py`
**Changes**: Add data fetching with error handling when fetch button is clicked

```python
import asyncio
import streamlit as st
import polars as pl
from datetime import datetime, timedelta
from dotenv import load_dotenv

from src.indicators.fred_indicator import FredIndicator
from src.indicators.growth import (
    INDICATOR_NAME_CODE_MAPPING,
    GrowthIndicatorName,
    GrowthIndicatorRegistry,
)

# ... (previous code remains the same)

# Fetch button
fetch_button = st.sidebar.button(
    "📈 Fetch Data",
    type="primary",
    use_container_width=True
)

# Main content area
if fetch_button:
    # Convert selected indicator string back to enum
    indicator_name = None
    for name in GrowthIndicatorName:
        if name.value == selected_indicator:
            indicator_name = name
            break

    if indicator_name is None:
        st.error(f"Invalid indicator selected: {selected_indicator}")
        st.stop()

    # Format date for FRED API
    observation_start = start_date.strftime("%Y-%m-%d")

    # Fetch data with loading indicator
    with st.spinner(f"Fetching {selected_indicator} data from FRED..."):
        try:
            # Get the indicator from registry
            indicator = registry._indicator_registry[indicator_name]

            # Fetch data (Streamlit handles async automatically)
            data = asyncio.run(indicator.get_series(observation_start=observation_start))

            if data.is_empty():
                st.warning("No data available for the selected date range")
                st.stop()

            # Success - store in session state
            st.session_state.data = data
            st.session_state.indicator_name = selected_indicator
            st.session_state.start_date = observation_start

        except Exception as e:
            st.error(f"Error fetching data: {str(e)}")
            st.error("Please check your FRED_API_KEY in the .env file")
            st.stop()

else:
    if not fetch_button:
        st.info("Select an indicator from the sidebar and click 'Fetch Data' to begin")
        st.stop()
```

**Rationale**:
- Uses `asyncio.run()` to handle async `get_series()` call
- Stores data in `st.session_state` to persist across reruns
- Provides helpful error messages for API failures
- Shows loading spinner during data fetch for better UX

#### 3.2 Add Visualization Components

**File**: `src/dashboard.py`
**Changes**: Add chart and summary statistics display after successful data fetch

```python
# ... (previous data fetching code)

# Display results if data exists
if 'data' in st.session_state:
    data = st.session_state.data
    indicator_name = st.session_state.indicator_name
    start_date = st.session_state.start_date

    # Display metadata
    st.subheader(f"📊 {indicator_name}")
    col1, col2, col3 = st.columns(3)

    with col1:
        st.metric("Start Date", start_date)
    with col2:
        latest_date = data.select(pl.col("date").max()).item()
        st.metric("Latest Date", latest_date.strftime("%Y-%m-%d") if latest_date else "N/A")
    with col3:
        st.metric("Data Points", f"{len(data):,}")

    # Create and display chart using existing visualise_series method
    st.markdown("### Time Series Chart")

    # Get the indicator to use its visualise_series method
    indicator_enum = None
    for name in GrowthIndicatorName:
        if name.value == indicator_name:
            indicator_enum = name
            break

    if indicator_enum:
        indicator_obj = registry._indicator_registry[indicator_enum]
        fig = indicator_obj.visualise_series(data)

        # Customize the figure
        fig.update_layout(
            title=f"{indicator_name} Over Time",
            height=500,
            hovermode='x unified'
        )

        st.plotly_chart(fig, use_container_width=True)

    # Summary statistics
    st.markdown("### Summary Statistics")

    # Calculate statistics
    stats = data.select([
        pl.col("value").min().alias("Minimum"),
        pl.col("value").max().alias("Maximum"),
        pl.col("value").mean().alias("Mean"),
        pl.col("value").median().alias("Median"),
        pl.col("value").std().alias("Std Dev"),
    ])

    # Get latest value
    latest_value = data.select(pl.col("value").tail(1)).item()

    # Display in columns
    stat_cols = st.columns(6)
    stat_names = ["Minimum", "Maximum", "Mean", "Median", "Std Dev", "Latest"]
    stat_values = list(stats.row(0)) + [latest_value]

    for col, name, value in zip(stat_cols, stat_names, stat_values):
        with col:
            st.metric(name, f"{value:.2f}")

    # Display raw data in expander
    with st.expander("📋 View Raw Data"):
        st.dataframe(
            data,
            use_container_width=True,
            height=300
        )
```

**Rationale**:
- Reuses existing `FredIndicator.visualise_series()` method for consistency
- Uses Streamlit's native `st.plotly_chart()` for full interactivity
- Displays key metadata (date range, data points) prominently
- Summary statistics use Polars' efficient operations
- Raw data hidden in expander to avoid cluttering the main view

### Success Criteria:

#### Automated Verification:
- [ ] Dashboard file compiles without errors: `python -m py_compile src/dashboard.py`
- [ ] All imports resolve correctly: `python -c "from src.dashboard import *"`
- [ ] Environment file exists: `test -f .env`
- [ ] FRED API key is set: `python -c "from dotenv import load_dotenv; import os; load_dotenv(); assert os.getenv('FRED_API_KEY')"`

#### Manual Verification:
- [ ] Select "FRED Manufacturing PMI (US)" and click "Fetch Data"
- [ ] Loading spinner appears during data fetch
- [ ] Chart displays with correct indicator name in title
- [ ] Chart is interactive (zoom, pan, hover tooltips work)
- [ ] Summary statistics show 6 metrics (Min, Max, Mean, Median, Std Dev, Latest)
- [ ] Raw data table appears in expander when clicked
- [ ] Select different date range (e.g., 2022-01-01) and verify chart updates
- [ ] Switch to "Nonfarm Payrolls (US)" and verify different data displays
- [ ] Verify error message appears if FRED_API_KEY is missing from .env

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation that the visualization works correctly and all features are functioning as expected.

---

## Testing Strategy

### Unit Tests:
Not included in MVP - existing `FredIndicator` and `GrowthIndicatorRegistry` classes already have implicit testing through `src/main.py`. Dashboard is primarily UI code that requires manual testing.

### Integration Tests:
- Verify dashboard launches: `just dashboard`
- Verify FRED API connection works with real API key
- Test all indicator selections with different date ranges

### Manual Testing Steps:
1. **Basic Flow**:
   - Launch dashboard: `just dashboard`
   - Select "FRED Manufacturing PMI (US)"
   - Keep default 5-year date range
   - Click "Fetch Data"
   - Verify chart displays with data from ~2020 to present
   - Verify summary statistics are reasonable (no NaN values)

2. **Date Range Validation**:
   - Set start date to 2022-01-01
   - Click "Fetch Data"
   - Verify chart only shows data from 2022 onwards
   - Verify data point count is lower than 5-year default

3. **Indicator Switching**:
   - Select "Nonfarm Payrolls (US)"
   - Click "Fetch Data"
   - Verify completely different data displays
   - Verify chart title updates to new indicator name

4. **Error Handling**:
   - Temporarily rename `.env` file
   - Restart dashboard
   - Click "Fetch Data"
   - Verify error message about FRED_API_KEY appears
   - Restore `.env` file

5. **Session Persistence**:
   - Fetch data for an indicator
   - Change sidebar controls without clicking "Fetch Data"
   - Verify previous chart remains visible
   - Click "Fetch Data" to verify new data replaces old

6. **UI Responsiveness**:
   - Verify chart is interactive (zoom in/out works)
   - Verify hover tooltips show date and value
   - Verify raw data table scrolls properly
   - Test on different browser window sizes

## Performance Considerations

- **FRED API Rate Limits**: FRED allows 120 requests per minute with standard API key. For MVP with 2 indicators and manual fetching, this is more than sufficient.
- **Data Size**: 5 years of daily data is ~1,825 data points, easily handled by Plotly and Polars
- **Caching**: `@st.cache_resource` used for registry initialization prevents recreating objects on every rerun
- **No caching for data**: Each fetch hits FRED API directly - acceptable for MVP, can add `@st.cache_data` in future if needed
- **Async Operations**: While async infrastructure exists, with only 1 indicator fetched at a time, the benefit is minimal. Architecture supports future multi-indicator fetching.

## Migration Notes

Not applicable - this is a new feature with no existing dashboard to migrate from. The dashboard integrates with existing indicator infrastructure without modifying it.

## References

- Research document: `thoughts/searchable/shared/research/2025-12-23-streamlit-dashboard-codebase-research.md`
- Indicator protocol: `src/indicators/protocol.py:6`
- FRED indicator implementation: `src/indicators/fred_indicator.py:10`
- Growth registry: `src/indicators/growth.py:19`
- Main entry point (CLI): `src/main.py:13`
- Streamlit documentation: https://docs.streamlit.io/
- FRED API documentation: https://fred.stlouisfed.org/docs/api/
