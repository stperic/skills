# ML Alpha Generation

## Premise

Use ML to discover nonlinear, high-dimensional patterns traditional statistical methods miss. Inputs are features (price, fundamental, alternative data); outputs are return forecasts, factor scores, or direct position sizes.

**The central problem** is not model choice — it's avoiding leakage, overfitting, and regime-shift fragility. Most "great ML alphas" are look-ahead bugs or data-snooping artifacts.

## Model hierarchy

| Model | Use case | Strength | Weakness |
|---|---|---|---|
| Linear / Ridge / Lasso | Factor construction, baselines | Interpretable, stable | Misses nonlinearity |
| Random Forest | Feature selection, baseline nonlinear | Handles noise, ranks features | Overfits on small data |
| XGBoost / LightGBM / CatBoost | Tabular alpha signals | SOTA for structured data | Needs regularization, tuning |
| LSTM / GRU | Sequential price patterns | Captures temporal dependence | Slow, overfits, hard to diagnose |
| Temporal CNN (TCN) | Sequential alternative to LSTM | Parallelizable, strong baselines | Fewer retail libraries |
| Transformers (Informer, TimesNet) | Long-horizon forecasting | Multi-scale, SOTA on benchmarks | Data-hungry, huge capacity |
| Graph Neural Networks | Asset relationships, sector links | Captures network effects | Complex training, graph construction |
| RL (PPO, DDPG, SAC) | Portfolio optimization, execution | Learns policies end-to-end | Reward design hard, unstable |
| Signature / path-signature methods | Path-dependent feature extraction | Mathematically principled, captures full path information | Tooling immature; less tooling and pedagogy than tabular methods |

**Default**: start with Lasso + XGBoost. If those don't find signal, deeper models rarely rescue it.

## Feature engineering

- **Price-based**: returns, log-returns, realized vol, high-low range, OHLCV ratios, z-scores at multiple lookbacks
- **Technical**: RSI, MACD, Bollinger width, ATR, volume profiles, candle patterns
- **Fundamental**: P/E, P/B, ROE, debt/equity, earnings surprise, revision spread
- **Alternative**: NLP sentiment (news, social, filings), satellite imagery, web traffic, credit card data
- **Microstructure**: bid-ask spread, order imbalance, VPIN, Kyle's lambda, trade size distribution
- **Cross-sectional**: rank transforms, sector-relative, industry-relative

Always transform features to be stationary. Raw prices are a bug; log-returns or z-scores are the fix.

## The leakage checklist

Before trusting any ML backtest result, verify:

1. **No future data in features.** Every feature at t must use strictly `< t` inputs. Rolling windows must not include t.
2. **No target leakage.** Future returns can sneak into features via forward-filled fundamentals, survivorship-biased universes, point-in-time fund databases.
3. **Survivorship bias.** The universe must be the *tradable universe at time t*, not today's tradable universe. Excluding delisted names biases up.
4. **Restatement bias.** Fundamental data must be point-in-time (what was known on that date), not restated later.
5. **Train-test split respects time.** No vanilla random k-fold on time-series with serial dependence. Walk-forward is the right default for anything path-dependent (intraday, momentum, autocorrelated labels). For genuinely cross-sectional, low-autocorrelation factor research (monthly rebalanced value, etc.), purged stratified k-fold with embargo is acceptable and gives more statistical power.
6. **Feature selection on train only.** Selecting features using the full sample is leakage.

## Cross-validation for time series

**Standard k-fold *without purging or embargo* is wrong** — it leaks future info into training folds. Walk-forward is the right default. Purged stratified k-fold is also valid for low-autocorrelation cross-sectional research. The "walk-forward only" framing is too absolute: the actual test is whether your CV scheme prevents label-horizon information from leaking, not the specific name of the scheme.

### Walk-forward (rolling / expanding)
Train on `[0, T]`, predict `T+1..T+h`. Step forward. Simple, correct, most widely used.

### Purged k-fold (López de Prado)
Remove training observations whose labels overlap with the test period's feature horizon. Prevents leakage from label horizon bleed-through.

### Combinatorial Purged Cross-Validation (CPCV)
Generate many non-overlapping train/test combinations to produce a *distribution* of backtest paths, not a single path. Robust to luck of a single split. Useful for distribution-of-Sharpe analysis and Deflated-Sharpe deflation.

**Caveat**: CPCV is more often interview-signaling than load-bearing. Most real alphas fail on Sharpe > 0, not on Sharpe-distribution width. CPCV is a useful diagnostic but it does not save a strategy whose underlying signal is weak.

CPCV implementation and theory: López de Prado, *Advances in Financial Machine Learning*, Ch. 12.

## Validation metrics

- **Information Coefficient (IC)**: rank correlation of predicted vs actual returns.
  - **Raw factor IC**: 0.02–0.03 is what serious shops fight for on a single signal. The 0.05+ target widely cited is for *combined / ensemble* signals, not raw factors.
  - **Combined / ensemble IC**: 0.05+ is the bar after combining multiple signals — don't apply this threshold to a raw factor.
- **Sharpe (OOS, net of costs)**: > 0.5 minimum for a single signal; > 1.0 for production strategies. **Internal consistency note**: the ≥ 1.5 number cited in `stat_arb.md` is a *portfolio-level* target for a fully constructed mature stat-arb book at top-tier shops, not a per-signal target. Don't conflate. For a small-to-mid stat-arb book in 2024–26, 0.8–1.2 net of real costs is realistic.
- **Sharpe is insufficient for non-Gaussian P&L** — see `metrics_beyond_sharpe.md` for Sortino, Calmar, CVaR, drawdown duration, and DSR caveats.
- **Max drawdown** + drawdown duration: does the strategy survive its worst historical period, and how long would you wait for recovery?
- **Turnover**: implied transaction cost
- **Deflated Sharpe** (López de Prado): corrects for multiple-testing inflation. **Caveat**: requires you to know K (effective number of trials), which nobody does honestly. The correction also assumes independent trials — related strategies share data and aren't independent. Use DSR as a *flag* for suspicious results, not as a *certificate* for clean ones.

## Interpretability (non-optional for risk management)

- **SHAP values**: attribute prediction to features per sample. Sanity check: do the top features make sense?
- **Feature importance drift**: if importance rankings shift drastically over time, the model is regime-fragile
- **Monotonicity constraints**: XGBoost supports forcing features to be monotonic in the target — constrains overfitting, preserves interpretability

## Alpha decay

Factors lose predictive power as capital chases them. Typical alpha half-life: 6–18 months.

Counter:
- **Originality**: measure AST / feature-expression similarity to known factors; flag near-duplicates
- **Hypothesis alignment**: tie features to economic reasoning; pure ML-discovered signals decay fastest
- **Complexity control**: penalize model size (regularization, pruning, small ensembles)

AlphaAgent (KDD 2025) explicitly built an LLM-driven alpha mining pipeline with these three anti-decay controls. Vendor-paper performance numbers from this line of research are not reproduced here — they tend to be cited out of context once they leak into reference documents. Read the paper for the methodology, not the headline number.

## The CPCV–Deflated Sharpe workflow

Standard institutional pipeline:

1. Generate K candidate signals (features, model hyperparams, target definitions)
2. Run CPCV on each → distribution of OOS Sharpes per signal
3. Select top signals by *median* OOS Sharpe (not max)
4. Apply Deflated Sharpe Ratio to correct for multiple testing
5. Combine surviving signals into ensemble
6. Live walk-forward monitoring — halt if OOS IC drifts significantly

## Python libraries

- `xgboost`, `lightgbm`, `catboost` — gradient boosting
- `scikit-learn` — baselines, Lasso, Ridge
- `pytorch`, `tensorflow` — deep learning
- `mlfinlab` — López de Prado algorithms (purged k-fold, CPCV, DSR)
- `shap` — interpretability
- `alphalens` — factor analysis
- `qlib` — Microsoft's end-to-end AI quant research platform

## References

- Marcos López de Prado — *Advances in Financial Machine Learning* (the essential book)
- López de Prado — *Machine Learning for Asset Managers*
- Bailey, Borwein, López de Prado, Zhu (2014) — "Pseudo-Mathematics and Financial Charlatanism" (DSR)
- Microsoft Qlib — https://github.com/microsoft/qlib
- awesome-quant-ai — https://github.com/leoncuhk/awesome-quant-ai
- AlphaAgent (KDD 2025) — decay-resistant LLM alpha mining
- Lyons et al. — *signature methods* for path-dependent features (Cuchiero, Salvi 2022 for a finance-focused intro)
- Bailey, Borwein, López de Prado, Zhu (2014) — "Pseudo-Mathematics and Financial Charlatanism" (Deflated Sharpe Ratio + caveats on knowing K)
