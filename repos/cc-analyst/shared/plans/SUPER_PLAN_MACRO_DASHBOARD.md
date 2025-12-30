# Super Plan: Macro Investing Dashboard - Vertical Slice Implementation

## Overview

This super plan outlines the phased implementation of a systematic macro investing dashboard. The approach prioritizes a **vertical slice** - completing the full feature stack (indicators → calculated fields → composite scores → regime detection → dashboard) for a subset of categories before expanding horizontally.

### Target Scope

**Categories (5 of 10)**:
1. Growth (partially implemented)
2. Inflation (partially implemented)
3. Interest Rates & Yield Curve (not started)
4. Equity Markets & Valuation (not started)
5. Monetary Policy & Liquidity (not started)

**Features**:
- Core calculated fields that feed composite scores
- Category-level composite scores (0-100)
- Overall macro regime indicator (Goldilocks/Reflation/Stagflation/Risk-Off)
- Dashboard overview with regime, scores, and sparklines
- Detailed analysis pages per category

**Constraints**:
- FRED data sources only (free, no API keys beyond existing)
- Local deployment
- Interactive manual analysis (no automation/alerts)
- Solo developer

---

## Phase 1: Foundation Hardening

### Objective
Solidify the existing codebase architecture to support composite scoring and regime detection. Ensure patterns are extensible and well-documented.

### Research & Planning Focus
- Review existing `CalculatedIndicator` implementation for multi-source calculations
- Identify any architectural gaps needed for:
  - Score calculation (indicators → normalized score)
  - Score aggregation (category scores → regime)
  - Time-series score storage (for sparklines)
- Determine if new base classes or protocols are needed
- Review existing test coverage and identify gaps

### Deliverables
- Architectural decision document for scoring system
- Any necessary refactoring of existing indicator infrastructure
- Pattern documentation for adding new categories
- Verification that existing growth/inflation indicators still work

### Success Criteria
- `just check` passes
- Existing dashboard functionality unchanged
- Clear extension points documented

---

## Phase 2: Complete Growth Category

### Objective
Finish the Growth indicator category with all FRED-available indicators and core calculated fields needed for the Growth composite score.

### Research & Planning Focus
- Review MACRO_DATA_SPECIFICATION.md Section 1 (Growth Indicators)
- Identify which of the 12 specified indicators are available in FRED
- Determine which calculated fields are required for Growth Score:
  - LEI 6-Month Growth Rate
  - PMI 3-Month Moving Average
  - Initial Claims 4-Week Average
  - Claims Deviation from 52-Week Low
- Design the Growth Score formula (weighted combination of indicators)
- Identify threshold values for growth rising/falling classification

### Deliverables
- All FRED-available growth indicators registered
- Core calculated fields implemented
- Growth Score calculation function
- Growth Score added to registry system

### Success Criteria
- `just check` passes
- Growth category shows all indicators in UI
- Growth Score calculates and displays correctly
- Score value between 0-100 with clear interpretation

---

## Phase 3: Complete Inflation Category

### Objective
Finish the Inflation indicator category with remaining FRED indicators and calculated fields needed for the Inflation composite score.

### Research & Planning Focus
- Review MACRO_DATA_SPECIFICATION.md Section 2 (Inflation Indicators)
- Identify remaining FRED-available indicators not yet implemented
- Determine which calculated fields are required for Inflation Score:
  - Supercore Services Inflation (if FRED-available)
  - Breakeven Inflation Spread (10Y vs 2Y)
  - Wage Growth Momentum
- Design the Inflation Score formula
- Identify threshold values for inflation rising/falling classification

### Deliverables
- All FRED-available inflation indicators registered
- Remaining core calculated fields implemented
- Inflation Score calculation function
- Inflation Score added to registry system

### Success Criteria
- `just check` passes
- Inflation category shows all indicators in UI
- Inflation Score calculates and displays correctly
- Score value between 0-100 with clear interpretation

---

## Phase 4: Interest Rates & Yield Curve Category

### Objective
Implement the Interest Rates category from scratch, including yield curve spread calculations critical for regime detection.

### Research & Planning Focus
- Review MACRO_DATA_SPECIFICATION.md Section 3 (Interest Rates)
- Identify all FRED codes for Treasury yields (2Y, 5Y, 10Y, 30Y, 3M)
- Identify TIPS yields for real rate calculations
- Design calculated fields:
  - 10Y-2Y Yield Curve Spread
  - 10Y-3M Yield Curve Spread
  - Real Interest Rate (10Y TIPS)
  - Yield Curve Steepening/Flattening Rate
- Consider how yield curve inversion signals feed into regime detection

### Deliverables
- New `src/indicators/interest_rates/` category directory
- All FRED yield indicators registered
- Yield curve spread calculations implemented
- Real rate calculations implemented
- Category registered in RegistryManager

### Success Criteria
- `just check` passes
- Interest Rates category appears in UI navigation
- Yield curve spreads calculate correctly
- Historical yield curve chart displays properly

---

## Phase 5: Equity Markets Category

### Objective
Implement the Equity Markets category with FRED-available valuation and volatility indicators.

### Research & Planning Focus
- Review MACRO_DATA_SPECIFICATION.md Section 6 (Equity Markets)
- Identify FRED-available indicators (note: many equity indicators may not be in FRED)
- Key FRED series to investigate:
  - S&P 500 index or proxy
  - VIX (CBOE Volatility Index)
  - Shiller CAPE ratio
  - Corporate earnings data
- Design calculated fields:
  - CAPE Ratio Percentile (historical ranking)
  - VIX Regime Indicator (low/normal/elevated/extreme)
  - Equity Risk Premium estimate
- Design Valuation Score formula

### Deliverables
- New `src/indicators/equity_markets/` category directory
- All FRED-available equity indicators registered
- Valuation-related calculated fields implemented
- Valuation Score calculation function
- Category registered in RegistryManager

### Success Criteria
- `just check` passes
- Equity Markets category appears in UI navigation
- VIX regime classification works correctly
- Valuation Score calculates between 0-100

---

## Phase 6: Monetary Policy & Liquidity Category

### Objective
Implement the Liquidity category with Fed balance sheet and money supply indicators.

### Research & Planning Focus
- Review MACRO_DATA_SPECIFICATION.md Section 4 (Monetary Policy & Liquidity)
- Identify FRED codes for:
  - Fed Total Assets (balance sheet)
  - M2 Money Supply
  - Bank Reserves
  - Reverse Repo facility
- Design calculated fields:
  - Fed Balance Sheet Rate of Change
  - M2 Growth Rate
  - Reverse Repo as % of Fed Balance Sheet
- Design Liquidity Score formula

### Deliverables
- New `src/indicators/liquidity/` category directory
- All FRED-available liquidity indicators registered
- Rate of change calculations implemented
- Liquidity Score calculation function
- Category registered in RegistryManager

### Success Criteria
- `just check` passes
- Liquidity category appears in UI navigation
- Balance sheet changes calculate correctly
- Liquidity Score calculates between 0-100

---

## Phase 7: Composite Score Infrastructure

### Objective
Create the infrastructure for calculating, storing, and displaying composite scores across all implemented categories.

### Research & Planning Focus
- Design the `CompositeScore` abstraction:
  - Input: multiple indicator values
  - Output: normalized 0-100 score
  - Method: weighted average with configurable weights
- Determine score normalization approach:
  - Z-score based (standard deviations from mean)
  - Percentile rank based (historical distribution)
  - Threshold based (predefined ranges)
- Design score registry/manager pattern
- Consider how scores update when underlying data refreshes

### Deliverables
- `CompositeScore` protocol or base class
- Score calculation implementations for all 5 categories:
  - Growth Score
  - Inflation Score
  - Interest Rate Score (or Yield Curve Score)
  - Valuation Score
  - Liquidity Score
- Score registry that parallels indicator registry
- API for retrieving current scores

### Success Criteria
- `just check` passes
- All 5 composite scores calculate without error
- Scores respond appropriately to indicator changes
- Score values are interpretable (documented meaning of ranges)

---

## Phase 8: Regime Detection Engine

### Objective
Implement the 4-quadrant macro regime classifier based on growth and inflation trends.

### Research & Planning Focus
- Review MACRO_INVESTING_FRAMEWORK.md regime definitions:
  - **Goldilocks**: Growth rising, Inflation falling
  - **Reflation**: Growth rising, Inflation rising
  - **Stagflation**: Growth falling, Inflation rising
  - **Risk-Off**: Growth falling, Inflation falling
- Design trend detection logic:
  - How to determine "rising" vs "falling"
  - Lookback period for trend calculation
  - Smoothing to avoid whipsaw
- Consider edge cases and transition logic
- Design regime output format and storage

### Deliverables
- `RegimeDetector` class or module
- Growth trend classifier (rising/falling/neutral)
- Inflation trend classifier (rising/falling/neutral)
- 4-quadrant regime mapper
- Current regime API endpoint
- Historical regime tracking (for future timeline view)

### Success Criteria
- `just check` passes
- Regime classifies correctly based on current data
- Regime label is one of: Goldilocks, Reflation, Stagflation, Risk-Off
- Classification logic is documented and testable

---

## Phase 9: Dashboard Overview Page

### Objective
Create the main dashboard view showing current regime, composite scores, and key indicator sparklines.

### Research & Planning Focus
- Design dashboard layout:
  - Prominent regime indicator (top of page)
  - Composite score cards (5 scores with gauges or progress bars)
  - Key indicator sparklines (mini time-series charts)
- Identify which indicators to show as sparklines:
  - 1-2 key indicators per category
  - Most actionable/frequently watched
- Design score card component:
  - Score value (0-100)
  - Score interpretation (e.g., "Expansionary", "Neutral", "Contractionary")
  - Trend arrow (up/down/flat)
- Consider data refresh and loading states

### Deliverables
- New dashboard overview page (`src/pages/dashboard.py`)
- Regime indicator component
- Composite score card component
- Sparkline chart component
- Dashboard registered as home/default page
- Original home page repurposed or removed

### Success Criteria
- `just check` passes
- Dashboard loads without error
- Regime displays current classification
- All 5 composite scores display with values
- Sparklines render for key indicators
- Page is responsive and performant

---

## Phase 10: Category Deep-Dive Pages

### Objective
Enhance the indicator explorer to support detailed category analysis with all indicators, calculated fields, and category-specific context.

### Research & Planning Focus
- Design category landing page layout:
  - Category composite score prominently displayed
  - All indicators in category listed with current values
  - Calculated fields shown with formulas/explanations
  - Historical chart for selected indicator
- Consider navigation flow:
  - Dashboard → Category → Individual Indicator
- Design indicator detail view:
  - Full historical chart
  - Key statistics (current, high, low, average)
  - Related indicators
- Determine if existing Indicator Explorer can be enhanced or needs replacement

### Deliverables
- Enhanced category view in indicator explorer
- Category header showing composite score
- Indicator list with current values and mini-trends
- Improved indicator detail view
- Navigation breadcrumbs

### Success Criteria
- `just check` passes
- Each category has a dedicated analysis view
- Category score displays on category page
- All indicators in category are accessible
- Navigation between dashboard and categories is intuitive

---

## Phase 11: Polish & Documentation

### Objective
Final polish pass on UI/UX, error handling, and documentation for maintainability.

### Research & Planning Focus
- Identify UI inconsistencies across pages
- Review error handling for:
  - FRED API failures
  - Missing data series
  - Calculation errors
- Document:
  - How to add new indicators
  - How to add new categories
  - How composite scores are calculated
  - How regime detection works
- Consider adding loading states and user feedback

### Deliverables
- Consistent styling across all pages
- Graceful error handling with user-friendly messages
- Updated CLAUDE.md with new architecture details
- Developer documentation for extending the system
- Any necessary code cleanup/refactoring

### Success Criteria
- `just check` passes
- No unhandled exceptions in normal usage
- FRED API failures display helpful error messages
- Documentation enables future development
- Code is clean and follows established patterns

---

## Dependencies & Sequencing

```
Phase 1 (Foundation)
    │
    ├─► Phase 2 (Growth) ─────────────────────┐
    │                                          │
    ├─► Phase 3 (Inflation) ──────────────────┤
    │                                          │
    ├─► Phase 4 (Interest Rates) ─────────────┼─► Phase 7 (Composite Scores)
    │                                          │           │
    ├─► Phase 5 (Equity Markets) ─────────────┤           │
    │                                          │           ▼
    └─► Phase 6 (Liquidity) ──────────────────┘   Phase 8 (Regime Detection)
                                                           │
                                                           ▼
                                                  Phase 9 (Dashboard)
                                                           │
                                                           ▼
                                                  Phase 10 (Deep-Dive Pages)
                                                           │
                                                           ▼
                                                  Phase 11 (Polish)
```

**Notes on Parallelization**:
- Phases 2-6 (category implementations) can be done in any order after Phase 1
- Phase 7 requires at least some category implementations to be useful
- Phase 8 requires Growth and Inflation scores from Phase 7
- Phases 9-11 are sequential and build on all previous work

---

## Risk Considerations

### Data Availability
- Some indicators in the spec may not be available in FRED
- Mitigation: During each phase's research, validate FRED codes exist and return expected data

### Calculation Complexity
- Some calculated fields have complex formulas (percentile ranks, z-scores)
- Mitigation: Start with simpler calculations, add complexity incrementally

### Score Calibration
- Composite scores need meaningful interpretation
- Mitigation: Use historical data to calibrate score ranges during Phase 7

### UI Performance
- Dashboard with multiple charts may be slow
- Mitigation: Leverage existing caching, consider lazy loading for sparklines

---

## Success Metrics for Complete Implementation

1. **Data Coverage**: 5 categories fully implemented with all FRED-available indicators
2. **Calculated Fields**: Core fields for each category that feed composite scores
3. **Composite Scores**: 5 category scores calculating correctly (0-100 scale)
4. **Regime Detection**: Current regime displays accurately based on growth/inflation trends
5. **Dashboard**: Single view showing regime + scores + sparklines
6. **Navigation**: Intuitive flow from overview to detailed analysis
7. **Code Quality**: All checks pass, patterns documented, extensible architecture
