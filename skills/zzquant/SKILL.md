---
name: zzquant
description: Senior quantitative trading and systematic investing expertise. Covers mean reversion, momentum, trend following, pairs trading, statistical arbitrage, factor investing, market making (Avellaneda-Stoikov, VPIN, PFOF flow segmentation), options premium selling (tastytrade, VRP, Greeks), gamma scalping, volatility arbitrage (GARCH, HAR-RV, rough vol), dispersion, volatility surface (SABR, SVI, skew, term structure), variance and volatility swaps, ML alpha (purged k-fold, CPCV caveats, SHAP, signature methods), regime detection (HMM, Markov switching), LLM-powered alpha research, LLM-in-the-loop trading operations (nightly/midday review, JSON contracts, outcome feedback), exit rule taxonomy (target/stop/time/signal/regime, trailing, credit-multiple, delta/gamma breach), backtest determinism (seeds, point-in-time, replay-live equivalence, survivorship), multi-strategy orchestration (allocation, risk parity, correlation, kill switches, haircut expansion), data quality (corporate actions, restatement, IVR vs IVP), event-driven and calendar risk (earnings, FOMC, dividends, index rebal, expiration), position lifecycle and OMS (fills, reconciliation, reconnect, idempotency), execution algorithms and realities (TWAP, VWAP, IS, SOR, microstructure, fees, broker quirks, Section 1256), post-2020 market structure (0DTE, dealer gamma/charm/vanna, GEX, PFOF), options operational risks (pin risk, early assignment, dividend exercise, IVR vs IVP), performance metrics beyond Sharpe (Sortino, Calmar, CVaR, drawdown duration, DSR caveats), position sizing, and strategy selection by regime. Use when designing a strategy, reviewing a backtest, debugging live PnL, structuring an options position, sizing a portfolio, forecasting volatility, building an execution algo, choosing a strategy for the current regime, operating an LLM inside a trading loop, running multiple strategies in one book, hardening a data pipeline, or answering "is this quant approach sound?".
---

# zzquant — Senior Quantitative Trading & Systematic Investing

Pure quant guidance. Covers strategy mechanics (what/how), math foundations (why), regime fit, risk controls, and execution. Project-specific conventions (brain logic, regime router, strategy configs) live in the project's CLAUDE.md, not here.

## When to use

Invoke whenever you are:

- Designing, reviewing, or debugging a systematic trading strategy
- Choosing between strategies for the current market regime
- Structuring an options position or managing Greeks
- Forecasting volatility or trading IV vs RV
- Building a factor portfolio or running stat arb
- Selecting execution algorithms or analyzing microstructure
- Evaluating an ML alpha pipeline (CV scheme, feature leakage, decay)
- Sizing positions, setting stops, or deciding on leverage
- Answering "would an institutional quant ship this?"

## How to navigate this skill

Each topic below is a self-contained reference file. Open only the files relevant to the current task — do not load them all. **Start with the topic that matches the concrete artifact you are touching** (e.g., editing a covered-call sizing rule → `options_premium.md`; sanity-checking a backtest → this file's §Cross-cutting principles first, then `ml_alpha.md`).

| Topic | File | When to open |
|---|---|---|
| Backtest soundness, leakage, regime fragility, decay | *this file* §Cross-cutting principles | "Is this result trustworthy?" — read before any specific reference |
| Mean reversion: OU, z-score, ADF, half-life, Bollinger, RSI, Kalman | `references/mean_reversion.md` | Range-bound signals, overshoot/revert setups, stationarity checks |
| Momentum and trend following: XS vs TS, MA crossover, dual momentum, crashes | `references/momentum.md` | Trend signals, lookback selection, momentum crash risk |
| Pairs trading: correlation vs cointegration, spread z-score, Kalman β | `references/pairs_trading.md` | Two-leg relative value, cointegration tests, pair selection |
| Statistical arbitrage: factor residuals, dollar/beta/sector neutral, Quant Quake | `references/stat_arb.md` | Portfolio-scale mean reversion, neutrality constraints |
| Factor investing: Fama-French, HML/UMD/QMJ, factor timing, alpha decay | `references/factor_investing.md` | Factor construction, multi-factor composites, decay management |
| Market making: Avellaneda-Stoikov, inventory skew, adverse selection, VPIN | `references/market_making.md` | Quote placement, inventory risk, HFT infrastructure |
| Options premium selling: tastytrade rules, VRP, IV rank, DTE, delta, management | `references/options_premium.md` | Short strangles/condors/verticals, 50% profit targets, 21 DTE rolls |
| Gamma scalping: delta hedging, realized vs implied vol, rebalance frequency | `references/gamma_scalping.md` | Long gamma + delta hedge, path-dependent PnL, MM-style scalping |
| Volatility arbitrage: IV vs RV forecasting, GARCH, HAR-RV, variance/vol swaps | `references/vol_arb.md` | Vol forecasting, VRP harvesting, variance swap replication |
| Dispersion trading: index vs single-name vol, implied correlation, PCA selection | `references/dispersion.md` | Correlation premium, index-versus-basket vega |
| Volatility surface: skew, smile, term structure, SABR, SVI, rough vol | `references/vol_surface.md` | Surface calibration, skew/term trades, no-arb constraints |
| ML alpha: model hierarchy, purged k-fold, CPCV, SHAP, walk-forward, decay | `references/ml_alpha.md` | ML signal research, CV leakage checks, feature engineering |
| Regime detection: HMM, Markov switching, clustering, change-point, adaptive allocation | `references/regime_detection.md` | Regime-aware strategy weighting, state detection, meta-strategies |
| LLM-powered alpha: Alpha-GPT, AlphaAgent, multi-agent, originality enforcement | `references/llm_alpha.md` | LLM factor mining, sentiment pipelines, agent frameworks |
| Execution algos: TWAP, VWAP, IS, POV, SOR, order book imbalance, VPIN, Kyle's lambda | `references/execution.md` | Minimizing impact, venue routing, slippage control |
| Position lifecycle: OMS, fills, reconciliation, reconnect, idempotency, corporate actions | `references/position_lifecycle.md` | Anything downstream of order submission; phantom positions; reconnect logic |
| Exit rule taxonomy: target/stop/time/signal/regime, trailing, credit-multiple, testing | `references/exit_rules.md` | Designing or debugging exits; comparing exit rule families; stop placement |
| Multi-strategy orchestration: allocation, risk parity, correlation, kill switches, attribution | `references/multi_strategy.md` | Running N strategies in one book; capital allocation; blast containment |
| Event-driven strategies and calendar risk: earnings, FOMC, dividends, expirations, index rebal | `references/events.md` | Trading around (or avoiding) scheduled events; earnings IV crush; calendar filters |
| Data quality defenses: corporate actions, survivorship, restatement, stale quotes, IV traps | `references/data_quality.md` | Hardening an ingest pipeline; debugging backtest-vs-live divergence |
| Backtest determinism: seeds, point-in-time plumbing, state hashing, replay-live equivalence | `references/backtest_determinism.md` | Backtest *infrastructure* — reproducibility, PIT plumbing, replay-live equivalence. For CV / leakage / decay, see `ml_alpha.md` |
| LLM-in-the-loop operations: nightly/midday review, JSON contracts, safety patterns, outcome feedback | `references/llm_in_the_loop.md` | Running an LLM as part of a trading loop (not for research — see llm_alpha.md) |
| Post-2020 market structure: 0DTE, dealer gamma/charm/vanna, GEX, PFOF flow segmentation | `references/market_structure_2020s.md` | Anything intraday SPX, MM strategy design, VRP magnitude calibration. Read alongside `vol_surface.md` and `market_making.md` |
| Options operational risks: pin, early assignment, dividend exercise, IVR vs IVP, what backtests don't show | `references/options_operational_risks.md` | Live options trading; the operational traps that bite junior traders first |
| Performance metrics beyond Sharpe: Sortino, Calmar, CVaR, drawdown duration, Omega, DSR caveats | `references/metrics_beyond_sharpe.md` | Evaluating any non-Gaussian strategy; choosing the right primary metric for short vol, stat arb, trend |
| Strategy selection decision tree by regime, edge, capital, constraints | `references/decision_framework.md` | Choosing between strategies; sanity-checking allocation decisions |

Open `decision_framework.md` when the question is "which strategy, given this market?".

## Cross-cutting principles

These show up in every reference file; worth internalizing before any strategy work:

1. **No look-ahead.** Every feature at time t must use data available strictly before t. Walk-forward validation only. Purged k-fold (López de Prado) for CV.
2. **Measure before optimizing.** Prove a result with a benchmark / backtest before tuning knobs. Sharpe ≥ 0.5 OOS before live deployment.
3. **Regimes change; edges decay.** Factor half-life is typically 6–18 months. Re-evaluate, rotate, or retire signals.
4. **Path matters.** Delta-hedged option PnL is path-dependent; a correct average view can still produce brutal paths. Stress-test the path, not just the expectation.
5. **Transaction costs are not frictionless.** Stat arb and gamma scalping live and die on execution. Model slippage, impact, and spread explicitly.
6. **Correlations spike in crises.** Diversification is a low-vol phenomenon. Dispersion, stat arb, and pairs all fail together when correlations jump to 1.
7. **Constraint before optimization.** Dollar/beta/sector neutrality, position caps, leverage limits are load-bearing. Optimization without constraints produces concentrated crash-prone portfolios.
8. **The VRP is the deepest edge in options, but it is not free money.** It compensates for negative gamma, tail risk, and forced-unwind scenarios. Respect the 2× credit stop and the 21 DTE roll.

## Operating rules for this skill

1. **Topic-first, not tier-first.** Open the reference by the artifact you're touching, not by "is this intermediate or advanced". Tiers are a rough complexity ordering, not a routing signal.
2. **Don't inline a reference into SKILL.md.** If a topic file is relevant, read it; don't paraphrase from memory.
3. **Don't fabricate thresholds or rules.** If a number (IV rank cutoff, delta target, DTE window, stop multiplier) isn't in the cited source, flag it as opinion. Do not attach made-up figures to named methodologies (Avellaneda-Stoikov, Fama-French, tastytrade) that they never prescribed.
4. **Project CLAUDE.md overrides this skill.** If AlphaSeeker (or any project) encodes a different rule, the project wins. Example: AlphaSeeker's brain annotations and regime presets override any generic factor-timing advice here.
5. **Formulas > prose.** When explaining a mechanic, show the equation and a minimal worked example; don't describe it abstractly.
6. **Distinguish premise from edge.** A premise ("prices mean-revert") is not an edge. An edge is a measured, out-of-sample Sharpe net of costs. Always be explicit about which one you're discussing.

## Canonical sources

Books
- Ernest P. Chan — *Algorithmic Trading* / *Quantitative Trading*
- Marcos López de Prado — *Advances in Financial Machine Learning*
- Sheldon Natenberg — *Option Volatility and Pricing*
- Colin Bennett — *Trading Volatility*
- Jim Gatheral — *The Volatility Surface*
- Kissell — *The Science of Algorithmic Trading and Portfolio Management*

Seminal papers
- Jegadeesh & Titman (1993) — momentum
- Moskowitz, Ooi, Pedersen (2012) — time-series momentum
- Gatev, Goetzmann, Rouwenhorst (2006) — pairs trading
- Avellaneda & Lee (2010) — stat arb in US equities
- Fama & French (1993, 2015) — 3-factor and 5-factor models
- Avellaneda & Stoikov (2008) — HFT market making
- Hamilton (1989) — regime switching
- Almgren & Chriss (2001) — optimal execution
- Derman (1999) — variance/volatility swaps
- Hagan et al. (2002) — SABR
- Gatheral (2004) — SVI

Ongoing references
- AQR research library — https://www.aqr.com/Insights
- Quantpedia — https://quantpedia.com/
- tastytrade Learn Center — https://tastytrade.com/learn/trading-products/options/
- Microsoft Qlib — https://github.com/microsoft/qlib
- gs-quant — https://github.com/goldmansachs/gs-quant
- awesome-quant — https://github.com/wilsonfreitas/awesome-quant
- awesome-quant-ai — https://github.com/leoncuhk/awesome-quant-ai
- ArXiv q-fin — https://arxiv.org/list/q-fin/recent
- AlphaAgent (KDD 2025), Alpha-GPT (EMNLP 2025) — LLM-driven alpha mining
