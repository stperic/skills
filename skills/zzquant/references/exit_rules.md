# Exit Rules: Taxonomy & Design

## Why this file exists

Most strategy literature obsesses over entries. Exits determine the P&L distribution. Two strategies with identical entries and different exit rules can differ by 5× in Sharpe, and the worst drawdown usually traces to the exit rule — not the signal.

This file is a taxonomy of exit rule *families*, what each optimizes for, and how they fail. Use it when designing a new strategy, debugging a losing P&L, or comparing exit rule variants in backtest.

## The five exit families

Every exit rule is one (or a combination) of these:

1. **Target-based** — exit at a predetermined profit level
2. **Stop-based** — exit at a predetermined loss level
3. **Time-based** — exit at a predetermined time/bar count
4. **Signal-based** — exit when the entry signal reverses or weakens
5. **Regime-based** — exit when the environment the strategy needs disappears

Combinations: target + stop + time is the baseline for most systematic strategies. Adding signal or regime exits is where sophistication lives.

## 1. Target-based exits

### Fixed fraction of maximum favorable excursion (MFE)
Exit at X% of theoretical max profit. Examples:
- **50% of max credit** (tastytrade short premium) — mechanical, robust, gives up upside for higher hit rate
- **25% of max credit** for low-premium iron condors (quicker turnover, protects against late gamma)
- **80% of theoretical target** for wide-body structures

### Absolute dollar targets
- **Dollar profit ceiling**: exit when unrealized P&L ≥ $X per position
- Used when position sizing is uneven across names; normalizes risk per trade in dollar terms
- Caution: converts a percentage rule into an absolute one; breaks if position size changes mid-trade

### Multiple-R targets
- Exit at `N × initial_risk` where R = distance to stop
- Popular in trend following: "take profit at 3R" means 3× the risk you accepted
- Ties profit target to risk taken, not arbitrary price level

### Volatility-scaled targets
- Target = `entry + k × σ`, scales with market volatility
- Prevents "one-size" targets being trivial in high-vol, unreachable in low-vol

## 2. Stop-based exits

### Fixed percentage / fixed dollar
- Simple, hardcoded ("exit at -2%" or "exit at -$500")
- Works if position sizes are normalized; fails under variable size
- Common bug: forgetting to update stops when position is added to

### ATR-based stops
```
stop = entry − k · ATR(n)
```
- Typical `k` ∈ {1.5, 2, 3}, `n` ∈ {14, 20}
- Scales with recent volatility — widens in noisy markets, tightens in calm
- Standard in trend following

### Credit-multiple stops (short premium)
- Exit short option when loss = `M × initial_credit`
- `M = 2` (tastytrade default) balances hit rate vs catastrophic loss
- `M = 3` or `4` for wider-wing structures where a 2× stop is inside normal vol noise
- **Hard failure mode 1**: gap opens past the stop; stop becomes a market order at a worse price
- **Hard failure mode 2 (often worse)**: dollar-multiple stops on **undefined-risk** shorts (strangles) routinely fire on **IV spikes without spot moves** — the position is still inside its theoretical breakevens but MTM is negative because vega exploded. The stop converts a temporary mark-to-market loss into a realized loss at the worst fill of the day. **Experienced short-vol desks do not use credit-multiple stops on undefined-risk structures.** Use either (a) defined-risk structures (iron condors) where the wing caps tail loss, or (b) delta/gamma breach triggers (see below) that actually measure positional risk rather than dollar P&L.

### Trailing stops
- **Fixed trail**: `stop = max_price_seen − k`
- **ATR trail**: `stop = max_price_seen − k · ATR`
- **Chandelier exit**: `stop = max_high_n − k · ATR_n`
- Locks in profit once in the money, lets winners run
- Biggest weakness: whipsaw in mean-reverting conditions; once stopped out, re-entry discipline is hard

### Parabolic SAR
- Accelerating stop that tightens toward price
- Good for capturing the tail end of a trend; bad in chop

### Delta-based stops (options)
- Exit when option delta exceeds threshold (e.g., short 16Δ put hits 40Δ = losing)
- Proxy for directional move + vol expansion combined
- More informative than dollar P&L for short premium — **the preferred trigger for undefined-risk shorts** because it actually measures position risk rather than mark-to-market noise from vega
- Common variants: short delta breach (e.g., short 16Δ → exit at 30Δ), short delta/gamma combination (delta breach OR gamma above threshold), tested-side delta (only the side under pressure, not both)

### Max dollar loss stops
- Absolute cap: "never lose more than $500 on any trade"
- Simple to enforce; disconnects from market-relative risk
- Useful as a backstop in addition to the primary stop, not as the primary rule

## 3. Time-based exits

### Fixed holding period
- Exit after N bars / days regardless of P&L
- Used when alpha has known decay horizon (post-earnings drift, post-merger arb, mean reversion)
- Caps opportunity cost of dead trades

### DTE-based (options)
- **21 DTE roll / close** (tastytrade) — exits before gamma acceleration
- **0 DTE** — time-of-day exit (commonly ~15:30 ET; never hold into settlement chaos)
- **30 DTE fresh entries only** — filter not exit

### Half-life time stop (mean reversion)
- Exit at `t = 2 × half_life` if spread hasn't reverted
- Matches time stop to the theoretical decay scale of the signal

## 4. Signal-based exits

### Reverse signal
- Exit when the entry condition flips (z-score crosses back, MA recrosses, RSI returns to neutral)
- Cleanest when entries and exits use the same indicator
- Failure mode: weakened but not reversed signal lets losers fester

### Weakening signal
- Exit when signal strength decays (|z| < 0.5, RSI returns to 40–60)
- Captures partial edge when full reversal is too slow
- Use for partial exits ("scale out")

### Opposite-strategy signal
- Momentum exit: exit when a mean-reversion signal fires
- Mean-reversion exit: exit when a trend signal fires
- Elegant symmetry; hard to tune

## 5. Regime-based exits

### Vol-regime exit
- Exit short premium when VIX term structure inverts
- Exit trend when realized vol spikes past threshold
- Exit mean-reversion when ADF fails (series lost stationarity)

### Correlation-spike exit
- Stat arb / pairs / dispersion: exit entire book when cross-correlation exceeds threshold (regime shift to "everything correlated")
- The "get flat on panic" rule — unfashionable but has saved several funds

### Cointegration-break exit
- Pairs: exit when rolling ADF on residuals fails at p > 0.10 for N consecutive days
- Prevents riding a broken pair all the way to infinity

## Combining rules

Production systems almost always combine families:

```
exit = target OR stop OR time OR regime_break
```

First-to-fire wins. Common production stack for short premium:
1. Profit target (50% credit)
2. Stop loss (2× credit)
3. Time stop (21 DTE)
4. Regime break (VIX spike, signal halt)

## Testing exit rules

Exit rules must be tested *independently* of entries. Common approach:

1. Hold entries constant
2. Grid-search exit rule parameters (stop multiple, target fraction, time limit)
3. Plot heatmap of Sharpe / max drawdown / win rate across the grid
4. Check for *stability*: a good exit rule has a broad plateau on the heatmap. A single peak is overfitting.
5. Confirm with walk-forward or CPCV (see `ml_alpha.md`)

Common anti-patterns:
- Tuning target and stop jointly until backtest Sharpe maxes — almost certainly overfit
- Using one-exit-rule-per-trade optimization — massively overfit
- Tuning exits on the same data used for entries — double-dipping

## Failure modes by family

| Family | Primary failure |
|---|---|
| Target | Leaves upside on the table; late-entry winners exit too fast |
| Stop | Whipsaws in chop; gap risk past the stop; correlated stops firing at once |
| Time | Exits still-profitable trades; arbitrary cutoff |
| Signal | Never triggers (signal never reverses); lags the turn |
| Regime | Defines "regime" too loosely; false halts |

## Risk controls specific to exits

1. **Exit logic ordering**: document which rule wins when multiple fire same bar (stop beats target; target beats time)
2. **Slippage budgeting**: stops are usually market orders — model slippage explicitly, do not assume limit fill
3. **Gap protection**: for overnight risk, consider closing vs rolling based on expected gap distribution
4. **Pre-exit hedging**: if close is slow (e.g., illiquid options), pre-hedge before unwind
5. **Bifurcated books**: once a trade hits the stop/target, move it to a "close-only" state; no new entries reference it
6. **Post-exit dead-zone**: don't re-enter the same name immediately after a stop (whipsaw filter)

## Exit rules ≠ exit *path*

An exit rule says *when* to exit. An exit *path* says *how*. These are separate concerns:

- Rule: "close at 50% max credit"
- Path: "close in 3 slices using midpoint limits with 10s timeout → market on last slice"

The path matters as much as the rule for any position size above minimal. See `execution.md` for execution algorithms and `position_lifecycle.md` for fill handling.

## References

- Van Tharp — *Trade Your Way to Financial Freedom* (R-multiples, exit as a separate discipline)
- Perry Kaufman — *Trading Systems and Methods* (comprehensive exit rule catalog)
- tastytrade research on mechanical management (50% / 21 DTE empirical studies)
- Jaffarian & Walsh — *The Leveraged Exchange-Traded Funds Manual* (stop placement in leveraged products)
- Chan — *Algorithmic Trading* (exits for mean reversion and momentum)
