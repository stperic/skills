# Pairs Trading

## Premise

Two historically cointegrated assets diverge temporarily. Go long the underperformer, short the outperformer, profit when they reconverge. A specific case of statistical arbitrage at N=2.

## Correlation is not cointegration

This is the most commonly botched distinction in retail pairs trading.

- **Correlation** (`ρ = cov(X,Y)/(σ_X σ_Y)`) measures co-movement of *returns*. Two assets can be highly correlated but drift apart forever.
- **Cointegration** means there exists a linear combination `Y_t - β X_t = ε_t` that is *stationary*. Their *levels* are tethered.

Cointegration is the necessary condition for a mean-reverting spread. Correlation alone is not enough.

## Tests

- **Engle-Granger**: regress `Y_t = α + β X_t + ε_t` via OLS, then ADF test on residuals. Reject null (unit root) at p < 0.05.
- **Johansen test**: for more than 2 assets; gives rank of cointegration (how many linearly independent mean-reverting combinations exist).
- **Rolling recalibration**: β is not static. Recompute via rolling OLS or Kalman Filter. Set a hard rule: if rolling ADF fails for N days, exit.

## Spread and z-score

```
spread_t = Y_t - β × X_t
z_t = (spread_t - rolling_mean(spread)) / rolling_std(spread)
half_life = -ln(2) / ln(AR(1) coef of spread)
```

Target half-life: 1–30 days. Under 1 → too noisy, likely artifact. Over 30 → too slow, capital inefficient.

## Pair selection criteria

- **Same sector / industry** (fundamental link; reduces risk that divergence is permanent)
- **Cointegration p < 0.05** on Engle-Granger or Johansen
- **Half-life between 1 and 30 days**
- **At least 2 years of cointegrated history** (not a recent coincidence)
- **Stable β** over rolling windows (unstable β = regime-sensitive relationship)
- **Liquid legs**: transaction costs on two-legged trades kill thin edges fast

## Execution rules (baseline)

- Enter at `|z| > 2.0`
- Exit at `z ≈ 0`
- Stop at `|z| > 3.5` or when cointegration breaks (rolling ADF fails)
- Recalculate β periodically (daily rolling OLS or continuously via Kalman)

## Dangers

1. **Permanent divergence**: M&A, bankruptcy, business model shift, spin-off. The cointegration *breaks* and never recovers. This is the single largest risk in pairs. Set a hard cointegration-break exit rule.
2. **Drawdown duration**: spread can stay wide for months. Capital committed, no PnL, optionality decay if hedging with options.
3. **Two-legged execution cost**: spreads pay half-bid-ask on each leg. A 1σ move has to cover 4× spread before you see P&L.
4. **Crowding**: widely known pairs become unprofitable. Watch for shrinking half-life *with* shrinking Sharpe — sign of decay.

## Kalman Filter for dynamic β

When β drifts, OLS on a window either lags (long window) or gets noisy (short window). Kalman Filter is the principled fix:

```
β_t = β_{t-1} + w_t          (state evolution)
Y_t = β_t × X_t + v_t        (observation)
```

`pykalman.KalmanFilter` implements this. Use it for any pair whose OLS β shows instability across rolling windows.

## Python libraries

- `statsmodels.tsa.stattools.coint` — Engle-Granger
- `statsmodels.tsa.vector_ar.vecm.coint_johansen` — Johansen
- `statsmodels.api.OLS` — static β
- `pykalman.KalmanFilter` — dynamic β
- `arch.unitroot.ADF` — ADF residual test

## References

- Gatev, Goetzmann, Rouwenhorst (2006) — "Pairs Trading: Performance of a Relative Value Arbitrage Rule"
- Ernest P. Chan — *Algorithmic Trading*, Ch. 4
- Vidyamurthy — *Pairs Trading: Quantitative Methods and Analysis*
- Elliott, van der Hoek, Malcolm (2005) — Kalman Filter pairs
