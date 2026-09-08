# Ranked Asset Allocation Model (RAAM)

**Please review `RAAM.ipynb`.** That notebook is the deliverable. Check that the pipeline below is implemented correctly. This README is only a short map.

The per-ticker OR ensembles are **not** chosen here. They come from `Multi-Asset Boruta Ensemble.ipynb` and are saved in `Ensemble_results/master_leaderboard_ensembles.json`. RAAM loads that file and allocates across those assets.

---

## What is the project?

RAAM ranks a mixed asset universe each day and holds up to four names. A name can be held only if it is both RSI-strong and currently long its frozen indicator ensemble. Leftover weight goes to gold (GLD) when GLD is eligible, otherwise cash.

Universe: `BTC-USD`, `TQQQ`, `UPRO`, `SMH.L`, `QTUM`, `GLD`, `MGK`, `^STOXX50E`.

## What is the research question?

Does this lagged, dual-gated allocation beat (or match) an in-sample static mix of the same assets, after trading costs?

## How it works

Run `RAAM.ipynb` top to bottom.

1. **Rebuild ensembles.** For each ticker in the leaderboard JSON, rebuild the saved OR ensemble on that ticker’s full Close series (signals lagged one bar). Write `RAAM_data/{slug}_raam.csv` with Date, Buy & Hold return, Strategy return, and 0/1 Position.
2. **Align.** Inner-join those files on Date. Correlation matrices are a check only.
3. **Rank.** RSI(28) on each ticker’s **cumulative Buy & Hold wealth**, not price. RSI below 50 → rank 0. Rank 1 = strongest remaining.
4. **Allocate.** Yesterday’s rank and position set today’s weights. Dual eligibility: rank in 1–4 **and** position == 1. Each selected name gets 25%. Leftover slots go to GLD if its rank > 0 and position == 1; else cash. Day 0 is cash.
5. **P&L and metrics.** Daily return = weight × **Buy & Hold** return (the ensemble is a gate, not the P&L series). Subtract 0.10% per side on weight turnover. Report net metrics vs in-sample max-Sharpe and max-Omega static portfolios.

## What data are you using?

- Yahoo Finance daily closes from 2018-01-01 (split/dividend-adjusted default).
- After the inner join the saved run is 2020-12-01 to 2026-09-08 (1390 days). Crypto weekends and non-overlapping holidays are dropped.
- Portfolio Sharpe uses 252 days. Commissions on the RAAM book are 0.10% per side on `sum(|Δw|)`.

## What are the folders and files?

- **`RAAM.ipynb`** — review this.
- **`RAAM_data/`** — per-ticker CSVs written by RAAM.
- **`Ensemble_results/master_leaderboard_ensembles.json`** — frozen ensembles (input).
- **`Multi-Asset Boruta Ensemble.ipynb`** — where that JSON is built (not the RAAM review).
- **`Indicator_sweeps/`** and **`Indicator_sweep_results/`** — used by the ensemble notebook, not by RAAM.

## What should I look at?

Open `RAAM.ipynb` and check two things: (1) is the code doing what the comments say, and (2) are the iffy design calls actually wrong?

**Implementation**

- Ensembles are rebuilt from the JSON (`members` + `param_display`, OR combine, `shift_signals=True`), not re-searched.
- Ranking and allocation share the inner-joined Date index.
- RSI is on cumulative BnH wealth, not price; RSI below 50 → rank 0; rank 1 = strongest.
- Weights use **yesterday’s** rank and position. Dual eligibility: rank in 1–4 **and** position == 1. Leftover to GLD or cash. Non-haven cap 25%. Day 0 is cash.
- Portfolio return is weight × **BnH**, not Strategy. Watch for look-ahead and the wrong return series.

**Please judge these (they may be wrong)**

- **Crypto weekends are dropped.** Files are inner-joined on Date, so BTC Saturday/Sunday bars disappear whenever equities have no print. Weekend crypto P&L is not folded into Friday–Monday. Is that the right mixed-calendar treatment, or does it understate BTC risk/return?
- **PSR / DSR.** Metrics use Bailey–López de Prado PSR (`P(true Sharpe > 0)`) and DSR with `N_TRIALS = 100000`. That N is a stress test against 100k zero-skill trials of this sample length — **not** a count of RAAM configs run in the notebook (there is one). Is the formula implemented correctly, and is that N an honest multiple-testing adjustment or an overstated hurdle?
- **Two cost layers.** Vectorbt fees/slippage (5 bp + 5 bp) shape the ensemble `Position` / Strategy path. RAAM then charges 10 bp per side on portfolio `sum(|Δw|)`. Is that double-counting, or correctly applying costs to two different series?
- **GLD leftover uses `rank > 0`**, not top-4. Gold can fill leftover slots whenever it is RSI-eligible and long its ensemble. Intentional safe-haven rule, or a loophole?
- **Static max-Sharpe / max-Omega see the whole sample.** They are an in-sample ceiling, not walk-forward. Sharpe annualization is 252 on the joined (mostly weekday) calendar even though BTC is in the book.

## How do I run it?

From the repo root, with `yfinance`, `TA-Lib`, `numpy`, `pandas`, `vectorbt`, `scipy`, and `matplotlib`. The leaderboard JSON must already exist. Open `RAAM.ipynb` and run all cells in order.
