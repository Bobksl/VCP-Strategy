# VCP — Volatility Contraction Pattern Strategy

A pure-technical, **long-only** breakout strategy on a broad US-equity universe (S&P-500 constituents), implemented as a Jupyter notebook. This is a professional-grade quantitative research project designed for systematic backtesting and trade analysis.

---

## Motivation

This project asks a simple question:

> Can a well-defined Volatility Contraction Pattern (VCP) breakout strategy, implemented with strict no-look-ahead rules on S&P 500 constituents, generate attractive risk-adjusted returns relative to a buy-and-hold benchmark?

The notebook is designed as a **research-grade backtest**, emphasising data integrity (CRSP), causal signal generation, and transparent trade logs over curve-fitting.

## Method and Key Results (2020–2024 Sample)

Using split-adjusted CRSP daily data for S&P 500 constituents (identified by PERMNO), the strategy:

- Scans for VCP patterns using an online directional-change algorithm with ATR-based thresholds
- Confirms signals only on causal data (DC confirmation bar)
- Constructs trades with fixed-fractional risk, stop at last contraction low, and exits at min(5R, next resistance)
- Benchmarks against a buy-and-hold ^GSPC strategy

### Backtest Summary (Example Numbers)

| Metric            | VCP Strategy | Buy & Hold (^GSPC) |
|-------------------|-------------:|-------------------:|
| CAGR              | 0.36         | 0.17               |
| Sharpe Ratio      | 1.85         | 0.75               |
| Max Drawdown      | -0.13        | -0.25              |
| Win Rate          | 0.75         | —                  |
| Average Profit    | 3.17         | —                  |

*Walk-forward split at PERMNO level to separate in-sample vs out-of-sample buckets and minimise data leakage.*

## Strategy Philosophy

- **Pure technical, no fundamentals** — only OHLCV price data is used.
- **Lenient VCP definition** — maximises pattern count with a reasonable win rate, rather than hunting for rare perfect Mark Minervini setups.
- **Long-only** with fixed-fractional position sizing.
- **No look-ahead bias is a hard constraint** — all signal generation uses only data that would have been available at the time.
- **Universe:** S&P 500 constituents (identified by CRSP PERMNO keys for robustness against ticker symbol changes).

---

## The VCP Pattern

The Volatility Contraction Pattern identifies stocks that have gone through a period of price consolidation (tightening ranges) on declining volume, followed by a breakout on above-average volume — a classic momentum setup.

### Pattern Detection Steps

1. **Directional Change (DC)** — Detects local pivot highs and lows using an adaptive threshold (sigma) derived from the stock's ATR. Each pivot is recorded as `[confirmation_bar, extreme_bar, extreme_price]`. The **confirmation bar** is the bar at which the pivot becomes knowable — critical for avoiding look-ahead bias.

2. **Rising Structure Detection** — From the DC pivots, identifies sequences of 3+ alternating tops and bottoms where each successive top and bottom is at least as high as the previous (rising structure). This is the "base" formation.

3. **Contraction Check** — The last swing (most recent top to most recent bottom) must contract to ≤ 12% of price (configurable via `last_contraction_pct`).

4. **Invalidation Check** — Price must not have closed below the last contraction low within the breakout window.

5. **Breakout Confirmation** — Price must close above resistance on ≥ 1.5× the pre-pattern volume baseline.

6. **Entry** — On the DC confirmation bar (the bar where the last pivot is confirmed), not the extreme bar. An additional 1-bar lag is applied (`buysignal_entry_lag = 1`).

---

## Data

### Source
CRSP (Center for Research in Security Prices) daily stock file, filtered to S&P 500 constituent PERMNOs. The data is pre-downloaded as a CSV file.

**Expected CSV path:** `C:\Users\bob.liang\Downloads\snp500_index_data.csv`

### Columns loaded
| Field | CRSP Column | Description |
|-------|-------------|-------------|
| Open | DlyOpen | Daily opening price |
| High | DlyHigh | Daily high price |
| Low | DlyLow | Daily low price |
| Close | DlyClose | Daily closing price |
| Volume | DlyVol | Daily volume |
| Adjustment Factor | DlyCumFacPr | Cumulative price adjustment (splits) |
| Share Factor | DlyCumFacShr | Cumulative share adjustment |
| MbrStartDt / MbrEndDt | Membership dates | S&P 500 index membership window |

### Instrument Keys
Tickers are identified by `PERMNO_<number>_<symbol>` (e.g. `PERMNO_12345_AAPL`) to survive ticker symbol changes over time.

### Split Adjustment
Raw CRSP prices are NOT split-adjusted. Prices are divided by `DlyCumFacPr` to produce a continuous split-adjusted series. Volume is multiplied by the same factor so dollar-volume (price × volume) stays consistent across splits.

### Date Range
The notebook defaults to data from `2020-01-01` onward (configurable via `BT_START_DATE`). The yfinance ^GSPC benchmark aligns automatically.

---

## Pipeline (Notebook Cells)

The notebook is organised into sequential cells that build on each other.

### 1. Data Loading (`load_snp500_data`)
- Reads the CRSP CSV, parses dates, filters by `start_date` if provided
- Performs split adjustment
- Pivots into a MultiIndex DataFrame: `columns = (field, Ticker)`, `index = DatetimeIndex`
- Prints summary (instruments, date range, row count)

### 2. Configuration
Key tunable parameters:
- `dc_sigma_warmup = 0.07` — seed sigma before ATR exists
- `dc_atr_multiplier = 1.5` — sigma = multiplier × ATR/close
- `dc_sigma_floor = 0.04`, `ceiling = 0.20` — sigma clipping bounds
- `dc_atr_window = 20` — rolling ATR window
- `last_contraction_pct = 0.12` — max allowed last-swing depth (12%)
- `breakout_window = 20` — bars forward to scan for breakout
- `breakout_buffer = 0.003` — price must exceed resistance by 0.3%
- `vol_surge_multiplier = 1.5` — volume surge threshold (1.5× baseline)
- `vol_surge_window = 20` — window for volume baseline

### 3. Pivot Detection
- `directional_change()` — Online algorithm that scans bar-by-bar and records local tops and bottoms using an adaptive sigma (from ATR). Each pivot entry stores `[confirmation_bar_index, extreme_bar_index, extreme_price]`.
- `compute_atr()` — Rolling ATR from OHLC data.

### 4. Pattern Detection
- `detect_all_rising_structures()` — Finds sequences of 3+ consecutive rising tops and bottoms from the DC pivot lists. Returns pattern dicts with start/end indices and pivot references.
- `extract_pattern_metadata()` — Computes last swing %, buy signal date (using the **confirmation bar**, not the extreme bar), and resistance level.

### 5. Rolling Bar-by-Bar Scanner (No Look-Ahead)
This replaces a naive full-history scan to eliminate look-ahead bias:
- DC is run once over the full series (it's inherently sequential/online).
- For each bar `t` from warmup onward:
  1. Filter DC pivots to only those confirmed at or before bar `t`
  2. Run rising-structure detection on these pivots only
  3. Check contraction tightness
  4. Check invalidation using data only up to bar `t`
  5. If valid → record the signal once (deduplicated by pattern `end_idx`)
- Produces `results_df` with columns: `VCP_ID`, `Ticker`, `PatternStart`, `PatternEnd`, `BuySignalDate`, `LastSwingPct`, `BreakoutDate`, plus internal metadata (`_df_full`, `_tops_full`, `_bottoms_full`, `_tops`, `_bottoms`, `_pattern`, `_meta`).

### 6. Trade Construction (`construct_trades`)
Converts `results_df` into `trades_df` with:
- Entry mode: `buysignal_preferred` (use buy-signal date if available, else breakout date)
- Entry timing: `confirm` mode uses the DC confirmation bar (no look-ahead)
- Stop loss: last contraction low, slipped downward
- Targets: primary = 5R (entry + 5 × risk), secondary = next resistance level
- Effective target = min(primary, secondary) when in `min_r_or_resistance` mode
- Walk-forward bucket assignment (PERMNO-level IS/OOS split)

### 7. Backtest Engine (`BacktestEngine` + `VCPStrategy`)
- `VCPStrategy` is a `Strategy` subclass that:
  - Groups signals by entry date for O(1) per-bar lookup
  - On each bar: resolves exits first (stops/targets/timeouts), then fires new entries
  - Entry sizing: `shares = (NAV × risk_per_trade_pct) / risk_per_share`
  - Enforces: one-position-per-ticker, `max_concurrent_positions` cap, cash sufficiency
  - Supports multiple exit modes: `fixed_r`, `min_r_or_resistance`, `partial_trail`
  - Logs every exit in `exit_log` with R-multiple, PnL%, holding bars, MAE/MFE
  - On the final bar, liquidates all remaining positions

- `BuyAndHold` benchmark strategy (buys ^GSPC on first bar and holds — exactly 1 trade)

### 8. Performance Analysis
- **R-multiple segment report** — Win rate, expectancy, profit factor, Sharpe-of-R sliced by: ALL TRADES, WF bucket (IS/OOS), EntryType
- **Trade export** — Comprehensive per-trade CSV (`vcp_trade_log.csv`) with 35+ columns: dates, prices, PnL, R-multiple, stop/target levels, MAE/MFE, volume data
- **Multi-ticker detail charts** — One candlestick chart per trade (VCP_ID) with DC pivot markers, connector lines, pattern rectangles, entry/exit diamonds, and holding-period lines

### 9. Signal Verification (`isBuySignal_check`)
A diagnostic function that takes a single PERMNO key and replots the VCP detection on the **causal data window** (pattern start → buy signal date), re-running DC from scratch on that window to verify that the correct local tops and bottoms were identified.

---

## Backtest Configuration (`BT` dict)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `initial_cash` | 1,000,000 USD | Starting equity |
| `risk_free_rate` | 0.04 | Annualised, used for Sharpe |
| `entry_mode` | `buysignal_preferred` | Trade entry priority |
| `buysignal_entry` | `confirm` | Use DC confirmation bar (no look-ahead) |
| `buysignal_entry_lag` | 1 | Extra bars after confirmation |
| `target_mode` | `min_r_or_resistance` | Exit at min(fixed R, resistance) |
| `R_target` | 5.0 | Base R-multiple target |
| `max_hold_bars` | 120 | ~6 month trading-day timeout |
| `slippage_pct` | 0.001 | Per side |
| `commission_pct` | 0.0005 | ~10 bps round-trip |
| `risk_per_trade_pct` | 0.01 | 1% of NAV risked per trade |
| `max_concurrent_positions` | 15 | Position cap |

---

## Look-Ahead Bias Prevention

Three layers of protection:

1. **ATR computation** — `compute_atr()` uses a rolling window, so bar `i`'s ATR only uses data up to bar `i`.

2. **DC confirmation bars** — Each pivot records its confirmation bar index (the bar at which the reversal is confirmed). The scanner uses `t[0] <= bar_t` to filter pivots, and `extract_pattern_metadata` uses `max(tp[-1][0], bp[-1][0])` for the buy signal date — i.e. the DC confirmation bar, not the extreme bar.

3. **Rolling pattern detection** — At each bar, pattern detection only runs on pivots already confirmed at that bar. Invalidation checks only use data up to the current bar. Breakout scans stop at the current bar.

---

## Output Files

| File | Description |
|------|-------------|
| `vcp_trade_log.csv` | Per-trade CSV with 35+ columns of trade data |
| Engine plots | NAV curve, drawdown, monthly return heatmap, information ratio vs ^GSPC |

---

## How to Run

1. Ensure the CRSP CSV exists at the expected path (or update `SNP500_CSV_PATH` in the notebook).
2. Open the notebook: `src/backtest/strategies/rule_based/vcp.ipynb`
3. Run all cells sequentially.
4. The notebook will:
   - Load and preprocess data
   - Run the rolling VCP scanner across all tickers (this is the longest step — ~500 tickers)
   - Construct trades and run the backtest
   - Display the R-multiple segment report
   - Export the trade CSV
   - Display multi-ticker detail charts

### Adjusting the Backtest Period

Change `BT_START_DATE` in the data-loading cell. Set to `None` for full history (from 1992).

### Charting Specific Trades

In the multi-ticker charts cell, set `CHART_TRADE_IDS = [0, 1, 5]` to plot specific VCP_IDs, or `CHART_TICKERS = ["PERMNO_..."]` to plot all trades for specific tickers.

### Verifying a Signal

Uncomment and call `isBuySignal_check("PERMNO_XXXXX")` in the verification cell.

---

## Architecture Notes

### Why PERMNO keys instead of ticker symbols?
CRSP PERMNOs are permanent identifiers that don't change when a company's ticker symbol changes (e.g. FB → META). The format `PERMNO_<id>_<ticker>` preserves both the stable identifier and the most recent readable symbol.

### Why a rolling scanner instead of a full-history scan?
A full-history scan can detect a VCP pattern using price bars that occur decades after the pattern — this is pure look-ahead. The rolling scanner simulates what would have been knowable at each point in time.

### Why two sets of DC pivots (`_tops_full` vs `_tops`)?
- `_tops_full` / `_bottoms_full` — Full-series pivots, used for complete chart rendering
- `_tops` / `_bottoms` — Pivots knowable at the signal time, used for zero-look-ahead pattern detection
