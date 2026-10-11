# Roostoo Quant Trading Hackathon — Autonomous Trading Agent

## Overview

This project is an autonomous cryptocurrency trading framework for the Roostoo competition environment. It connects asset-universe discovery, historical-data preparation, momentum research, portfolio construction, backtesting, robustness diagnostics, and live execution.

The core investment idea is to allocate capital to relatively strong assets while considering their absolute trends, volatility, and the broader market regime. Portfolio sizing and trading frequency are intended to balance opportunity against concentration risk and transaction costs.

This README explains the strategy design, its relationship to the code, the research workflow, and the information needed to evaluate the submission.

### Documentation status

This document is based on the supplied project README. The repository source code, configuration values, execution logs, and backtest reports were not supplied for this revision. Accordingly:

- The modules and capabilities below reflect the original project description, not an independent code audit.
- Mathematical expressions explain the intended concepts; they are not verified reproductions of the implementation.
- Items marked `[TO CONFIRM]` or `[TO FILL]` require repository or report evidence before submission.
- No performance figures, fee rates, or risk thresholds have been invented.

## 1. Investment thesis and strategy rationale

The strategy combines two different questions:

1. Cross-sectional momentum: which eligible assets have performed better than their peers?
2. Time-series momentum: is an individual asset itself trending positively or negatively?

These questions are complementary. An asset can outperform its peers while still losing value in a falling market. Relative strength alone therefore does not establish that holding it is attractive. Absolute-trend information provides an additional perspective on whether the strategy should maintain exposure.

The design rationale is to use relative momentum for asset selection and absolute trend, volatility, and market-regime information to inform filtering and exposure. The research framework evaluates candidate signals rather than assuming that every candidate must be traded simultaneously.

This is a trading hypothesis to be tested, not a claim that momentum always persists. Reversals, sideways markets, and abrupt volatility changes can invalidate the signal.

### Candidate signals

| Component | Investment interpretation | Project reference |
| --- | --- | --- |
| Cross-sectional momentum (`cs_mom`) | Rank eligible assets by historical relative performance | `src/roostoo_quant/research/candidates.py` |
| Time-series momentum (`ts_mom`) | Measure each asset's own historical trend | `src/roostoo_quant/research/candidates.py` |
| Breakout and volatility filters | Evaluate trend-related conditions and avoid or reduce exposure to noisy conditions | Candidate definitions and configuration; exact rules `[TO CONFIRM]` |
| Realized-volatility rank (`realized_volatility_rank_24h`) | Compare volatility across the eligible universe | Candidate definitions; estimator and usage `[TO CONFIRM]` |
| Regime scaling | Adjust total exposure according to market trend and volatility | Exact implementation and thresholds `[TO CONFIRM]` |

For intuition, a momentum measure can be represented as:

\[
m_{i,t}(L) = \frac{P_{i,t}}{P_{i,t-L}} - 1,
\]

where \(P_{i,t}\) is the price of asset \(i\) at time \(t\), and \(L\) is the lookback window. Cross-sectional momentum compares this measure across assets; time-series momentum evaluates an asset's own trend. The implemented return definition, lookbacks, score transformations, and signal-combination rules must be confirmed from the code.

### Final strategy selection

A list of research candidates is not the same as a final trading strategy. Before submission, document:

- Final candidate or combination used in live trading: `[TO FILL]`.
- Momentum lookback windows and data-bar frequency: `[TO FILL]`.
- Whether absolute momentum is a filter, a score component, or both: `[TO FILL]`.
- Breakout and volatility-filter rules: `[TO FILL]`.
- Selection criterion and supporting validation results: `[TO FILL]`.

Explain which alternatives were rejected and why, using actual research results rather than retrospective explanations based on the holdout period.

## 2. How the portfolio is curated

Portfolio construction follows the decision chain below:

```text
Exchange-supported assets
    -> eligibility and volume filtering
    -> historical/live feature preparation
    -> candidate signal calculation
    -> asset ranking and selection
    -> target weights and exposure limits
    -> comparison with current holdings
    -> cost-aware rebalancing
    -> order execution and balance reconciliation
```

### 2.1 Eligible asset universe

The project discovers supported pairs through Roostoo's exchange-information interface and applies volume-based eligibility criteria. The resulting universe is configured in `config/universe.yaml`.

The distinction between eligible assets and selected holdings matters: a pair may be tradable without receiving a positive target weight.

Confirm the volume metric, eligibility threshold, refresh frequency, and mapping between historical Binance symbols and Roostoo pairs. Historical evaluation should also disclose whether it uses a fixed present-day universe or point-in-time membership; a present-day universe can introduce survivorship bias.

### 2.2 Selection and weighting

Eligible assets are evaluated using the configured strategy. The original project description includes equal-weight and risk-adjusted sizing, subject to position and total-exposure limits.

For an equal-weight portfolio, an illustrative target is:

\[
w_{i,t} = \frac{G_t}{N_t},
\]

where \(G_t\) is target gross exposure and \(N_t\) is the number of selected assets. Risk-adjusted sizing instead modifies allocation according to the implemented risk estimate. The exact risk-adjustment formula is `[TO CONFIRM]`; it should not be assumed to be inverse-volatility weighting without checking the engine.

`src/roostoo_quant/backtest/engine.py` is the documented portfolio-sizing and backtesting reference. The final submission must specify which weighting method is actually used, how caps are enforced, and whether residual capital remains in cash or is redistributed.

### 2.3 Exposure and cash allocation

The design scales gross exposure with market conditions, including trend and volatility regimes. Lower exposure leaves more capital uninvested rather than forcing full allocation at every rebalance.

Holding cash reduces the amount exposed to market movements, but does not guarantee a drawdown limit. The regime definition and mapping from regime to exposure must be documented numerically from the configuration or code.

### 2.4 Rebalancing and exits

The project describes configurable rebalancing intervals and a rank-buffer mechanism, `hysteresis_rank_buffer`, to reduce unnecessary position changes.

Conceptually, hysteresis lets an existing holding tolerate small ranking changes instead of immediately replacing it with a marginally higher-ranked asset. This is intended to reduce turnover; the exact entry and retention thresholds must be confirmed.

Record the implemented exit conditions: loss of eligibility, failure of a signal filter, movement outside the retention band, reduced target exposure, or another coded rule. Do not describe stop-loss or take-profit exits as implemented unless the repository contains them.

## 3. How the code reflects the investment idea

The architecture separates research decisions from exchange communication and runtime operations.

| Path | Documented responsibility | Relationship to the strategy |
| --- | --- | --- |
| `scripts/discover_universe.py` | Discover supported trading pairs | Defines the opportunity set |
| `config/universe.yaml` | Store asset-universe settings | Controls which assets may be considered |
| `scripts/download_data.py` | Download/cache historical Binance data | Supplies historical observations for research |
| `src/roostoo_quant/research/candidates.py` | Define research candidates | Converts the investment hypothesis into numerical signals |
| `config/default.yaml` | Store strategy and research configuration | Controls configurable signal, portfolio, and cost settings |
| `src/roostoo_quant/backtest/engine.py` | Simulate portfolio sizing and trading | Translates signals into historical holdings and returns |
| `scripts/run_backtest.py` | Run a research backtest | Evaluates a configured strategy |
| `scripts/run_full_research.py` | Run the broader research suite | Evaluates validation and robustness diagnostics |
| `src/roostoo_quant/live/` | Manage live loops and state | Applies trading decisions during operation |
| `src/roostoo_quant/roostoo/` | Handle REST requests and HMAC signing | Communicates with the competition exchange |
| `scripts/run_live.py` | Start live execution | Runs one cycle or continuous operation |

### Live execution flow

The documented live architecture includes exchange-data retrieval, strategy evaluation, order execution, and reconciliation of local positions with exchange balances.

The important implementation relationship is that the live system should translate target portfolio weights into required trades, rather than repeatedly buying selected assets without accounting for existing inventory. Confirm the exact implementation of this translation and the handling of outstanding orders, rejected orders, partial fills, rounding, and minimum order sizes.

A single-cycle live run is not documented as a dry run. Treat `run_live.py --once` as potentially order-producing.

## 4. Data sources and their limitations

The research pipeline uses cached Binance historical archives under `data/binance` to provide OHLCV observations. Live operation uses Roostoo exchange-information and ticker endpoints.

This creates an important evaluation limitation: research and execution use different venues. Historical Binance prices and volumes do not establish that the same fills or liquidity would have been available on Roostoo.

Before submission, document:

- Historical period, bar interval, and timezone: `[TO FILL]`.
- Price field used for each feature and simulated execution: `[TO FILL]`.
- Missing-bar, duplicate, and stale-data handling: `[TO FILL]`.
- Cross-venue symbol mapping and unavailable-asset treatment: `[TO FILL]`.
- How live features obtain sufficient history, including warm-up: `[TO FILL]`.

Ticker snapshots should not be described as equivalent to a complete OHLCV series without documenting how the required history is constructed.

## 5. Backtesting and research methodology

Run the documented research entry points:

```bash
python scripts/run_backtest.py
python scripts/run_full_research.py
```

### 5.1 Chronological evaluation

The original README describes walk-forward evaluation with training, validation, and holdout data. These periods serve different purposes:

- Training/research: develop candidate definitions and parameter choices.
- Validation: compare candidates and assess whether results persist outside the development period.
- Holdout: evaluate the frozen strategy without using its results to tune the model.

The final README should record the actual dates, window sizes, refitting schedule, candidate-selection metric, and whether the final holdout remained untouched during development.

### 5.2 Timing and look-ahead prevention

The documented framework shifts bar-close signals before execution to avoid using information that would not have been available when trading.

Confirm the precise sequence in the engine: when a bar becomes observable, when the signal is computed, which price is used for execution, and when costs are deducted. Using a closing price to form a signal and also assuming an earlier fill at that same close would require an explicit, defensible execution assumption.

### 5.3 Robustness diagnostics

The project describes:

- Block-bootstrap return diagnostics.
- Market-gap shock scenarios.
- Asset-staleness injection scenarios.

These diagnostics investigate sensitivity to alternative return paths and operational disruptions. They do not prove that all future risks are covered. Report scenario parameters and actual outcomes rather than only listing the test names.

### 5.4 Results to report

No numerical backtest results were supplied with the source README. Complete the table below from reproducible reports.

| Metric | Validation | Final holdout |
| --- | --- | --- |
| Evaluation dates | `[TO FILL]` | `[TO FILL]` |
| Net total return | `[TO FILL]` | `[TO FILL]` |
| Annualized volatility | `[TO FILL]` | `[TO FILL]` |
| Sharpe ratio | `[TO FILL]` | `[TO FILL]` |
| Maximum drawdown | `[TO FILL]` | `[TO FILL]` |
| Turnover, with definition | `[TO FILL]` | `[TO FILL]` |
| Fees and modeled slippage | `[TO FILL]` | `[TO FILL]` |
| Number of trades | `[TO FILL]` | `[TO FILL]` |
| Benchmark return and assumptions | `[TO FILL]` | `[TO FILL]` |

Specify annualization conventions, the risk-free-rate assumption for Sharpe, and whether benchmark returns include costs. Keep backtest results separate from realized live performance.

## 6. Maker/taker fees and slippage

The original project description states that maker/taker fees and slippage are configurable and applied to traded notional. The final submission must disclose the actual configured rates and how each simulated or live fill is classified.

### 6.1 Cost accounting

An illustrative per-fill fee calculation is:

\[
\mathrm{Fee}_j = |q_j p_j| f_j,
\]

where \(q_j\) is filled quantity, \(p_j\) is fill price, and \(f_j\) is the applicable maker or taker fee rate. Charges should be based on traded notional, including both purchases and sales, rather than on the whole portfolio at every timestamp.

If slippage is modeled separately in basis points, an illustrative cost is:

\[
\mathrm{SlippageCost}_j = |q_j p_j|\frac{s_j}{10{,}000}.
\]

These expressions explain the accounting concept; confirm whether the engine represents slippage as a cash deduction or as an adjusted execution price. Do not charge it twice.

### 6.2 Maker versus taker assumptions

Do not assume that every limit order earns maker treatment. The relevant distinction is whether execution provides liquidity or immediately consumes available liquidity, subject to the venue's classification.

Document the following from the actual implementation:

| Setting | Value or evidence required |
| --- | --- |
| Maker fee rate | `[TO FILL]` |
| Taker fee rate | `[TO FILL]` |
| Simulated maker/taker classification rule | `[TO FILL]` |
| Live order types and fee handling | `[TO FILL]` |
| Slippage assumption in basis points | `[TO FILL]` |
| Partial-fill and unfilled-order treatment | `[TO FILL]` |

If passive fills are not modeled, disclose that limitation and state the fee assumption used. Do not present a best-case maker rate as a demonstrated execution outcome.

### 6.3 Turnover control

Rank buffers and configurable rebalancing reduce the frequency of marginal portfolio changes. Evaluate their benefit after costs: lower turnover is useful only if the retained portfolio remains effective. Report net performance alongside gross performance to show the contribution of transaction friction.

## 7. Risk management

The original README describes exposure limits, position caps, an open-position limit, turnover controls, regime scaling, and stress diagnostics.

| Control | Purpose | Required implementation detail |
| --- | --- | --- |
| Gross-exposure cap | Limit total market exposure | Maximum value and enforcement point |
| Per-asset weight cap | Limit concentration in one asset | Maximum weight and redistribution rule |
| Open-position limit | Bound the number of holdings | Maximum count and selection rule |
| Regime scaling | Reduce allocation in specified conditions | Regime thresholds and exposure multipliers |
| Rank hysteresis | Reduce repeated switching | Entry/retention logic and buffer value |
| Balance reconciliation | Keep local inventory aligned with the exchange | Frequency and mismatch handling |
| Stress diagnostics | Evaluate adverse scenarios | Shock sizes, staleness assumptions, and outcomes |

These controls reduce selected risks; they do not eliminate correlated losses, liquidity risk, sudden gaps, or execution failures. An open-position limit alone does not guarantee diversification.

Operational handling of stale data, API failures, duplicate submissions, and restart recovery should be described only after checking the code. A kill switch, drawdown-triggered shutdown, or stop-loss is not established by the supplied README and should not be claimed without evidence.

## 8. Setup, credentials, and operation

Copy `.env.example` to `.env` and set the required credentials:

```dotenv
ROOSTOO_API_KEY=your_api_key_here
ROOSTOO_SECRET_KEY=your_secret_key_here
ROOSTOO_BASE_URL=https://mock-api.roostoo.com
```

API credentials must not be hardcoded into strategy code or committed to version control. Document the required Python version and dependency-installation command from the repository: `[TO FILL]`.

### Read-only preflight

```bash
python scripts/preflight_competition.py
```

The original README identifies preflight as read-only, with no order submission. It checks credentials, clock, and venue connectivity. Keep the host clock synchronized for authenticated requests.

### Research workflow

```bash
python scripts/discover_universe.py
python scripts/download_data.py
python scripts/run_backtest.py
python scripts/run_full_research.py
```

### Live operation — may place orders

```bash
# One live cycle; do not assume this is a dry run.
python scripts/run_live.py --once

# Continuous live operation.
python scripts/run_live.py
```

Confirm the intended competition endpoint and review strategy/risk configuration before starting live execution.

## 9. Reproducibility and judge-facing evidence

To make the strategy assessable, include or identify:

1. The repository commit used for the submitted results.
2. The exact configuration for the final strategy, including universe, signals, weights, risk limits, and costs.
3. The historical-data period and data-preparation procedure.
4. Commands needed to reproduce each reported experiment.
5. Paths to generated reports and logs: `[TO FILL]`.
6. Gross and net performance, validation/holdout separation, and benchmark assumptions.
7. Example live decisions showing the connection from signal to target weight, required trade, fill, fee, and reconciled balance.

A useful decision example follows this structure:

```text
Timestamp and information available
    -> eligible assets and scores
    -> selected holdings and target exposure
    -> constrained target weights
    -> differences from current holdings
    -> submitted orders and execution assumptions
    -> fills, fees, and reconciled balances
```

Populate this example with an actual logged cycle, not fabricated values.

## 10. Limitations and interpretation

- The project targets the Roostoo mock competition environment; results should not be treated as demonstrated execution performance on another exchange.
- Binance historical observations are a research proxy, not a guarantee of Roostoo liquidity or matching fills.
- Momentum may fail during reversals or sideways conditions; volatility and regime rules may also react late.
- Repeated candidate testing can overfit validation data even when basic look-ahead bias is avoided.
- Simulated costs and fills remain assumptions unless reconciled with execution evidence.
- Past backtest performance does not guarantee future live performance.

The submission should be evaluated on the coherence of its hypothesis, the fidelity of its implementation, the credibility of its out-of-sample evaluation, and the transparency of its execution and risk assumptions—not on unsupported performance claims.
