# Treasury Vol Dashboard

Treasury Vol Dashboard is a single-page, browser-based dashboard for monitoring realized volatility across the U.S. Treasury yield curve. It fetches the latest official Treasury yield curve observations, converts daily yield changes into basis-point moves, and summarizes current volatility against historical reference periods.

The dashboard is designed to reproduce and extend the volatility framework from the Excel workbook that originally motivated the project, while keeping the calculations transparent and refreshable from the official source.

## Data Source

The dashboard uses the official U.S. Department of the Treasury Daily Treasury Yield Curve XML feed:

`https://home.treasury.gov/resource-center/data-chart-center/interest-rates/pages/xml`

For each calendar year from the selected start year through the current year, the page requests:

`?data=daily_treasury_yield_curve&field_tdr_date_value=<YEAR>`

The XML fields are mapped into the dashboard tenor set:

- `BC_1MONTH` -> `1M`
- `BC_1_5MONTH` -> `1.5M`
- `BC_2MONTH` -> `2M`
- `BC_3MONTH` -> `3M`
- `BC_4MONTH` -> `4M`
- `BC_6MONTH` -> `6M`
- `BC_1YEAR` -> `1Y`
- `BC_2YEAR` -> `2Y`
- `BC_3YEAR` -> `3Y`
- `BC_5YEAR` -> `5Y`
- `BC_7YEAR` -> `7Y`
- `BC_10YEAR` -> `10Y`
- `BC_20YEAR` -> `20Y`
- `BC_30YEAR` -> `30Y`

The page does not rely on a bundled historical CSV. Each launch refreshes from Treasury.gov. If any yearly request fails, the dashboard keeps the successful years and reports the partial failures in the message banner.

## Core Calculation Method

### Daily Basis-Point Moves

For each tenor and date, the dashboard calculates the daily yield move as:

```text
Daily BP Move = (Current Yield - Previous Observation Yield) * 100
```

Treasury yields are quoted in percentage points. Multiplying by `100` converts the difference into basis points. For example, a move from `4.20%` to `4.25%` is `5 bp`.

The first observation has no prior value, so its daily move is blank. Missing tenor observations are also skipped.

### RMS Realized Volatility

The dashboard uses RMS realized volatility:

```text
Daily RMS BP Vol = SQRT(SUMSQ(Daily BP Moves) / COUNT(Daily BP Moves))
```

Then it annualizes the value:

```text
Annualized RMS BP Vol = Daily RMS BP Vol * SQRT(252)
```

The annualization convention is `252` trading days.

### Why RMS Instead of Standard Deviation

The original Excel workbook used formulas like:

```text
SQRT(SUMSQ(range) / COUNT(range))
```

The dashboard follows that same realized-volatility logic. RMS is useful here because it measures the typical magnitude of daily rate moves directly, without first subtracting the sample mean. That makes the measure simple, auditable, and close to a realized-volatility calculation: "how large have daily moves been?"

RMS also works cleanly with sparse Treasury tenors because blank observations are removed before the calculation. The denominator is the actual count of valid daily moves, so missing values do not get treated as zero moves and do not dilute the volatility estimate.

Standard deviation is still a valid statistical measure, but it answers a slightly different question: how dispersed are moves around their average move? For Treasury rate volatility monitoring, the RMS method is better aligned with the Excel target and with the practical question of daily move magnitude.

### Rolling Windows

The dashboard supports three rolling windows:

- `1M`: 21 observations
- `3M`: 63 observations
- `1Y`: 252 observations

For each date, tenor, and selected window, the dashboard calculates annualized RMS BP volatility over the trailing window. A window requires at least `80%` of its observations, with a minimum of `5` valid moves, before a value is displayed.

## Dashboard Sections

### Treasury Yield Curve

The Treasury Yield Curve is an embedded companion chart for the underlying U.S. Treasury term structure. It lets you inspect the latest curve, move across historical dates, and pin prior yield curves for direct comparison.

This section is useful as the level backdrop for the volatility views below. It helps explain whether a volatility change happened during inversion, steepening, front-end repricing, or a long-end selloff.

### Current Volatility Curve

The Current Volatility Curve shows the latest annualized RMS BP volatility by tenor for the selected rolling windows.

Displayed tenors are intentionally reduced for readability:

```text
1M, 3M, 6M, 1Y, 2Y, 3Y, 5Y, 10Y, 20Y, 30Y
```

The x-axis uses a Bloomberg-style exponential front-loaded time scale, `u(t) = t + (8.41 / pi) * (1 - exp(-pi * t))`, where `t` is tenor in years. The extra front-end spacing fades quickly so the long end approaches a linear year scale.

Although the calculation is built from rolling windows, this panel shows only the latest available point from each selected window. In practice, it reads as a current snapshot by lookback length:

- `1M`: latest curve using the trailing `21` observations
- `3M`: latest curve using the trailing `63` observations
- `1Y`: latest curve using the trailing `252` observations

### Treasury Volatility Curve

The Treasury Volatility Curve is an embedded custom comparison tool for realized volatility curves across user-defined date ranges.

The user can:

- choose a `From` date
- choose a `To` date
- choose a rolling window: `1M`, `3M`, or `1Y`
- pin multiple volatility curves for comparison

The default setup uses the last `3M` window. Pinned comparisons keep a common rolling window so that different historical periods can be compared on a like-for-like basis.

This section answers a different question from the Current Volatility Curve. Instead of asking "what does volatility look like right now across different lookback lengths?", it asks "for this chosen historical period, what did the volatility curve look like under one common rolling window?"

### Percentile Comparison

The Percentile Comparison uses the primary rolling window, normally `3M`, and compares the latest volatility level against the historical distribution over the configured lookback period.

The dashboard calculates:

- `P20`
- `P50`
- `P80`
- `P95`
- current percentile
- regime label

Regime labels are assigned as:

- `Low`: percentile at or below `20`
- `Normal`: between `20` and `80`
- `High`: at or above `80`
- `Extreme`: at or above `95`

This section helps answer whether today's volatility is low, normal, high, or extreme relative to its own recent history.

### Volatility References

This chart recreates the spirit of the original Excel volatility comparison. It compares annualized RMS BP volatility across fixed and trailing reference periods.

Fixed reference periods:

- `Last 20 Years`
- `2012-2021 (Low Rate Period)`
- `2022-2023 (High Rate Period)`
- `Last 10 Years`

Trailing reference periods:

- `Last 3 Years`
- `Last Quarter`
- `1Y`
- `3M`
- `1M`

The reference-tenor set is:

```text
3M, 1Y, 2Y, 3Y, 5Y, 7Y, 10Y, 20Y, 30Y
```

This section is useful for comparing current volatility against recognizable rate regimes. For example, it can show whether current 10Y volatility looks closer to the low-rate era or to the more recent higher-volatility regime.

### Volatility History

Volatility History plots the full time series of rolling annualized RMS BP volatility for the selected window.

This section shows persistence and regime shifts. For example, it can identify whether a high current reading is a short spike or part of a sustained volatility regime.

### Treasury Rates Over Time

Treasury Rates Over Time plots the underlying yield levels across selectable date ranges:

- `1M`
- `3M`
- `6M`
- `YTD`
- `1Y`
- `3Y`
- `5Y`
- `All`

This section provides context for volatility. A sharp change in yield levels, inversion, steepening, or bear steepening can explain why a volatility reading changed.

### Regime Table

The Regime Table combines the current primary-window volatility, historical percentile, percentile thresholds, and regime label by tenor.

It is the most compact way to see which parts of the curve are currently high or extreme.

### Latest Official Observations

This table shows the latest Treasury yield observations from the official feed. It is a quick audit trail for the current dashboard state and helps confirm that the page refreshed to the expected date.

## Excel Export

The `Export Excel report` button generates a new workbook from the current dashboard data. It does not export or copy the original Excel file.

The exported workbook includes:

- `Summary`
- `Current Curve`
- `Percentiles`
- `Fixed Regimes`
- `Latest Observations`
- `Yield History`
- `Daily BP Changes`
- `Methodology`

The export is intended to preserve the dashboard's calculated outputs and methodology in a portable workbook, using the latest Treasury data loaded in the browser.

## Configuration

The dashboard configuration is embedded in `index.html`.

Important settings:

```text
annualization_days = 252
1M window = 21 observations
3M window = 63 observations
1Y window = 252 observations
percentile lookback = 20 years
low percentile threshold = 20
high percentile threshold = 80
extreme percentile threshold = 95
outlier threshold = 75 bp
```

The default history start year is controlled by the input at the top of the dashboard. A longer history gives better regime and percentile context but takes longer to fetch.

## Interpreting the Dashboard

Use the dashboard as a realized-volatility monitor for the Treasury curve:

- Rising front-end volatility can indicate policy-rate uncertainty.
- Rising belly volatility can indicate repricing of medium-term inflation or growth expectations.
- Rising long-end volatility can indicate term-premium, fiscal, supply, or duration-risk stress.
- High percentiles show tenors where current volatility is unusual relative to history.
- Volatility increase ratios show how far current conditions are from calmer reference regimes.

The dashboard does not forecast future yields or volatility. It summarizes realized historical moves from official Treasury observations.

## Running the Dashboard

Open `index.html` in a browser. The page will fetch the latest official Treasury data on launch.

An internet connection is required for:

- Treasury XML data
- Plotly chart library
- XLSX export library
- Google Fonts

If the Treasury feed is unavailable or a yearly request fails, the dashboard displays a message banner describing the issue.
