# Initial Backtest Framework — MA Crossover Strategy

> *Can a simple, rule-based momentum signal demonstrate genuine statistical edge? And if it can, how do we know we're measuring it honestly?*

This repository is the attempt to answer those questions rigorously. It is not a finished strategy — it is a structured investigation. The goal was never to build the most profitable system, but to construct a framework disciplined enough to tell the difference between real edge and backtest artefact.

---

## What Are We Actually Testing?

A 9-day / 21-day moving average crossover on four US technology equities (AAPL, META, MSFT, TSLA), applied long-only with an ATR-based stop and a daily equal-weight portfolio structure. The deeper question underneath that setup is this: *does a mechanical trend signal — one of the oldest in systematic trading — still generate positive expectancy on large-cap US equities across a full market cycle?*

The answer emerging from the framework is cautiously yes — a portfolio Sharpe of 1.18, positive expectancy on all four tickers, and an 85.9% probability of a positive annual return. But those numbers are only credible if the framework that produced them is airtight. That is what the nine notebooks are really about.

---

## Pipeline Architecture

The framework is structured as nine sequential modules. Each one produces a persisted `.parquet` output, creating a clean data handoff that prevents any stage from contaminating another and allows any individual module to be re-run or modified in isolation.

---

### Notebook 1 — Data Loading & Preprocessing

*Question: Can we trust the data before we trust the signal?*

Raw OHLCV data is pulled from Yahoo Finance and immediately subjected to a tiered quality audit. The NYSE trading calendar is used to distinguish between legitimate market closures (holidays, Hurricane Sandy) and genuine data defects — a distinction that matters because treating a holiday as missing data introduces phantom gaps into the signal series. A masking rule flags any row with fewer than three valid OHLCV fields as a fatal error; rows with partial data are forward-filled and preserved rather than dropped.

META's pre-IPO window is explicitly null-masked. Duplicate index entries are checked. All adjustment methodology (splits, dividends via `auto_adjust=True`) is logged to a `metadata.json` audit trail.

**Output:** `ohlcv_{ticker}.parquet` with an `is_valid_day` validity mask; `metadata.json`.

---

### Notebook 2 — Signal Construction

*Question: How do we ensure the signal only ever sees the past?*

The crossover logic itself is straightforward — a long signal fires when the 9-day average is above the 21-day average. The discipline comes from the implementation. A mandatory one-period lag (`signal.shift(1)`) is applied to every final position column, ensuring that a crossover detected at the close of day *T* is only actionable at the open of day *T+1*. Without this, every backtest result is contaminated by look-ahead bias.

A 200-day trend filter is computed as a modular, optional column rather than a hard gate — this allows A/B testing in later notebooks without restructuring the signal pipeline. Both SMA and EMA variants are computed and persisted, giving the downstream engine two independently testable strategies.

**Output:** `signals_{ticker}.parquet` containing MA series, lagged position flags, and strategy returns for both MA types.

---

### Notebook 3 — Backtest Engine & Trade Sheet

*Question: What does execution actually cost, and does our stop behave the way we think it does?*

This is the core simulation. The engine iterates event-by-event, entering at the next open after a signal and exiting at whichever condition fires first — a stop breach (checked against the intraday low, not the close) or a signal reversal. Two stop regimes are evaluated: a fixed 2% percentage stop, and an ATR-based stop set at 2× the 14-period average true range. The ATR stop is strictly superior in high-volatility environments — it widens for TSLA during earnings events and tightens for MSFT during quiet periods, adapting to the instrument rather than applying a one-size-fits-all rule.

Transaction costs (5 bps) and slippage (1 bp) are applied per leg. Beyond raw return, each trade records MAE, MFE, entry and exit efficiency, holding days, R-multiple, and exit type. These fields make Notebook 4's statistical interrogation possible.

**Output:** 16 trade sheets — 4 tickers × 2 stop modes × 2 MA variants.

---

### Notebook 4 — Trade-Level Analytics

*Question: Is positive total return the result of genuine edge, or a few lucky outliers pulling the average up?*

This notebook disaggregates the trade sheet to challenge the headline returns. Win rates are low (36–38%), which is expected for trend-following — the question is whether the winners are large enough to justify it. Profit factors ranging from 1.96 to 4.36, and payoff ratios from 3.19 to 7.72, confirm that the answer is yes across all four tickers.

The return distributions are examined for skewness (all positive — a desirable trait for momentum strategies) and kurtosis (TSLA's reading of 40.35 is a warning flag — extreme fat tails mean its volatility metrics alone do not capture its true risk profile). The Kelly criterion is computed per ticker. TSLA's 27.8% Kelly fraction exceeds the 25% safety threshold, triggering an explicit warning. Half-Kelly fractions are persisted for downstream use.

**Output:** `trade_analytics_summary.parquet`; `kelly_fractions.json`.

---

### Notebook 5 — Equity Curves & Portfolio Construction

*Question: Does combining these four strategies produce genuine diversification, or do they all move together when it matters most?*

A daily-rebalanced equal-weight portfolio (25% per asset) is constructed. The mechanics are intentionally simple — equal weighting avoids the false precision of in-sample covariance optimisation. Capital not deployed in an active position is held as cash at 0%, enforcing strict isolation between assets.

The 6-month rolling correlation heatmap is the most revealing output: pairwise correlations spike sharply during the COVID crash and the 2022 selloff — the exact periods when diversification is most needed. All four are US technology stocks, and their correlated drawdowns are the primary structural risk of this portfolio. The equity curves are plotted across four timeframes (all-time, 5Y, 1Y, 1M) to stress-test whether the performance is evenly distributed or concentrated in a single favourable window.

**Output:** `strategy_returns_all.parquet`; `portfolio_returns.parquet`.

---

### Notebook 6 — Performance Metrics

*Question: How much of the return is compensation for risk, and how does it compare to doing nothing?*

A full risk-adjusted metrics suite is computed — Sharpe (1.18), Sortino (1.46), Calmar (0.93), Omega, historical and parametric VaR/CVaR, tail ratio, and an extended drawdown analysis tracking duration and threshold breach counts. The buy-and-hold benchmark is constructed from the same four tickers for an apples-to-apples comparison.

Active return versus buy-and-hold is negative for all instruments (−12% to −30%). This is structurally expected for a trend-following strategy in a predominantly bull market — the strategy exits positions during corrections and misses portions of the recovery. The negative information ratio is not evidence of a broken strategy; it is the quantified cost of downside protection. The relevant question is whether that protection was worth the premium, which the stress test results in Notebook 9 help answer.

**Output:** `performance_summary.parquet`.

---

### Notebook 7 — Risk Management, Sizing & Stop Calibration

*Question: How sensitive is performance to how we size positions, and what is the true cost of each stop-loss regime?*

Three sizing methods are evaluated side-by-side: static equal-weight, volatility-targeted (15% annualised target, 20-day rolling vol, capped at 2× leverage), and fractional Kelly (Half-Kelly per ticker). The portfolio's annualised volatility of 16.4% — materially below the 22.5% weighted average of individual stock volatilities — confirms that correlation structure is providing genuine risk reduction. The diversification ratio quantifies this.

The ATR versus fixed stop comparison frames a key design question: a fixed 2% stop would have generated significantly more stop-outs on TSLA, whose ATR-based thresholds typically sit at 5–10% during high-volatility windows. The ATR stop's ability to adapt to the volatility regime is its core advantage, though it comes with wider individual trade losses during stress events.

**Output:** `risk_calibration_summary.parquet`.

---

### Notebook 8 — Parameter Optimisation & Robustness

*Question: Is the 9/21 MA combination genuinely robust, or did we happen to choose a combination that looks good in-sample?*

This is the framework's most critical intellectual checkpoint. The dataset is split 70/30 chronologically (in-sample / out-of-sample), and a grid search evaluates Sharpe and Calmar across short MA windows (5, 9, 12, 15, 20) against long MA windows (21, 50, 100, 150, 200), and SL/TP combinations across five stop-loss and six take-profit levels.

The key diagnostic is Sharpe degradation from in-sample to out-of-sample. Degradation above 40% signals severe overfitting. The heatmaps are read for stable plateaus rather than isolated peaks — a parameter combination that sits within a broad region of similar performance is far more trustworthy than one that outperforms only at a single point in parameter space.

**Output:** `optimization_ma_results.parquet`; `optimization_sltp_results.parquet`.

---

### Notebook 9 — Monte Carlo & Stress Testing

*Question: How does the strategy behave under conditions the historical sample may not fully represent?*

Two complementary approaches to tail risk are applied. The bootstrap Monte Carlo simulation (10,000 paths, 252-day horizon, seed 42) resamples historical daily returns with replacement, preserving the empirical distribution's fat tails and skewness while randomising the sequence. The median outcome (×1.20, approximately +20% annualised) aligns well with the empirical CAGR of 21.3% — a reassuring consistency check. The 85.9% probability of positive annual return means roughly one in six simulated years still produces a loss.

The historical stress tests slice performance across three documented regime shifts: COVID (Feb–Mar 2020, −15.4% total return, −17.2% max drawdown), the 2022 Fed rate hike cycle (−15.8% over the full year), and the 2015 China devaluation shock (−6.2%). Each event reveals something different about the strategy's failure modes — fast systemic crashes trigger ATR stops calibrated to pre-crisis volatility, while slow grinding bear markets generate costly false re-entry signals.

**Output:** `monte_carlo_results.parquet`; `stress_test_summary.parquet`.

---

## Key Findings

| Metric | Value |
|--------|-------|
| Portfolio CAGR | 21.3% |
| Portfolio Sharpe Ratio | 1.18 |
| Portfolio Sortino Ratio | 1.46 |
| Portfolio Max Drawdown | −23.0% |
| Total Trades | 391 |
| Win Rate Range | 36–38% across all tickers |
| Profit Factor Range | 1.96 (MSFT) – 4.36 (TSLA) |
| Probability of Positive Return (MC) | 85.9% |
| Active Return vs Buy-and-Hold | −12% to −30% (structurally expected) |

---

## What This Framework Does Not Resolve

Honest accounting of the open questions this version leaves on the table:

- **Survivorship bias.** AAPL, META, MSFT, and TSLA are among the strongest-performing equities of the 2010–2025 period. The results cannot be generalised to the broader market without testing on a pre-determined, point-in-time universe.
- **Single-asset optimisation.** The MA parameter grid search runs on AAPL only. Whether 9/21 is robust across the other three tickers has not been independently confirmed.
- **Temporal concentration.** Whether the portfolio Sharpe of 1.18 is evenly earned across the full 2010–2025 window, or disproportionately concentrated in the 2019–2021 bull run, remains an important decomposition exercise.
- **Walk-forward validation.** A single 70/30 split is a starting point, not a verdict. A rolling walk-forward with multiple folds would produce a distribution of out-of-sample Sharpe ratios rather than a single data point.
- **Transaction cost sensitivity.** 5 bps per leg is a reasonable estimate for large-caps, but a sensitivity sweep at 10 bps and 20 bps would establish the minimum edge threshold required to survive higher-cost environments.

---

*The framework is designed to be interrogated. Every design decision — the stop type, the lag, the cost model, the split ratio — is a hypothesis. The right response to these results is not to deploy the strategy; it is to keep asking harder questions.*
