# Multi-Asset Boruta Ensemble

## What is the project?

This project backtests technical-indicator trading strategies across a mixed multi-asset universe (crypto, leveraged equity ETFs, sector/theme ETFs, and commodities). For each ticker it sweeps parameter grids over thirteen indicators, then builds small OR ensembles (2–3 indicators) inside a development window using purged walk-forward folds. A final untouched lockbox is scored once after the pipeline is frozen.

Ensemble candidates are optimized on fold in-sample Sharpe and screened with Boruta-style validation on fold-validation slices only (same OR signal combine as the traded ensembles; path-preserving circular-shift nulls with `ENSEMBLE_BORUTA_SHUFFLE_FRAC = 0.75` kept alignment, so max `|shift|` is 25% of the slice; binomial decisions and the stored hit-rate score are read against the max-shadow null `1/(m+1)`, not a coin-flip; leaderboard selection is restricted to the top `ENSEMBLE_SHOWCASE_TOP_N` (currently `10`) ensembles by IS Sharpe among those with finite OOS Sharpe `>= ENSEMBLE_LEADERBOARD_MIN_OOS_SHARPE` (currently `1.0`), then walks the same gates as before — `_oos_gate`, then `_gate` (OOS and Boruta; the score must be finite and exceed stored `_boruta_null = 100/(m+1)`), then highest OOS trades (`oos_total_trades`) instead of highest IS Sharpe — and falls back with a WARNING if no row in that top-N pool clears those gates; an empty finite-IS-Sharpe top-N pool is a `ValueError`; failing the Boruta gate does not drop the ticker; selection is re-run each replicate; one shared RNG). Per-asset detailed showcase tables reuse those same top-N frames (`_ticker_oos_top`) and list them by IS Sharpe; a ticker with no OOS passer gets a WARNING and an empty showcase entry. Indicators and ensembles must also clear `ENSEMBLE_MIN_TRADES_PER_YEAR = 2` (strict `>`): indicators are gated on param-matched in-sample `trades_per_year` from the sweep JSON before they enter the OR grid, and ensembles are gated on IS trades/year before Boruta/OOS and again on OOS trades/year after validation; a ticker with no qualifying ensembles is recorded under ticker failures. Lockbox Sharpe is report-only and is not used for selection. Backtests are run in vectorbt on daily closes with fixed fees and slippage from 2018 through the latest available download date.

## What is the research question?

Do small OR ensembles of technical indicators, selected with nested walk-forward validation inside a development window and Boruta-style fold screening, produce more reliable lockbox out-of-sample risk-adjusted performance than single-indicator strategies alone — and does that edge hold across crypto, leveraged equity ETFs, sector ETFs, and gold?

## What data are you using?

- **Data source:** Yahoo Finance via `yfinance`
- **Assets:** The `TICKERS` list in `Multi-Asset Boruta Ensemble.ipynb` (order preserved): BTC-USD, TQQQ, UPRO, SMH.L, QTUM, GLD — crypto (BTC-USD), 3× leveraged equity ETFs (TQQQ, UPRO), a London-listed semiconductor ETF (SMH.L), a US-listed quantum/next-gen tech ETF (QTUM), and gold (GLD).
- **Time period:** 2018-01-01 through the latest available daily bar at download time (currently through 2026-08-29 in the saved run)
- **Frequency:** Daily (`interval="1d"`); Sharpe annualization uses 252 trading days for stocks/ETFs and 365 for crypto on standalone series. The multi-asset notebook uses one weekday portfolio NAV calendar (`PORTFOLIO_NAV_CALENDAR = "weekday"`): weekend crypto P&L is kept inside the Friday-to-Monday close (weekend bars are not inner-joined away and equities are not forward-filled onto Saturday/Sunday). The weekday index is validated (no weekends, no NaT); both sides of the cut need at least two dates. Development/lockbox is cut on a single declared date (`LOCKBOX_START`) from that calendar (`LOCKBOX_RATIO = 0.20`), not by per-ticker row count
- **Preprocessing:** Drop infinite/missing closes; sort by date; normalize timestamps to UTC-naive midnight and deduplicate again after that collapse; chronological development/lockbox split on the common weekday NAV date `LOCKBOX_START` (`LOCKBOX_RATIO = 0.20` of `portfolio_nav_index`); `portfolio_nav_close` aligns native closes onto that weekday grid without forward-fill and requires finite marks per ticker; purged expanding walk-forward folds inside development (`DEV_WF_FOLDS = 4`, `DEV_PURGE_BARS = 21`); indicators from TA-Lib plus custom implementations (KAMA, SuperTrend, Kalman, ALMA, STC, etc.); entry/exit signals lagged one bar (`shift_signals=True`) to reduce same-bar look-ahead
- **Dividends:** Equity/ETF closes from yfinance are used as returned `Close` series (default auto-adjust behavior), so dividend effects are reflected in the adjusted price path rather than modeled as separate cash distributions
- **Price adjustment:** Split- and dividend-adjusted closes (yfinance default); no additional custom adjustment layer is applied in the notebook

## What are the folders and files?

- **`Multi-Asset Boruta Ensemble.ipynb`:** Primary multi-asset Boruta ensemble backtesting notebook; loads saved sweep results and builds OR ensembles across the ticker universe
- **`Indicator_sweeps/`:** Per-asset notebooks that sweep indicator parameter grids. The ensemble currently loads results for BTC-USD, TQQQ, UPRO, SMH.L (`SMH indicator sweep.ipynb`), QTUM, and GLD. Other sweep notebooks in this folder are not enrolled in `TICKERS`.
- **`Indicator_sweep_results/`:** Saved sweep JSON for each swept ticker (`{ticker_slug}_indicator_sweep_results.json`). The ensemble notebook consumes the six enrolled slugs: `btc_usd`, `tqqq`, `upro`, `smh_l`, `qtum`, `gld`.
- **`README.md`:** Project overview, research framing, and folder/file map
- **`.gitignore`:** Ignore rules for virtual environments, Jupyter checkpoints, secrets, and local scratch notebooks

## What should I look at?

Please focus your review on:

1. **Indicator parameter sweeps** — Review one of the indicator sweep notebooks inside the 'Indicator_sweeps' folder and review the full notebook. All of the notebooks present inside the folder are the same notebook, except each one got a different asset.
2. **Multi-Asset Boruta Ensemble** — Review the full Multi-Asset Boruta Ensemble notebook.

## Changes since last review

| Date | Changes |
| --- | --- |
| 2026-08-25 | Added `ENSEMBLE_MIN_TRADES_PER_YEAR = 2` in `Multi-Asset Boruta Ensemble.ipynb`: ensembles with IS trades/year `<= 2` are excluded from Boruta/OOS; if none remain, the ticker is recorded as a failure. |
| 2026-08-25 | Extended that gate to single indicators: an indicator must clear OOS Sharpe and IS trades/year `> 2` before it is eligible for ensembles. |
| 2026-08-25 | Review hardening on that gate: indicator eligibility prefers param-matched IS `trades_per_year` (not full-sample); threshold must be finite and `>= 0`; missing `is_trades_per_year` on IS ensemble results is a `KeyError`; shared `_metric_matching_params` helper keeps the Sharpe lookup path. |
| 2026-08-25 | Updated this README so the screening description and the changes table match the trades-per-year indicator and ensemble gates. |
| 2026-08-25 | Per-asset ensemble showcases in `Multi-Asset Boruta Ensemble.ipynb` now only show ensembles that clear `ENSEMBLE_LEADERBOARD_MIN_OOS_SHARPE` (reuse section A's `_oos_ok`; empty ticker → WARNING). |
| 2026-08-25 | Updated this README so the screening description and the changes table match the per-asset OOS Sharpe showcase filter. |
| 2026-08-25 | Extended `ENSEMBLE_MIN_TRADES_PER_YEAR` in `Multi-Asset Boruta Ensemble.ipynb` to OOS: ensembles with OOS trades/year `<= 2` are dropped after Boruta/OOS; if none remain, the ticker is recorded as a failure. |
| 2026-08-25 | Updated this README so the screening description and the changes table match the OOS trades-per-year ensemble gate. |
| 2026-08-25 | Review hardening on the OOS trades/year gate: filter before Boruta (skip shuffles on sparse OOS); track/print filtered count; empty-result errors distinguish TPY filters from Boruta failures; coerce `oos_trades_per_year` with `safe_float` before the safety mask. |
| 2026-08-25 | Updated this README so the changes table matches that OOS trades/year review hardening. |
| 2026-08-26 | Limited per-asset detailed showcases in `Multi-Asset Boruta Ensemble.ipynb` to the top `ENSEMBLE_SHOWCASE_TOP_N = 10` OOS-eligible ensembles by IS Sharpe. |
| 2026-08-26 | Review hardening on that showcase cap: validated `ENSEMBLE_SHOWCASE_TOP_N` (integer `>= 1`); `_showcase_top_n` mirrors `displayed_is_top_n` (`KeyError` + `dropna` + `nlargest`); OOS-eligible rows are filtered once into `_showcase_eligible` before the per-ticker loop. |
| 2026-08-26 | Updated this README so the screening description and the changes table match the top-N per-asset showcase cap. |
| 2026-08-28 | Master leaderboard in `Multi-Asset Boruta Ensemble.ipynb` still walks `_oos_gate` then `_gate` (OOS and Boruta) per ticker, but only inside each asset's top `ENSEMBLE_SHOWCASE_TOP_N` OOS-eligible rows; the last sort key is `oos_total_trades` instead of IS Sharpe. |
| 2026-08-28 | Review hardening on that pick: empty top-N pool is a `ValueError`; `_oos_ok` length is checked before the pick; showcase reuses `_ticker_oos_top` instead of recomputing `_showcase_top_n`. |
| 2026-08-28 | Updated this README so the screening description and the changes table match the top-N OOS-and-Boruta walk with most-trades as the last key. |
| 2026-08-31 | Updated this README so the documented universe matches `TICKERS` in `Multi-Asset Boruta Ensemble.ipynb`: BTC-USD, TQQQ, UPRO, SMH.L, QTUM, GLD (QTUM replaces QNTM.L; ETH-USD, AAPL, GOOG, QQQ, and SPY are no longer enrolled). Saved-run date range is through 2026-08-29. Removed the deleted `Single_Asset_Ensembles/` folder from the file map. |

