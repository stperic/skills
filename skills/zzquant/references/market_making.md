# Market Making

## Premise

Provide liquidity by continuously posting bid and ask quotes, profiting from the spread while managing inventory risk and adverse selection. Edge is operational (speed, inventory skill) rather than predictive.

## Core mechanics

Post bid at `mid - δ_bid` and ask at `mid + δ_ask`. Profit ≈ captured spread − inventory holding cost − adverse-selection cost.

Three things must all be true for the quote to be profitable:
1. The mid-price is correct (no stale quote vs fair value)
2. Fills come from noise traders more often than informed traders
3. Inventory can be offloaded before the market moves against it

## Avellaneda-Stoikov model

> **Important**: AS is a **pedagogical skeleton**, not production code. It assumes Poisson order-arrival with exponential intensity, constant volatility, a single asset, no queue position, no latency, and an adverse-selection model that collapses into the single parameter `k`. **No real market-making desk uses AS as-written.** Every production MM uses it (or its derivatives) as a starting point, then layers on: empirically calibrated fill models, microprice fair-value estimates, queue-position tracking, latency-aware quote ladders, toxicity overlays (VPIN, order-flow imbalance), and dealer-flow / GEX context. A junior who builds an AS engine without these layers gets adversely selected and concludes the data is wrong.

The seminal HFT market-making framework. Given inventory `q`, time-to-horizon `T−t`, volatility `σ`, risk aversion `γ`, order-flow intensity `k`:

**Reservation price** (the MM's private fair value, shifted by inventory):
```
r = s − q · γ · σ² · (T − t)
```
When long (`q > 0`), r shifts below mid → quotes get skewed to sell. When short, opposite.

**Optimal half-spread**:
```
δ = (γ · σ² · (T − t)) / 2 + (1/γ) · ln(1 + γ/k)
```

**Bid and ask**:
```
bid = r − δ
ask = r + δ
```

Interpretation: spread grows with volatility and risk aversion, shrinks when fill intensity is high. Inventory is mean-reverted toward zero by the `q·γ·σ²` skew.

Guéant, Lehalle, Fernandez-Tapia (2012) extended to infinite-horizon and multi-asset. Gives closed-form for bounded inventory.

## Inventory management

Inventory is the core risk. Options:

- **Skew quotes** (Avellaneda-Stoikov style): shift r based on q
- **Hard inventory caps**: stop quoting one side at limit
- **Target-zero strategy**: aggressively hit the offsetting side when |q| exceeds threshold
- **Hedge externally**: futures or ETF hedge against accumulated inventory
- **End-of-day flat**: reduce or eliminate overnight inventory risk

## Adverse selection & toxic flow

If your quote gets hit, it's because someone wanted your side at that price. Informed traders are the enemy — they fill you, then the market moves against you.

**VPIN** (Volume-synchronized Probability of Informed Trading, Easley-López de Prado-O'Hara 2012):
- Bucket trades into volume-equal buckets
- Measure buy vs sell imbalance per bucket
- High VPIN = high toxic flow
- When VPIN spikes, widen spreads or pull quotes

**Kyle's lambda**: price impact per unit volume. Estimate from historical fills. If lambda is high on your recent trades, you're trading with informed flow — back off.

## Where it's used

- **Equity MM**: Citadel, Virtu, Jane Street
- **ETF market making**: Jane Street is widely reported as the largest ETF market maker; specific volume-share numbers vary by source and year — treat with skepticism
- **Options market making**: captures bid-ask spread *plus* the VRP (structural edge)
- **Crypto MM**: wide spreads, high vol, high fragmentation, high opportunity
- **Futures MM**: on-exchange programs, rebate-based economics

### The actual moat: flow segmentation, not latency

For US equity MM in particular, the dominant economic edge post-2010 is **access to retail order flow** routed via wholesalers (PFOF / payment-for-order-flow). Retail flow is structurally less informed than institutional flow, so a wholesaler that fills retail orders captures a positive expected spread that is wider than what's offered to institutional flow. Brokers route to wholesalers and receive PFOF; wholesaler and broker share the rents.

Latency still matters for residual lit-market flow but it is not the moat. A new entrant cannot compete with Citadel Securities on retail flow without becoming a wholesaler themselves (regulatory approval, capital, broker relationships).

For **options and futures** MM, the picture is different: PFOF in options exists but is smaller relative to spread; latency, SPAN margining, and pricing skill still matter. Futures MM has minimal retail flow.

See `market_structure_2020s.md` §PFOF for the full picture.

## Infrastructure realities

MM is an infrastructure sport. Winning shops:
- Co-locate at exchanges
- Use kernel bypass networking (DPDK, Solarflare)
- Tail-optimized languages: C++, Rust, and more recently Go for non-critical paths
- FPGA for the hottest path
- Sub-microsecond quote-to-ack targets on major venues

**Go notes**: Go trades well in the 10–500 μs tier. Goroutines handle concurrent venue streams; `sync.Pool` eliminates hot-path allocation; static binaries ease colocation deployment. GC pause < 500 μs post-1.14 is acceptable for most non-FPGA MM. Below ~5 μs E2E, C++/Rust dominate.

## Risk controls

1. **Inventory cap** — hard stop on |q|
2. **Spread floor** — never quote tighter than fair-value uncertainty
3. **Circuit breaker on drawdown** — kill switch when session P&L breaches threshold
4. **Staleness monitor** — drop quotes if mid-price feed lags
5. **Cancel on loss of connectivity** — exchange-side auto-cancel; also client-side dead-man switch
6. **Per-symbol/per-venue limits** — don't let one symbol or venue dominate risk

## References

- Avellaneda & Stoikov (2008) — "High-frequency trading in a limit order book"
- Guéant, Lehalle, Fernandez-Tapia (2012) — optimal MM with general execution
- Easley, López de Prado, O'Hara (2012) — VPIN and flow toxicity
- Stoikov (2018) — "The Micro-Price" (microprice estimator)
- Cartea, Jaimungal, Penalva — *Algorithmic and High-Frequency Trading*
- `go-hft-orderbook` (GitHub) — Go LOB reference implementation
