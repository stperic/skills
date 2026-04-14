# Options Operational Risks

## Why this file exists

`options_premium.md` covers the strategy-level rules of selling premium. This file covers what can go wrong **operationally** — the ways live options trading produces P&L surprises that no backtest captures and no Greeks model predicts. These are what bite junior options traders first, and most short-premium blowups are operational, not strategic.

## Pin risk

### What it is
At expiration, if the underlying closes very close to a strike where you have open contracts, you can be partially or fully assigned (or not), and you don't know which until well after the close. The position is "pinned" to the strike.

### Failure modes
- **Short call near strike**: closes 1 cent ITM → likely assigned → you're suddenly short 100 shares per contract Monday morning
- **Short put near strike**: closes 1 cent ITM → likely assigned → you're suddenly long 100 shares per contract
- **Long option near strike**: broker default is auto-exercise of ITM longs by 0.01; you can override with DNE (Do-Not-Exercise) instructions before the cutoff

### What goes wrong
- Friday close at $450.01 on a stock with $450 short calls
- You expect the calls to expire worthless (1 cent OTM at the close)
- After-hours print at $450.10 → ITM, OCC may auto-exercise → you may be assigned
- Or: stock gaps Sunday on news; Monday open is $440 → you're short stock at $450, $10 unrealized loss before you even know
- Or: long-call holder pays the exercise fee anyway because they want the stock — assignment isn't a function of intrinsic value alone

### Defense
- **Close any short option that closes within 0.5% of strike before market close** — eat the bid-ask, sleep at night
- **Know your broker's auto-exercise / DNE policy** — it varies
- **Document a pin-risk threshold** (e.g., "close any short within X cents of strike at 15:50 ET")
- **Index options (SPX) are European cash-settled** — no pin risk on assignment, only on settlement price (still a risk for very near-the-money positions at AM-settled cash close)

## Early assignment on short American options

### When it happens
American-style options (single-name equity, ETFs except SPX) can be exercised by the holder at any time. **You can be assigned at any time, not just at expiration.**

### Triggers in order of frequency

#### 1. Day before ex-dividend on short ITM calls (the biggest single source)
- A short ITM call holder will rationally exercise the day before ex-dividend if `dividend > remaining_extrinsic_value`
- You wake up assigned: short 100 shares per contract, owing the dividend, no upside left from time premium
- **This is the most common tactical bug in retail short premium** — junior traders forget to check ex-div dates and get blown up on covered calls and short call verticals

#### 2. Deep ITM with little extrinsic left
- A short put deep ITM with extrinsic < commission to close → holder may exercise
- A short call deep ITM with extrinsic < dividend → holder will exercise
- Mechanism: the long holder rationally captures intrinsic and avoids the time-decay risk of waiting for expiration

#### 3. Hard-to-borrow squeezes
- If your assigned short shares are HTB, the broker can charge huge borrow rates or buy you in
- During squeezes (GME 2021), borrow rates hit thousands of percent annualized
- A squeeze on a stock you don't even own can cost you tens of thousands

### Defense
- **Maintain an ex-dividend calendar** for any name with open short ITM calls
- **Close short ITM calls 2+ days before ex-div** if extrinsic < dividend
- **Avoid short premium on names paying high dividends near expiration**
- **Avoid undefined-risk shorts on hard-to-borrow names entirely**
- **Prefer index options (SPX) over single names for short premium** — European cash-settled, no early assignment, no dividend exercise risk. This is a structural reason to prefer SPX over SPY at any meaningful size.

## Dividend exercise math

```
exercise value to long call holder = dividend captured
loss to long call holder = extrinsic value extinguished
exercise is rational when: dividend > extrinsic
```

For a short call holder (you):
1. Check every short call against the next ex-dividend date
2. Check if the call has more extrinsic than the dividend
3. If not, expect to be assigned
4. Brokers don't warn you. The assignment notice arrives the next morning.

## IV Rank vs IV Percentile

This belongs in operational risks because the wrong choice produces consistently wrong sizing decisions across every short-premium strategy.

### Definitions
- **IV Rank (IVR)**: `(current_IV − 52wk_min_IV) / (52wk_max_IV − 52wk_min_IV)`. Linear scaling between observed extremes.
- **IV Percentile (IVP)**: fraction of trading days in the past year where IV was lower than current. Distribution-free; rank-based.

### Why IVP is statistically better
- IVR is **dominated by extremes**. One vol spike (March 2020, August 2024 yen carry unwind) sets the max; for ~6 months afterward IVR sits near zero even when current IV is well above its long-run median.
- IVP is **distribution-free**. It tells you "current IV is at the 60th percentile of the past year" — robust to outliers.
- IVR is **easier to compute** (one division, two extreme lookups). That is its only advantage.

### Why this matters operationally
- Strategies that gate entries on `IVR > 50` were systematically *out of the market* for the 6+ months following March 2020 — exactly when short-vol harvesting was most profitable
- A portfolio sized on IVR will under-allocate during the rebuild phase after a vol shock
- Switching to IVP for sizing immediately reveals premium-rich periods that IVR was hiding

### Practical recommendation
- **Use IVP as the entry / sizing metric**
- Compute IVR alongside it for legacy compatibility and tooling that uses it
- If the two disagree substantially (e.g., IVR = 5, IVP = 50), trust IVP — the disagreement means a stale extreme is dominating IVR
- The tastytrade preference for IVR is a tooling preference, not a statistical one

## Margin and haircut surprises

### What backtests don't show
- Brokers can raise initial and maintenance margin at any time
- During stress events, brokers raise haircuts on volatile names by 2–3× *exactly when you most need capacity*
- Several short-vol funds blew up in March 2020 not from realized losses but from forced de-grossing as haircuts expanded
- LJM Preservation & Growth Fund (Feb 2018) is a textbook example: short-vol strategy, single bad day, forced liquidation by FCM, fund ceased operations

### Defense
- **Keep a haircut buffer**: maintain liquid cash equal to expected stress-haircut expansion
- **Monitor broker margin notices**: subscribe to margin policy updates
- **Stress-test under expanded haircuts**: run your portfolio through 1.5× and 2× current margin
- **Diversify brokers** at scale: a single broker's policy change shouldn't take you out
- **Defined-risk by default in small accounts**: caps the haircut surprise

## What you cannot backtest

A backtest cannot model:

1. **Borrow rates and locate availability** — historical borrow data is poor; recall events are not well documented
2. **PFOF wholesaler fills** — your retail order would have routed through a wholesaler; backtest assumes lit-market fills
3. **OCC exercise cutoffs** — corner cases in exercise mechanics
4. **Margin call timing** — you might have been forced to close before your strategy wanted to
5. **Auto-liquidation triggers** — broker-side risk thresholds you don't control
6. **Queue position at the exchange** — you might not have been filled where backtest assumes
7. **Auction-cross fills** — opening and closing auctions have different dynamics from continuous trading
8. **Halt and circuit breaker behavior** — your strategy can't trade halted names
9. **Pin and assignment** as discussed above
10. **Dividend exercise** as discussed above
11. **Haircut expansion in stress** as discussed above
12. **Broker outages** — your strategy might have been unable to manage positions during a feed outage
13. **Section 1256 vs short-term tax treatment** — affects after-tax Sharpe materially for short premium (SPX gets 60/40, SPY does not)
14. **Exchange and OCC fees** — per-contract fees, especially around 0DTE, can make or break thin strategies

A backtest can be perfectly deterministic, point-in-time, and realistic in every modelable dimension and still be wrong about live P&L by 20–50% because of the above. The defense is paper trading, cautious sizing during ramp, and explicit stress reserve — not better backtesting.

## Risk controls specific to operational risks

1. **Pre-close pin sweep**: at 15:50 ET, scan all positions for pin proximity; close any within X cents of strike
2. **Ex-div calendar integration**: every short ITM call has an associated ex-div date; flag if extrinsic < dividend
3. **Borrow monitor**: for short positions (including assignments), track borrow rate and recall risk
4. **Margin headroom alert**: if margin utilization > X%, auto-degross
5. **Exercise / assignment audit**: every morning, reconcile assignments with expectations; investigate surprises
6. **HTB exclusion**: maintain a list of HTB names; exclude from undefined-risk short premium
7. **Cash buffer**: hold N% of equity in cash to absorb haircut expansion
8. **IVP-based sizing**: replace IVR-based gates with IVP to avoid post-shock dead zones

## References

- OCC (Options Clearing Corporation) — exercise and assignment rules, https://www.theocc.com
- SEC Rule 605/606 — broker execution quality and routing disclosure
- Battalio, Corwin, Jennings — "Can Brokers Have It All?" (PFOF execution quality)
- Natenberg — *Option Volatility and Pricing* (assignment mechanics, pin risk)
- McMillan — *Options as a Strategic Investment* (operational chapters)
- broker-specific margin policy documents (IBKR, Tastyworks, Schwab) — read the fine print
- LJM Preservation & Growth Fund (Feb 2018) — case study in margin-call cascade
