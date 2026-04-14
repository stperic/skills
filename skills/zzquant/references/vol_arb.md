# Volatility Arbitrage (incl. Variance & Volatility Swaps)

## Premise

Trade the spread between implied volatility (the market's forecast embedded in options) and your estimate of future realized volatility. If your forecast of RV beats the market's, you profit.

Direction:
- `IV > fair_RV` → sell options (short vol), delta-hedge
- `IV < fair_RV` → buy options (long vol), delta-hedge

Hold until expiration or target P&L.

## Delta-hedged P&L (reminder)

```
daily_P&L ≈ 0.5 · Γ · S² · (σ²_implied − σ²_realized) · dt
```

Average sign depends on `σ_implied − σ_realized`. For index options, VRP is positive on average → short vol wins on average. Path and tail make it not a free lunch.

> **Caveat**: this is a continuous-time identity. Real-world P&L depends on hedge frequency, bid-ask cost on each rebalance, gap jumps the continuous formula misses, and the correlation between Γ and the realized-variance path. A correct continuous-time *average* can co-exist with brutal realized paths that wipe months of premium in a single session. The recipe "sell when IV > fair_RV, delta-hedge, hold to expiration" is the classical statement and is also a great way to lose money if you ignore path. See `gamma_scalping.md` for the same caveat from the long-vol side.

## Volatility forecasting models

Listed roughly in order of model complexity:

### GARCH(1,1)
```
σ²_t = ω + α · ε²_{t-1} + β · σ²_{t-1}
```
Workhorse baseline. Captures volatility clustering. Weakness: symmetric — treats up-moves and down-moves the same.

### EGARCH
Log-variance specification; captures the leverage effect (vol responds more to down-moves than up-moves). More accurate for equity indices.

### HAR-RV (Heterogeneous Autoregressive — Corsi 2009)
```
RV_t = β_0 + β_d · RV^d_{t-1} + β_w · RV^w_{t-1} + β_m · RV^m_{t-1} + ε_t
```
Uses daily, weekly, monthly realized vol as predictors. Surprisingly strong; one of the best parametric forecasters.

### Rough volatility (Gatheral-Jaisson-Rosenbaum 2018)
Vol paths follow fractional Brownian motion with Hurst exponent `H ≈ 0.1` — far rougher than classic models assume. Rough Bergomi model prices this. Practical implication: ATM skew decays faster than classical models predict; trade models that ignore roughness at short tenors.

### ML forecasters
LSTM, TCN, transformer on high-frequency returns. Often beat GARCH in backtest, often lose OOS due to non-stationarity. Need purged CV and careful feature engineering.

## The VRP decomposition

VRP is not a single number; it has structure:
- **Average level premium**: `E[IV − RV]` ≈ 2–4 vol points for SPX *unconditional historical mean*; post-2018 conditional has compressed to ~1–1.5 vol points with negative episodes. See `market_structure_2020s.md`.
- **Skew premium**: OTM puts overpriced vs ATM (crash insurance demand)
- **Term structure premium**: contango in calm markets (front cheap vs back; or vice versa for backwardation)
- **Convexity premium**: variance swaps price higher than `(vol swap)²` due to Jensen's

Different vol-arb flavors harvest different components.

## Variance swaps

**Payoff** (at expiry):
```
N_var · (σ²_realized − K_var)
```
where `K_var` is the strike variance (set at inception so PV = 0) and `N_var` is notional in variance units.

**Replication**: a static portfolio of options weighted inversely proportional to strike² (`1/K²`), plus a delta-hedge. Derman-Demeterfi-Kamal-Zou (1999) derived this. Key insight: you can replicate realized variance *without* discrete-hedging errors.

**Fair strike**:
```
K_var ≈ (2/T) · ∫ call(K)/K² dK
```
For SPX in idealized conditions, `K_var ≈ VIX²`.

**Convexity bias**: long variance swap benefits from `σ²` convexity in `σ` → the fair variance strike sits slightly above squared at-the-money vol.

## Volatility swaps

**Payoff**: `N_vol · (σ_realized − K_vol)`

- **Cannot be statically replicated** (unlike variance swaps) because vol is a concave function of variance
- Must be dynamically hedged using variance swaps as the underlying
- Approximate relation:
  ```
  K_vol ≈ √K_var · (1 − σ²/(8·K_var))
  ```
  (convexity adjustment from Jensen)

Volatility swaps have tighter P&L paths (linear in vol) but trade at a discount to the implied `√K_var` reflecting the convexity gap.

## Forward variance

```
forward_var(t1, t2) = (K_var(t2) · t2 − K_var(t1) · t1) / (t2 − t1)
```

Trade views on future vol regimes without taking spot vol exposure. Term structure of forward variance tends to flatten at longer horizons; trading near-term vs forward is a common structural trade.

## Variance swap dispersion

Sell index variance swap vs buy weighted single-stock variance swaps. The cleanest expression of the correlation trade — see `dispersion.md`.

## Who uses vol arb

- Volatility-dedicated hedge funds (Capstone, Parallax, LMR, etc.)
- Options market-making desks (it's their natural by-product)
- Variance swap dealers at banks
- Systematic VRP harvesting funds (simpler, directional short-vol)

## Risk controls

1. **Tail hedge overlay** — cheap OTM puts or VIX calls as crash protection
2. **Vega budget** — cap net vega per underlying and at portfolio level
3. **Stress test** — model a 2008/2020-style vol spike on the book; ensure survivable
4. **Liquidity** — variance swap market is dealer-intermediated, thinner than listed options; plan for wide unwind spreads in stress
5. **Basis risk** — when replicating variance swap with listed options, strike discretization creates residual error

## Python libraries

- `arch` — GARCH family (symmetric, EGARCH, GJR-GARCH, APARCH)
- `QuantLib-Python` — option pricing, local vol
- `pyvolib` / `py_vollib` — fast Greeks
- `tensorflow` / `pytorch` — ML vol forecasting
- `statsmodels` — HAR-RV via OLS

## References

- Colin Bennett — *Trading Volatility* (the vol trading bible)
- Derman, Demeterfi, Kamal, Zou (1999) — "More Than You Ever Wanted to Know About Volatility Swaps" (GS Research)
- JPMorgan — "Just What You Need to Know About Variance Swaps"
- Corsi (2009) — HAR-RV
- Gatheral, Jaisson, Rosenbaum (2018) — "Volatility is Rough"
- Bossu, Strasser, Guichard — practical variance swap guide
- Quantpedia — Volatility Risk Premium Effect (curated papers)
