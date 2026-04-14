# Statistical Arbitrage

> **Regime-anchor disclaimer**: numerical thresholds in this file (Sharpe targets, max drawdown, win rate, position counts) are *industry mature-book* anchors that apply to top-tier shops with full infrastructure. Realistic Sharpe for a well-run small-to-mid stat-arb book in 2024–26 is closer to 0.8–1.2 net of real costs, not 1.5+. Treat the higher number as aspirational. See `metrics_beyond_sharpe.md` for why Sharpe alone is the wrong metric for a left-skewed book.

## Premise

Extension of pairs trading to portfolios of dozens to hundreds of securities. Exploit small, temporary mispricings via factor residuals while maintaining market and factor neutrality. Historically the bread and butter of equity quant.

## Core architecture

1. **Alpha signal generation** — identify mispriced securities as residuals from a factor model
2. **Portfolio construction** — long undervalued, short overvalued, constrained for neutrality
3. **Risk management** — monitor factor exposures, sector tilts, drawdowns, crowding
4. **Execution** — smart routing, slippage control, participation caps

Each step can fail independently. Industry shops separate research, risk, and execution teams for this reason.

## Factor model approach

Estimate expected returns via multi-factor regression:
```
r_i = α_i + β_1 F_1 + β_2 F_2 + ... + ε_i
```

Residual `ε_i` is the "alpha" — returns not explained by systematic factors. Trade: long assets with positive recent residuals that are expected to revert (contrarian stat arb), or trend residuals (momentum stat arb).

Common factors:
- **Market** (MKT) — equity risk premium
- **Size** (SMB — small minus big)
- **Value** (HML — high minus low book-to-price)
- **Momentum** (UMD — up minus down)
- **Quality** (RMW — robust minus weak profitability)
- **Low volatility** (BAB — betting against beta)
- **Investment** (CMA — conservative minus aggressive)

Avellaneda & Lee (2010) use PCA to extract latent factors rather than pre-specified ones, then mean-revert the residuals.

## Portfolio constraints (load-bearing)

- **Dollar-neutral**: `Σ w_long ≈ Σ w_short`
- **Beta-neutral**: `Σ w_i β_i ≈ 0` — net beta in ±0.10
- **Sector-neutral**: balance long/short inside each sector (GICS level 2 or 3)
- **Position limits**: no single name > 2–5% of GMV
- **Turnover cap**: target turnover based on alpha half-life and transaction costs

These constraints are not optional. Unconstrained stat arb concentrates into a handful of stocks with the most extreme residuals, and those are precisely the ones most likely to have a real story (earnings surprise, M&A rumor).

## Performance targets (industry norms)

| Metric | Target |
|---|---|
| Sharpe (OOS, net of costs) | ≥ 1.5 for *top-tier mature* strategies; 0.8–1.2 is realistic for well-run smaller books |
| Max drawdown | ≤ 10–15% |
| Win rate | 55–65% |
| Turnover | weekly to monthly |
| Number of positions | 100–500+ |

Sharpe < 1.0 net of costs is typically not worth the infrastructure.

## Historical context (learn from failures)

- **Morgan Stanley** (Tartaglia, mid-1980s) — pioneered stat arb
- **LTCM (1998)** — correct models can still fail when liquidity evaporates; cross-correlations jumped to 1.
- **August 2007 "quant quake"** — multiple stat arb funds simultaneously deleveraged over 3 days, driving 20%+ drawdowns in strategies that had never had a losing week. Classic crowding + forced-unwind cascade.
- **2015 China deleveraging** — similar dynamic, smaller scale.

The lesson: stat arb works until everyone has the same book, and then it breaks catastrophically in a cascade.

## When it works, when it fails

**Works**: high cross-sectional dispersion, moderate vol, mean-reverting spreads, quiet macro.

**Fails**:
- Correlation spikes (risk-off events)
- Forced deleveraging cascades
- Liquidity crises
- Regime changes that invalidate the factor model
- Crowding (alpha decay as more capital chases the same residuals)

## Risk controls specific to stat arb

1. **Factor exposure monitor**: real-time net exposure to each factor; automatic rebalance if beyond bounds
2. **Crowding monitor**: track correlation of your P&L to other known quant funds' public returns or industry indices
3. **Liquidity filter**: exclude bottom quartile ADV names
4. **Borrow availability and recall risk**: shorts must be borrowable at stable rates. **The largest live risk in cash-equity stat arb.** Hard-to-borrow names can have borrow rates spike from <1% to 100%+ annualized in days; recalls force buy-ins at the worst possible price. The 2007 quant quake involved borrow stress, and the 2021 GME / meme-stock cascade was a textbook borrow-recall blowup. Track HTB status daily; exclude bottom-quartile borrowability from the universe; carry a borrow-cost cushion in your alpha threshold.
5. **Drawdown halt**: cut gross exposure by 50% at X% drawdown; halt new trades at Y%
6. **Haircut-expansion buffer**: brokers / prime brokers raise haircuts on volatile names by 2–3× during stress events, *exactly when you need capacity*. Several short-vol funds blew up in March 2020 from forced de-grossing, not from realized losses (LJM Preservation & Growth Fund 2018 is the textbook case). Maintain liquid cash equal to expected stress-haircut expansion; stress-test under 1.5× and 2× current margin; diversify prime brokers at scale.

## Python libraries

- `statsmodels` — Fama-MacBeth regressions, OLS
- `alphalens` — factor analysis, IC, quantile returns
- `portfoliolab` / `riskfolio-lib` — portfolio construction with constraints
- `scipy.optimize` — custom optimization (cvxpy for convex problems)
- `cvxpy` — convex optimization for long-short portfolio construction

## References

- Avellaneda & Lee (2010) — "Statistical Arbitrage in the US Equities Market"
- Marcos López de Prado — *Advances in Financial Machine Learning*, Ch. 7–9
- Khandani & Lo (2007) — "What Happened to the Quants in August 2007?"
- Ernest P. Chan — *Algorithmic Trading*, Ch. 5 (risk management)
