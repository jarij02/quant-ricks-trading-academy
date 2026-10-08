# Ranked Asset Allocation Model (RAAM)

RAAM ranks a mixed asset universe each day and holds up to four names. A name can be held only if it is both RSI-strong and currently long its frozen indicator ensemble. Leftover weight goes to gold (GLD) when GLD is eligible, otherwise cash.

**Universe:** `BTC-USD`, `TQQQ`, `UPRO`, `SMH.L`, `QTUM`, `GLD`

The per-ticker OR ensembles are frozen in `Ensemble_results/master_leaderboard_ensembles.json`. The notebooks load that file and allocate across those assets; they do not re-search ensembles.

---

## Repository layout

### Notebooks

| File | Role |
|---|---|
| **`RAAM.ipynb`** | Research / backtest notebook. Rebuilds each frozen OR ensemble on full Close (signals lagged one bar), writes per-ticker CSVs, inner-joins on Date, ranks by RSI(28) on cumulative buy-and-hold wealth, allocates with the dual gate, applies 0.10% per-side commissions on weight turnover, and reports net metrics vs in-sample max-Sharpe and max-Omega static portfolios. |
| **`Live Trading.ipynb`** | Daily operator notebook for live use after the close. Same dual-gate allocation as RAAM, but outer-joins on equity session dates (holiday fills), applies signals at each ticker’s next native session (`europe` / `na` / `crypto`), and emits next-session **BUY / HOLD / SELL / REBALANCE / FLAT** actions. |

### Data and ensembles

| Path | Role |
|---|---|
| **`Ensemble_results/master_leaderboard_ensembles.json`** | Frozen best OR ensemble per ticker (members, parameters, IS/OOS metrics, Boruta flags). Input to both notebooks. |
| **`RAAM_data/`** | Per-ticker CSVs written by RAAM / Live Trading. Each file has `Date`, `{TICKER}_BnH` (buy-and-hold return), `{TICKER}_Strategy` (ensemble strategy return), and `{TICKER}_Position` (0/1 long gate). |

Active universe CSVs:

| File | Ticker |
|---|---|
| `btc_usd_raam.csv` | BTC-USD |
| `tqqq_raam.csv` | TQQQ |
| `upro_raam.csv` | UPRO |
| `smh_l_raam.csv` | SMH.L |
| `qtum_raam.csv` | QTUM |
| `gld_raam.csv` | GLD |

Other CSVs in `RAAM_data/` (`eth_usd_raam.csv`, `qqq_raam.csv`, `stoxx50e_raam.csv`) are leftover from earlier experiments and are not in the current leaderboard or `TICKERS` list.

### Upstream research (local / gitignored)

These are not re-run by RAAM; they produced the frozen leaderboard:

| Path | Role |
|---|---|
| **`Indicator_sweeps/`** / **`Indicator_sweep_results/`** | Per-ticker parameter sweeps for the active indicators. |
| **`Multi-Asset Boruta Ensemble.ipynb`** | Builds OR ensembles from sweep results and writes `master_leaderboard_ensembles.json`. |
| **`Pinescript/`** | TradingView ports of the frozen per-ticker ensembles. |

---

## How allocation works

1. **Rebuild ensembles.** For each ticker in the leaderboard JSON, rebuild the saved OR ensemble on that ticker’s Close series. Write `RAAM_data/{slug}_raam.csv`.
2. **Align.** RAAM inner-joins on Date; Live Trading outer-joins equity session dates with holiday fills.
3. **Rank.** RSI(28) on each ticker’s cumulative buy-and-hold wealth (not price). RSI below 50 → rank 0. Rank 1 = strongest remaining.
4. **Allocate.** Yesterday’s rank and position set today’s weights. Dual eligibility: rank in 1–4 **and** position == 1. Each selected name gets 25%. Leftover slots go to GLD if eligible, else cash.
5. **P&L.** Daily return = weight × buy-and-hold return (the ensemble is a gate, not the P&L series).

Active indicator builders: `Triple EMA`, `MACD`, `AROON`, `STC`, `KAMA`, `SUPERTREND`, `KALMAN`, `RSI`, `ADX`, `DONCHIAN`, `TRIX`, `VORTEX`, `ALMA`.

Frozen leaderboard members in use: `Triple EMA`, `MACD`, `STC`, `RSI`, `ADX`, `VORTEX`, `ALMA`, `TRIX`.

---

## How to run

From the repo root, with `yfinance`, `TA-Lib`, `numpy`, `pandas`, `vectorbt`, `scipy`, and `matplotlib`. The leaderboard JSON must already exist.

- **Research / backtest:** open `RAAM.ipynb` and run all cells in order.
- **Live positions:** open `Live Trading.ipynb` after the market close and run all cells in order.

Data: Yahoo Finance daily closes from 2021-01-01. Portfolio Sharpe uses 252 days. Commissions on the RAAM book are 0.10% per side on `sum(|Δw|)`.
