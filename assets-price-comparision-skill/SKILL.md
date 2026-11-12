---
name: assets-price-comparision-skill
description: Compare user-specified assets or market indices with Yahoo Finance daily data, preserving CSV data, producing annotated multi-period charts, and assembling a two-page PDF report. Use when a user asks to compare asset price performance or volatility across time windows.
---

# Assets Price Comparision Skill

Use this skill when the user supplies asset names, Yahoo Finance tickers, or a mixture of both and requests a reproducible price/performance comparison. The deliverable is a self-contained output directory; do not require credentials or upload data anywhere.

## Workflow

1. Resolve each input to a Yahoo Finance ticker. Accept explicit tickers unchanged; support common names such as Bitcoin (`BTC-USD`), Ethereum (`ETH-USD`), S&P 500 (`^GSPC`), Nasdaq-100 (`^NDX`), gold futures (`GC=F`), and Apple (`AAPL`). For an ambiguous name, state the chosen ticker in the output and ask for clarification only when multiple interpretations would materially change the result. Preserve the user’s display name separately from the ticker.
2. Download daily history from Yahoo Finance with `yfinance` (or the project’s existing Yahoo Finance client), using adjusted close for investable securities when available and the close/index level for indices and crypto. Fetch at least one calendar year plus a small buffer, then normalize dates to timezone-naive calendar dates, sort ascending, remove non-positive values, and drop rows without a usable price. Record the retrieval timestamp, source, ticker, and price field in the report or summary CSV.
3. Create the output tree below. Use a fresh run directory when the requested output location is not explicit, so earlier results are not silently overwritten:

   ```text
   asset_analysis/
   ├── csv_data/
   │   ├── <asset>.csv
   │   ├── all_assets_close_prices.csv
   │   ├── normalized_1_week.csv
   │   ├── normalized_1_month.csv
   │   ├── normalized_6_months.csv
   │   └── normalized_1_year.csv
   ├── single_asset_charts/
   ├── comparison_charts/
   └── asset_comparison_report.pdf
   ```

   Each per-asset CSV must contain a `Date` column and the source price column. The combined CSV must retain one column per asset. Also write `summary_statistics.csv` at the output root with one row per asset and period, including ticker, actual first/last dates, observations, start/end prices, absolute change, percentage change, and annualized volatility.
4. Analyze exactly these calendar windows relative to the run date: recent week (7 days), month (1 calendar month), half-year (6 calendar months), and year (1 calendar year). Use the first and last available observations inside each window; do not invent weekend or holiday prices. If a window has fewer than three observations, keep the row with a clear `insufficient_data` status and skip its statistic rather than failing the entire run.
5. Calculate percentage change as `last / first - 1`. Calculate annualized historical volatility from simple daily returns as `std(ddof=1) * sqrt(annualization_days)`, using 365 for assets that trade every day (for example crypto) and 252 for exchange-traded securities and indices. Label this explicitly as annualized volatility; never present it as implied or forward volatility. For comparisons, align assets on common dates before normalizing each series to 100 at the first common observation.
6. Produce four single-asset line charts per successfully analyzed asset (`1_week`, `1_month`, `6_months`, `1_year`). Every chart must include the asset/ticker, actual date range, units, and an in-plot annotation containing start price, end price, percentage change, and annualized volatility. Use readable date ticks, a grid, a legend only when useful, and close figures after saving.
7. Produce four normalized comparison line charts, one for each period. All lines must start at 100 on the same common initial date for that chart; include a dashed 100 baseline and a legend showing each asset’s period-end percentage change. Annotate the chart with the common date range and a note that series are normalized to the first common observation. Do not compare raw BTC, stock, and index levels on one y-axis.
8. Assemble `asset_comparison_report.pdf` with exactly two pages using the generated data and images. Page 1 should contain the run metadata, data coverage, and a compact comparison table for the four periods showing both period-end change and annualized volatility for each asset (for example, `+12.3% / 28.4%`). Page 1 must also include concise methodology/limitations. Page 2 should show the four normalized comparison charts in a readable 2×2 layout plus a short interpretation of the strongest/weakest changes and highest volatility. If there are many assets, keep the table readable by abbreviating values and move detailed rows to `summary_statistics.csv`; do not add a third page.
9. Return the output directory and a short result summary. Mention any ticker that failed, any period with insufficient data, the data source/retrieval time, and that the result is historical analysis rather than investment advice. Do not claim a successful download or report until the files exist and can be opened.

## Implementation guidance

Prefer a one-off Python execution or an existing project runner over adding a new permanent script to the skill. Typical open-source dependencies are `yfinance`, `pandas`, `numpy`, `matplotlib`, and (if convenient for PDF layout) `Pillow`; use Matplotlib’s `PdfPages` or another installed local PDF writer to guarantee the report is exactly two pages. Keep network access limited to Yahoo Finance data retrieval, make failures per ticker non-fatal, and avoid embedding credentials or machine-specific paths.

Before presenting results, verify that:

- every successful ticker has a per-asset CSV and a non-empty price series;
- `summary_statistics.csv` contains all four period labels for each asset with enough data;
- each normalized CSV and comparison chart starts at 100 for every included asset;
- the four single-asset chart types and four comparison chart types are present when data permits;
- the PDF opens and has exactly two pages, with no clipped or overlapping text after rendering each page to PNG;
- the page-one table includes both price change and annualized volatility, and all report prose stays inside the page margins.

The reference notebook `Compare Assets.ipynb` in the project demonstrates the intended data cleaning, fallback ticker, normalization, annotation, and annualization patterns. Improve on it where needed for robust input resolution, provenance, missing-data handling, and the fixed two-page report; do not copy its assumptions blindly.
