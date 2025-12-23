---
date: 2025-12-23T15:39:10+0000
researcher: Claude Sonnet 4.5
git_commit: No commits yet (repository not initialized)
branch: main
repository: cc-analyst
topic: "Streamlit Dashboard for Macro Indicators - Codebase Research"
tags: [research, codebase, streamlit, dashboard, macro-indicators, fred-api]
status: complete
last_updated: 2025-12-23
last_updated_by: Claude Sonnet 4.5
---

# Research: Streamlit Dashboard for Macro Indicators - Codebase Research

**Date**: 2025-12-23T15:39:10+0000
**Researcher**: Claude Sonnet 4.5
**Git Commit**: No commits yet (repository not initialized)
**Branch**: main
**Repository**: cc-analyst

## Research Question

Understand the existing codebase to design and implement a configurable Streamlit dashboard where users can select macro economic indicators from a dropdown and filter the dashboard to display the desired indicator with visualizations.

## Summary

The cc-analyst project is a macro investing analysis system built with Python 3.11+, focused on tracking and visualizing 20-30 core economic indicators. The project has well-defined specifications but minimal implementation so far. Currently, there is:

- A protocol-based architecture for indicators using Python's `Protocol` type
- A working `FredIndicator` class that fetches data from the Federal Reserve Economic Data (FRED) API
- A registry pattern for managing growth indicators
- Two implemented growth indicators (PMI and Nonfarm Payrolls)
- Comprehensive markdown specifications for macro indicators and investing frameworks
- **No Streamlit dashboard implementation yet** - this needs to be built from scratch

The foundation is solid for building a configurable dashboard, with clear separation of concerns between data fetching, indicator management, and visualization.

## Detailed Findings

### Project Structure

```
cc-analyst/
├── src/
│   ├── indicators/
│   │   ├── __init__.py
│   │   ├── protocol.py          # Defines Indicator protocol
│   │   ├── fred_indicator.py     # FRED API integration
│   │   └── growth.py             # Growth indicator registry
│   └── main.py                   # Entry point (CLI demo)
├── MACRO_DATA_SPECIFICATION.md   # Comprehensive indicator specs
├── MACRO_INVESTING_FRAMEWORK.md  # Investment decision framework
├── pyproject.toml                # Project dependencies
├── justfile                      # Task runner (just run)
└── .env                          # Environment variables (FRED API key)
```

### Core Architecture

#### 1. Indicator Protocol (`src/indicators/protocol.py:6`)

The codebase uses a protocol-based design pattern:

```python
class Indicator(Protocol):
    async def get_series(
        self,
        observation_start: str | None = None,
    ) -> pl.DataFrame: ...
```

This protocol defines the contract that all indicators must implement, enabling polymorphism and extensibility.

#### 2. FredIndicator Implementation (`src/indicators/fred_indicator.py:10`)

The `FredIndicator` class is the primary data fetcher:

- **Location**: `src/indicators/fred_indicator.py:10`
- **FRED API Integration**: Uses `pyfredapi` library to fetch data (`src/indicators/fred_indicator.py:14-21`)
- **Async Data Fetching**: Wraps synchronous FRED API calls in `asyncio.to_thread()` for non-blocking operations
- **Default Time Window**: Fetches last 5 years of data by default (`src/indicators/fred_indicator.py:37-39`)
- **Data Format**: Returns `polars.DataFrame` with columns: `date`, `value`
- **Visualization Method**: Includes `visualise_series()` method using Plotly (`src/indicators/fred_indicator.py:27-33`)

Key methods:
- `get_series()`: Fetches time series data from FRED
- `visualise_series()`: Creates a Plotly line chart (returns `go.Figure`)
- `set_default_observation_start()`: Handles date validation and defaults

#### 3. Growth Indicator Registry (`src/indicators/growth.py:19`)

Implements a registry pattern for managing multiple indicators:

- **Location**: `src/indicators/growth.py:19`
- **Pattern**: Registry pattern with fluent interface
- **Current Indicators**:
  - `GrowthIndicatorName.PMI_US` → FRED code `MANEMP` (Manufacturing PMI)
  - `GrowthIndicatorName.PAYEMS` → FRED code `PAYEMS` (Nonfarm Payrolls)
- **Mapping**: `INDICATOR_NAME_CODE_MAPPING` dictionary links friendly names to FRED codes (`src/indicators/growth.py:13-16`)

Key methods:
- `register_indicator()`: Adds indicator to registry (fluent interface, returns self)
- `get_registered_indicators()`: Returns list of registered indicator names
- `get_indicator_series()`: Fetches data for a specific indicator

#### 4. Main Entry Point (`src/main.py:13`)

Current implementation is a simple CLI demonstration:

- Creates a `GrowthIndicatorRegistry`
- Registers two indicators (PMI and PAYEMS)
- Fetches and prints data using `describe()` method
- Loads environment variables from `.env` file for FRED API key

**No Streamlit dashboard exists yet** - the main.py file is purely for testing.

### Technology Stack

#### Dependencies (from `pyproject.toml:7`)

| Package | Version | Purpose |
|---------|---------|---------|
| `streamlit` | ≥1.52.2 | Web dashboard framework (not yet used) |
| `plotly` | ≥6.5.0 | Interactive visualizations |
| `polars` | ≥1.36.1 | Fast DataFrame library (alternative to pandas) |
| `pyfredapi` | ≥0.10.2 | FRED API client with Polars support |
| `python-dotenv` | ≥1.2.1 | Environment variable management |

#### Why These Choices?

- **Polars over Pandas**: Modern, faster DataFrame library with better memory efficiency
- **pyfredapi with Polars**: Integrated FRED API client that directly returns Polars DataFrames
- **Plotly**: Already used in `FredIndicator.visualise_series()`, interactive charts work well with Streamlit
- **Streamlit**: Rapid dashboard development with native support for Plotly

### Data Specifications

The project includes two comprehensive specification documents that define the macro investing framework:

#### MACRO_DATA_SPECIFICATION.md

Defines 20-30 core indicators across 11 categories:

1. **Growth Indicators** (12 indicators)
   - Conference Board LEI, ISM PMI, Jobless Claims, Housing Starts, etc.
   - Each has FRED codes, release schedules, and calculated fields

2. **Inflation Indicators** (11 indicators)
   - Core PCE, CPI, PPI, Employment Cost Index, Breakeven Inflation
   - Includes wage growth and supercore services inflation

3. **Interest Rates & Yield Curve** (9 indicators)
   - Treasury yields (2Y, 5Y, 10Y, 30Y), TIPS, international bonds
   - Yield curve spreads and term premium calculations

4. **Monetary Policy & Liquidity** (7 indicators)
   - Fed Balance Sheet, RRP, M2, Bank Reserves
   - Global liquidity index calculations

5. **Credit Markets & Spreads** (8 indicators)
   - High Yield spreads, Investment Grade, BBB/AAA corporates, TED spread
   - Financial Conditions Index

6. **Equity Markets & Valuation** (9 indicators)
   - S&P 500, Forward P/E, CAPE, VIX, Put/Call ratios

7. **Sentiment & Positioning** (6 indicators)
   - AAII Sentiment, CNN Fear & Greed, CFTC COT, Fund Flows

8. **Commodities** (7 indicators)
   - Gold, Oil (WTI/Brent), Copper, Bloomberg Commodity Index

9. **Foreign Exchange** (6 indicators)
   - DXY, EUR/USD, GBP/USD, USD/JPY, USD/CNY

10. **Geopolitical Risk** (3 indicators)
    - EPU Index, GPR Index, Sovereign CDS

11. **Composite Scores** (5 calculated scores)
    - Growth, Inflation, Liquidity, Valuation, Sentiment scores (0-100)

#### MACRO_INVESTING_FRAMEWORK.md

Provides investment decision frameworks:

- Four-quadrant regime classification (Goldilocks, Reflation, Stagflation, Risk-Off)
- Asset allocation grids by regime
- Decision trees for recession risk, inflation positioning, liquidity regimes
- Critical thresholds and warning levels for each indicator
- Historical case studies (2008 crisis, 2020 COVID, 2022 inflation)

### Configuration & Environment

#### .env File

The project expects a `.env` file with FRED API credentials:
- Location: `/Users/elliotsteene/Documents/claude-finance/cc-analyst/.env`
- Required variable: `FRED_API_KEY` (likely)
- Loaded via `python-dotenv` in `main.py:47`

#### justfile

Simple task runner with one command:
```
run:
    @ PYTHONASYNCIODEBUG=1 && uv run src/main.py
```

Sets async debug mode and runs main.py using `uv` (fast Python package installer).

### What's Missing for the Dashboard

To build the Streamlit dashboard, the following needs to be implemented:

1. **No Streamlit App File**: Need to create `src/dashboard.py` or `streamlit_app.py`

2. **No Multi-Category Support**: Currently only growth indicators exist. Need registries for:
   - Inflation indicators
   - Interest rates
   - Monetary policy
   - Credit markets
   - Equities
   - Sentiment
   - Commodities
   - FX
   - Geopolitical risk

3. **No Calculated Fields**: The specification defines many calculated fields (e.g., LEI 6-month growth, PMI 3-month MA) that aren't implemented

4. **No Data Caching**: Each request fetches from FRED API (could hit rate limits)

5. **No Indicator Metadata**: No descriptions, units, or interpretation guidance stored with indicators

6. **No Composite Scores**: The 5 composite scores (0-100) from the specification aren't implemented

7. **No Multi-Indicator Views**: Dashboard should support comparing multiple indicators

## Architecture Patterns Found

### 1. Protocol-Based Design

The codebase uses Python's `Protocol` type (structural subtyping) rather than abstract base classes. This allows:
- Duck typing with type safety
- Easy extension without inheritance
- Clear contracts between components

### 2. Registry Pattern

`GrowthIndicatorRegistry` implements a registry pattern:
- Centralized indicator management
- Fluent interface (method chaining)
- Type-safe indicator name enumeration

### 3. Async/Await for I/O

FRED API calls use async patterns:
- Non-blocking data fetching
- Potential for concurrent requests
- Better scalability for dashboard with multiple indicators

### 4. Data-First with Polars

Choice of Polars over Pandas indicates:
- Performance-oriented design
- Modern approach to data processing
- Memory efficiency for large time series

## Code References

- `src/indicators/protocol.py:6` - Indicator protocol definition
- `src/indicators/fred_indicator.py:10` - FredIndicator class
- `src/indicators/fred_indicator.py:14` - Async get_series method
- `src/indicators/fred_indicator.py:27` - Plotly visualization method
- `src/indicators/growth.py:8` - GrowthIndicatorName enum
- `src/indicators/growth.py:13` - FRED code mapping
- `src/indicators/growth.py:19` - GrowthIndicatorRegistry class
- `src/main.py:13` - Main entry point
- `pyproject.toml:7` - Dependencies list

## Related Research

(No prior research documents found - this is the first research document for the project)

## Open Questions

1. **API Rate Limits**: What are FRED API rate limits? Need caching strategy?
2. **Data Storage**: Should fetched data be persisted locally (SQLite/CSV) or always fetch fresh?
3. **Update Frequency**: How often should dashboard refresh data? (Daily, on-demand, cached?)
4. **Indicator Categories**: Should we implement all 11 categories immediately or start with a subset?
5. **Calculated Fields**: Should calculations happen at fetch time or display time?
6. **Multi-Indicator Views**: Should dashboard support comparing multiple indicators on one chart?
7. **Date Range Selection**: Should users control observation_start parameter via UI?
8. **Composite Scores**: Implement the 0-100 scoring system from the specification?
