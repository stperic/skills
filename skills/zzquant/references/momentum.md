# Momentum & Trend Following

## Premise

Assets that have been rising tend to keep rising; assets that have been falling tend to keep falling. Momentum persists over 3–12 month horizons across essentially every asset class studied since 1800. The most robust anomaly in empirical finance, yet it crashes hard in regime changes.

## Math

### Cross-sectional momentum (XS)
Rank N assets by trailing return `r_{t-L, t-1}` over lookback L (typical L = 6–12 months, skip most recent month to avoid short-term reversal). Long top decile, short bottom decile.

### Time-series momentum (TSMOM)
Per asset, independently:
```
signal_i = sign(r_{i, t-L, t-1})
position_i = signal_i × target_vol / realized_vol_i
```
Moskowitz, Ooi, Pedersen (2012) showed this works across 58 futures markets over 25 years.

### Moving average crossover
```
signal_t = +1 if SMA_fast(t) > SMA_slow(t) else -1
```
Typical fast/slow: 20/50, 50/200, 10/30. MA crossover is mathematically a filtered version of TSMOM.

### Exponential moving average
```
EMA_t = α · price_t + (1-α) · EMA_{t-1}
α = 2 / (N+1)
```
EMA reacts faster to recent data than SMA. More responsive, more whipsaw.

## Position sizing

- **Equal weight**: simplest, fine for small N
- **Inverse volatility**: `w_i = (1/σ_i) / Σ(1/σ_j)` — normalizes risk contribution
- **Signal-strength weighting**: size ∝ |z-score| or |momentum score|
- **Target-vol**: scale whole portfolio to a fixed annualized vol (e.g., 10%)

Inverse-vol is the baseline for cross-asset trend portfolios (AQR's approach).

## When it works, when it fails

**Works**: sustained trends, high cross-sectional dispersion, crisis selloffs *after* the initial shock (TSMOM got short and profited in 2008).

**Fails**:
- Choppy range-bound markets (whipsaws eat the P&L)
- **Momentum crashes**: sharp reversals at regime turns. March 2009 was the worst — long-short momentum lost ~30% in a few weeks as beaten-down stocks rallied violently. Daniel & Moskowitz (2016) documented this as a skew problem: momentum has negative tail skew.

## Key variants

- **Dual momentum** (Antonacci): combine absolute (is return > 0?) and relative (is it top ranked?)
- **Adaptive momentum**: adjust L based on regime (shorter in high-vol, longer in low-vol)
- **Vol-target momentum**: skip signals when VIX or realized vol exceeds threshold
- **Sector momentum rotation**: apply XS momentum to sector ETFs instead of single names
- **Risk-managed momentum** (Barroso & Santa-Clara): scale exposure down when momentum itself is volatile

## Risk controls

1. **Trailing stop**: ATR-based (e.g., `entry - 2 × ATR_14`)
2. **Time stop**: exit after N periods regardless
3. **Diversification**: trend portfolios need 30–50+ markets to dilute single-market drawdowns
4. **Vol targeting**: scale gross exposure to a target vol; reduces crash risk mechanically
5. **Crash filter**: disable or invert when realized vol of momentum factor itself spikes

## Python libraries

- `pandas` — rolling returns, ranking
- `numpy` — ranking, percentiles
- `vectorbt` — fast multi-asset backtesting
- `riskfolio-lib` — portfolio construction with constraints

## References

- Jegadeesh & Titman (1993) — the seminal paper
- Moskowitz, Ooi, Pedersen (2012) — "Time Series Momentum" (TSMOM)
- Asness, Moskowitz, Pedersen (2013) — "Value and Momentum Everywhere"
- Daniel & Moskowitz (2016) — "Momentum Crashes"
- AQR — "A Century of Evidence on Trend-Following Investing"
- Barroso & Santa-Clara (2015) — "Momentum Has Its Moments"
