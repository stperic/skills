# Data Quality & Pipeline Defenses

## Why this file exists

Bad data kills strategies silently. A robust strategy on corrupt inputs is still wrong; a modest strategy on clean inputs is often fine. More backtests are broken by data quality bugs than by modeling errors — and the bugs are frequently invisible until after money is deployed. This file is a catalog of the common failure modes and the defenses against them.

## The hierarchy of data quality problems

1. **Missing data** — rows that should exist don't
2. **Wrong data** — rows exist but values are incorrect
3. **Stale data** — values exist but weren't updated on schedule
4. **Misaligned data** — values exist but are mapped to the wrong date / symbol / field
5. **Fabricated data** — synthetic fill-ins that are indistinguishable from real
6. **Leaked data** — data exists in backtest that wouldn't have existed in live

(6) is the most dangerous because it looks like a strategy working; see `backtest_determinism.md`.

## Equities / price data

### Corporate actions
- **Splits**: prices pre-split must be adjusted; unadjusted raw prices in a backtest produce phantom gaps
- **Cash dividends**: adjusted back-prices subtract dividend from pre-ex-date levels; unadjusted leave a phantom drop
- **Stock dividends**: treated similarly to splits
- **Spinoffs**: parent's post-spinoff price has the spun-off value removed
- **Rights issues**: similar mechanics, different adjustment factor
- **Mergers**: target stock typically jumps to deal price and goes flat

Defense: use a vendor's **total-return-adjusted** series for research, and **unadjusted** for order size calculation. Mixing the two is a classic bug.

### Survivorship bias
- Your dataset today does not include names that were delisted
- Any universe backtest that queries "the S&P 500 stocks" without historical membership dates is survivorship-biased

Defense: PIT index membership tables, explicit delisting records. See `backtest_determinism.md`.

### Stale quotes
- Low-volume names can have quotes that haven't updated in minutes or hours
- Backtesting with stale quotes ≈ backtesting with arbitrage opportunities that didn't exist
- Live detection: if `last_trade_time > N` minutes old, do not trade the name

### Trading halts
- Halted names may still appear in the feed with a stale last price
- Halt status must be honored — no orders, no fills, no signals

### Price glitches
- Flash crashes, fat-finger trades, bad prints
- A single outlier can trip a rolling-window calculation and propagate for N bars
- Robust statistics: prefer median over mean for rolling calcs; use Hampel filter for outlier detection

## Options / derivatives data

Options data quality is roughly 10× harder than equities. Every failure mode below is common:

### Chain completeness
- Are all strikes for a given expiration present?
- Are all expirations for a given underlying present?
- Vendor coverage varies; assume holes

### Bid-ask sanity
- **Wide spreads**: options with bid < 0.05 or ask / bid > 5 are effectively untradeable; filter
- **Zero bid**: `bid = 0` means no one wants it at any price; typically far-OTM, near-expiry
- **Crossed markets**: `bid > ask` — data bug, filter the row
- **Locked markets**: `bid = ask` — possible, but suspicious on retail-visible chains

### IV calculation consistency
- Different vendors compute implied vol with different assumptions (American vs European, dividend model, rate, root-finder tolerance)
- Your own IV must be recomputed from the same price series you use for signals — don't mix vendor IV with raw prices

### IV rank traps
- **IV Rank vs IV Percentile**: different definitions, different values
  - IV Rank: `(current − 52wk_min) / (52wk_max − 52wk_min)`
  - IV Percentile: fraction of days in past year with lower IV
- Document which one is in use; don't switch mid-study

### Volume and open interest
- **Zero volume / zero OI**: not tradeable; filter
- **Stale OI**: OI lags by a day; don't use today's OI for today's signals

### Historical options data gotchas
- Many providers only have end-of-day snapshots; intraday quotes are expensive
- Weekly, quarterly, LEAP expirations are sometimes missing from free feeds
- Corporate actions apply to options chains differently from stocks (strike adjustment, symbol changes)
- Post-2019 SPX weekly expansion increased chain complexity; older data may be sparser

### Earnings adjustments
- Options around earnings have known IV crush patterns
- Backtest that accidentally uses post-earnings IV as entry IV is leaking future information
- Defense: mark earnings dates explicitly; handle as events

## Fundamentals data

### Restatement bias
- Companies restate earnings; databases replace the original numbers with restated ones
- Backtest using restated data is using future information
- Defense: PIT databases (Compustat PIT, Refinitiv Point-in-Time)

### Reporting lag
- Fundamentals are known to traders ~45–90 days after period end
- Using "Q3 earnings" in a backtest on the last day of Q3 is wrong — they weren't published until November
- Defense: explicit reporting lag or use the as-of-report-date field

### Currency / unit changes
- Companies change reporting currency or units (millions vs thousands)
- Ratios computed across the boundary are garbage
- Defense: consistency checks (year-over-year change bounds, ratio plausibility)

### Corporate restructurings
- Segment reporting can change year-over-year; comparables break
- Defense: trust only fields that are consistently defined across periods

## Alternative data

### Backfill
- Alt data vendors often deliver historical data that was backfilled, not captured in real time
- Backfilled data may include information that wasn't actually available at the timestamp
- Defense: require explicit "as-of-delivery" timestamps; only use data with delivery time ≤ decision time

### Point-in-time sentiment
- News articles, social media posts have publication timestamps
- Crawls that collect them have crawl timestamps
- Your decision time must be ≥ crawl time, not just ≥ publication time (the data didn't exist in your system until the crawl)

### Geographic / holiday effects
- Satellite imagery has cloud cover, orbit gaps
- Credit card data has holiday and day-of-week effects
- Always check for periodic patterns before treating as pure signal

### Sample bias
- Alt data covers a subset of the population (e.g., one credit card network, one weather service)
- Conclusions generalize only if the sample is representative

## Sentiment / NLP data

### Encoding issues
- UTF-8 vs Latin-1, escaped characters, JSON re-encoding
- Always normalize to UTF-8 on ingest; validate byte order mark absence

### Ticker disambiguation
- $AAPL (Apple) vs apple-the-company-in-other-context vs apple-the-fruit
- FB → Meta; TWTR → X; symbol changes orphan historical content
- Defense: map symbols over time; use a resolved-entity database

### Duplicate content
- Same news article republished across outlets
- Counts as one signal, not many
- Defense: content deduplication (hashing, simhash, MinHash for near-duplicates)

### Temporal skew
- Articles written late at night but timestamped as "next day" in some feeds
- Regular maintenance windows leave gaps
- Validate timestamp continuity

## Reference data

### Symbol mapping
- Tickers change: FB → META, TWTR → X, delisted names get recycled
- CUSIP / ISIN / RIC / Bloomberg IDs are more stable than tickers
- Store mappings with effective date ranges

### Holidays and half-days
- Market holiday calendars are timezone-specific and change year to year
- Half-days (early close before a holiday) are often missed
- Use an authoritative calendar (Pandas `pandas_market_calendars`, or vendor-provided)

### Time zone handling
- US equities: America/New_York
- Futures: exchange-specific
- Crypto: 24/7 UTC
- Store everything in UTC internally, convert only at display / calendar boundaries

### Fiscal year vs calendar year
- Companies have varying fiscal year ends
- "Q3 earnings" for one company is a different calendar quarter than for another
- Use fiscal-period-end and report-date fields explicitly

## Defensive checks (the standard toolkit)

Every data pipeline should have a set of automatic checks that run on ingest and flag anomalies. Minimum set:

### Row-level
- **Null check**: required fields present
- **Type check**: numeric fields parseable
- **Range check**: prices > 0, volumes ≥ 0, IV ∈ [0, 5], bid ≤ ask
- **Freshness check**: timestamp within expected range

### Series-level
- **Continuity check**: no unexpected gaps
- **Growth check**: day-over-day change within bounds (5σ flag)
- **Outlier check**: Hampel filter or MAD-based outlier detection
- **Coverage check**: symbol count matches expected universe

### Cross-source reconciliation
- Two independent sources for critical fields (e.g., price from broker + price from data vendor)
- Alert when they diverge beyond tolerance
- Most silent data bugs are caught this way

### Monitoring over time
- **Distribution drift**: KS test on key fields vs last week's distribution
- **Counts**: row counts by day/symbol — drops flag coverage issues
- **Latency**: data arrival vs expected schedule

## The "it worked in backtest but not live" checklist

When a backtest result doesn't reproduce in live, data quality is usually the cause. Check, in order:

1. **Point-in-time**: did backtest use data that wasn't available at decision time?
2. **Survivorship**: did backtest universe differ from live universe?
3. **Costs**: did backtest model realistic costs including spreads?
4. **Corporate actions**: are splits / dividends handled consistently?
5. **Synthetic data**: did backtest silently fill in missing data?
6. **Restatements**: did backtest use updated fundamentals instead of original vintage?
7. **Timing**: did backtest fill at prices that wouldn't have been achievable in reality?
8. **Universe filtering**: did backtest quietly exclude hard-to-trade names?

## Risk controls for live data pipelines

1. **Input validation at the boundary**: every external source is checked on ingest; bad rows are quarantined, not silently dropped
2. **Circuit breaker on anomalous data**: if ingest volume or statistics drift past threshold, halt downstream use
3. **Cross-source check**: critical fields have a second source for reconciliation
4. **Latency alarm**: data arriving later than expected pages the on-call
5. **Fail-safe default**: when data is missing, strategies either skip or halt — never substitute
6. **Audit log**: every ingest transformation is logged with a row count before/after
7. **Periodic re-check**: re-validate historical data on a schedule (vendor backfills can change history)

## References

- López de Prado — *Advances in Financial Machine Learning* (Ch. 2 on financial data structures)
- Bailey & López de Prado — "The Deflated Sharpe Ratio" (multiple testing on noisy data)
- CFA Institute — "Ethical Principles for Data Use" (practitioner guidelines)
- WRDS (Wharton Research Data Services) — documentation on PIT databases
- Compustat PIT / I/B/E/S Estimate History / CRSP — canonical PIT data providers
- Great Expectations — Python data validation framework
