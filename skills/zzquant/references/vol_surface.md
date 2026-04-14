# Volatility Surface Trading

## Premise

Exploit mispricings across the volatility surface — the 3D relationship between implied vol, strike, and expiration. The surface is not a single number; it's a structured object with shape priors (skew, smile, term) that can be traded directly.

## Surface dimensions

- **Skew (strike axis)**: IV varies by moneyness. Equity indices show negative skew — OTM puts are more expensive than OTM calls (crash-insurance demand). Single stocks often show smiles or positive skew (acquisition optionality).
- **Term structure (time axis)**: IV varies by expiration. Normal shape = contango (upward sloping); stress shape = backwardation (inverted).
- **Smile curvature**: convexity of skew. High curvature ↔ high kurtosis expectation ↔ tail event pricing.

Together these form a surface `σ(K, T)`. No-arbitrage constraints (see below) restrict allowable shapes.

> **Important caveat**: the models in this file (SVI, SABR, rough vol) describe the *static* surface fit to options prices. They do not model the **dealer hedging feedback** that drives intraday spot in post-2020 SPX. For intraday work in 2024+, you need both the classical surface and a dealer-flow / GEX overlay. See `market_structure_2020s.md` §Dealer gamma, charm, vanna.

## Skew trading

- **Sell steep skew**: when put skew is historically elevated (high vs own 1Y rank), sell OTM puts and buy closer-to-ATM puts. Put spread. Bets on skew mean-reverting.
- **Buy cheap upside**: when call skew is flat (upside not priced in), buy OTM calls. Cheap convexity before a rally.
- **Risk reversal**: sell OTM put + buy OTM call (or vice versa). Trades skew *direction*.
- **Skew-variance decomposition**: variance swap fair strike is insensitive to skew shape, so skew trades vs variance swaps isolate skew P&L.

## Term structure trading

- **Calendar spread**: sell near-term, buy back-month. Profits from front-month time decay + back-month stability. Works in contango.
- **Earnings term trade**: pre-earnings, front-month IV elevates. Sell the front, buy the back (weighted by vega). After earnings, front-month IV crushes → profit. Standard retail/semi-pro structure.
- **Forward volatility trades**: compute the implied forward vol between two expirations; trade views on the shape of the forward curve.

## Surface arbitrage (no-arb constraints)

The surface must satisfy constraints to be arbitrage-free. Violations are trade opportunities.

- **Butterfly arbitrage**: `σ(K, T)` must produce a convex call curve in K. If `call(K−ΔK) − 2·call(K) + call(K+ΔK) < 0`, a butterfly is free money (modulo liquidity).
- **Calendar arbitrage**: total implied variance `w(K, T) = σ²(K, T) · T` must be non-decreasing in T (Gatheral 2004). If `w(K, T1) > w(K, T2)` for `T1 < T2`, the forward variance is negative — arbitrage.
- **Lee's moment formula**: limits on wing behavior of implied vol as `K → 0` or `K → ∞`; violations imply infinite moments, which is unphysical.

Most "arbitrages" found in listed markets are really liquidity-premium or bid-ask-spread artifacts. True surface arbs are rare in liquid indices; more common in illiquid single-name chains.

## Parametric surface models

Practitioners fit a parametric model to the surface and trade deviations.

### SABR (Hagan-Kumar-Lesniewski-Woodward 2002)
Stochastic volatility model with 4 parameters per expiration: `α` (atm vol), `β` (skew shape exponent), `ρ` (correlation of vol with spot), `ν` (vol of vol). Industry standard for rates and commodities surfaces. Hagan formula gives closed-form implied vol.

### SVI (Gatheral 2004)
Stochastic Volatility Inspired parametrization: total variance as a function of log-moneyness. 5 parameters per slice:
```
w(k) = a + b · (ρ(k − m) + √((k − m)² + σ²))
```
Guarantees no butterfly arbitrage with parameter constraints. Fast, robust, the go-to for equity index surfaces.

### SSVI / eSSVI
Surface-level extensions of SVI (one set of parameters for the whole surface). Calendar-arbitrage-free by construction.

## Rough volatility

Recent research (Gatheral-Jaisson-Rosenbaum 2018) shows implied vol paths are *rough* — Hurst exponent `H ≈ 0.1`. Consequences:

- ATM skew decays as `τ^(H − 0.5)` instead of `τ^(-0.5)` — faster decay at short tenors than classical models predict
- Rough Bergomi model for pricing
- Trading signal: compare implied roughness (from surface fit) vs realized roughness (from high-frequency underlying data)

## Higher-order moments

Options implicitly price higher-order moments of the return distribution:

- **Variance** (2nd moment): ATM IV
- **Skewness** (3rd moment): via risk reversal / skew
- **Kurtosis** (4th moment): via butterfly / smile curvature
- **5th, 6th moments**: hyper-skew and hyper-kurtosis for exotic pricing

Bakshi, Kapadia, Madan (2003) give model-free formulas for risk-neutral moments from option prices.

## Variance swap replication: classical vs current

The Carr-Madan / Derman-Demeterfi-Kamal-Zou static replication of variance swaps via a strip of options weighted by `1/K²` is the classical reference and is correct in continuous strikes with continuous trading. **In practice it has well-known frictions**:

- **Strike discretization** — listed strikes are not continuous; the replication has discretization error
- **Wing truncation** — you cannot trade arbitrarily far OTM; the truncation introduces a jump-risk bias
- **Liquidity** — far OTM strikes have wide spreads that destroy the replication economics

Post-2008 dealer desks calibrate variance using forward-variance and local-stochastic-vol (LSV) models off listed prices, not the pure static replication. The static formula is the right pedagogical anchor and the wrong production tool. See `vol_arb.md` for the variance and volatility swap mechanics; treat the replication formula as the intuition for *what variance swaps are*, not as the recipe for *how to price them today*.

## Calibration workflow

1. Pull option chain snapshot (strike, expiration, bid, ask, IV)
2. Filter illiquids (wide spreads, zero open interest)
3. Fit per-slice SVI (or SABR) for each expiration
4. Check no-arb constraints (butterfly, calendar)
5. Compute residuals — signal for surface trades
6. Re-fit continuously intraday; track parameter time series for regime shifts

## Python libraries

- `QuantLib-Python` — full surface construction, SABR, Heston
- `py_lets_be_rational` — fast Black-Scholes implied vol inversion
- `volatility_smile` / custom — SVI fitters
- `scipy.optimize` — custom calibration routines
- `rbergomi` / `roughbergomi` — rough vol models

## References

- Hagan, Kumar, Lesniewski, Woodward (2002) — SABR
- Gatheral (2004) — SVI
- Gatheral (2006) — *The Volatility Surface* (definitive textbook)
- Gatheral, Jaisson, Rosenbaum (2018) — "Volatility is Rough"
- Derman & Kani (1994) — local volatility
- Dupire (1994) — local vol formula
- Bakshi, Kapadia, Madan (2003) — model-free moments
- PyQuant News — practical surface analysis walkthroughs
