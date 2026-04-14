# Performance Metrics Beyond Sharpe

## Why this file exists

Most files in this skill anchor on Sharpe ratio. Sharpe is the most-used number in quant finance, and for many strategy classes it is the wrong number. For strategies with non-Gaussian return distributions (short vol, stat arb, momentum, illiquid carry), Sharpe systematically flatters the strategy by underweighting tail risk. This file collects the alternatives, when each is appropriate, and the failure modes of each.

## Why Sharpe is insufficient

Sharpe ratio:
```
Sharpe = (R − R_f) / σ
```

Three structural problems:

1. **σ assumes Gaussian returns.** Short-vol P&L is left-skewed and fat-tailed. A strategy with negative skew and high kurtosis produces a higher Sharpe than its true risk-adjusted return suggests.
2. **σ treats upside and downside symmetrically.** A high-vol strategy with most variance on the upside is penalized as if the variance were dangerous.
3. **σ is a *point* statistic.** It says nothing about drawdown depth, drawdown duration, or path of P&L delivery.

A 1.5 Sharpe strategy with a 40% max drawdown is, for almost every real allocator, *worse* than a 1.0 Sharpe strategy with a 12% max drawdown. Sharpe doesn't see the difference.

## Sortino ratio

```
Sortino = (R − R_target) / σ_downside
```

Where `σ_downside` is the standard deviation of returns below `R_target` (typically 0 or risk-free rate).

### When it helps
- Strategies with asymmetric return distributions (short vol, stat arb)
- Penalizes downside vol but credits upside vol
- More honest than Sharpe for non-Gaussian strategies

### Failure modes
- Sample size on the downside is half (or less) the full sample → noisier estimate
- Choice of `R_target` matters; document it
- Doesn't capture path / drawdown duration

## Calmar ratio

```
Calmar = annualized_return / max_drawdown
```

### When it helps
- Strategies where drawdown depth is the primary risk concern
- Allocators who care about peak-to-trough loss (most do)
- Long-only and trend-following strategies; CTA standard

### Failure modes
- Single max drawdown is a one-event statistic — high-variance estimator
- A strategy with a deep drawdown 10 years ago has a permanently bad Calmar even if it has improved
- Doesn't distinguish a quick recovery from a slow grind back

## MAR ratio

```
MAR = CAGR / max_drawdown
```

Variant of Calmar using CAGR. Same strengths and weaknesses.

## Sterling ratio

```
Sterling = CAGR / (avg_drawdown − threshold)
```

Averages drawdowns rather than taking the max. More stable than Calmar, less sensitive to one worst event. Common in CTA evaluation.

## Conditional Value at Risk (CVaR / Expected Shortfall)

```
CVaR_α = E[loss | loss > VaR_α]
```

The expected loss conditional on being in the worst α% of outcomes.

### When it helps
- Risk-aware position sizing
- Tail-conscious strategies
- Regulatory frameworks (Basel, Solvency II) prefer CVaR over VaR
- Strategies with negative skew

### Practical use
- Define a CVaR budget (e.g., "expected loss in worst 5% of weeks ≤ 3% of NAV")
- Size positions so portfolio CVaR matches the budget
- Reject strategies whose CVaR exceeds budget regardless of Sharpe

### Failure modes
- Tail estimation is statistically hard — needs many observations or a parametric tail model
- Choice of α matters (5% vs 1% vs 0.5%)
- Backward-looking; doesn't predict future tails

## CVaR-scaled return ("Conditional Sharpe")

```
CSR = (R − R_f) / CVaR_α
```

Sharpe but with CVaR in the denominator. Penalizes left-tail risk explicitly. The most direct upgrade from Sharpe for fat-tailed strategies.

## Omega ratio

```
Omega(τ) = (∫_τ^∞ (1 − F(r)) dr) / (∫_{-∞}^τ F(r) dr)
```

Where `F` is the CDF of returns and `τ` is a threshold. Ratio of probability-weighted gains above τ to losses below τ.

### When it helps
- Captures the full return distribution, not just first two moments
- Distribution-free; no Gaussian assumption
- Useful for highly non-normal P&L

### Failure modes
- Hard to communicate to non-quant allocators
- Requires enough data to estimate the full distribution
- Sensitive to choice of τ

## Information Ratio

```
IR = (R_strategy − R_benchmark) / σ_(R_strategy − R_benchmark)
```

Sharpe of the *active* return (vs benchmark). Standard for benchmarked active managers.

### When it helps
- Long-only or benchmarked strategies
- Distinguishes alpha from beta

### Failure modes
- Same Gaussian assumption as Sharpe
- Benchmark choice is political; can be gamed

## Drawdown duration

Not a ratio — a separate dimension of risk.

```
drawdown_duration = max(t_recovery − t_peak)
```

How long, in calendar time, between the peak and the eventual recovery to that peak.

### Why it matters
- A strategy with 20% drawdown that recovers in 3 months is fundamentally different from one with 20% drawdown that takes 4 years
- Allocators redeem on duration, not depth — patience runs out before money does
- Long drawdowns are the actual cause of fund closures, not deep ones

### Use it as a co-metric
Always report Calmar + max drawdown duration together. The pair tells you both the depth and the patience required.

## Deflated Sharpe Ratio (DSR)

Bailey & López de Prado (2014). Adjusts a reported Sharpe for the multiple-testing inflation that comes from trying many strategies and reporting the best.

### When it helps
- You ran K candidate strategies and want to know if the best is *actually* better than chance
- Counter to "data mining produces a high Sharpe"

### Failure modes (load-bearing — read these)
- **You must know K** (the effective number of trials), and nobody does. Researchers report whatever K makes their result survive. Garbage-in, garbage-out.
- **It assumes the trials are independent.** In practice, related strategies share data and aren't independent — DSR over-corrects.
- **The correction is large.** For K = 100 trials, the threshold for "real" Sharpe is much higher than naive — but if the trials weren't independent, the threshold is wrong by an unknown amount.
- DSR is a sanity check, not a precision instrument. Use it to *flag* suspicious results, not to *certify* clean ones.

## Practical multi-metric reporting

For any strategy, report all of:

- Sharpe (with standard errors)
- Sortino
- Calmar
- Max drawdown depth and duration
- Win rate
- Average win / average loss
- Skewness, kurtosis of returns
- CVaR at 5%
- Sample size / number of independent observations

A strategy that looks great on Sharpe but bad on most others is suspicious. A strategy that looks good across the panel is more credible.

## Anti-patterns

1. **Reporting only Sharpe** — hides distribution shape
2. **Annualizing Sharpe naively** by `× √252` when returns are autocorrelated — overstates by 20–50%
3. **Using max drawdown alone** — single-event statistic, high variance
4. **Using a long lookback for max drawdown** — penalizes strategies that have improved
5. **Ignoring drawdown duration** — depth without duration is half the picture
6. **Not deflating for multiple testing** — every paper reporting "Sharpe 2.5 OOS" without DSR is suspect
7. **Trusting DSR as if K were known** — see DSR failure modes above

## Recommended primary metric by strategy type

| Strategy class | Primary metric | Why |
|---|---|---|
| Trend following / CTA | Calmar + drawdown duration | Trends compound; deep DDs are normal; recovery time matters |
| Stat arb | Sortino + CVaR + drawdown duration | Negative skew; tail events dominant |
| Short premium | Sortino + CVaR + max DD | Left-skew; need explicit tail penalty |
| Long-only beta | Information ratio + max DD | Benchmark-relative; alpha + beta separation |
| Carry strategies | Calmar + Sortino + DSR | Path-dependent; carry crashes are tail events |
| HFT / market making | Sharpe + drawdown duration | Returns are closer to Gaussian at high frequency |
| ML alpha | DSR (with caveats) + walk-forward consistency + IC stability | Multiple-testing risk; need deflation |

Sharpe is fine for HFT and high-frequency mean reversion where returns are roughly Gaussian. For everything else, it is the wrong primary.

## References

- Bailey & López de Prado (2014) — "The Deflated Sharpe Ratio"
- Sortino & Price (1994) — Sortino ratio original
- Young (1991) — Calmar ratio original
- Keating & Shadwick (2002) — Omega ratio
- Acerbi & Tasche (2002) — Expected Shortfall foundations
- López de Prado — *Advances in Financial Machine Learning* (Ch. 14, backtest statistics)
- Pedersen — *Efficiently Inefficient* (allocator perspective on metrics)
- Lo (2002) — "The Statistics of Sharpe Ratios" (Sharpe under autocorrelation)
