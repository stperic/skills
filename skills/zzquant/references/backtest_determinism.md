# Backtest Determinism & Reproducibility

## Why this file exists

`ml_alpha.md` covers cross-validation discipline (walk-forward, purged k-fold, CPCV). This file covers the *infrastructure* under backtesting: making a backtest deterministic, reproducible, and honest. A non-deterministic or non-reproducible backtest is unfalsifiable — you cannot debug it, you cannot trust it, you cannot ship decisions from it.

## The three questions every backtest must answer

1. **Is it deterministic?** Run it twice with the same inputs → do you get byte-identical output? If not, something is unbounded.
2. **Is it point-in-time?** Does every decision use only data that was *actually available* at that decision time?
3. **Does it match live?** Can the same code path run in replay and live mode and produce consistent logic?

Failing any of these produces results that look scientific but aren't.

## Determinism checklist

### Seeds everywhere
- Python: `random.seed`, `numpy.random.seed`, `torch.manual_seed`, `tf.random.set_seed`
- Every stochastic component (bootstrap, MC, ensemble sampling) takes a seed parameter
- Log the master seed in the run manifest; allow override via CLI

### Set-order determinism
- Python `set()` iteration order is implementation-defined across versions — never iterate a raw set for logic
- Dict iteration is insertion-ordered since 3.7, but don't rely on it across library boundaries
- Use `sorted()` or `OrderedDict` for any iteration that affects output

### Floating-point determinism
- Different BLAS backends (OpenBLAS, MKL, Accelerate) can produce different results on the same input
- Multi-threading + floating-point reductions = non-deterministic (reduction order varies)
- For strict determinism: pin library versions, pin BLAS backend, set `OMP_NUM_THREADS=1` where needed

### Time sources
- `datetime.now()`, `time.time()` are the enemy. Every use of wall-clock time in business logic is a determinism bug.
- All timestamps come from the *data* (bar timestamp, event timestamp, received-at from the feed), never from `now()`
- If you need a "current time" in the logic, pass it as a parameter from the simulation clock

### Hash-based iteration
- Python 3.3+ randomizes string hash seed per-run
- `PYTHONHASHSEED=0` to fix it
- Or: never use dict/set iteration order for logic that affects output

### Parallelism
- Multi-process / multi-thread can produce non-deterministic event ordering
- If you parallelize for speed, do so at a level where merge is deterministic (e.g., partition by symbol, merge by sorted key)

## Point-in-time correctness

### The look-ahead taxonomy

1. **Future-in-features**: computing a feature with any data post-decision-time (rolling window that includes t, not just up to t-1)
2. **Survivorship bias**: backtest universe is "stocks in the S&P 500 *today*" rather than "stocks in the index on that date"
3. **Restatement bias**: fundamentals data has been restated post-hoc; backtest uses the restated number as if it had been known
4. **Delisting gaps**: delisted names vanish from the dataset rather than closing out at delisting price
5. **Index reconstitution leak**: knowing tomorrow's index composition today
6. **Look-ahead labels**: training targets that use future returns without a proper label lag

### Defenses

- **As-of queries**: every data fetch takes an `as_of` timestamp; no query without one is allowed in backtest paths
- **Point-in-time data vendor**: use vendors with explicit PIT snapshots (Compustat PIT, I/B/E/S Estimate History, etc.) for fundamentals
- **Frozen indexes**: store historical index constituents by date, query them point-in-time
- **Delisting handling**: explicit delisting records with liquidation price, not silent drop
- **Label lag**: `label(t) = f(data[t+1 : t+h])` with explicit lag; walk-forward CV enforces this

### The "would this data have existed on that date?" test

For every feature your backtest uses, answer: *was this exact value computable on that date with only data published before that date?*

- Earnings estimate from I/B/E/S — yes, if you use the vintage snapshot from that date, not the current consensus
- Price data — yes, assuming corporate-action-adjusted back-data is correct (see `data_quality.md`)
- Sentiment from a news article — only if the article was published before the decision time; watch for backfill
- An alternative data feed — only if its delivery timestamp ≤ decision time (many alt data vendors deliver with days of delay)

If you can't answer "yes" with evidence, the feature is suspect.

## State and replay

### State hashing
Checksum the state of the simulator at every step: open positions, cash, parameters, regime state. Compare across runs. If two runs diverge, the hash tells you the exact step of divergence — log-diffing by hand is not feasible for long runs.

### Event ordering
When multiple events happen at the "same" timestamp (bar close, signal, fill), the processing order matters. Define it explicitly:
```
Priority: data arrival → pre-trade risk → signal → order → fill → post-trade risk
```
Never rely on the order events happen to arrive in memory.

### Reproducibility manifest
Every backtest run writes a manifest:
```json
{
  "run_id": "uuid",
  "code_git_sha": "abc123",
  "config_hash": "...",
  "data_vintage": "2024-11-01",
  "seed": 42,
  "start": "2023-01-01",
  "end": "2023-12-31",
  "python_version": "3.11.4",
  "key_library_versions": {...}
}
```
Given the manifest, you can reproduce the run byte-for-byte six months later.

## Replay-live equivalence

The single most valuable property of a trading system: **the same code path runs in replay and live**. Strategies cannot have an `if replay:` branch.

### How to achieve it

1. **Abstract the data source**: `DataService` interface with two implementations, `LiveDataService` and `ReplayDataService`. The strategy sees only `DataService`.
2. **Abstract time**: `Clock` interface with `LiveClock` (wall clock) and `SimulationClock` (advances explicitly).
3. **Abstract the broker**: `OMS` interface with `LiveOMS` (real broker API) and `SimulatedOMS` (fills from historical quotes).
4. **No direct imports of `datetime.now()`, `time.time()`, or `random.random()` in strategy code.**

### Replay fidelity gradations

Levels of replay realism, from cheapest to most expensive:

1. **Bar-level replay**: trade at OHLC prices, assume infinite liquidity. Fast, useful for signal research.
2. **Quote-level replay**: use actual bid/ask snapshots. Realistic for options and illiquid names.
3. **L1 replay**: every top-of-book update. Appropriate for short-horizon strategies.
4. **L2 / full depth**: entire LOB replay. Needed for execution research and market making.
5. **Event-level**: every tick, every quote, every trade, in source order. Used for latency-sensitive strategies.

Match the level to the strategy's latency requirements. A swing strategy on bar-level replay is fine; a market maker on bar-level replay is nonsense.

## The "no synthetic data" rule

If your backtest quietly fabricates data when real data is missing (e.g., synthetic option chains, synthetic bars, forward-filled fundamentals beyond normal reporting lag), your backtest is not reproducing a market — it's running a fictional one. Common traps:

- **Synthetic option chains**: generating fake strikes/expirations when the historical chain is missing. The strategy appears to trade every day; it actually trades garbage.
- **Forward-filled fundamentals**: a company's P/E stays at last-known value forever if they delist or stop reporting
- **Inferred splits**: corporate actions not in the data are "reconstructed" from price gaps, with errors

Rule: if the data isn't there, the strategy either *skips* that day/symbol or *halts* with a WARNING. Never silently substitute.

## Survivorship bias in practice

Equity universes are the worst. A backtest of "S&P 500 stocks" over 20 years on today's constituents excludes every stock that was dropped from the index — typically the losers. Results are dramatically biased upward.

Defenses:
- Use vendor PIT index membership tables
- Include delisted names with liquidation prices
- Track why a name exited (M&A, bankruptcy, index rebalance) and handle each case explicitly

Similar problem for:
- Hedge fund returns (self-reported; survivors overrepresented)
- Crypto (dead tokens)
- Bonds (called, defaulted, matured)

## Bar-level traps

Bar-level backtesting is the most common mode and has subtle gotchas:

1. **Open price on signal**: signal is computed at close of bar t, order placed at open of bar t+1. Using close of t for both signal *and* fill is a leak.
2. **High/low fills**: assuming "if the high exceeded my limit, I got filled at the limit" — only true for pure limit orders in a deeply liquid market; wrong for stops, wrong for illiquid names.
3. **Intra-bar order**: was the high or low reached first? Affects whether a stop or target fired. Vendor data doesn't tell you — make a conservative assumption.
4. **Bar aggregation**: 5m bars from a vendor may not align with your simulator's 5m bars. Check boundaries.

## Costs and slippage

A backtest without costs is a research toy. Minimum realistic model:

- **Commissions**: per-share, per-contract, or per-trade
- **Exchange fees**: especially for options
- **Bid-ask spread**: half-spread on entry, half on exit is a baseline; widen for illiquid names
- **Market impact**: size-dependent, concave in size (Almgren-Chriss square-root model: `impact ≈ k · σ · √(Q/ADV)`)
- **Borrow costs**: for shorts, daily rate
- **Financing**: for leveraged positions

A rule of thumb: if your backtest's Sharpe drops by more than 30% when you switch from zero costs to realistic costs, you had a cost-sensitive strategy and the original result was misleading.

## What you cannot backtest

Even a perfectly deterministic, point-in-time, replay-live-equivalent backtest **cannot** model:

1. **Borrow rates and locate availability** — historical borrow data is poor; recall events are not well documented
2. **PFOF wholesaler fills** — your retail order would have routed through a wholesaler; backtest assumes lit-market fills
3. **OCC exercise cutoffs and corner cases**
4. **Margin call timing** — you might have been forced to close before your strategy wanted to
5. **Auto-liquidation triggers** — broker-side risk thresholds you don't control
6. **Queue position at the exchange** — you might not have been filled where backtest assumes
7. **Auction-cross fills** — opening and closing auctions have different dynamics from continuous trading
8. **Halt and circuit breaker behavior**
9. **Pin risk and assignment** on options near expiration
10. **Dividend exercise** on short ITM American calls
11. **Haircut expansion in stress** — brokers raise margin requirements 2–3× exactly when you need capacity
12. **Broker outages** — your strategy might have been unable to manage positions during a feed outage
13. **Section 1256 vs short-term tax treatment** — affects after-tax Sharpe materially
14. **Exchange and OCC fees** — per-contract fees can dominate edge at 0DTE prices
15. **Your own market impact** — backtests assume your orders didn't move the market

A backtest can be perfectly deterministic and still be wrong about live P&L by 20–50% because of the above. Determinism is **necessary, not sufficient** for trustworthiness. The defenses are paper trading, cautious sizing during ramp, explicit stress reserves, and a healthy skepticism of "but the backtest said..." See `options_operational_risks.md` for the options-specific items and `execution.md` §Execution realities for the cost/fee items.

## Backtest hygiene checklist (pre-publication)

Before acting on a backtest result:

- [ ] Run twice; output is byte-identical
- [ ] Seeded; master seed in manifest
- [ ] No `now()`, no `time.time()` in strategy code
- [ ] All data fetches have explicit `as_of`
- [ ] Universe is point-in-time; delistings included
- [ ] Fundamentals are point-in-time (PIT database or explicit vintage)
- [ ] Labels have explicit lag; CV is walk-forward or purged
- [ ] Costs are modeled; slippage is realistic for size
- [ ] Survivorship audited (dropped names accounted for)
- [ ] Same code path runs in replay and live
- [ ] Run manifest includes git SHA, config hash, data vintage, Python + key library versions
- [ ] State hashes logged per step for debug

## References

- Marcos López de Prado — *Advances in Financial Machine Learning* (Ch. 7 on backtest overfitting, Ch. 11 on backtest statistics)
- Bailey, Borwein, López de Prado, Zhu (2014) — "Pseudo-Mathematics and Financial Charlatanism" (deflated Sharpe, multiple-testing)
- Arnott, Harvey, Markowitz (2019) — "A Backtesting Protocol in the Era of Machine Learning"
- Harvey, Liu, Zhu (2016) — on false discoveries in factor research
- Pardo — *The Evaluation and Optimization of Trading Strategies* (walk-forward analysis)
- López de Prado — *Tactical Investment Algorithms* (operational aspects of backtest-to-production)
