# Multi-Asset Boruta Ensemble

## What is the project?

This project backtests technical-indicator trading strategies across a mixed multi-asset universe (equities, leveraged ETFs, commodities, and crypto). For each ticker it sweeps parameter grids over thirteen indicators, then builds small OR ensembles (2–3 indicators) inside a development window using purged walk-forward folds. A final untouched lockbox is scored once after the pipeline is frozen.

Ensemble candidates are optimized on fold in-sample Sharpe and screened with Boruta-style validation on fold-validation slices only (same OR signal combine as the traded ensembles; path-preserving circular-shift nulls with `ENSEMBLE_BORUTA_SHUFFLE_FRAC = 0.75` kept alignment, so max `|shift|` is 25% of the slice; binomial decisions and the stored hit-rate score are read against the max-shadow null `1/(m+1)`, not a coin-flip; leaderboard selection walks IS ranks until OOS Sharpe and the Boruta score both clear — `_gate` sits next to `_oos_gate`; the score must be finite and exceed stored `_boruta_null = 100/(m+1)` — and falls back with a WARNING if no row clears those gates; failing the Boruta gate does not drop the ticker; selection is re-run each replicate; one shared RNG). Lockbox Sharpe is report-only and is not used for selection. Backtests are run in vectorbt on daily closes with fixed fees and slippage from 2018 through the latest available download date.

## What is the research question?

Do small OR ensembles of technical indicators, selected with nested walk-forward validation inside a development window and Boruta-style fold screening, produce more reliable lockbox out-of-sample risk-adjusted performance than single-indicator strategies alone — and does that edge hold across equities, leveraged products, and crypto?

## What data are you using?

- **Data source:** Yahoo Finance via `yfinance`
- **Assets:** BTC-USD, ETH-USD, TQQQ, UPRO, SMH.L, QNTM.L, AAPL, GOOG, GLD, QQQ, SPY
- **Time period:** 2018-01-01 through the latest available daily bar at download time (currently through 2026-08-11 in the saved run)
- **Frequency:** Daily (`interval="1d"`); Sharpe annualization uses 252 trading days for stocks/ETFs and 365 for crypto on standalone series. The multi-asset notebook uses one weekday portfolio NAV calendar (`PORTFOLIO_NAV_CALENDAR = "weekday"`): weekend crypto P&L is kept inside the Friday-to-Monday close (weekend bars are not inner-joined away and equities are not forward-filled onto Saturday/Sunday). The weekday index is validated (no weekends, no NaT); both sides of the cut need at least two dates. Development/lockbox is cut on a single declared date (`LOCKBOX_START`) from that calendar (`LOCKBOX_RATIO = 0.20`), not by per-ticker row count
- **Preprocessing:** Drop infinite/missing closes; sort by date; normalize timestamps to UTC-naive midnight and deduplicate again after that collapse; chronological development/lockbox split on the common weekday NAV date `LOCKBOX_START` (`LOCKBOX_RATIO = 0.20` of `portfolio_nav_index`); `portfolio_nav_close` aligns native closes onto that weekday grid without forward-fill and requires finite marks per ticker; purged expanding walk-forward folds inside development (`DEV_WF_FOLDS = 4`, `DEV_PURGE_BARS = 21`); indicators from TA-Lib plus custom implementations (KAMA, SuperTrend, Kalman, ALMA, STC, etc.); entry/exit signals lagged one bar (`shift_signals=True`) to reduce same-bar look-ahead
- **Dividends:** Equity/ETF closes from yfinance are used as returned `Close` series (default auto-adjust behavior), so dividend effects are reflected in the adjusted price path rather than modeled as separate cash distributions
- **Price adjustment:** Split- and dividend-adjusted closes (yfinance default); no additional custom adjustment layer is applied in the notebook

## What are the folders and files?

- **`Multi-Asset Boruta Ensemble.ipynb`:** Primary multi-asset Boruta ensemble backtesting notebook; loads saved sweep results and builds OR ensembles across the ticker universe
- **`Indicator_sweeps/`:** Per-asset notebooks that sweep indicator parameter grids for each multi-asset ensemble ticker only (BTC-USD, ETH-USD, TQQQ, UPRO, SMH.L, QNTM.L, AAPL, GOOG, GLD, QQQ, SPY)
- **`Indicator_sweep_results/`:** Top indicator results for those same ensemble assets (`{ticker_slug}_indicator_sweep_results.json`); consumed as inputs by the multi-asset ensemble notebook
- **`Single_Asset_Ensembles/`:** Single-ticker Boruta ensemble notebooks (for example SPY and BTC-USD)
- **`README.md`:** Project overview, research framing, and folder/file map
- **`.gitignore`:** Ignore rules for virtual environments, Jupyter checkpoints, secrets, and local scratch notebooks

## What should I look at?

Please focus your review on:

1. **Indicator parameter sweeps** — Review one of the indicator sweep notebooks inside the 'Indicator_sweeps' folder and review the full notebook. All of the notebooks present inside the folder are the same notebook, except each one got a different asset.
2. **Multi-Asset Boruta Ensemble** — Review the full Multi-Asset Boruta Ensemble notebook.

## Changes since last review

| Date | Changes |
| --- | --- |
| 2026-08-17 | Softened the OOS Boruta circular-shift null in `Multi-Asset Boruta Ensemble.ipynb`: `ENSEMBLE_BORUTA_SHUFFLE_FRAC = 0.75` is kept alignment, so max `|shift|` is 25% of the series (a 75% roll still allowed the harshest 50% rotation). |
| 2026-08-17 | Review hardening on that shuffle: empty shift ranges raise instead of silently using `k = 1`; `rng` must provide `integers()`; `min_shift_frac` must be a finite fraction in `(0, 1)`. |
| 2026-08-17 | Updated this README so the Boruta null description and the changes table match the softened OOS shuffle. |
| 2026-08-17 | Boruta in `Multi-Asset Boruta Ensemble.ipynb` now scores the same OR combine the backtest trades (`combine="or"`). It previously passed `combine="vote"` (2-of-3 on three-member ensembles — a different strategy than OR). |
| 2026-08-17 | Vote `k` is hoisted from the full pool so leave-one-out rest/shadow slices keep the original threshold; `_or_combine_member_signals` now validates members the same way vote does (1-D bool, lengths, empty/missing); deterministic full/rest Sharpes are computed once per Boruta round, not every shuffle. |
| 2026-08-17 | Updated this README so the Boruta description states that screening uses the same OR combine as the traded ensembles. |
| 2026-08-17 | Boruta binomial tests in `Multi-Asset Boruta Ensemble.ipynb` now use the max-shadow null `p0 = 1/(m+1)` instead of `0.5`. `p0` is computed once per round and must lie in `(0, 1)`. |
| 2026-08-17 | The OOS Boruta hit-rate score is a binding leaderboard gate: `ENSEMBLE_BORUTA_NULL = 100/(len(members)+1)`; a row clears it when the score is finite and `> 2 ×` that null. Failing the gate no longer skips the ticker. The leaderboard walks IS ranks until OOS Sharpe and Boruta both clear, then WARNINGs if a pick missed a gate (same fallback pattern as OOS Sharpe). |
| 2026-08-17 | Review hardening on that gate: `_as_bool_flag` coerces stored `passes_boruta` flags so NaN/invalid are False (`bool(np.nan)` is True); `len(members) < 2` raises before the OOS backtest; a missing `passes_boruta` column is a `KeyError`; the best-row WARNING uses that helper instead of `bool(.get())`. |
| 2026-08-17 | Updated this README so the Boruta screening description matches the max-shadow null, the binding `2 × 100/(m+1)` leaderboard gate, and that review hardening. |
| 2026-08-17 | Leaderboard Boruta gate in `Multi-Asset Boruta Ensemble.ipynb` is now `_gate` next to `_oos_gate`: OOS Sharpe and finite `boruta_score > _boruta_null`, with `_boruta_null = 100/(len(members)+1)` stored per row. It previously required `> 2 ×` that null via `passes_boruta`. |
| 2026-08-17 | Review hardening on that gate: score and null go through `safe_float` / `isfinite` like the OOS mask; `_lb_required` includes `boruta_score`, `_boruta_null`, and `n_members`; stored null must equal `100/(n_members+1)` and be finite in `(0, 100)` at write time; the best-row WARNING uses `score > _boruta_null` instead of the stored `passes_boruta` flag. |
| 2026-08-17 | Updated this README so the Boruta screening description matches the binding `_boruta_null = 100/(m+1)` leaderboard gate (`_gate` next to `_oos_gate`) and that review hardening. |

