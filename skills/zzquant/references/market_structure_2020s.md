# Market Structure: Post-2020 Realities

## Why this file exists

Most quant references — including most of this skill — describe markets as they were 2010–2019. Post-2020, three structural changes have rewritten the intraday behavior of US equity index options and the economics of liquidity provision:

1. **The 0DTE explosion in SPX**
2. **Dealer gamma / charm / vanna flows as dominant intraday drivers**
3. **PFOF (payment for order flow) and retail flow segmentation**

If you are trading SPX options, providing liquidity in equities, or running any intraday vol strategy, ignoring these is malpractice. Classical references that don't address them are correct in the textbook sense and incomplete in the live sense.

## 0DTE: the dominant flow in SPX

### Scale
- 0DTE = options expiring same day
- Pre-2022: SPX had Mon/Wed/Fri weeklies; 0DTE was a niche product
- May 2022: CBOE added Tue/Thu expirations → SPX has 5-day weekly cycles
- 2024–25 industry estimates: 0DTE ≈ 45–50% of SPX options volume
- Comparable but slower expansion in QQQ, SPY

### What it broke

- **Front-month VRP compression.** Realized hedging by 0DTE participants compressed the implied–realized gap on near-dated SPX. The unconditional historical mean (2–4 vol points) is no longer a reliable point estimate for current conditional VRP — recent regimes have been closer to 1–1.5 vol points with frequent negative episodes.
- **Calendar structure can invert intraday.** 1DTE can be cheaper than 0DTE on quiet mornings (no time premium left in 0DTE), more expensive on event days.
- **Theta curve is no longer smooth.** 0DTE theta is concentrated in the last 3 hours of the session, not exponentially distributed across DTE as classical models assume.
- **The 50% profit rule is meaningless on 0DTE.** There is no "next day" to take profit.
- **The 21 DTE roll rule is meaningless on 0DTE.** No roll horizon exists.
- **Most tastytrade-derived rules** were calibrated on 30–50 DTE strangles in pre-2018 SPY data and do not transfer to 0DTE without recalibration.

### Trading implications

- **Short 0DTE iron condor at the open**: works in low-vol mornings, blows up in directional sessions. Negative tail.
- **Long 0DTE straddle into Fed/CPI minutes**: accepted retail vol play; generally negative-EV due to IV crush at the announcement.
- **Dealer-flow-aware 0DTE**: institutional approach; trade direction predicted by dealer gamma profile (see below). Requires GEX-style data and a thesis about dealer hedging behavior.

### Operational cautions

- **Liquidity is fragmented across strikes** — many 0DTE strikes have wide spreads
- **Pin risk concentrates** at popular strikes near close
- **Settlement**: SPX is European cash-settled (no early exercise); SPY is American physical (assignment risk on shorts)
- **Fees** at exchange and OCC level are non-trivial relative to per-contract premium at 0DTE prices

## Dealer gamma, charm, vanna

### The setup
Market makers and dealers are systematically short gamma against typical retail-facing flow (long stock positioning + short calls + short puts in net). When dealers are short gamma, they must hedge directionally:

- Stock rises → dealers buy stock to delta-neutral → amplifies the move
- Stock falls → dealers sell stock → amplifies the move

This is the **dealer gamma hedging flow**. Post-2020, with 0DTE volume amplifying it, it has become a dominant intraday driver of SPX price action.

### Net dealer gamma exposure (GEX)

- Aggregate dealer gamma at every strike, weighted by open interest
- **Negative GEX**: dealers are short gamma → moves amplify, vol-of-vol high, trends extend
- **Positive GEX**: dealers are long gamma → moves dampen, intraday range compresses, mean-reversion works
- **Inflection level**: the price at which net dealer gamma flips sign — historically a magnet

### Charm (delta decay over time)

- ∂Δ/∂t — how delta changes as time passes, holding spot constant
- For dealers short ATM options near expiration, charm becomes large in the last hours of the session
- Drives the "afternoon drift" that some 0DTE strategies attempt to predict

### Vanna (delta sensitivity to vol)

- ∂Δ/∂σ
- When vol drops, ATM call deltas rise and ATM put deltas fall — dealers must rebalance
- Drives the post-2020-strengthened "vol-down → equity-up" linkage

### Practical use

- **GEX-based regime classifier**: positive vs negative GEX as a regime variable for any intraday strategy
- **Avoid mean-reversion in negative-GEX regimes**: moves are amplified and trending, not reverting
- **Sources**: SpotGamma, GEX Index (CBOE), Goldman / Nomura research notes, your own computation from chain data
- **None of these are perfect** — dealer positioning is opaque; what's measured is *retail-facing* dealer positioning, not the full picture (institutional vega, structured products, etc.)

### Why classical vol surface references miss this

`vol_surface.md` describes SVI, SABR, Gatheral's classical models. They are correct in their domain but they describe *static* surfaces fit to options prices. They don't model the dealer hedging feedback that drives intraday spot. For intraday work in 2024+, you need *both* the classical surface and a flow / GEX overlay.

## PFOF and retail flow segmentation

### What changed

US equity market making used to be primarily a latency-and-inventory game. Post-2010 (and especially post-2020 retail boom), the dominant economic edge in equity MM is **access to retail order flow** routed via wholesalers (Citadel Securities, Virtu, G1X, Jane Street).

- Retail flow is structurally less informed than institutional flow
- A wholesaler that fills retail orders captures a positive expected spread that's wider than the spread offered to institutional flow
- Brokers route to wholesalers and receive payment per share/contract — payment for order flow (PFOF)
- Wholesaler and broker share the rents

### Implications

- **Equity MM economics are now flow-segmentation economics, not pure latency economics.** Latency still matters for residual lit-market flow, but the moat is the broker relationship.
- **A new entrant cannot compete with Citadel Securities on retail flow** without becoming a wholesaler themselves (regulatory approval, capital, broker relationships).
- **For options and futures**, the picture is different:
  - **Options**: PFOF exists but is smaller relative to spread; latency, SPAN margining, and pricing skill still matter
  - **Futures**: minimal retail flow; latency and SPAN dominate

### `market_making.md` over-indexes on latency

The MM file in this skill describes Avellaneda-Stoikov and inventory management as if they were the core edge. They are necessary but not sufficient for equities. The actual edge is access to non-toxic flow.

### Cautions for backtesting

- A backtest of an MM strategy on lit-market data is testing the *residual* flow, not the wholesaler-routed flow
- A profitable backtest on TAQ data is testing something different from what Citadel does
- Retail traders cannot replicate this edge — they have no flow to internalize

## Implications for `zzquant` users

If you are using this skill to design a strategy in 2024+, the relevant questions:

- **Does your strategy depend on the pre-2018 VRP magnitude?** If yes, recalibrate to the compressed regime.
- **Does it trade SPX intraday without considering 0DTE flow?** If yes, you are trading against participants who do.
- **Does it assume static vol surface dynamics?** If yes, you are missing dealer feedback effects.
- **Does it assume MM strategies generalize from a lit-market backtest?** If yes, you are testing the wrong sample.

The classical references (Natenberg, Bennett, Gatheral, Avellaneda-Stoikov) are not wrong. They are *incomplete* for current market structure. Use them as foundation; layer the post-2020 realities on top.

## References

- CBOE — research notes on 0DTE growth and volume composition
- Brogaard, Ringgenberg, Sovich — "The Economic Impact of Index Investing" and follow-ups on dealer flows
- SpotGamma — independent dealer flow analytics (subscription service)
- Goldman Sachs FICC — regular vol/gamma research notes
- Nomura QIS — published GEX framework
- SEC Order Type Definitions and Rule 605/606 reports for PFOF transparency
- Battalio, Corwin, Jennings — academic critiques of PFOF impact on execution quality
- Bryzgalova, Pavlova, Sikorskaya (2023) — "Retail Trading in Options and the Rise of the Big Three Wholesalers"
