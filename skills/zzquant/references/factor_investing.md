# Factor Investing

## Premise

Systematic exposure to return-driving characteristics (factors) that have historically delivered excess returns, grounded in decades of academic research. Distinct from stat arb: factor investing is lower-turnover, longer-horizon, and typically long-only or long-biased.

## Established factors

| Factor | Long | Short | Rationale |
|---|---|---|---|
| Market (MKT) | Stocks | Risk-free | Equity risk premium |
| Size (SMB) | Small cap | Large cap | Illiquidity / neglect premium |
| Value (HML) | High B/P | Low B/P | Distress / contrarian premium |
| Momentum (UMD) | Winners | Losers | Behavioral under-reaction |
| Quality (RMW) | High ROE, low debt | Low ROE, high debt | Flight to quality |
| Low volatility (BAB) | Low-vol | High-vol | Lottery preference bias |
| Investment (CMA) | Conservative | Aggressive | Empire-building penalty |

Fama-French started with 3 (MKT, SMB, HML), extended to 5 (+ RMW, CMA). Carhart added momentum. AQR added BAB. Post-2010 factor zoo has 300+ proposed factors; most are data-mined and do not survive out-of-sample (Harvey, Liu, Zhu 2016).

## Construction

Classic academic construction (Fama-French):
1. Rank universe on the factor (e.g., book-to-price)
2. Form quintile or decile portfolios
3. Long top decile, short bottom decile
4. Rebalance monthly or quarterly
5. Value-weight within each bucket

Practical long-only tilt:
1. Overweight the target factor's exposure vs a benchmark (e.g., 1.3× value exposure)
2. Constrain tracking error to benchmark
3. Use optimization (cvxpy, riskfolio) for multi-factor combinations

## Multi-factor composites

Combining 3–5 factors diversifies factor-specific drawdowns. Rules:
- **Equal risk weighting** (not equal dollar weighting): each factor contributes equal volatility to the composite
- **Low-correlation factors preferred**: value + momentum is the canonical pair (correlation near 0)
- **Avoid redundancy**: many "new" factors are repackaged momentum or quality

## Factor timing signals (use with caution)

Factor timing is notoriously hard — Asness (2016) argued it's essentially another factor subject to the same behavioral biases.

- **Value spread** (valuation gap between cheap and expensive bucket): wide → increase value tilt
- **Factor momentum** (recent factor returns): factors that performed well tend to continue over 1–6 months
- **Macro regime**: value in recoveries, momentum in expansions, quality in downturns (weak empirical support)
- **Crowding indicators**: reduce exposure when factor gets crowded (short interest in basket, fund flows)

Most practitioners now recommend *minor* factor timing, not rotation. Strategic over tactical.

## Alpha decay

Factor returns erode over time as more capital chases the same signals.

- Typical factor half-life: **6–18 months** before re-evaluation needed
- Size factor (SMB) nearly disappeared post-1980s
- Value had a decade-long drawdown 2010–2020 before recovering
- Low-vol got crowded in mid-2010s, then reverted

Counter with:
- **Novel factor construction** (alternative data, NLP, unconventional signals)
- **Regime-aware timing** (light touch)
- **Originality enforcement** — ensure signals aren't syntactically similar to known factors (AST similarity checks in ML alpha pipelines)

## Implementation approaches

1. **Long-only factor tilt**: overweight factor exposures vs benchmark; used by smart-beta ETFs
2. **Long-short factor portfolio**: classic academic; requires shorting infrastructure
3. **Multi-factor composite**: combine 3–5 with equal risk weighting
4. **Factor timing overlay**: dial exposure up/down based on spreads or macro regime

## Python libraries

- `alphalens` — factor tear sheets, IC, quantile spread, turnover
- `statsmodels` — Fama-MacBeth regressions
- `riskfolio-lib`, `skfolio` — portfolio optimization with factor constraints
- `zipline`, `vectorbt` — backtesting frameworks

## References

- Fama & French (1993, 2015) — 3-factor, 5-factor
- Carhart (1997) — 4-factor with momentum
- Frazzini & Pedersen (2014) — "Betting Against Beta"
- Harvey, Liu, Zhu (2016) — "...and the Cross-Section of Expected Returns" (factor zoo critique)
- Asness, Frazzini, Israel, Moskowitz, Pedersen (2015) — "Fact, Fiction, and Value Investing"
- Asness (2016) — "The Siren Song of Factor Timing"
- AQR research library — extensive open factor research
