# Dispersion Trading

## Premise

Exploit the gap between index implied volatility and the weighted implied vol of constituents. Index options price in higher correlation than is typically realized → sell index vol, buy single-stock vol, profit from the correlation risk premium.

## The math

Index variance decomposes as:
```
σ²_index = Σ w_i² · σ²_i + Σᵢ Σⱼ≠ᵢ wᵢ · wⱼ · ρᵢⱼ · σᵢ · σⱼ
```

Implied correlation (extracted from implied vols):
```
ρ_implied = (σ²_index − Σ w_i² · σ²_i) / (Σᵢ Σⱼ≠ᵢ wᵢ · wⱼ · σᵢ · σⱼ)
```

Empirically, `ρ_implied > ρ_realized` on average. That gap is the **correlation risk premium**.

## The trade

Two legs, both delta-hedged:

1. **Short index vol**: sell ATM straddle on the index (SPX, NDX, etc.), or sell index variance swap
2. **Long single-stock vol**: buy ATM straddles on N constituents, weighted to replicate index exposure

Vega-neutral at inception: the sum of single-stock vegas matches the index vega. Gamma: long on singles, short on index. Correlation: short — profits when stocks move independently, loses when correlations spike.

Variance-swap-based dispersion is the cleanest: no path dependence from discrete hedging, pure exposure to realized variance and correlation.

## Why it works

- Institutional demand for index puts (tail hedging) inflates index implied vol
- Single-stock options less subject to this demand
- The gap is the structural edge — not mispricing, but risk premium

## When it fails

- **Correlation spikes** (risk-off events): in crises, correlations jump toward 1, and the short-index leg loses more than single-stock legs gain. 2008, 2020, 2022 — all brutal for dispersion.
- **Dispersion shock at low vol**: in a calm market, single-stock vols can decay faster than index vol, eroding the trade without a correlation move
- **Idiosyncratic single-name vol**: a single constituent's earnings surprise can move its vol independently, creating legging P&L
- **Costs**: two-sided transaction costs on N+1 legs eat thin edges

## PCA-based stock selection

You rarely trade all 500 names in SPX. Select a subset that best replicates index variance:

1. Run PCA on constituent returns
2. Select stocks with highest loadings on the first few principal components
3. These carry the most "systematic" variance, which is what the index is exposed to
4. Typical subset: 30–50 names capture most of the index factor structure

Published research on PCA-based subsetting for S&P 500 dispersion reports 14.5–26.5% p.a. returns with Sharpe ~0.34–0.40 (varies by period and method). Take specific numbers with skepticism — period-dependent.

## Instruments compared

| Instrument | Pros | Cons |
|---|---|---|
| Variance swaps | Pure variance exposure, no path dependence | OTC, dealer markup, limited venues |
| Listed straddles + delta hedge | Accessible, liquid on majors | Path-dependent, hedge cost, basis |
| Strangles | Cheaper entry | Need larger moves to pay |
| Combo: index VS + single listed options | Practical hybrid | Basis risk, complexity |

## Risk exposures (summary)

- **Short correlation**: central bet
- **Near-zero vega at inception**: by construction
- **Long gamma on singles, short gamma on index**: mechanical P&L driver
- **Tail-risk short**: correlations → 1 in panics is the nightmare scenario

## Risk controls

1. **Correlation stop**: pre-defined ρ_realized level at which to unwind
2. **Stress test** to a correlation shock (e.g., ρ → 0.9); ensure survivable loss
3. **Basis monitor**: track constituent weighting drift vs index rebalance dates
4. **Unwind plan**: identify which legs are most liquid; unwind order matters in stress

## Python libraries

- `scipy.linalg.eigh` / `sklearn.decomposition.PCA` — PCA for name selection
- `QuantLib-Python` — variance swap fair value
- `arch` — correlation forecasting via DCC-GARCH
- `cvxpy` — weight optimization for basket replication

## References

- Bossu (2006) — "A New Approach For Modelling and Pricing Correlation Swaps"
- JPMorgan — "Correlation Trading"
- MDPI paper on S&P 500 dispersion (2020) — PCA-based subsetting
- Derman (1999) — variance swap fundamentals
- QuantVPS / QuantInsti — practitioner dispersion explainers
