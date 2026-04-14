# Regime Detection & Adaptive Strategies

## Premise

Markets cycle through distinct regimes — bull / bear / crisis / recovery / high-vol / low-vol / trending / ranging. Strategies that adapt to the current regime outperform static approaches in most backtests. The hard part is detecting regimes *in real time* without look-ahead bias.

## Models

### Hidden Markov Models (HMM)
Discrete latent states with transition probabilities. Canonical choice:
- Typically 2–4 states (low-vol trending, high-vol trending, mean-reverting, crisis)
- Observables: returns, realized vol, VIX, yield curve, credit spreads
- Viterbi algorithm → most likely state sequence
- Baum-Welch → parameter estimation

Renaissance Technologies is widely reported to use HMMs for regime classification (secondhand sources; treat as industry lore, not hard fact).

Library: `hmmlearn`.

### Markov-Switching Regression
Parameters of a regression switch between regimes. Hamilton (1989) is the foundational paper. Good for modeling regime-dependent return distributions. `statsmodels.tsa.regime_switching.MarkovRegression`.

### Clustering (K-means, GMM)
Unsupervised grouping of market features. Features: VIX level, yield curve slope, credit spreads, cross-asset correlations, realized vol.

- **K-means**: hard assignments, fast, but you must pick k
- **Gaussian Mixture Model**: soft assignments with probabilities; better uncertainty quantification

Weakness: no temporal structure — clusters get assigned point-wise, ignoring regime persistence. Use HMM or add temporal smoothing.

### Change-point detection
Detect structural breaks in a time series without pre-specifying the number of regimes.
- **CUSUM** — classical, simple
- **Bayesian online change-point** (Adams & MacKay 2007) — real-time, probabilistic
- **PELT** (Killick et al. 2012) — fast exact offline detection
- Library: `ruptures`

### ML classifiers
XGBoost / neural nets trained on hand-labeled regime data. Labels are subjective — the biggest pitfall. Works when you have a robust, pre-agreed labeling scheme (e.g., NBER recession dates for macro regimes).

## Feature set for regime detection

Standard features for equity regimes:
- VIX level and term structure slope
- SPX realized volatility (rolling 20d)
- Trailing SPX return (5d, 20d, 60d)
- Credit spreads (IG OAS, HY OAS)
- Yield curve slope (10Y − 3M)
- USD index (DXY)
- Gold, oil (risk asset proxies)
- Cross-asset correlations (SPX-bonds, SPX-gold)

For commodity or rates regimes, substitute appropriate factors.

## Adaptive strategy framework

```
1. Classify current regime (HMM state probability, or cluster ID)
2. Select strategy allocation:
   Bull / low-vol       → momentum + premium selling
   Bear / high-vol      → defensive, reduce leverage, increase hedges
   Mean-reverting       → pairs trading, stat arb
   Crisis               → cash, tail hedges, long vol
3. Adjust position sizing by regime confidence (soft allocation)
4. Rebalance when regime probability shifts significantly
```

Hard switching is noisy — prefer soft weighting by state probability. Add hysteresis: require the new regime's probability to persist N days before rebalancing.

## Meta-strategy (strategy of strategies)

Rather than switch strategies, run all of them simultaneously and dynamically weight allocation. Analogous to a multi-manager fund.

- Each sub-strategy produces a return stream
- Allocator optimizes weights based on recent performance, regime, diversification
- Methods: mean-variance with shrinkage, risk parity, regime-conditional weighting, online learning (exponential weights, regret minimization)
- Reduces blow-up risk of any single sub-strategy

## Pitfalls

1. **Look-ahead in labeling**: if you label regimes ex-post and train a classifier, the classifier sees the future. Only use causal labels (e.g., "state probability at time t given data up to t").
2. **Overfitting regime count**: more regimes fit history better but hurt OOS. Start with 2–3.
3. **Regime oscillation / whipsaw**: unsmoothed HMM state jumps every day. Add persistence (minimum dwell time) or smoothing.
4. **Unstable parameters**: HMM with many states and few observations is fragile. Regularize or reduce complexity.
5. **Regime definitions are not universal**: "bull market" means different things to a stat arb shop vs a trend follower. Define regimes relative to *which strategies you're switching between*.

## Python libraries

- `hmmlearn` — HMMs
- `statsmodels.tsa.regime_switching` — Markov-switching models
- `ruptures` — change-point detection
- `scikit-learn` — K-means, GMM, classifiers
- `pymc` — Bayesian regime models

## References

- Hamilton (1989) — regime-switching models
- Ang & Bekaert (2002) — regime switches in international equity returns
- Adams & MacKay (2007) — Bayesian online change-point detection
- Killick, Fearnhead, Eckley (2012) — PELT
- awesome-quant-ai — HMM quant trading guides
- López de Prado — meta-strategy and ensemble approaches in *Advances in Financial Machine Learning*
