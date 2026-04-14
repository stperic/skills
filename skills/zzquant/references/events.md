# Event-Driven Strategies & Calendar Risk

## Premise

Certain dates and announcements are *known in advance* to move prices: earnings, FOMC, CPI, dividends, index reconstitution, expirations. Event-driven strategies either trade these directly or actively avoid them. The calendar is itself a signal.

This file covers both: strategies that exploit events (earnings crush, FOMC drift, index rebal) and defenses for strategies that don't want to be exposed to them.

## Event categories

### Company-specific
- **Earnings announcements** (quarterly, scheduled)
- **Dividends** (ex-date, pay-date)
- **Stock splits**, **spinoffs**, **mergers**
- **Management changes**, **guidance revisions**
- **Clinical trial results** (biotech)
- **Product launches**, **FDA decisions**

### Macro
- **FOMC meetings** (8× per year)
- **CPI / PPI / employment reports**
- **GDP releases**
- **Central bank speeches**
- **Treasury auctions** (affect rates)
- **OPEC meetings** (affect oil)

### Market structure
- **Index reconstitution** (Russell rebalance in June, S&P quarterly)
- **Options expiration** (third Friday monthly; now weekly and 0DTE daily)
- **Futures roll** (quarterly for equity index, monthly for some commodities)
- **Rebalancing dates** (quarter-end, month-end window dressing)
- **Triple/quadruple witching**

### Geopolitical
- **Elections**, **referendums**
- **Regulatory announcements**
- **Tariff decisions**
- **Central bank crises**

## Earnings: the biggest single event

### The IV crush mechanic
Leading into earnings:
- Implied vol rises ~2–5 points above normal
- The weekly expiration closest to earnings absorbs most of the premium
- ATM straddle implies an expected move (`~0.85 × straddle_price / stock_price`)

On the announcement:
- Realized move is distributed around the expected move
- Implied vol *collapses* (IV crush) — front-month can drop 30–60% overnight
- Move magnitude determines directional P&L; IV crush determines vol P&L

### Strategies for earnings

1. **Long straddle / strangle into earnings**
   - Bet on the realized move exceeding the implied move
   - Pays off if the stock gaps significantly
   - Loses to IV crush if the move is small
   - Historically negative expected value on average: the expected move is well-priced

2. **Short straddle / strangle into earnings**
   - Harvest the IV crush
   - Profits if the realized move is inside the expected move
   - Catastrophic if earnings shock (10σ tails happen)
   - Must be defined-risk (iron condor) for survivability

3. **Calendar spread**
   - Short front-month (high IV), long back-month (lower IV)
   - Profits from front-month IV crushing while back-month holds
   - Standard retail approach; limited risk, limited reward

4. **Earnings drift (PEAD)**
   - Post-earnings announcement drift: stocks that surprise positively continue up for days; surprise down continues down
   - Academic anomaly documented since Bernard & Thomas (1989)
   - Decayed substantially since publication; still mentioned as a reference for calendar-aware momentum

### Earnings data traps

- **Pre-market vs after-market announcements**: a company reporting before the open has its earnings "for" day T but the first tradeable price is open of T. A company reporting after close on T has earnings "for" T but the first tradeable is open of T+1. Backtests routinely get this wrong.
- **Estimate vintage**: "consensus estimate" as reported today is not what it was before the announcement. Use estimate history (I/B/E/S Detail History, WRDS)
- **Surprise definition**: actual − estimate / estimate vs actual − estimate / price — define explicitly and stick with it
- **Whisper numbers**: not in consensus; affect reaction but hard to source reliably
- **Reaction windows**: immediate reaction (market open / close) differs from drift (days 1–60)

## FOMC and macro events

### Price behavior around FOMC

- **Pre-FOMC drift**: Lucca & Moench (2015) documented an anomalous average equity return in the 24 hours *before* each FOMC announcement. Decayed but still referenced.
- **Release minute**: equity vol spike at exactly 2pm ET (announcement time), again at 2:30pm (press conference)
- **Press conference tail**: majority of post-release moves happen during the press conference, not at the statement release
- **Next-day drift**: direction of drift varies; no stable pattern post-2015

### Trading FOMC

- **Long vol into**: buy straddles a few days before, exit before release (IV tends to expand into the release, crush after)
- **Short vol after**: sell premium after the event to harvest the crush
- **Avoid altogether**: many systematic strategies simply halt on FOMC day — the cleanest solution

### CPI / employment

Similar structure to FOMC but less pronounced. Release times are known and consistent; vol spikes at release minute. Defenses: halt new entries N minutes before; wait for the first N minutes of post-release price action to normalize.

## Dividends

### Ex-dividend date mechanics
- Stock drops by dividend amount on ex-date
- Options are dividend-adjusted: long call loses intrinsic value equal to dividend on ex-date
- Early exercise of ITM American calls becomes optimal just before ex-date for high-dividend stocks

### Dividend capture
- Buy just before ex-date, sell just after, collect the dividend
- Price drop typically captures the dividend, leaving no net profit
- Historical anomalies exist for very short holding periods in specific structures; mostly closed
- Tax implications dominate returns (qualified vs ordinary)

### Dividend traps in options strategies
- Short calls can get early-exercised the day before ex-date if deep ITM
- Covered call sellers lose the dividend if assigned
- Bid-ask spreads widen on ex-date morning

## Index reconstitution

### Russell annual reconstitution
- Early June: preliminary index changes announced
- Late June: rebalance takes effect (last Friday)
- Predictable flow: funds tracking the index must buy new additions, sell deletions
- Pre-announcement positioning was highly profitable; crowded out post-2000

### S&P 500 additions/deletions
- Announced on irregular schedule, typically 5 business days before effective date
- Additions: indexers must buy, typically 3–5% premium in the window
- Deletions: indexers must sell, typically negative pressure
- "Index effect" has decayed substantially (Patel & Welch 2017) but still observable in liquid retail names

### Defensive notes
- Names being added or removed have atypical flow in the reconstitution window
- Backtests that don't account for the rebalance date can misattribute returns
- Reference data must track index membership over time (see `data_quality.md`)

## Options expiration

### Pin risk
- On expiration Friday, there is pressure for SPX and popular strikes to "pin" to round-number strikes
- Stocks with large open interest concentrated at a strike often close at that strike
- Partly a myth (most "pinning" is noise), partly real (market makers hedging gamma near expiration)

### Gamma concentration
- Near expiration, option gamma explodes
- Large dealer positions in short-dated options drive intraday price action
- 0DTE options have amplified this effect post-2022 SPX daily expirations

### Triple/quadruple witching
- Quarterly expiration of index futures, index options, stock options, stock futures
- Elevated volume and volatility at the open
- Historically significant, less so in recent years

### Defensive notes
- Strategies that don't want expiration exposure should close positions by 2 DTE or before the weekly expiry
- Strategies that *want* gamma exposure (scalpers, MMs) trade around expiration deliberately
- Backtests must handle expiration as discrete events, not just time decay

## Calendar-aware strategy design

### Event avoidance (defensive)
- Maintain a calendar of upcoming events
- Flag positions that will be held over an event
- Either close before, hedge, or accept
- Most systematic short-premium strategies avoid new entries into earnings and FOMC

### Event-targeted (offensive)
- Strategies that *specifically* trade around events (earnings calendars, FOMC straddles)
- Require clean event data (dates, times, consensus estimates)
- Backtest sensitivity to event mis-dating is extreme — a single misaligned earnings date can flip the result

### Event window definitions
- **Pre-event window**: N days before the event
- **Event day**: the announcement date (+ reaction period, often to next day open)
- **Post-event window**: N days after
- Document the windows in the backtest; don't let them shift under you

## Data requirements

Event-driven strategies need more and cleaner data than signal strategies:

- **Earnings calendar** with pre-market / after-market timing
- **Estimate history** (I/B/E/S Detail History via WRDS)
- **Macro release schedule** (Bloomberg, BEA, BLS feeds)
- **Dividend calendar** (ex-date, amount)
- **Index membership history** (Compustat, CRSP)
- **Options chain with expiration type** (weekly vs monthly vs quarterly)

A hole in any of these → specific false backtest results.

## Risk controls specific to events

1. **Event calendar monitor**: all upcoming events surfaced in strategy context
2. **Auto-halt pre-event**: configurable halt window (e.g., "no new entries 24h before FOMC")
3. **Event-aware exits**: existing positions closed or hedged before overnight exposure to a scheduled event
4. **Sizing cap during event windows**: reduce risk around volatile dates
5. **Post-event re-entry filter**: wait for first N minutes of post-event price action before re-enabling
6. **Earnings-aware short premium**: no new short premium on names with earnings inside DTE
7. **Calendar reconciliation**: daily check that the event calendar matches the broker / vendor view

## References

- Bernard & Thomas (1989) — "Post-Earnings-Announcement Drift"
- Fama (1998) — "Market Efficiency, Long-Term Returns, and Behavioral Finance"
- Lucca & Moench (2015) — "The Pre-FOMC Announcement Drift"
- Patel & Welch (2017) — "Index Changes and Unexpected Losses to Investors"
- Cooper, Gulen, Vanden (2011) — "Mutual Fund Performance Around Events"
- Savor & Wilson (2013) — "How Much Do Investors Care About Macroeconomic Risk?"
- So & Wang (2014) — "News-driven return reversals"
- CBOE research — 0DTE and weekly expiration behavior post-2022
