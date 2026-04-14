# Gamma Scalping

## Premise

Maintain a delta-neutral options position and repeatedly rebalance the delta hedge to harvest oscillation. Profits when **realized vol exceeds implied vol** at entry. The long-gamma mirror image of premium selling.

## Setup

1. Buy an ATM straddle (or strangle): long gamma, long vega, short theta
2. Delta-hedge with the underlying (stock or futures)
3. As price rises, position goes delta-positive → sell underlying to re-neutralize
4. As price falls, position goes delta-negative → buy underlying
5. Each round-trip locks in a small scalping profit

Net P&L = gamma scalping gains − theta decay − transaction costs.

## The core math

For an infinitesimal time step with a delta-hedged long option (continuous-time, frictionless):
```
dP&L ≈ 0.5 · Γ · (dS)² − Θ · dt
```

Aggregating over a day:
```
daily_P&L ≈ 0.5 · Γ · S² · (σ²_realized − σ²_implied_at_entry) · dt
```

Profitable iff `σ²_realized > σ²_implied_at_entry`.

> **Caveat**: this is a continuous-time identity (El Karoui-Jeanblanc-Shreve 1998), not a real-world P&L predictor. Live P&L depends on the rebalance frequency you choose, the bid-ask cost on each rebalance, gap jumps the continuous formula misses, and the path correlation between Γ and the realized variance you actually capture. The "realized variance" term is also sensitive to the sampling interval — finer sampling captures more variance but pays more in transaction costs. Treat the formula as an intuition for *sign* and *direction*, not a forecast of magnitude.

Key properties:
- **Path-dependent**: more oscillations at the same realized vol = more scalping income. A straight-line move realizes vol but doesn't pay scalping.
- **Gamma decays to zero away from ATM**: as underlying drifts, gamma migrates and you need to re-strike (roll into a new ATM straddle)
- **Theta accelerates near expiration**: the clock is always ticking

## Rebalancing approaches

No single optimal rule; trade off gamma capture vs transaction costs.

- **Time-based**: rebalance every N minutes/hours. Simple, predictable cost.
- **Delta-threshold**: rebalance when `|delta| > threshold` (e.g., ±0.10). Scales with moves.
- **Variance-based**: rebalance when `|dS| > k·σ·√dt`. Accounts for vol regime.
- **Adaptive**: Whalley-Wilmott optimal hedging bands — explicit tradeoff between gamma capture and cost, derived from utility maximization.

Rule of thumb: in liquid underlyings with tight spreads, more frequent rebalancing wins. With wide spreads or low volumes, cost eats the scalping.

## When it works

- **Realized > Implied**: the whole premise
- **Oscillating markets**: range-bound with intraday reversals
- **Liquid underlyings**: tight bid-ask, low slippage on rebalances
- **Futures preferred over stocks**: the real reasons are (a) **SPAN portfolio margining** (much more capital-efficient than equity margin), (b) **no locate / borrow** required for the directional hedge, and (c) **23h trading liquidity** for hedge execution outside RTH. Note that 23h trading does *not* eliminate gap risk — overnight ES/SPX moves are large relative to the realized vol you're harvesting; the benefit is hedge-execution flexibility, not gap immunity.
- **Near-term, ATM options**: highest gamma per dollar of theta

## When it fails

- **Implied priced the move correctly**: straight-line trends without oscillation pay zero scalping even if vol is realized
- **Single large gap**: gap-and-grind markets; you pay theta all day and the gamma capture arrives as one big hedge, often at the wrong time
- **Wide bid-ask or low liquidity**: rebalance costs exceed gamma income
- **Post-event vol crush**: buying straddles before an event, getting the direction right, but losing on the IV collapse

## Practical notes

- **Vega risk**: gamma scalping is long vega; a drop in IV while holding causes mark-to-market losses independent of scalping P&L
- **Theta budgeting**: before opening, compute daily theta cost and compare to expected gamma P&L at current realized vol
- **Execution automation**: manual gamma scalping is infeasible at any reasonable scale. Requires:
  - Real-time Greeks recomputation
  - Low-latency hedge execution
  - Automated rebalance triggers
  - This is why options market makers essentially *are* gamma scalpers — infrastructure is the moat

## Go implementation notes

Gamma scalping is a natural fit for Go's concurrency model:
- One goroutine per symbol for quote ingestion
- Pricing service goroutine for Greeks
- Hedge executor goroutine with channel-based trigger
- `sync/atomic` for delta aggregation without locks on the critical path

## Python libraries

- `py_vollib`, `py_vollib_vectorized` — fast Greeks
- `QuantLib-Python` — for exotic structures and term vol
- `numpy`, `pandas` — simulation and backtest

## References

- Natenberg — *Option Volatility and Pricing* (gamma scalping chapter)
- Colin Bennett — *Trading Volatility* (Ch. 8)
- Whalley & Wilmott (1997) — "An asymptotic analysis of an optimal hedging model with transaction costs"
- Ahmad & Wilmott (2005) — "Which free lunch would you like today?"
- `GammaScalping` (GitHub: michaelsyao) — reference implementation
