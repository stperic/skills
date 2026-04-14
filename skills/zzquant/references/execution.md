# Execution Algorithms & Market Microstructure

## Premise

How you execute is as important as what you trade. Sophisticated execution minimizes market impact and slippage, preserving the alpha your research earned. Edge at the signal level commonly dies at the execution level — a 0.3 Sharpe backtest can go negative after realistic costs.

## Classic execution algorithms

### TWAP — Time-Weighted Average Price
Split order evenly over a time window. Benchmark = average price during the window.

```
for each slice (at time t_i):
    send order of size Q/N
```

- **Pro**: simple, predictable, benchmark-comparable
- **Con**: ignores volume patterns, signals intention if order is large relative to volume
- **When**: small orders, illiquid names, even exposure preferred

### VWAP — Volume-Weighted Average Price
Trade proportional to historical intraday volume profile. Benchmark = volume-weighted average of the day.

```
slice_i = Q · (expected_volume_i / total_expected_volume)
```

- **Pro**: blends into natural volume, minimizes signaling
- **Con**: assumes historical volume profile holds; brittle on news days
- **When**: standard agency execution, large institutional orders

### Implementation Shortfall (IS) — Almgren-Chriss
Minimize expected deviation from decision price. Aggressive early, passive later. Trades off market impact against opportunity cost.

```
minimize  E[impact] + λ · Var[impact]
```

Closed-form trajectory for given urgency parameter `λ`. Used when entry price matters — the benchmark is the price at decision time.

- **Pro**: theoretically optimal under Almgren-Chriss assumptions
- **Con**: assumptions (linear impact, constant vol) often violated
- **When**: alpha has a short decay horizon; entry price is the alpha's baseline

### POV — Percentage of Volume
Participate at X% of realized market volume. Scales aggressiveness with available liquidity.

- **Pro**: self-adapting to liquidity
- **Con**: can't finish in fixed time; signals intention if X is high
- **When**: discretionary urgency without hard deadline

## Advanced execution

- **Adaptive algorithms**: adjust aggressiveness based on real-time spread, depth, and vol. Hybrid of IS + POV.
- **Smart Order Routing (SOR)**: split orders across lit and dark venues for best price. Routes to ECN with best quote net of fees.
- **Dark pools**: non-displayed liquidity, reduces leakage on large orders. Risk: adverse selection inside the pool (being a liquidity provider to informed flow).
- **Liquidity-seeking**: probe multiple venues opportunistically for hidden liquidity
- **RL-based execution**: Deep Q-Learning or PPO to dynamically optimize slice placement. Nevmyvaka et al. (2006) was early; modern work uses LOB snapshots as state.

## Microstructure signals

### Order book imbalance
```
imbalance = (bid_depth − ask_depth) / (bid_depth + ask_depth)
```
Positive imbalance → short-term upward drift pressure. Widely used intraday signal for market makers and short-horizon execution.

### VPIN (Volume-synchronized PIN, Easley-López de Prado-O'Hara 2012)
Bucket trades into equal-volume buckets. Measure buy-vs-sell imbalance per bucket. High VPIN = high toxic flow = informed trading active.

Use: widen quotes or throttle participation when VPIN spikes.

### Kyle's lambda
Price impact coefficient: `ΔP = λ · V` where V is signed volume. Estimate via regression on recent trades. High lambda = thin book; slow down aggressive orders.

### Microprice (Stoikov 2018)
Volume-weighted midpoint:
```
microprice = (bid · ask_size + ask · bid_size) / (bid_size + ask_size)
```
Better fair-value estimator than simple mid. Leans toward the thicker side of the book (where the next trade is more likely).

### Effective spread
```
effective_spread = 2 · |trade_price − mid_at_trade_time|
```
Actual cost paid vs quoted spread. Often wider than quoted due to hidden orders and slippage.

## Transaction cost analysis (TCA)

Post-trade, decompose execution cost:

- **Bid-ask spread cost**: half-spread paid
- **Market impact**: price move attributable to the order
- **Timing cost**: alpha lost while waiting to execute
- **Opportunity cost**: unexecuted portion that subsequently moved favorably

TCA reports inform algorithm selection for future orders.

## Go for execution systems

Execution is one of the sweet spots for Go:

- **Goroutines** for concurrent venue monitoring (N venues × M symbols)
- **Channels** for pipeline stages: signal → router → executor → confirm
- **Sub-microsecond latency achievable** with tuning (avoid allocation, use `sync.Pool`, pin GC, avoid reflection)
- **FIX protocol**: `go-trader` (robaho) demonstrates 90K+ quotes/sec at sub-ms latency
- **gRPC alternative**: 400K+ quotes/sec, sub-600μs per published benchmarks
- Coinbase reports sub-50μs E2E for order path in Go

Below ~5μs, C++/Rust still dominate. Above that, well-tuned Go is competitive.

## Execution realities: fees, brokers, taxes

Academic execution literature ignores three things that dominate live P&L for retail and small-prop:

### Fees (the line item you'll forget)

- **Equity commissions**: most US retail brokers are now $0 commission on stocks
- **Options commissions**: $0.50–$0.65 per contract is the retail standard; "$0.65/contract" is *one part of the cost*
- **Exchange fees**: per-contract fees charged by the exchange where the option trades (ranges from $0.05 to $0.50+ per contract depending on exchange and customer/professional status)
- **OCC fees**: clearing fees per contract (small but non-zero)
- **ORF (Options Regulatory Fee)**: another per-contract charge
- **Section 31 fee** (sales side only on equity): SEC transaction fee
- **Locate / borrow fees** for shorts: invisible until you check the statement

A high-turnover backtest using "$0.65/contract" as the total cost is **off by 30–50%**. For 0DTE strategies where premium is small, total per-contract fees can exceed expected edge. Check broker statements for actual realized cost per trade, not headline commission.

### Broker reality

- **IBKR**: low-cost, professional-friendly, but interface and risk controls are unforgiving for new users. SPAN portfolio margin available with qualification.
- **Tastytrade / Tastyworks**: short-premium-friendly, mechanical-rule-friendly UI, professional-status fee structure can hit active traders
- **Schwab (TD Ameritrade legacy)**: retail-friendly, less aggressive on margin policy
- **Robinhood**: PFOF wholesaler routing, opaque fill quality, not appropriate for serious systematic
- **Margin policy varies wildly**: same trade, same account size, different initial and maintenance requirements at different brokers
- **Auto-liquidation thresholds vary**: some brokers liquidate at maintenance breach immediately, others give you intraday window
- **Order routing**: IBKR routes to lit venues by default; retail-facing brokers route to wholesalers (PFOF). This affects fill quality measurably for marketable orders.
- **API stability**: TWS / IB API has historical outages; Tastytrade API is newer but improving; Schwab/TDA API has migration risk

For a live system spec, pick one broker and instrument the integration to handle that broker's specific quirks. Don't write portable code until you understand the abstraction is leaking.

### Taxes

- **Section 1256 contracts** (broad-based index options including SPX, SPY index futures): 60/40 long-term/short-term tax treatment regardless of holding period. Materially better after-tax Sharpe than equity options for high-turnover strategies.
- **Equity options** (single-name, ETF options like SPY, QQQ, IWM): short-term capital gains if held < 1 year — almost always for active strategies
- **Wash sale rule**: realized losses on substantially identical securities within 30 days are disallowed; can produce nasty year-end surprises for high-turnover strategies
- **Constructive sale rule**: short-against-the-box and similar structures can trigger constructive realization
- **Tax-loss harvesting** at year-end is a legitimate optimization for taxable accounts but adds operational complexity

Section 1256 alone is a structural reason to prefer SPX over SPY for high-frequency premium selling at meaningful size. **This affects after-tax Sharpe by 5–15 percentage points** for short-term strategies — larger than most "alpha" improvements.

### See also

- `options_operational_risks.md` for assignment, pin risk, IVR/IVP, and other operational traps
- `position_lifecycle.md` for fill handling and reconciliation

## Risk controls specific to execution

1. **Max order size vs ADV** — cap single order to X% of average daily volume
2. **Price band** — reject orders outside ±Y% of mid; protects against fat-finger
3. **Cancel-on-disconnect** — exchange and client-side dead-man switches
4. **Rate limiting** — per-symbol, per-venue, per-account
5. **Kill switch** — instant flatten-and-halt on anomaly detection
6. **Fill monitoring** — alert on execution latency spikes (bad venue health)
7. **TCA feedback loop** — track slippage per symbol; deprioritize venues with consistent bad fills

## Python libraries (for research / analysis)

- `ccxt` — unified exchange API (crypto primarily)
- `ibapi` — Interactive Brokers API
- `backtrader`, `zipline`, `vectorbt` — backtesting with execution models
- `pyfolio` — portfolio tear sheets including execution analysis

For production execution, Python is typically too slow; use Go, C++, or Rust.

## References

- Almgren & Chriss (2001) — optimal execution with market impact
- Bertsimas & Lo (1998) — earlier execution framework
- Nevmyvaka, Feng, Kearns (2006) — RL for execution
- Easley, López de Prado, O'Hara (2012) — VPIN
- Stoikov (2018) — microprice
- Cartea, Jaimungal, Penalva — *Algorithmic and High-Frequency Trading*
- Kissell — *The Science of Algorithmic Trading and Portfolio Management*
- `go-trader` (GitHub: robaho) — Go FIX reference impl
- USENIX SREcon23 — ultra-low-latency Go trading systems
