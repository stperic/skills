# Mean Reversion

## Premise

Prices tend to revert to a long-run average after temporary deviations. When an asset moves too far from its mean, it is likely to snap back. Premise ≠ edge: mean reversion only pays when the series is *stationary* and the deviation is large relative to its noise.

## Math

**Ornstein-Uhlenbeck process** (continuous-time mean reversion):
```
dX_t = θ(μ - X_t) dt + σ dW_t
```
- `θ` — speed of mean reversion (higher = faster snap-back)
- `μ` — long-run mean
- `σ` — volatility
- `W_t` — standard Brownian motion

**Discrete AR(1) equivalent**: `X_t = α + β X_{t-1} + ε_t`. Stationary iff `|β| < 1`.

**Half-life of mean reversion** (how long to close half the gap to the mean):
```
half_life = -ln(2) / ln(β)
```
Strategies only work when half-life is short enough to trade but long enough to survive noise. Rule of thumb: 1–30 days for daily data.

**Z-score**: `z = (price - rolling_mean) / rolling_std`. The canonical entry/exit signal.

**Augmented Dickey-Fuller (ADF)**: null hypothesis is unit root (non-stationary). Reject at p < 0.05 → series is mean-reverting enough to trade. Do this on the *spread* for pairs, on *residuals* for factor models, on *price levels* only for a few stationary series (vol, spreads, ratios — almost never a stock price).

## Entry/exit rules (baseline)

- Enter long when `z < -2.0`
- Enter short when `z > +2.0`
- Exit at `z ≈ 0` (full) or `z = ±0.5` (partial)
- Stop-loss at `|z| > 3.0` or time stop at `2 × half_life`

These are textbook defaults, not edges. Real strategies calibrate thresholds per series.

## When it works, when it fails

**Works**: range-bound, low-vol, sideways markets with a stable mean. Sector-neutral spreads, funding-rate spreads, calendar spreads, ETF-vs-basket arbs, mean-reverting vol term structure.

**Fails**: strong trends, structural breaks, regime shifts. A mean-reverting strategy during a trend *compounds losses* by doubling down against the move. The ADF test is backward-looking — passing ADF yesterday doesn't mean stationary today.

## Key variants

- **Bollinger Bands**: enter at price crossing mean ± 2σ (equivalent to |z|=2)
- **RSI reversal**: buy RSI < 30, sell RSI > 70
- **Kalman Filter**: adaptive mean estimation for time-varying μ
- **GARCH-dynamic thresholds**: scale z-score bands by forecasted σ
- **Hurst exponent filter**: only trade when H < 0.5 (mean-reverting regime)

## Risk controls

1. **Stationarity monitor**: rolling ADF on the traded series; exit or halt on p > 0.10
2. **Time stop**: don't hold past `2 × half_life`; if it hasn't reverted, something changed
3. **Regime filter**: disable in trending regimes (e.g., ADX > 25, or a trend-detection model)
4. **Capital scaling**: position size ∝ |z| but capped at a max (prevents all-in at z=5 during a blow-up)

## Python libraries

- `statsmodels.tsa.stattools.adfuller` — ADF test
- `statsmodels.tsa.ar_model.AutoReg` — AR(1) β estimation
- `pykalman` — Kalman Filter for adaptive μ
- `arch` — GARCH models for dynamic σ
- `pandas` — rolling mean/std

## References

- Ernest P. Chan — *Algorithmic Trading*, Ch. 3
- Quantpedia — Mean Reversion strategies catalog
- Poterba & Summers (1988) — "Mean Reversion in Stock Prices"
