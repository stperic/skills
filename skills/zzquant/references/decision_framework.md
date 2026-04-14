# Strategy Selection Decision Framework

## When to open this file

You're choosing between strategies for a given regime, capital base, or edge — or sanity-checking whether the strategy you're running is appropriate for current conditions.

## Decision tree

```
Market Assessment
│
├── Volatility regime?
│   ├── Low IV (VIX < 15)   → Trend following, momentum, calendar spreads
│   ├── Moderate IV (15–25) → Stat arb, factor investing, premium selling
│   └── High IV (> 25)      → Premium selling (defined-risk), vol arb, dispersion
│
├── Market structure?
│   ├── Trending             → Momentum, TSMOM, breakout
│   ├── Range-bound          → Mean reversion, premium selling, pairs
│   └── Transitioning        → Regime detection + adaptive allocation
│
├── Where is your edge?
│   ├── Speed                → Market making, HFT, execution algos (Go/C++/Rust)
│   ├── Data                 → ML alpha, alternative data, NLP (Python ecosystem)
│   ├── Mathematics          → Vol surface, dispersion, variance swaps
│   └── Persistence / patience → Factor investing, systematic premium selling
│
└── Capital & constraints?
    ├── < $50K               → Defined-risk options, small pairs
    ├── $50K – $500K         → Multi-strategy, small stat arb, premium selling
    ├── > $500K              → Full quant infrastructure, market making, dispersion
    │
    └── No overnight?         → Intraday momentum, gamma scalping, 0DTE options
```

These are orientation defaults, not rules. Many successful shops break them.

## Strategy × regime fit matrix

| Strategy | Low vol | Moderate vol | High vol | Crisis |
|---|---|---|---|---|
| Mean reversion | ✅ | ✅ | ⚠️ | ❌ |
| Momentum / TSMOM | ✅ | ✅ | ⚠️ | ✅ (after shock) |
| Pairs trading | ✅ | ✅ | ⚠️ | ❌ |
| Stat arb | ✅ | ✅ | ⚠️ | ❌ |
| Factor investing | ✅ | ✅ | ✅ | ⚠️ |
| Market making | ✅ | ✅ | ⚠️ | ❌ |
| Short premium | ⚠️ | ✅ | ✅ | ❌ |
| Long vol / gamma scalp | ❌ | ⚠️ | ✅ | ✅ |
| Vol arb | ⚠️ | ✅ | ✅ | ⚠️ |
| Dispersion | ✅ | ✅ | ⚠️ | ❌ |

- ✅ good fit, expected to work
- ⚠️ marginal, requires tighter risk controls
- ❌ typically loses; avoid or size minimally

## Edge-type guide

Different quant shops compete on fundamentally different edges. Be honest about which you actually have.

### Speed edge
- **Who has it**: colocated HFTs with sub-μs infrastructure
- **Strategies**: market making, latency arb, statistical arbitrage on intraday
- **Cost**: millions in infrastructure, fee arrangements, hardware
- **Language**: C++, Rust, FPGA for hot path; Go for warm path

### Data edge
- **Who has it**: firms with unique alt data (satellite, credit card, web scraping, sentiment)
- **Strategies**: ML alpha, alternative-data factor construction
- **Cost**: data vendor fees, legal review, engineering
- **Language**: Python for research; Go / Rust / C++ for production

### Math edge
- **Who has it**: shops with deep quant researchers (derivatives modeling, optimization, stochastic calculus)
- **Strategies**: vol surface arbs, dispersion, variance swaps, exotic structuring
- **Cost**: PhD-quality headcount, calibration infrastructure
- **Language**: Python + C++ for pricing, Julia in some shops

### Persistence / patience edge
- **Who has it**: capital willing to sit through drawdowns; long investment horizon
- **Strategies**: factor investing, systematic premium harvesting, carry trades
- **Cost**: behavioral discipline, investor base that tolerates long drawdowns
- **Language**: Python for research, whatever for execution (not latency-sensitive)

**If you don't have one of these, you don't have an edge.** Discretionary "gut feel" is not in the list — that's not a quant strategy.

## Common misallocations

1. **Trend-following in choppy markets** — every trend-follower knows they'll whipsaw; still underestimates it
2. **Short premium through vol expansions** — the 2018 XIV blowup, the March 2020 crush
3. **Mean reversion into trends** — "this dip is bigger than normal" until it isn't
4. **Stat arb in correlation spikes** — the 2007 quant quake pattern
5. **Dispersion in crisis** — correlations → 1, both legs lose
6. **Pairs on cointegration break** — the spread goes to infinity, not to zero
7. **ML alpha without OOS discipline** — beautiful backtest, live P&L near zero
8. **Factor timing as main strategy** — Asness's siren song; it rarely works

## Diagnostic questions before sizing up

Before increasing allocation to a strategy, answer these:

1. **Why does it work?** If you can't state the economic rationale in one sentence, don't scale.
2. **When will it fail?** You should be able to name the specific regime or event that breaks it. Not "bad luck" — a mechanism.
3. **What's the current regime?** Classify it before sizing, not after.
4. **Who is on the other side?** If you can't identify the natural counterparty, you may be the liquidity, not the edge.
5. **Is it decaying?** Rolling Sharpe over the last 6–12 months vs full-sample Sharpe — if materially lower, decay is happening.
6. **What's the tail?** Not just average P&L, but worst-case path in a stress scenario.
7. **Can you survive a 2× worst historical drawdown?** If not, size down.

## Cross-reference

- Mechanics and math of each strategy → corresponding reference file
- Avoiding backtest artifacts → `ml_alpha.md` (CV, leakage)
- Execution-level costs → `execution.md`
- Regime classification → `regime_detection.md`
