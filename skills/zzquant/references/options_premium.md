# Options Premium Selling (tastytrade Methodology)

> **Regime-anchor disclaimer**: numerical thresholds in this file (VRP magnitude, IV Rank cutoff, 50% profit target, 2× stop, 16–30 delta, 30–50 DTE) are *historical anchors* derived from pre-2018 SPY/IWM strangle backtests. Treat each as a starting point to interrogate against current regime data, not as a live parameter. See `market_structure_2020s.md` for what changed post-2018 (VRP compression, 0DTE, dealer flows) and `options_operational_risks.md` for what backtests don't show (pin risk, early assignment, IVR vs IVP).

## Premise

Systematically sell options premium to harvest the **Volatility Risk Premium (VRP)**: implied volatility is structurally higher than subsequent realized volatility because institutions persistently buy puts for tail protection.

Historically (multi-decade unconditional mean), SPX IV has exceeded RV by ~2–4 vol points. **The conditional, current-regime number is materially lower** — post-2018, with vol-of-vol compression and the 0DTE explosion, recent rolling VRP on SPX has averaged closer to 1–1.5 vol points with frequent negative episodes (Feb 2018, Mar 2020, parts of 2023–24). The structural edge still exists; the magnitude shrank. You are paid this premium for bearing negative gamma and tail risk. See `market_structure_2020s.md` for the post-2018 picture.

## The VRP in one equation

For a delta-hedged short option in *continuous-time, frictionless* settings:
```
daily_P&L ≈ 0.5 · Γ · S² · (σ²_implied − σ²_realized) · dt
```
Positive on average because `σ_implied > σ_realized`. **This is a limiting-case identity, not a P&L predictor in real markets.** Real P&L depends on hedge frequency, bid-ask cost on each rebalance, gap jumps the continuous formula doesn't see, and the correlation between Γ and the realized-variance path. A correct continuous-time average can co-exist with brutal realized paths. Treat the formula as an intuition for the *sign* of edge, not as a forecast of magnitude.

## tastytrade entry rules

| Rule | Value | Why |
|---|---|---|
| **IV Rank / Percentile** | sizing input, not gate | IV Percentile is the statistically better normalizer; IVR > 50 was the original tastytrade gate but it produces dead-zones for ~6 months after vol shocks (March 2020, August 2024). See `options_operational_risks.md` §IV Rank vs IV Percentile |
| **DTE** | 30–50 | Sweet spot for theta decay acceleration |
| **Delta** | 16–30 | ~1 SD OTM, 68–84% probability of profit |
| **Buying power per trade** | ≤ 3–5% | Cap per-trade concentration |
| **One trade per underlying** | yes | Avoid correlated concentration on a single name |

IV Rank = `(current_IV − 52wk_min_IV) / (52wk_max_IV − 52wk_min_IV)`. IV Percentile = fraction of days in the past year with lower IV. IVR is preferred by tastytrade because it is faster to compute, **but IVP is the statistically correct distribution-free choice** — IVR is dominated by extremes and goes to zero mechanically after a vol shock rolls off the 52-week window. Use IVP for sizing decisions; compute IVR only for legacy compatibility.

## tastytrade management rules

| Rule | Action | Why |
|---|---|---|
| **Profit target** | Close at 50% of max credit | Mechanical — locks in majority of edge, frees buying power |
| **Roll point** | 21 DTE | Avoid gamma acceleration near expiration |
| **Defense** | Roll for credit (later exp, same or new strike) | Extend duration to let it work out |
| **Stop loss** | 2× credit (defined-risk only); for *undefined-risk* shorts prefer delta/gamma breach | Dollar-multiple stops on undefined-risk shorts often fire on IV spikes *without* spot moves and convert MTM losses into realized losses at the worst fill of the day. Experienced short-vol desks use defined-risk structures *or* delta/gamma breach triggers, not dollar stops on uncapped strangles |

These rules are not theoretically derived — they are the empirical outcome of tastytrade's backtesting across 15+ years of SPY/IWM strangle data, mostly in the low-vol 2010s. Mechanical management outperformed discretionary management *in their backtested regime*. **Caveats**: (1) the 50% profit rule generalizes poorly to iron condors (25–35% is closer for narrow wings) and is meaningless for 0DTE; (2) "mechanical beats discretionary" is sample-specific — discretionary traders who flatten into vol spikes have outperformed mechanical ones across 2018, 2020, and 2022 (while plenty of others have blown up). The honest statement: mechanical rules dominate *within their backtested regime*; regime-aware overlays help outside it.

## Strategy menu

| Strategy | Direction | Risk | When |
|---|---|---|---|
| Short strangle | Neutral | Undefined | High IV, liquid, PM account |
| Iron condor | Neutral | Defined | Moderate IV, IRA-friendly |
| Short put vertical | Bullish | Defined | Bullish bias, moderate IV |
| Short call vertical | Bearish | Defined | Bearish bias, moderate IV |
| Iron butterfly | Neutral | Defined | Very high IV, expecting pin |
| Jade lizard | Bullish | Partially defined | Bullish, no upside risk |
| Calendar spread | Neutral | Defined | Low near-term IV, higher back-month |
| Diagonal spread | Directional | Defined | Directional + time decay |
| Covered call / CSP | Mild bullish | Stock | Income on existing positions |

**Jade lizard**: short put + short call spread, credit ≥ call spread width → zero upside risk. Specific structure worth knowing by name.

## Greeks management (portfolio level)

- **Delta**: keep portfolio delta near zero, beta-weighted to SPY. E.g., `Σ delta_i · β_i ≈ 0`.
- **Theta**: positive theta is the point. Aim for 0.1–0.2% of portfolio per day (tastytrade typical).
- **Vega**: negative vega — short premium benefits from IV contraction. Biggest P&L swings come from vol changes, not price.
- **Gamma**: negative gamma is the biggest risk; accelerates near expiration. Root cause of the 21 DTE roll rule.

**Beta-weighted delta**: `delta_i · β_i` normalizes per-underlying delta to SPY-equivalent units. Summing gives portfolio exposure expressed as SPY shares.

## Sizing

- **Per trade**: ≤ 3–5% of portfolio buying power
- **Per underlying**: one trade at a time (avoids correlated stacking on the same name)
- **Total short premium**: target a risk budget (e.g., max loss in a 3-sigma move ≤ X% of NAV)
- **Correlation**: cap the number of correlated short strangles (all tech, all semis, etc.)

## When it works, when it fails

**Works**: high-IV environments, range-bound markets, post-shock IV crush, mechanical discipline.

**Fails**:
- **Vol expansions**: 2018 vol-mageddon, March 2020, 2022 bear market. Short vol gets crushed as IV and RV both spike.
- **Gap risk**: overnight news, earnings, Fed announcements can move past stops before they trigger.
- **Discretionary meddling**: the rules work because they're mechanical. Adjustments typically lose alpha.
- **Over-concentration**: one bad underlying can wipe months of premium.

## Risk controls

1. **Defined-risk only in small accounts** — uncapped strangles need PM margin and a tail-hedge program
2. **Tail hedges**: buy cheap far OTM puts on SPX as overlay (costs ~10–15% of premium harvest)
3. **Event avoidance**: no new positions into FOMC, CPI, earnings without a specific thesis
4. **Regime filter**: some operators pause new trades when VIX term structure inverts (contango → backwardation)
5. **Correlation cap**: diversify across sectors, not just names

## Python / tools

- `py_vollib` — Black-Scholes pricing, Greeks
- `QuantLib-Python` — full options pricing library
- tastytrade backtesting tool — native to the broker, used internally
- `pandas_datareader` — historical chains (limited)
- AlphaDB (for this project) — historical options chains

## Related files in this skill

- `market_structure_2020s.md` — 0DTE, dealer gamma flows, VRP compression
- `options_operational_risks.md` — pin risk, early assignment, dividend exercise, IVR vs IVP, what backtests can't see
- `exit_rules.md` — exit rule taxonomy (target/stop/time/signal/regime), delta-breach stops
- `metrics_beyond_sharpe.md` — why Sortino + CVaR + drawdown duration matter more than Sharpe for short-vol P&L
- `events.md` — earnings, FOMC, dividend dates as event-driven risks

## References

- tastytrade Learn Center — https://tastytrade.com/learn/trading-products/options/
- Natenberg — *Option Volatility and Pricing*
- Colin Bennett — *Trading Volatility*
- Quantpedia — "Volatility Risk Premium Effect" (curated VRP papers)
- Cboe research on VIX futures term structure
