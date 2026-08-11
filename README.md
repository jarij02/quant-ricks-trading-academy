# Multi-Asset Boruta Ensemble

## What is the project?

This project backtests technical-indicator trading strategies across a mixed multi-asset universe (equities, leveraged ETFs, commodities, and crypto). For each ticker it sweeps parameter grids over thirteen indicators, ranks candidates by risk-adjusted performance with an in-sample / out-of-sample split, then builds small OR ensembles (2–3 indicators) from the OOS survivors.

Ensemble candidates are optimized on in-sample Sharpe and further screened with a Boruta-style validation step (shadow/permutation tests on OOS returns and leave-one-out Sharpe deltas). Backtests are run in vectorbt on daily closes with fixed fees and slippage from 2018 through the latest available download date.

## What is the research question?

Do small OR ensembles of technical indicators, selected with an IS/OOS Sharpe filter and Boruta-style validation, produce more reliable out-of-sample risk-adjusted performance than single-indicator strategies alone — and does that edge hold across equities, leveraged products, and crypto?

## What data are you using?

- **Data source:** Yahoo Finance via `yfinance`
- **Assets:** BTC-USD, ETH-USD, TQQQ, UPRO, SMH.L, NFLX, AAPL, GOOG, GLD, QQQ, SPY
- **Time period:** 2018-01-01 through the latest available daily bar at download time (currently through 2026-08-11 in the saved run)
- **Frequency:** Daily (`interval="1d"`); Sharpe annualization uses 252 trading days for stocks/ETFs and 365 for crypto calendars
- **Preprocessing:** Drop infinite/missing closes; sort by date; deduplicate timestamps; chronological 60/40 train/validation split per ticker (`TRAIN_RATIO = 0.60`); indicators from TA-Lib plus custom implementations (KAMA, SuperTrend, Kalman, ALMA, STC, etc.); entry/exit signals lagged one bar (`shift_signals=True`) to reduce same-bar look-ahead
- **Dividends:** Equity/ETF closes from yfinance are used as returned `Close` series (default auto-adjust behavior), so dividend effects are reflected in the adjusted price path rather than modeled as separate cash distributions
- **Price adjustment:** Split- and dividend-adjusted closes (yfinance default); no additional custom adjustment layer is applied in the notebook

## What are the folders and files?

- **`Multi-Asset Boruta Ensemble.ipynb`:** Primary multi-asset Boruta ensemble backtesting notebook; loads saved sweep results and builds OR ensembles across the ticker universe
- **`Indicator_sweeps/`:** Per-asset notebooks that sweep indicator parameter grids for a single ticker
- **`Indicator_sweep_results/`:** Top indicator results for each given asset (`{ticker_slug}_indicator_sweep_results.json`); consumed as inputs by the multi-asset ensemble notebook
- **`Single_Asset_Ensembles/`:** Single-ticker Boruta ensemble notebooks (for example SPY and BTC-USD)
- **`README.md`:** Project overview, research framing, and folder/file map
- **`.gitignore`:** Ignore rules for virtual environments, Jupyter checkpoints, secrets, and local scratch notebooks

## What should I look at?

Please focus your review on:

1. **Indicator parameter sweeps** — Review one of the indicator sweep notebooks inside the 'Indicator_sweeps' folder and review the full notebook. All of the notebooks present inside the folder are the same notebook, except each one got a different asset.
2. **Multi-Asset Boruta Ensemble** — Review the full Multi-Asset Boruta Ensemble notebook.



