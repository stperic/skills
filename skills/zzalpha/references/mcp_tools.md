# AlphaMCP Specification — 76 Tools

This document defines the complete tool inventory for AlphaMCP, the Model Context Protocol server that provides unified market data, analytics, and search to AlphaSeeker's AI trading agents.

**Consumers:** Strategist Daily Mode, Strategist Live/Tactical Mode, Financial Analyst Agent.

**Design principle:** AlphaMCP is the single data gateway. It proxies Polygon, Alpha Vantage, FRED, tastytrade, Schwab, and SearXNG behind one MCP interface and computes derived analytics (IV rank, skew, IV/RV spread, etc.) server-side. Agents never call upstream providers directly.

**Switchable quote provider:** The `QUOTE_PROVIDER` env var (`schwab` default, `polygon`, `tastytrade`) controls which upstream is used for `bars`, `quotes`, `options_chain`, `options_expirations`, and composite tools (`symbol_context`, `market_context`). `bars` uses Schwab (default) or Polygon (tastytrade has no bars). `market_metrics` always uses tastytrade (unique data). Switching requires only changing `QUOTE_PROVIDER` and restarting — no code changes.

---

## Prerequisites

AlphaMCP is an **stdio MCP server** — the host spawns it as a subprocess and communicates over stdin/stdout. It requires the **AlphaDB gateway** running on port 8080. All provider API keys are server-side; agents never need them directly.

73 tools are registered on connect. Call `track_status` to verify gateway connectivity. Call `list_accounts` to verify OMS connectivity. See the [AlphaDB HTTP API](http_api.md) for the full HTTP API reference.

### Authentication

The gateway supports **tiered API keys**. When keys are configured on the gateway, the MCP server must provide them via env vars in the MCP client config:

| Env Var | Purpose | Required When |
|---------|---------|---------------|
| `ALPHA_API_KEY` | General access — all `/v1/*` endpoints, including paper/sim trading | `ALPHA_API_KEY` is set on the gateway |
| `ALPHA_OMS_KEY` | Schwab live trading only — `place_order` and `place_spread` on schwab accounts | `ALPHA_OMS_KEY` is set on the gateway and schwab live trading is needed |

The MCP server sends `X-API-Key` on every HTTP request to the gateway. Trading tools use `ALPHA_OMS_KEY` if set (falls back to `ALPHA_API_KEY`). Paper and sim trading (paper, schwab-sim, alpaca accounts) only require `ALPHA_API_KEY`. The OMS key check is per-account: only accounts with `provider=schwab` require the elevated key.

If no keys are configured on the gateway, these env vars can be omitted.

---

## Global Response Contract

Most analytics and composite tools return a `data`-wrapped envelope:

```json
{
  "data": { "...tool-specific fields..." },
  "as_of": "2026-02-14T14:30:00Z"
}
```

Market data tools (`bars`, `quotes`, `options_chain`, `options_expirations`) return **flat responses** — fields at the top level, no `data` wrapper.

**Batch mode (global convention):** All single-symbol analytics tools accept comma-separated `symbols` for batch mode (up to 50). A single symbol returns the normal flat response shape; multiple symbols return `{"data": {"SYM1": {...}, "SYM2": {...}}}` with per-symbol results (failed symbols get `{"error": "..."}` entries instead of their data object). Batch-enabled tools: `iv_metrics`, `iv_term_structure`, `iv_rv_spread`, `skew`, `earnings_implied_move`, `sentiment`, `volatility`, `beta`. `iv_metrics` batch is optimized with a single DB query across all requested symbols. See individual tool specs for exact response shapes.

On error:

```json
{
  "error": "Human-readable error message"
}
```

**Truncation:** `bars` and `options_chain` always include a `"truncated"` boolean. When `true`, the result was capped and your request should be narrowed:

| Tool | Cap | Action |
|------|-----|--------|
| `bars` | 5000 rows | Increase `limit` or narrow `start`/`end` date range |
| `options_chain` | 200 contracts | Add `expiry` + `min_strike`/`max_strike` to narrow to ATM range |

**Data freshness metadata:** Analytics and composite tools include metadata so agents can reason about data recency:

| Field | Present In | Meaning |
|-------|-----------|---------|
| `as_of` | All analytics responses | Actual data timestamp (e.g., date of the ATM IV record), NOT query time |
| `market_closed` | All analytics responses | `true` when US equity markets are closed |
| `freshness` | Composite tools only | Breakdown of which fields are `live`, `daily`, or `synced` (see §5) |

The server does NOT compute staleness — agents compare `as_of` vs current time to determine if data is stale.

**Partial data handling:** When some data sources are unavailable (e.g., symbol not tracked for IV metrics), return partial data with `null` fields and an explicit `_missing` array explaining what's missing and why:

```json
{
  "data": {
    "symbol": "SMCI",
    "price": 45.00,
    "iv_rank": null,
    "beta": null,
    "sentiment_score": 0.35
  },
  "_missing": [
    "iv_rank: symbol not tracked, call track_symbols to start tracking",
    "beta: requires 252 days of history, symbol has 45 days"
  ],
  "as_of": "2026-02-14T14:30:00Z"
}
```

Never skip fields silently. Always return the field as `null` so the agent's parsing logic is consistent. The `_missing` array tells the agent why and what to do about it.

**Symbols:** Tools use either `symbol` (singular) or `symbols` (plural) — the parameter name indicates the response shape. Single-symbol tools (`symbol`) return a flat entity. Multi-symbol tools (`symbols`) return a keyed map `{"data": {"AAPL": {...}, "MSFT": {...}}}`. Analytics tools (§4.1–4.9) accept **both** — `symbol=AAPL` returns a flat entity, `symbols=AAPL,MSFT` returns a keyed map. All symbols are normalized to uppercase automatically.

**Dates:** Accept `YYYY-MM-DD` strings. Default to sensible ranges when omitted (bars: 30 trading days, FRED: 90 days, earnings: 90 days forward).

**Point-in-time replay (`as_of`):** 14 tools support an optional `as_of` parameter (RFC3339 or `YYYY-MM-DD`) for historical replay. Most tools source data from stored snapshots at or before the specified time. The sentiment detail endpoint fetches historical articles from Alpha Vantage on demand. Requires tracked symbols with synced data. Tools supporting `as_of`: `quotes`, `options_chain`, `options_expirations`, `iv_metrics`, `iv_rv_spread`, `skew`, `sentiment`, `volatility`, `beta`, `earnings`, `dividends`, `scan_symbols`, `symbol_context`, `market_context`. Composite tools (`symbol_context`, `market_context`) thread `as_of` into all sub-queries automatically. Sections that cannot be served historically (e.g., liquidity in `scan_symbols`) gracefully degrade to null. In as of mode, `as_of` is **required** on all requests and must not exceed the boundary date.

**Data routing:** Data source is determined automatically. Without `as_of`, data comes live from the upstream provider (Schwab, Alpha Vantage, tastytrade, FRED). With `as_of`, data comes from the local database. In as-of replay mode, `as_of` is required on all date-aware endpoints — all data comes from the database.

**Size limits:** Cap large responses (bars: 5000 rows max, default 100; options_chain: 200 contracts). Set `"truncated": true` when capped. Always pass `limit` explicitly for bars — the default (100) may not match your analysis horizon.

---

## 1. Market Data — 7 tools

Data source: Polygon.io (default), tastytrade REST, or Schwab REST (switchable via `QUOTE_PROVIDER`). `bars` uses Polygon (default) or Schwab. `quotes`, `options_chain`, and `options_expirations` switch between all three providers. Schwab provides proper float64 values and full options Greeks (delta, gamma, theta, vega, rho) in chain responses.

**HTTP endpoints:** Market data tools call provider-agnostic paths (`/v1/bars`, `/v1/quotes`, `/v1/options/...`, `/v1/market/...`) that normalize responses across providers. Do **not** use Polygon proxy paths (`/v1/proxy/polygon/...`) for these — those paths are Polygon-specific and break when `QUOTE_PROVIDER` changes.

### 1.1 `bars`

Historical OHLCV price bars. **REST:** `GET /v1/bars?symbol=SPY&timeframe=1d&start=2026-01-01`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbol` | string | yes | — |
| `timeframe` | string | yes | `1m`, `5m`, `15m`, `1h`, `1d` |
| `start` | string | no | 30 trading days ago |
| `end` | string | no | now |
| `limit` | number | no | 100 (max 50000 for tracked symbols from DB, 5000 for untracked via proxy) |

**Returns:**

```json
{
  "symbol": "SPY",
  "bars": [
    {"t": "2026-02-13T00:00:00Z", "o": 518.50, "h": 521.20, "l": 517.80, "c": 520.00, "v": 85000000, "vw": 519.40, "n": 750000},
    {"t": "2026-02-14T00:00:00Z", "o": 520.00, "h": 522.50, "l": 518.00, "c": 519.50, "v": 72000000, "vw": 520.10, "n": 680000}
  ],
  "count": 2,
  "truncated": false,
  "provider": "polygon"
}
```

Bar fields use compact names: `t` (RFC3339 timestamp), `o` (open), `h` (high), `l` (low), `c` (close), `v` (volume). Optional: `vw` (VWAP, present when available), `n` (trade count, present when non-zero). Index symbols (`$VIX`, `$SPX`, etc.) return OHLC only — no `v`, `vw`, or `n`. `truncated: true` means more bars exist — increase `limit` or narrow the date range. `provider` reflects the active `QUOTE_PROVIDER`. When served from DB (via `as_of`), includes `"timeframe"` field.

### 1.2 `quotes`

Latest price snapshot for one or more symbols. **REST:** `GET /v1/quotes?symbols=SPY,AAPL`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | comma-separated, up to 50 |
| `as_of` | string | no | — Point-in-time timestamp (RFC3339 or YYYY-MM-DD). Returns historical snapshot from local DB. Requires tracked symbols. |

**Returns:**

```json
{
  "data": {
    "SPY": {"price": 520.00, "change": -2.30, "change_pct": -0.44, "volume": 72000000, "prev_close": 522.30, "high": 522.50, "low": 518.00},
    "TSLA": {"price": 250.00, "change": -3.00, "change_pct": -1.19, "volume": 45000000, "prev_close": 253.00, "high": 254.00, "low": 248.50}
  },
  "as_of": "2026-02-14T14:30:00Z",
  "provider": "polygon"
}
```

When `as_of` is set, returns the latest bar snapshot at or before the specified time from the local DB. Only works for tracked symbols with stored bar data.

### 1.3 `options_chain`

Options contracts with Greeks, bid/ask, volume, and open interest. **REST:** `GET /v1/options/chain?underlying=SPY&expiry=2026-03-20&type=put&min_strike=510&max_strike=530`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `underlying` | string | yes | — |
| `expiry` | string | no | nearest monthly — **always specify to avoid cross-expiry bloat** |
| `type` | string | no | `all` |
| `min_strike` | number | no | — set to ~price × 0.85 |
| `max_strike` | number | no | — set to ~price × 1.15 |
| `as_of` | string | no | — Point-in-time snapshot from local DB. Returns latest option data at or before timestamp. Requires tracked underlying. |

**Returns:**

```json
{
  "underlying": "SPY",
  "contracts": [
    {"strike": 520.0, "type": "put", "expiry": "2026-03-20", "bid": 5.20, "ask": 5.40, "last": 5.30, "volume": 12500, "oi": 85000, "delta": -0.45, "gamma": 0.012, "theta": -0.08, "vega": 0.15, "iv": 0.18}
  ],
  "count": 150,
  "truncated": false,
  "provider": "polygon"
}
```

Each contract contains up to 13 fields: `strike`, `expiry`, `type`, `bid`, `ask`, `last`, `volume`, `oi`, `iv`, `delta`, `gamma`, `theta`, `vega`. Raw Polygon nested structures are stripped. Greeks are null for tastytrade (not available via REST). **Historical (`as_of`) responses omit fields with no data** — `bid`, `ask`, and `oi` are absent for bars sourced from Polygon flat files (OHLCV only); Greeks fields are absent when the IV solver could not converge. Live responses always include all 13 fields. Hard cap: **200 contracts**. `truncated: true` signals the cap was hit — add `expiry` and narrow `min_strike`/`max_strike`. Without filters, SPY and AAPL typically return 200 contracts and truncate.

### 1.4 `options_expirations`

Available expiration dates for an underlying. **REST:** `GET /v1/options/expirations?underlying=SPY`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `underlying` | string | yes | — |
| `as_of` | string | no | — Point-in-time date (RFC3339 or YYYY-MM-DD). Returns historical expirations from the local DB. Requires tracked underlying with synced options data. |

**Returns:**

```json
{
  "underlying": "SPY",
  "expirations": ["2026-02-21", "2026-02-28", "2026-03-20", "2026-06-19"],
  "count": 4,
  "provider": "polygon"
}
```

When `as_of` is provided, returns expirations that were available at the specified point in time (sourced from stored options data). When called without `as_of`, fetches live from the active quote provider.

### 1.5 `market_metrics`

tastytrade-only market metrics: IV rank, IV percentile, liquidity rating, implied volatility, earnings date, and dividend info. Always uses tastytrade regardless of `QUOTE_PROVIDER`. **REST:** `GET /v1/market/metrics?symbols=AAPL,SPY` (normalized) or `GET /v1/proxy/tastytrade/market-metrics?symbols=AAPL,SPY` (raw, string-typed numerics).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | comma-separated, up to 100 |

**Returns:**

```json
{
  "data": {
    "AAPL": {
      "iv": 0.3266,
      "iv_rank": 0.2641,
      "iv_percentile": 0.7201,
      "liquidity": 4,
      "liquidity_value": 85689.99,
      "earnings_date": "2026-01-29",
      "earnings_timing": "AMC",
      "ex_dividend_date": "2026-02-07",
      "dividend_amount": 0.25
    },
    "SPY": {
      "iv": 0.2249,
      "iv_rank": 0.3328,
      "iv_percentile": 0.7258,
      "liquidity": 4,
      "liquidity_value": 120000.00
    }
  },
  "as_of": "2026-02-14T14:30:00Z"
}
```

Fields: `iv` (30-day implied volatility index), `iv_rank` (0-1 scale, where current IV sits in 52-week range), `iv_percentile` (0-1 scale, % of days IV was lower), `liquidity` (1-5 rating), `liquidity_value` (dollar liquidity). `earnings_date`, `earnings_timing`, `ex_dividend_date`, `dividend_amount` are included when available.

**Note:** This is independent of AlphaDB's computed IV rank/percentile (`iv_metrics` tool). tastytrade computes these from their own data and methodology — useful as an independent validation source.

### 1.6 `market_hours`

Market hours, session schedules, and trading day status. **REST:** `GET /v1/market/hours?markets=equity&date=2026-02-20`

Supports any date (past or future). Recent/future dates use Schwab API (full session hours). Historical dates fall back to the market calendar DB. The `source` field indicates which was used (`schwab` or `calendar`).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `markets` | string | no | `equity` — one of: `equity`, `option`, `bond`, `future`, `forex` |
| `date` | string | no | Today (Eastern) — `YYYY-MM-DD`, any date |

**Returns:**

Always includes `is_trading_day`, `is_early_close`, `date`, and `source` at the top level. Schwab-sourced responses include full session hours in the `schwab` field.

```json
{
  "date": "2026-02-20",
  "is_trading_day": true,
  "is_early_close": false,
  "source": "schwab",
  "schwab": {
    "equity": {
      "EQ": {
        "date": "2026-02-20",
        "marketType": "EQUITY",
        "isOpen": true,
        "sessionHours": {
          "preMarket": [{"start": "2026-02-20T07:00:00-05:00", "end": "2026-02-20T09:30:00-05:00"}],
          "regularMarket": [{"start": "2026-02-20T09:30:00-05:00", "end": "2026-02-20T16:00:00-05:00"}],
          "postMarket": [{"start": "2026-02-20T16:00:00-05:00", "end": "2026-02-20T20:00:00-05:00"}]
        }
      }
    }
  }
}
```

Calendar-sourced responses (historical dates) include a synthetic `sessionHours` block on trading days, matching the Schwab response shape:

```json
{
  "date": "2025-11-28",
  "is_trading_day": true,
  "is_early_close": true,
  "early_close_time": "13:00",
  "source": "calendar",
  "sessionHours": {
    "preMarket": [{"start": "2025-11-28T07:00:00-05:00", "end": "2025-11-28T09:30:00-05:00"}],
    "regularMarket": [{"start": "2025-11-28T09:30:00-05:00", "end": "2025-11-28T13:00:00-05:00"}]
  }
}
```

`preMarket` (7:00–9:30 ET) is always present on trading days. `postMarket` (16:00–20:00 ET) is present on normal days but omitted on early close days. `early_close_time` is in ET 24h format. Holidays return `is_trading_day: false` with no `sessionHours`. Use `is_trading_day` to determine if market data exists for a date.

### 1.6b `market_hours_range`

Bulk trading day lookup for a date range. **REST:** `GET /v1/market/hours/range?start=2025-01-01&end=2025-12-31`

Returns one entry per date with the same shape as `market_hours` calendar responses. Replaces client-side day-by-day loops (one call instead of 365 sequential round-trips).

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `start` | string | yes | Start date `YYYY-MM-DD` |
| `end` | string | yes | End date `YYYY-MM-DD` (must be >= start, max 730 days) |

**Returns:** `{start, end, count, source, days: [{date, is_trading_day, is_early_close, sessionHours?, ...}]}`

### 1.7 `list_indices`

Returns all supported market index symbols with canonical names, provider aliases, and data availability notes. **REST:** `GET /v1/market/indices`

No parameters.

**Returns:**

A list of supported index symbols with their canonical names, Polygon/Schwab symbol mappings, and notes on data availability.

**Supported indices** (use `$` prefix in `bars`, `quotes`, and analytics tools):

| Symbol | Name | Daily History | Minute History |
|--------|------|---------------|----------------|
| `$SPX` | S&P 500 | 2006+ | 2023+ |
| `$NDX` | NASDAQ 100 | 2006+ | 2023+ |
| `$DJI` | Dow Jones Industrial | 2006+ | 2023+ |
| `$RUT` | Russell 2000 | 2006+ | 2026+ |
| `$MRUT` | Mini Russell 2000 | 2021+ | 2023+ |
| `$SOX` | Philadelphia Semiconductor | 2006+ | 2023+ |
| `$COMP` | NASDAQ Composite | 2006+ | 2026+ |
| `$VIX` | CBOE Volatility Index | 2006+ | 2023+ |
| `$VIX3M` | CBOE 3-Month VIX | 2007+ | 2023+ |
| `$VIX9D` | CBOE 9-Day VIX | 2013+ | 2023+ |
| `$VVIX` | VIX of VIX | 2012+ | 2023+ |
| `$VXN` | NASDAQ Volatility | 2006+ | 2023+ |
| `$RVX` | Russell 2000 Volatility | 2006+ | 2023+ |
| `$GVZ` | Gold Volatility | 2008+ | 2023+ |
| `$OVX` | Crude Oil Volatility | 2008+ | 2023+ |
| `$SKEW` | CBOE Skew | 2011+ | daily only |
| `$PUT` | CBOE Put/Call Ratio | 2007+ | 2023+ |
| `$MOVE` | ICE BofA MOVE Index (bond volatility) | 2021+ | 2026+ |
| `$IRX` | 13-Week T-Bill Yield | 2006+ | 2023+ |
| `$FVX` | 5-Year Treasury Yield | 2006+ | 2023+ |
| `$TNX` | 10-Year Treasury Yield | 2006+ | 2023+ |
| `$TYX` | 30-Year Treasury Yield | 2006+ | 2023+ |

`$MOVE` is the VIX equivalent for bonds — measures Treasury rate volatility. Not available on FRED; use `bars` or `quotes` with `$MOVE`, not `fred_series`.

---

## 2. Fundamentals (Alpha Vantage) — 6 tools

Data source: Alpha Vantage Premium (75 req/min).

### 2.1 `earnings`

Next earnings date with `days_until`/`timing`, plus last 4 quarters of EPS data. **REST:** `GET /v1/fundamentals/earnings?symbols=AAPL,MSFT,GOOGL`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (1 to 50). |
| `as_of` | string | no | — Filters to quarters reported on or before this date. `days_until` computed relative to `as_of` instead of now. |

**Response** (always per-symbol map, whether 1 or many symbols):

```json
{
  "data": {
    "AAPL": {"next_date": "2026-04-29", "days_until": 73, "timing": "AMC", "quarters": [...], "source": "db"},
    "MSFT": {"next_date": "2026-04-22", "days_until": 66, "quarters": [...], "source": "db"},
    "SMCI": {"error": "not tracked"}
  }
}
```

DB first for tracked symbols (2 batch queries via LATERAL join — safe for 500+ symbols). Alpha Vantage fallback for untracked symbols (individual calls). Unavailable symbols return `{"error": "not tracked"}` or `{"error": "not available"}`.

`next_date`, `days_until`, and `timing` are sourced from AlphaDB's earnings calendar (DB). They are `null` when no upcoming earnings date is available or when the value is unknown (empty strings are normalized to `null`). `timing` values: `AMC` (after market close), `BMO` (before market open), `TNS` (time not supplied), or `null`. Returns the last 4 reported quarters (most recent first). `next_date`/`days_until`/`timing` are `null` when no upcoming date is on record.

### 2.2 `dividends`

Next ex-dividend date, days until, amount, yield, and payment frequency. **REST:** `GET /v1/fundamentals/dividends?symbols=AAPL,MSFT,JNJ`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (1 to 50). |
| `as_of` | string | no | — Filters to dividends with ex-date on or before this date. Uses historical price for yield calculation. |

**Response** (always per-symbol map, whether 1 or many symbols):

```json
{
  "data": {
    "AAPL": {"symbol": "AAPL", "next_ex_date": "2026-02-20", "days_until": 6, "amount": 0.25, "yield_pct": 0.48, "frequency": "quarterly", "source": "db"},
    "MSFT": {"symbol": "MSFT", "next_ex_date": "2026-03-12", "days_until": 26, "amount": 0.83, "yield_pct": 0.78, "frequency": "quarterly", "source": "db"},
    "SMCI": {"error": "not tracked"}
  }
}
```

DB first for tracked symbols (2 batch queries — safe for 500+ symbols). Alpha Vantage fallback for untracked symbols (individual calls). Unavailable symbols return `{"error": "not tracked"}` or `{"error": "not available"}`.

`yield_pct` = annualized dividend / current price (Polygon snapshot). `frequency` detected from ex-date intervals: `monthly`, `quarterly`, `semi-annual`, `annual`, `irregular`. All fields present in response (`null` when unavailable — no future ex-date, no price data, etc.).

### 2.3 `company_overview`

Company profile, sector, market cap, and financial ratios. **DB-first** for tracked symbols (nightly sync from Alpha Vantage, refreshed every 30 days). AV fallback for untracked symbols. Accepts comma-separated symbols for batch mode (up to 50). **REST:** `GET /v1/fundamentals/company?symbols=AAPL`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |

**Returns:**

```json
{
  "data": {
    "AAPL": {
      "symbol": "AAPL",
      "name": "Apple Inc.",
      "sector": "Technology",
      "industry": "Consumer Electronics",
      "market_cap": 3200000000000,
      "pe_ratio": 32.5,
      "eps": 6.73,
      "revenue": 394000000000,
      "revenue_growth_pct": 0.05,
      "description": "Apple Inc. designs, manufactures, and markets smartphones...",
      "source": "db"
    }
  }
}
```

Returns 10 fields per symbol. `source` is `"db"` for tracked symbols (nightly sync) or `"alphavantage"` for untracked symbols (live fallback). Batch queries against the DB are safe for 500+ symbols.

### 2.4 `insider_transactions`

Recent insider buys and sells. Accepts comma-separated symbols for batch (up to 10). **REST:** `GET /v1/fundamentals/insiders?symbol=AAPL&days=90`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — |
| `days` | number | no | 90 |

**Returns:**

```json
{
  "data": [
    {"date": "2026-02-10", "name": "Tim Cook", "title": "CEO", "type": "SELL", "shares": 50000, "value": 11250000}
  ],
  "symbol": "AAPL",
  "as_of": "2026-02-14T14:30:00Z"
}
```

`type`: `BUY` (AV `A`) or `SELL` (AV `D`). `value` = `shares × share_price`. Filtered to `days` lookback window.

### 2.5 `earnings_transcript`

Earnings call transcript text. Accepts comma-separated symbols for batch (up to 10). **REST:** `GET /v1/fundamentals/transcript?symbol=AAPL&quarter=2026Q1`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — |
| `quarter` | string | no | latest (e.g. `2026Q1`) |

**Returns:**

```json
{
  "data": {
    "quarter": "2026Q1",
    "date": "2026-01-30",
    "transcript_text": "Good afternoon. Welcome to Apple's fiscal first quarter...",
    "truncated": true
  },
  "symbol": "AAPL",
  "as_of": "2026-02-14T14:30:00Z"
}
```

Transcript capped at ~4000 chars (`truncated: true` when hit). `date` resolved from AV EARNINGS `reportedDate`. Returns `null` for `data` when no matching quarter found.

### 2.6 `alpha_vantage_query`

Raw pass-through to the full Alpha Vantage API. Use for any AV function not covered by dedicated tools. See [Alpha Vantage documentation](https://www.alphavantage.co/documentation/) for all 100+ available functions. API key injected server-side. Rate limited at 75 req/min (shared pool with all AV calls). **REST:** `GET /v1/alphavantage/query?function={FUNCTION}&...`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `function` | string | yes | AV function name (e.g. `INCOME_STATEMENT`, `BALANCE_SHEET`, `CASH_FLOW`, `HISTORICAL_OPTIONS`, `REAL_GDP`, `CPI`, `TREASURY_YIELD`) |
| `params` | object | no | Additional query parameters as key-value pairs (e.g. `{"symbol": "AAPL"}`, `{"maturity": "10year"}`). Do NOT include `apikey`. |

**Example calls:**
- Income statement: `{"function": "INCOME_STATEMENT", "params": {"symbol": "AAPL"}}`
- GDP: `{"function": "REAL_GDP"}`
- Treasury yield: `{"function": "TREASURY_YIELD", "params": {"maturity": "10year"}}`
- Historical options: `{"function": "HISTORICAL_OPTIONS", "params": {"symbol": "AAPL", "date": "2025-06-15"}}`

Returns raw Alpha Vantage JSON. Prefer dedicated tools (`earnings`, `dividends`, `company_overview`, etc.) when available — they add batch support, `as_of`, DB caching, and normalized responses.

---

## 3. Indicators & Macro (Alpha Vantage + FRED) — 2 tools

### 3.1 `technical_indicator`

Pre-computed technical indicators from Alpha Vantage. **REST:** `GET /v1/fundamentals/indicator?symbol=SPY&indicator=SMA`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbol` | string | yes | — |
| `indicator` | string | yes | `SMA`, `EMA`, `RSI`, `MACD`, `ATR`, `BBANDS`, `STOCH`, `ADX`, `CCI`, `AROON`, `OBV`, `VWAP` |
| `interval` | string | no | `daily` (`1min`, `5min`, `15min`, `30min`, `60min`, `daily`, `weekly`, `monthly`) |
| `time_period` | number | no | standard per indicator (SMA/EMA: 20, RSI: 14, ATR: 14, BBANDS: 20) |
| `series_type` | string | no | `close` (`open`, `high`, `low`, `close`) |
| `limit` | number | no | 30 (most recent data points) |

**Returns:**

```json
{
  "symbol": "SPY",
  "indicator": "SMA",
  "data": [
    {"date": "2026-02-14", "sma": "518.45"},
    {"date": "2026-02-13", "sma": "517.90"},
    {"date": "2026-02-12", "sma": "516.20"}
  ],
  "count": 3
}
```

Flattened to `{date, ...values}` array sorted most-recent-first. Field names lowercased (`sma`, `rsi`, `macd_signal`, `macd_hist`, etc.).

**⚠ Values are strings, not numbers.** Alpha Vantage encodes all indicator values as JSON strings (e.g., `"sma": "518.45"`). Parse to float before arithmetic: `if rsi < 30` must be `if parseFloat(data[0].rsi) < 30`.

When `time_period` is omitted, sensible defaults are applied: SMA/EMA/BBANDS: 20, RSI/ATR/ADX/CCI: 14, STOCH: 5, AROON: 25. Indicators that don't use `time_period` (OBV, VWAP, MACD) ignore the parameter. Alpha Vantage error responses (`{"Error Message": "..."}`) are caught and returned as spec error envelopes (`{"error": "..."}`).

Multi-value indicators (MACD, BBANDS, STOCH, AROON) include all sub-fields per point:
```json
{"date": "2026-02-14", "macd": "2.15", "macd_signal": "1.80", "macd_hist": "0.35"}
```

### 3.2 `fred_series`

Economic time series served from the AlphaDB local `fred_1d` database (pre-synced from FRED). **REST:** `GET /v1/fred?series_id=DGS10&start=2026-01-01&end=2026-02-14`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `series_id` | string | yes | e.g. `DGS10`, `DGS2`, `DFF`, `T10Y2Y`, `CPIAUCSL`, `UNRATE`, `VIXCLS` |
| `start` | string | no | 90 days ago |
| `end` | string | no | now |

**Common series IDs:**

| Series | Description |
|--------|-------------|
| `VIXCLS` | CBOE Volatility Index |
| `DGS10` | 10-Year Treasury Yield |
| `DGS2` | 2-Year Treasury Yield |
| `DFF` | Federal Funds Effective Rate |
| `T10Y2Y` | 10Y-2Y Treasury Spread |
| `CPIAUCSL` | Consumer Price Index |
| `UNRATE` | Unemployment Rate |
| `BAMLH0A0HYM2` | High Yield Spread |

**Returns:**

```json
{
  "series_id": "DGS10",
  "data": [
    {"date": "2026-02-13", "value": "4.35"},
    {"date": "2026-02-14", "value": "4.32"}
  ],
  "count": 2,
  "as_of": "2026-02-14"
}
```

Each observation: `date` and `value` only. Missing FRED data points (`"."`) excluded from results. `as_of` is the date of the most recent observation in the response.

**Latest-value fallback:** Some FRED series are published weekly or monthly (e.g., `CPIAUCSL`, `UNRATE`). If the requested date range contains no observations, the response automatically returns the most recent available value with its actual date in `as_of`. This prevents 404s when using short lookbacks like `days=1`.

**⚠ Values are strings, not numbers.** FRED returns all values as JSON strings (e.g., `"value": "4.35"`). Parse to float before arithmetic.

---

## 4. Analytics (AlphaDB-Computed) — 14 tools

These are insights AlphaDB computes from its own database. Most require the symbol to be tracked (see `track_symbols`). Proxy tools (bars, quotes, earnings, etc.) work for any symbol without analytics enabled.

**HTTP endpoints:** Tools 4.1–4.9 dispatch through `GET /v1/alpha/analytics?metric={name}`. The analytics dispatcher accepts both `symbol` (single, flat response) and `symbols` (comma-separated up to 50, keyed map response `{"data": {"SYM1": {...}, "SYM2": {...}}}`). `symbols=AAPL` (single value via plural param) is normalized to a flat response. IV metrics batch uses a single optimized DB query. Exceptions: `portfolio_greeks` → `POST /v1/portfolio/greeks`, `scan_symbols` → `GET /v1/alpha/scan`, `earnings_calendar` → `GET /v1/alpha/analytics?metric=earnings_calendar`.

**Live vs DB:** During market hours, current-value fields (e.g. `current_iv`, `realized_vol`, volume) are replaced with live Polygon data; historical context (52-week IV history, baselines) stays DB-sourced. After close, pure DB reads. On live fetch failure, silently falls back to DB. All responses include `market_closed: bool` and `as_of` (data timestamp, not query time).

### 4.1 `iv_metrics`

IV rank, IV percentile, current IV from 52-week options history. Single-point mode (default or `as_of`) returns a snapshot. Time-series mode (`start`/`end`) returns daily IV rank/percentile history. Accepts comma-separated symbols for batch mode (up to 50).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `as_of` | string | no | — Point-in-time IV rank/percentile computed against 52-week history ending at `as_of`. |
| `start` | string | no | — Start date for time-series mode (YYYY-MM-DD). Returns daily IV history. |
| `end` | string | no | — End date for time-series mode (YYYY-MM-DD). Defaults to today. |

**Returns:**

```json
{
  "data": {
    "iv_rank": 62,
    "iv_percentile": 68,
    "current_iv": 0.55,
    "iv_52w_high": 0.85,
    "iv_52w_low": 0.30,
    "as_of": "2026-02-13"
  },
  "symbol": "TSLA",
  "market_closed": false,
  "as_of": "2026-02-14T14:30:00Z"
}
```

Requires 52 weeks of IV history for accurate rank/percentile. The `data.as_of` field reflects the date of the ATM IV record used.

**During market hours:** `current_iv` is replaced with a live ATM IV from Polygon options snapshot. `iv_rank` and `iv_percentile` are recomputed against the same 52-week DB history using the live IV value. `as_of` reflects current time.

### 4.2 `iv_term_structure`

IV across expiration dates — contango, backwardation, or flat. Accepts comma-separated symbols for batch mode (up to 50).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `min_dte` | number | no | — |
| `max_dte` | number | no | — |
| `as_of` | string | no | — Point-in-time date (YYYY-MM-DD). Returns historical term structure snapshot. |

**Returns:**

```json
{
  "data": {
    "structure": "contango",
    "near_term_iv": 0.22,
    "near_term_dte": 7,
    "far_term_iv": 0.18,
    "far_term_dte": 125,
    "expirations": [
      {"expiry": "2026-02-21", "dte": 7, "iv": 0.22},
      {"expiry": "2026-03-20", "dte": 34, "iv": 0.19},
      {"expiry": "2026-06-19", "dte": 125, "iv": 0.18}
    ],
    "data_quality": {
      "price_source": "close",
      "iv_source": "computed",
      "note": "Greeks computed from daily close prices via Black-Scholes; bid/ask not available"
    }
  },
  "symbol": "SPY",
  "as_of": "2026-02-14T14:30:00Z"
}
```

Structure: `contango` (front > back, normal), `backwardation` (front < back, fear), or `flat`. `near_term_iv`/`far_term_iv` are convenience fields for the nearest and furthest expirations.

`data_quality` is present when fallback data sources were used (see [Data Quality](#data-quality) below). When live quote data with provider-supplied Greeks is available, `data_quality` is omitted.

**Historical mode (`as_of`):** Returns the term structure as it existed on the given date, using stored options data. DTE is computed relative to `as_of`, not today. Useful for replaying strategy decisions or comparing term structure evolution over time.

**During market hours:** Options chains are fetched live from Polygon snapshots instead of DB. IV values reflect real-time bid/ask midpoints and provider-supplied Greeks.

### 4.3 `iv_rv_spread`

Implied vs realized volatility spread. Identifies options mispricing. Accepts comma-separated symbols for batch mode (up to 50).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `as_of` | string | no | — Historical IV/RV spread at the specified point in time. |

Realized volatility window is fixed at 20 trading days.

**Computed fields:**

| Field | Formula | Significance |
|-------|---------|-------------|
| `current_iv` | ATM IV from options chain | What the market expects |
| `realized_vol` | StdDev(log returns) x sqrt(252) over rv_window | What actually happened |
| `iv_rv_spread` | current_iv - realized_vol | Positive = options overpriced (edge for selling) |
| `iv_rv_ratio` | current_iv / realized_vol | >1.2 = meaningful edge |
| `iv_rv_percentile` | Where current spread sits in 52-week history | Is this spread unusual? |

**Guardrails:**
- If realized_vol is computed from fewer than 15 days of data, flag as unreliable in `_missing`.
- `iv_rv_percentile` requires 52 weeks of spread history. Returns `null` with `_missing` note when history is insufficient.

**During market hours:** `current_iv` is replaced with a live ATM IV from Polygon. `realized_vol` remains from DB (HV is close-to-close, not intraday — standard practice). `iv_rv_spread` and `iv_rv_ratio` are recomputed using the live IV.

**Returns:**

```json
{
  "data": {
    "current_iv": 0.18,
    "realized_vol": 0.12,
    "iv_rv_spread": 0.06,
    "iv_rv_ratio": 1.50,
    "iv_rv_percentile": 75,
    "rv_window": 20
  },
  "symbol": "SPY",
  "as_of": "2026-02-14T14:30:00Z"
}
```

When `iv_rv_percentile` is unavailable:

```json
{
  "data": {
    "current_iv": 0.18,
    "realized_vol": 0.12,
    "iv_rv_spread": 0.06,
    "iv_rv_ratio": 1.50,
    "iv_rv_percentile": null,
    "rv_window": 20,
    "_missing": ["iv_rv_percentile: requires 52-week spread history"]
  },
  "symbol": "SPY",
  "as_of": "2026-02-14T14:30:00Z"
}
```

### 4.4 `skew`

Put/call volatility skew at 25-delta. Measures downside fear premium. Accepts comma-separated symbols for batch mode (up to 50).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `as_of` | string | no | — Historical skew from stored data (skips live options chain). |

**Computed fields:**

| Field | Formula | Significance |
|-------|---------|-------------|
| `put_25d_iv` | IV at 25-delta put | Downside protection cost |
| `call_25d_iv` | IV at 25-delta call | Upside speculation cost |
| `skew_25d` | put_25d_iv - call_25d_iv | Positive = put skew (fear premium) |
| `skew_percentile` | Where current skew sits in 52-week history | Is skew unusually steep/flat? |
| `atm_iv` | IV at 50-delta | Reference point |

**Guardrails:**
- Requires liquid options chain. If 25-delta strike has zero OI or bid=0, return `null` with note in `_missing`. Use 30-delta as fallback if 25-delta is illiquid.
- `put_25d_iv`, `call_25d_iv`, and `expiry` require per-strike IV interpolation from live options chains. When only close-price-derived aggregate ATM IV is available (DB source), these return `null` with `_missing` notes.
- `skew_percentile` requires 52 weeks of skew history. Returns `null` with `_missing` note when history is insufficient.

**Returns (live data with full chain):**

```json
{
  "data": {
    "put_25d_iv": 0.20,
    "call_25d_iv": 0.14,
    "skew_25d": 0.06,
    "skew_percentile": 65,
    "atm_iv": 0.17,
    "expiry": "2026-03-20"
  },
  "symbol": "SPY",
  "as_of": "2026-02-14T14:30:00Z"
}
```

**Returns (DB source — no liquid 25-delta contracts in chain):**

```json
{
  "data": {
    "skew_25d": 0.06,
    "put_25d_iv": null,
    "call_25d_iv": null,
    "atm_iv": 0.17,
    "skew_percentile": null,
    "expiry": null,
    "_missing": [
      "put_25d_iv: no liquid 25-delta options found in DB chain (add options data or wait for backfill)",
      "call_25d_iv: no liquid 25-delta options found in DB chain (add options data or wait for backfill)",
      "skew_percentile: requires 52-week skew history",
      "expiry: skew computed from aggregate ATM IV, not per-expiry chain"
    ]
  },
  "symbol": "SPY",
  "as_of": "2026-02-14T14:30:00Z"
}
```

When options history is backfilled and liquid 25-delta contracts exist in the DB chain, `put_25d_iv`, `call_25d_iv`, and `expiry` are populated. `skew_25d` is always populated (falls back to aggregate ATM IV difference when per-strike data is unavailable).

### 4.5 `earnings_implied_move`

Options market's expected move around earnings vs historical reality. Accepts comma-separated symbols for batch mode (up to 50).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `as_of` | string | no | — Point-in-time date (YYYY-MM-DD or RFC3339). When set, returns pre-computed implied/actual move from DB. |
| `history_n` | int | no | `8` — Quarters of history to include in the distribution. Clamped to `[1, 20]`. |

**Two modes:**

- **Live (no `as_of`):** Computes implied move from the current options chain for the next upcoming earnings. Requires earnings <=90 days away.
- **Historical (`as_of` set):** Returns pre-computed data from `earnings_quarterly` (implied move derived from ATM straddle on the day before earnings, actual move from close-to-close around earnings). No live options chain needed. The historical distribution is built from quarters strictly **prior** to the anchor quarter so the anchor's own realized move never leaks into its baseline.

**Computed fields:**

| Field | Formula | Significance |
|-------|---------|-------------|
| `implied_move_pct` | ATM straddle price / stock price | What options market expects |
| `historical_move_mean_pct` | Mean of \|move\| over last `history_n` quarters | Average realized reaction |
| `historical_move_median_pct` | Median (type-7 interpolation) | Robust central tendency — preferred over mean for skewed distributions |
| `historical_move_p75_pct` | 75th percentile | Tail of typical reactions |
| `historical_sample_size` | Actual usable quarters | May be less than `history_n` if backfill incomplete |
| `implied_vs_historical_ratio` | implied / historical_median | **Preferred filter.** >1.0 = market pricing more than typical → positive EV for premium sellers |
| `implied_vs_actual_ratio` | implied / historical_mean | Back-compat alias (mean-based) |
| `avg_historical_move_pct` | = historical_mean | Back-compat alias |
| `next_earnings_date` | From earnings calendar | Context |
| `days_to_earnings` | Computed | Context |

**Guardrail (live mode only):** Only compute when `days_to_earnings <= 90`. Beyond that, the straddle doesn't meaningfully reflect earnings pricing. Return `null` fields with a note if earnings are too far out.

**Returns (live mode):**

```json
{
  "data": {
    "implied_move_pct": 4.2,
    "historical_move_mean_pct": 3.5,
    "historical_move_median_pct": 3.1,
    "historical_move_p75_pct": 4.0,
    "historical_sample_size": 8,
    "implied_vs_historical_ratio": 1.35,
    "implied_vs_actual_ratio": 1.20,
    "avg_historical_move_pct": 3.5,
    "next_earnings_date": "2026-04-29",
    "days_to_earnings": 74,
    "expiry_used": "2026-04-17",
    "expiry_match": "nearest_before",
    "data_quality": {
      "price_source": "close",
      "iv_source": "computed",
      "note": "Greeks computed from daily close prices via Black-Scholes; bid/ask not available"
    }
  },
  "symbol": "AAPL",
  "as_of": "2026-02-14T14:30:00Z"
}
```

**Returns (historical mode with `as_of`):**

```json
{
  "data": {
    "earnings_date": "2024-10-31",
    "as_of": "2024-11-15",
    "implied_move_pct": 3.82,
    "actual_move_pct": 3.12,
    "straddle_price": 8.48,
    "underlying_price": 233.67,
    "historical_move_mean_pct": 3.15,
    "historical_move_median_pct": 2.95,
    "historical_move_p75_pct": 3.80,
    "historical_sample_size": 8,
    "implied_vs_historical_ratio": 1.30,
    "implied_vs_actual_ratio": 1.21,
    "avg_historical_move_pct": 3.15,
    "source": "db"
  }
}
```

Historical data covers earnings from 2014 onward (limited by `options_1d` availability). Returns the most recent earnings with pre-computed implied move on or before the `as_of` date.

**Additional response fields (live mode):**
- `expiry_used`: The actual options expiration used for the straddle calculation.
- `expiry_match`: `"exact"` if the expiry falls on or after earnings, or `"nearest_before"` if no post-earnings expiry was available and the closest prior expiry was used instead.
- `data_quality`: Metadata about the data sources used (see [Data Quality](#data-quality) below). Present when fallback data sources were used.

Returns `null` data with `_missing` note if earnings are >90 days away (live mode only).

**During market hours:** The ATM straddle is priced from a live Polygon options snapshot instead of DB. `implied_move_pct` reflects real-time bid/ask midpoints. Historical earnings moves remain from DB.

### 4.6 `sentiment`

Sentiment dashboard with two signal groups: **news** (article-derived QSE scores) and **options** (IV rank, skew, IV-HV spread, put/call IV ratio). Each group carries per-group freshness metadata. Accepts comma-separated symbols for batch mode (up to 50). **REST:** `GET /v1/alpha/analytics?metric=sentiment&symbol={symbol}`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `as_of` | string | no | — Point-in-time date (YYYY-MM-DD). Historical mode uses DB data only. |

**Returns:**

```json
{
  "data": {
    "symbol": "AAPL",
    "news": {
      "score": 0.15,
      "magnitude": 0.15,
      "direction": "bullish",
      "confidence": 0.82,
      "confidence_flag": "NORMAL",
      "article_count": 42,
      "source_diversity": 8,
      "dispersion": 0.31,
      "bullish_count": 28,
      "bearish_count": 8,
      "neutral_count": 6,
      "sma_5": 0.12,
      "sma_20": 0.08,
      "score_delta": 0.03,
      "streak": 3,
      "freshness": {
        "as_of_date": "2026-04-10",
        "stale_days": 0,
        "data_quality": "fresh",
        "source": "live"
      }
    },
    "options": {
      "atm_iv": 0.2834,
      "iv_rank": 45.5,
      "iv_percentile": 52.3,
      "iv_high_52w": 0.55,
      "iv_low_52w": 0.18,
      "skew_25d": -0.042,
      "term_slope": 0.012,
      "put_call_iv_ratio": 1.05,
      "iv_hv_spread": 0.035,
      "freshness": {
        "as_of_date": "2026-04-10",
        "stale_days": 0,
        "data_quality": "fresh",
        "source": "live"
      }
    }
  },
  "market_closed": false
}
```

**Two modes:** Without `as_of` (live mode), news is fetched live from Alpha Vantage and options from the live chain, falling back to DB when unavailable. With `as_of` (historical/replay mode), both groups come from DB only.

**Freshness tiers:** `fresh` (0-1 trading days), `stale` (2-5), `very_stale` (6-30), `none` (>30 or no data — all value fields become `null`). For ETFs (QQQ, IWM, TLT), news is often stale or absent — the options group provides the primary sentiment read.

**Partial success:** Returns 200 if at least one group has data. 404 only if both groups are empty. The `source` field indicates data origin: `"live"` (real-time provider), `"db"` (stored historical), `"db_carry_forward"` (last known value, not from the requested date).

### 4.7 `volatility`

Full volatility profile: HV10/20/30, IV30, IV rank/percentile, IV-HV spread, skew, term slope, 52-week IV high/low, and market context (price, earnings proximity). Accepts comma-separated symbols for batch mode (up to 50).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `as_of` | string | no | — Point-in-time snapshot (YYYY-MM-DD or RFC3339). |
| `start` | string | no | — Start date for time-series mode (YYYY-MM-DD). |
| `end` | string | no | — End date for time-series mode (YYYY-MM-DD). Defaults to today. |

Single-point mode (default or `as_of`) returns `vol_profile` (historical, implied, structure sections), `premium` (IV-HV spread, recommendation), and `market_context`. Time-series mode (`start`/`end`) returns daily records with IV30, HV10/20/30, rank/percentile, skew, term slope, and stock close.

### 4.8 `beta`

Stock beta relative to SPY (S&P 500). Accepts comma-separated symbols for batch mode (up to 50).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `days` | number | no | 252 (1 year). Range: 20-252. Use 20 for short-term beta (recent regime). |
| `as_of` | string | no | — End date for the regression window (YYYY-MM-DD or RFC3339). Regression runs over `[as_of - days .. as_of]`. Without this, runs through the most recent available data. |

**Returns:** `symbol`, `beta`, `r_squared`, `benchmark` (SPY), `period_days`, `data_points`.

### 4.9 `correlation`

Pairwise correlation matrix between symbols.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | comma-separated, 2-20 symbols |
| `lookback_days` | number | no | 30 |

**Returns:**

```json
{
  "data": {
    "matrix": {
      "AAPL": {"MSFT": 0.85, "TSLA": 0.42},
      "MSFT": {"AAPL": 0.85, "TSLA": 0.38},
      "TSLA": {"AAPL": 0.42, "MSFT": 0.38}
    },
    "period_days": 30
  },
  "as_of": "2026-02-14T14:30:00Z"
}
```

### 4.10 `unusual_volume`

Institutional-grade volume anomaly detection with statistical guardrails.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | comma-separated |
| `threshold` | number | no | 2.0 (RVOL minimum) |
| `min_dollar_volume` | number | no | 5000000 |

**Computed fields:**

| Field | Formula | Significance |
|-------|---------|-------------|
| `rvol` | Volume_now / AvgVol_30d | The "In Play" signal. >2.0 = institutions are active |
| `z_score` | (Volume_now - Mean_30d) / StdDev_30d | Volatility-scaled. Prevents false positives on volatile names |
| `dollar_volume` | Price x Volume | Liquidity filter. Rejects ghost tickers |
| `vwap_dist_pct` | (Price - VWAP) / VWAP | Sentiment. Positive = accumulation, negative = distribution |
| `relative_range` | (Day High - Day Low) / AvgRange_10d | Volatility expansion. High vol + high range = true breakout |
| `obv_trend` | On-Balance Volume direction | Flow direction. Is money staying in or exiting? |

**Guardrails:**
- Symbols with `dollar_volume` below `min_dollar_volume` are excluded (prevents illiquid false signals).
- Only symbols exceeding the `threshold` RVOL are returned.
- Z-score provides volatility scaling: a 2.0x RVOL in a utility stock (z=3.5) is more significant than 2.0x in a meme stock (z=0.8).

**Returns:**

```json
{
  "data": [
    {
      "symbol": "NVDA",
      "rvol": 3.2,
      "z_score": 2.8,
      "dollar_volume": 45000000000,
      "vwap_dist_pct": 1.5,
      "relative_range": 1.8,
      "obv_trend": "accumulating",
      "price": 890.00,
      "volume": 52000000,
      "avg_volume_30d": 16250000
    }
  ],
  "filters_applied": {
    "threshold": 2.0,
    "min_dollar_volume": 5000000
  },
  "as_of": "2026-02-14T14:30:00Z"
}
```

OBV trend: `accumulating`, `distributing`, or `neutral`.

**During market hours:** Today's volume is fetched from a live Polygon ticker snapshot (`Day.Volume`) instead of summing DB minute bars. The 30-day average baseline remains from DB.

### 4.11 `portfolio_greeks`

Aggregate portfolio Greeks with optional what-if scenario analysis.

**REST:** `POST /v1/portfolio/greeks`. Note: REST uses `option_symbol` (OCC format, e.g. `"O:AAPL250321C00185000"`) and `side` (`"long"`/`"short"`); MCP accepts `option_type`/`strike`/`expiry` for LLM usability.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `positions` | string | yes | JSON array: `[{"symbol": "SPY", "quantity": -2, "type": "put", "strike": 510, "expiry": "2026-03-20", "cost_basis": 3.50}]`. Stocks need only `symbol`/`type`/`quantity`. |
| `what_if` | string | no | JSON: `{"underlying_move_pct": -2.0, "iv_change": 3.0, "days_forward": 7}` |

**Returns:**

```json
{
  "data": {
    "as_of": "2026-02-14T14:30:00Z",
    "net_delta": -45.2,
    "net_gamma": 2.1,
    "net_theta": 12.5,
    "net_vega": -85.0,
    "what_if_pnl": {
      "added_position": "SPY 2026-03-20 510P x-2",
      "new_net_delta": -52.4,
      "new_net_gamma": 3.0,
      "new_net_theta": 15.2,
      "new_net_vega": -92.0,
      "delta_change": -7.2,
      "gamma_change": 0.9,
      "theta_change": 2.7,
      "vega_change": -7.0
    }
  }
}
```

`what_if_pnl` is only present when the `what_if` parameter is provided. It shows the new aggregate Greeks after adding the hypothetical position, plus the change from current values.

### 4.12 `scan_symbols`

Cross-symbol screener. Filters all tracked symbols by IV rank, days to earnings, RVOL, IV/RV ratio, and options liquidity. Returns matching symbols with their current analytics snapshot. **REST:** `GET /v1/alpha/scan`

This is the **opportunity discovery** tool. Without it, agents can only evaluate symbols they already know. Use it as the first step in any new-position workflow.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `min_iv_rank` | number | no | — |
| `max_iv_rank` | number | no | — |
| `min_days_to_earnings` | number | no | — |
| `max_days_to_earnings` | number | no | — |
| `min_days_to_ex_div` | number | no | — (exclude symbols with ex-div within N days; symbols with no dividend data pass) |
| `min_rvol` | number | no | — |
| `max_rvol` | number | no | — |
| `min_iv_rv_ratio` | number | no | — |
| `max_iv_rv_ratio` | number | no | — |
| `min_liquidity` | number | no | — (1–5 tastytrade liquidity rating; 3 = reasonable minimum for retail) |
| `sort_by` | string | no | — (`iv_rank`, `rvol`, `days_to_earnings`, `iv_rv_ratio`, `days_to_ex_div`) |
| `order` | string | no | `desc` (`asc` or `desc`) |
| `as_of` | string | no | — Historical scan at a point in time. All analytics are sourced from DB at `as_of`. Liquidity data (tastytrade) is unavailable and returns null. |

All filter params are optional. Omitting all returns the full tracked universe with IV rank snapshots. Only symbols for which the requested metric data is available pass through — if IV history is not yet built for a symbol, it won't appear in `min_iv_rank` filtered results.

**Missing-data transparency:** When a filter excludes symbols because analytics data is not yet available (not because the value failed the threshold), the response includes `excluded_no_data` — the count of symbols excluded for this reason. This lets the agent know it's operating on a partial universe.

**`min_days_to_earnings` behavior:** Symbols with no earnings data in the DB are conservatively treated as passing (we don't know earnings are imminent). `max_days_to_earnings` excludes symbols with no data (we can't confirm earnings are within the window).

**`min_days_to_ex_div` behavior:** Symbols with no dividend history (non-payers or data not yet synced) are conservatively treated as passing. Only symbols with a known upcoming ex-dividend date closer than the threshold are rejected. Example: `min_days_to_ex_div=3` hard-rejects any symbol going ex-div within 3 days while passing all non-dividend payers.

**Returns:**

```json
{
  "data": [
    {"symbol": "NVDA", "iv_rank": 81.0, "current_iv": 0.63, "days_to_earnings": 21, "earnings_timing": "AMC", "days_to_ex_div": 45, "rvol": 1.1, "iv_rv_ratio": 1.55, "iv_rv_spread": 0.11},
    {"symbol": "TSLA", "iv_rank": 74.2, "current_iv": 0.58, "days_to_earnings": 34, "earnings_timing": "AMC", "rvol": 1.2, "iv_rv_ratio": 1.42, "iv_rv_spread": 0.09}
  ],
  "count": 2,
  "scanned": 18,
  "excluded_no_data": 3,
  "excluded": ["SMCI", "GME", "MSTR"],
  "filters": {"min_iv_rank": 70, "min_days_to_earnings": 14, "sort_by": "iv_rank", "order": "desc"},
  "as_of": "2026-02-14T14:30:00Z"
}
```

`excluded_no_data` is only present when > 0. `excluded` lists the actual symbol names that were excluded for missing data (same count as `excluded_no_data`). Response fields per symbol are present only when that metric was fetched (driven by which filters are active). With no filters, only `iv_rank` and `current_iv` are included. Results are sorted descending by `sort_by` field when specified; symbols missing that field sort to the bottom.

**Liquidity filter:** When `min_liquidity` is set, AlphaDB makes one batch tastytrade `market_metrics` call for the entire symbol universe and filters by the `liquidity-rating` field (1–5 scale). This adds one upstream call per scan but keeps the universe size unchanged. Symbols with no tastytrade liquidity data are excluded when the filter is active (`excluded_no_data` is incremented). Rating guide: 1 = wide spreads, hard to trade; 3 = liquid enough for retail; 5 = institutional-grade.

**Typical premium-selling screen:**
```
scan_symbols(min_iv_rank=70, min_days_to_earnings=14, min_days_to_ex_div=3, min_iv_rv_ratio=1.2, min_liquidity=3, sort_by=iv_rank)
→ High IV, not near earnings or ex-div, options overpriced vs realized vol, liquid enough to trade — sorted highest IV first
```

**Mean-reversion / low-IV screen:**
```
scan_symbols(max_iv_rank=20, min_days_to_ex_div=3, min_liquidity=3, sort_by=iv_rank, order=asc)
→ Depressed IV names, not near ex-div, sorted cheapest IV first
```

### 4.13 `earnings_calendar`

Upcoming earnings across all tracked symbols within `days_ahead` days. No symbol required — scans the entire tracked universe. **REST:** `GET /v1/alpha/analytics?metric=earnings_calendar&days_ahead=7`

**Run this daily as a risk check** before entering new positions, sizing adjustments, or going over weekends. Every portfolio manager runs this every morning.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `days_ahead` | number | no | 7 (max: 90) |

**Returns:**

```json
{
  "data": {
    "earnings": [
      {"symbol": "AAPL", "report_date": "2026-02-20", "days_until": 6, "timing": "AMC"},
      {"symbol": "MSFT", "report_date": "2026-02-22", "days_until": 8, "timing": "AMC", "eps_estimate": 3.12}
    ],
    "count": 2,
    "days_ahead": 7
  },
  "as_of": "2026-02-14T14:30:00Z"
}
```

Returns only symbols present in AlphaDB's earnings calendar (tracked symbols with earnings data synced). `timing` values: `AMC`, `BMO`, `TNS`, or `null`. `eps_estimate` is included when available.

### 4.14 `put_call_ratio`

Put/call volume ratio for near-term options (0-45 DTE). Measures market sentiment by comparing how much put volume vs. call volume is trading. Live data from Polygon options snapshot — most meaningful during market hours. Accepts comma-separated symbols for batch mode (up to 50). **REST:** `GET /v1/alpha/analytics?metric=put_call_ratio&symbol=SPY`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |

**Interpretation:**

| Ratio | Signal |
|-------|--------|
| > 1.2 | Bearish — heavy put buying, downside hedging or speculation |
| 0.7 – 1.2 | Neutral — balanced put/call activity |
| < 0.7 | Bullish — call-heavy, speculative or momentum-driven |

**Returns:**

```json
{
  "data": {
    "symbol": "SPY",
    "put_call_ratio": 0.82,
    "put_volume": 1450000,
    "call_volume": 1768000,
    "total_volume": 3218000,
    "dte_range": "0-45",
    "as_of": "2026-02-14T14:30:00Z"
  },
  "symbol": "SPY",
  "as_of": "2026-02-14T14:30:00Z"
}
```

`put_call_ratio` is `null` when no call volume is found (market closed or symbol has no options). `dte_range` indicates the expiration window used (always `0-45`). This covers the options most actively traded for directional bets and hedges — longer-dated contracts are dominated by structured products and less informative for sentiment.

---

## 5. Composite — 2 tools

Composite tools fan out across multiple gateway endpoints (quotes, analytics, fundamentals) and combine results into a single response. No single HTTP equivalent — for scripted use call the underlying endpoints individually.

**Freshness metadata:** Both composite tools include a `freshness` object that categorizes every field by its data recency:

| Category | Meaning | Example fields |
|----------|---------|---------------|
| `live` | Real-time from Polygon snapshot (during market hours, includes live-eligible analytics) | `price`, `change_pct`, `volume`, `iv_rank`, `current_iv`, `iv_rv_spread`, `skew_25d`, `rvol`, `iv_structure` |
| `daily` | AlphaDB-computed from last synced DB snapshot (only present when market is closed) | `iv_rank`, `current_iv`, `iv_rv_spread`, `skew_25d`, `rvol`, `iv_structure` |
| `synced` | Periodically synced from Alpha Vantage / FRED | `sector`, `market_cap`, `pe_ratio`, `vix`, `us10y`, `sentiment_score`, `beta` |

During market hours, live-eligible analytics fields shift from `daily` to `live` and the `daily` category may be absent entirely. Both tools also include `market_closed: bool` extracted from the gateway's analytics interceptor.

### 5.1 `symbol_context`

Everything the Strategist needs to evaluate a symbol in one call. Includes price, IV metrics, IV/RV spread, skew (spread + per-strike IVs), put/call ratio, term structure shape (`contango`/`backwardation`/`flat`), earnings, dividends, beta, sentiment, volume, and fundamentals. Accepts comma-separated symbols for batch mode (up to 50).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbols` | string | yes | — Comma-separated ticker symbols (up to 50). Single symbol also works. |
| `as_of` | string | no | — Full symbol snapshot at a historical point in time. Threads `as_of` into all sub-queries (quote, analytics, fundamentals). |

**Sources bundled:**

| Field | Source | Category |
|-------|--------|----------|
| `price`, `change_pct`, `volume` | quotes (Polygon, tastytrade, or Schwab) | Market Data |
| `iv_rank`, `current_iv` | iv_metrics (AlphaDB) | Analytics |
| `iv_rv_spread`, `iv_rv_ratio` | iv_rv_spread (AlphaDB) | Analytics |
| `skew_25d`, `put_25d_iv`, `call_25d_iv` | skew (AlphaDB) | Analytics |
| `put_call_ratio` | put_call_ratio (AlphaDB / Polygon live) | Analytics |
| `days_to_earnings`, `earnings_timing` | earnings (Alpha Vantage) | Fundamentals |
| `implied_move_pct`, `implied_vs_actual_ratio` | earnings_implied_move (AlphaDB) | Analytics (only if earnings <=30d) |
| `days_to_ex_div`, `dividend_yield` | dividends (Alpha Vantage) | Fundamentals |
| `beta` | beta (AlphaDB) | Analytics |
| `sentiment_score` | sentiment (AlphaDB) | Analytics |
| `rvol` | unusual_volume (AlphaDB) | Analytics |
| `iv_structure` | iv_term_structure (AlphaDB) | Analytics |
| `sector`, `market_cap`, `pe_ratio` | company_overview (AlphaDB DB / Alpha Vantage fallback) | Fundamentals |

**Returns:**

```json
{
  "data": {
    "symbol": "TSLA",
    "price": 250.00,
    "change_pct": -1.2,
    "volume": 45000000,
    "iv_rank": 62,
    "current_iv": 0.55,
    "iv_rv_spread": 0.08,
    "iv_rv_ratio": 1.35,
    "skew_25d": 0.04,
    "put_25d_iv": 0.58,
    "call_25d_iv": 0.54,
    "put_call_ratio": 0.92,
    "days_to_earnings": 34,
    "earnings_timing": "AMC",
    "implied_move_pct": null,
    "implied_vs_actual_ratio": null,
    "days_to_ex_div": null,
    "dividend_yield": null,
    "beta": 1.85,
    "sentiment_score": 0.15,
    "rvol": 1.4,
    "iv_structure": "contango",
    "sector": "Consumer Cyclical",
    "market_cap": 800000000000,
    "pe_ratio": 65.2
  },
  "_missing": [
    "implied_move_pct: earnings >30 days away, straddle does not reflect earnings pricing"
  ],
  "freshness": {
    "live": ["price", "change_pct", "volume", "iv_rank", "current_iv", "iv_rv_spread", "iv_rv_ratio", "skew_25d", "implied_move_pct", "rvol", "iv_structure"],
    "synced": ["sector", "market_cap", "pe_ratio", "days_to_earnings", "days_to_ex_div", "dividend_yield", "sentiment_score", "beta"]
  },
  "market_closed": false,
  "as_of": "2026-02-14T14:30:00Z"
}
```

During market hours (`market_closed: false`), analytics fields that support live data (`iv_rank`, `current_iv`, `iv_rv_spread`, `iv_rv_ratio`, `skew_25d`, `implied_move_pct`, `rvol`, `iv_structure`) are in the `live` category. After market close, they shift to `daily`. The `daily` category is absent from the response when all analytics fields are live.

### 5.2 `market_context`

Macro/regime snapshot: index prices (SPY, QQQ, IWM), VIX, yield curve (2Y/10Y), DXY, and AlphaDB-computed SPY analytics (IV/RV spread, skew, put/call ratio, VIX term structure).

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `as_of` | string | no | — Historical market regime snapshot. Index prices from DB, FRED from DB, SPY analytics from DB. |

**Sources bundled:**

| Field | Source | Category |
|-------|--------|----------|
| `spy`, `qqq`, `iwm` (price + change_pct) | quotes (Polygon, tastytrade, or Schwab) | Market Data |
| `vix` (level + change) | quotes/FRED | Market Data |
| `vix_term_structure` | iv_term_structure (AlphaDB) | Analytics |
| `spy_iv_rv_spread`, `spy_iv_rv_ratio` | iv_rv_spread for SPY (AlphaDB) | Analytics |
| `spy_skew_25d` | skew for SPY (AlphaDB) | Analytics |
| `spy_put_call_ratio` | put_call_ratio for SPY (AlphaDB / Polygon live) | Analytics |
| `us2y`, `us10y`, `us10y_us2y_spread` | fred_series (FRED) | Macro |
| `dxy` | quotes/FRED | Macro |
| `market_status` | Polygon | Market Data |

**Returns:**

```json
{
  "data": {
    "spy": {"price": 520.00, "change_pct": -0.45},
    "qqq": {"price": 440.00, "change_pct": -0.62},
    "iwm": {"price": 210.00, "change_pct": -0.30},
    "vix": {"level": 18.5, "change": 1.2},
    "vix_term_structure": "contango",
    "spy_iv_rv_spread": 0.06,
    "spy_iv_rv_ratio": 1.50,
    "spy_skew_25d": 0.06,
    "spy_put_call_ratio": 1.05,
    "us2y": 4.80,
    "us10y": 4.35,
    "us10y_us2y_spread": -0.45,
    "dxy": 104.2,
    "market_status": "open"
  },
  "freshness": {
    "live": ["spy", "qqq", "iwm", "vix_term_structure", "spy_iv_rv_spread", "spy_iv_rv_ratio", "spy_skew_25d"],
    "synced": ["vix", "us10y", "us2y", "us10y_us2y_spread", "dxy"]
  },
  "market_closed": false,
  "as_of": "2026-02-14T14:30:00Z"
}
```

---

## 6. Search (SearXNG + Alpha Vantage) — 3 tools

Data source: Self-hosted SearXNG instance (privacy-preserving, no tracking).

**HTTP note:** Search queries are executed directly from the MCP binary to the SearXNG instance — they do not route through the AlphaDB gateway. No HTTP gateway equivalent exists for these tools.

### 6.1 `search_news`

Financial news search.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `query` | string | yes | — |
| `time_range` | string | no | `day` (`day`, `week`, `month`) |
| `max_results` | number | no | 10 |

**Returns:**

```json
{
  "data": [
    {"title": "Fed Signals Pause in Rate Cuts", "url": "https://...", "date": "2026-02-14", "snippet": "Federal Reserve officials indicated..."}
  ],
  "as_of": "2026-02-14T14:30:00Z"
}
```

### 6.2 `search_ticker`

Ticker-focused news and analyst sentiment via Alpha Vantage `NEWS_SENTIMENT`. Returns articles with per-article and per-ticker sentiment scores.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `symbol` | string | yes | — |
| `time_range` | string | no | `day` |
| `max_results` | number | no | 10 |

**Returns:**

```json
{
  "data": [
    {
      "title": "TSLA Deliveries Beat Estimates",
      "url": "https://...",
      "date": "2026-02-14T14:30:00Z",
      "snippet": "Tesla reported Q1 deliveries of...",
      "source": "Reuters",
      "sentiment_score": 0.35,
      "sentiment_label": "Somewhat-Bullish",
      "ticker_relevance": "0.98",
      "ticker_sentiment_score": "0.42",
      "ticker_sentiment_label": "Bullish"
    }
  ],
  "count": 1,
  "symbol": "TSLA",
  "provider": "alphavantage",
  "as_of": "2026-02-14T14:30:00Z"
}
```

**Provider:** Alpha Vantage (counts against AV rate limit). `time_range` maps to AV's `time_from` parameter.

### 6.3 `search_general`

General web search for SEC filings, analyst reports, macro research.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `query` | string | yes | — |
| `time_range` | string | no | `week` |
| `max_results` | number | no | 10 |

**Returns:**

```json
{
  "data": [
    {"title": "SEC Filing: Form 10-K Annual Report", "url": "https://...", "date": "2026-02-10", "snippet": "Annual report for fiscal year..."}
  ],
  "as_of": "2026-02-14T14:30:00Z"
}
```

---

## 7. Tracking & Admin — 5 tools

**Market data tools (bars, quotes, options, earnings, etc.) work for any symbol without tracking.** Tracking is required only for AlphaDB-computed analytics: IV rank/percentile, volatility, IV/RV spread, skew, IV term structure, earnings implied move, sentiment, beta, correlation, unusual volume. AlphaDB decides start dates, intervals, and feature flags internally — the agent only provides symbols.

**MCP tools:** `track_status` · `system_status` · `restart_server` · `trigger_sync` · `sync_status`

**HTTP-only endpoints (not exposed as MCP tools):** `POST /v1/track/batch` (track_symbols) · `DELETE /v1/track/batch` (untrack_symbols) — these are admin operations called via HTTP, not via MCP.

### 7.1 `track_symbols` (HTTP only — not an MCP tool)

Enroll symbols for computed analytics. AlphaDB validates each ticker against Polygon (rejects invalid/delisted), then enqueues backfill jobs for ~1 year of bars, ATM IV history, and dividends. **REST:** `POST /v1/track/batch`

| Param | Type | Required | |
|-------|------|----------|-|
| `symbols` | array of strings | yes | 1–1000; uppercased and deduplicated |
| `stock_interval` | string | no | `"1m"` (default) or `"1d"` — bar granularity for stock data. Applies to all symbols in the call. |
| `options_interval` | string | no | `"1m"` or `"1d"` — bar granularity for options data. Omit to disable options collection. |

**Interval override:** Analytics that require a specific granularity will override the user's choice. For example, if Greeks are enabled and the user sets `stock_interval: "1m"`, AlphaDB automatically adds `1d` because ATM IV computation needs daily closes. The user's requested interval is always collected; the system only adds additional intervals when analytics require them.

**Returns (202 when ≥1 accepted, 200 when all already tracked, 422 when all failed):**

```json
{
  "results": [
    {"symbol": "AAPL",    "status": "already_tracked"},
    {"symbol": "NVDA",    "status": "accepted", "sync_status": "pending", "jobs_count": 8},
    {"symbol": "INVALID", "status": "error",    "error": "symbol not found on provider"}
  ],
  "accepted": 1,
  "already_tracked": 1,
  "failed": 1
}
```

Per-symbol `status`: `accepted` (new, `jobs_count` River backfill jobs enqueued), `already_tracked` (no-op), `error` (see `error` field — symbol not found, delisted, fetch disabled). Poll with `track_status(summary=true)` until `pending == 0` before calling analytics tools.

### 7.2 `untrack_symbols` (HTTP only — not an MCP tool)

Remove symbols from the tracked universe. Stored bars and insights are retained; market data tools (bars, quotes, options, etc.) keep working for removed symbols. **REST:** `DELETE /v1/track/batch`

| Param | Type | Required | |
|-------|------|----------|-|
| `symbols` | array of strings | yes | uppercased automatically |

**Returns:**

```json
{
  "results": [
    {"symbol": "AAPL", "status": "removed"},
    {"symbol": "FAKE", "status": "not_found"},
    {"symbol": "SPY",  "status": "error", "error": "failed to untrack symbol"}
  ],
  "removed": 1,
  "not_found": 1,
  "failed": 1
}
```

`not_found` = was not tracked (safe to ignore). After removal, analytics tools return 404 for that symbol.

### 7.3 `track_status`

Readiness check for tracked symbols. Two modes via `summary` param. **REST:** `GET /v1/track`

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `summary` | boolean | no | `false` |
| `symbol` | string | no | all symbols (ignored when `summary=true`) |
| `status` | string | no | filter: `ready`, `pending`, `disabled` (ignored when `summary=true`) |
| `metric` | string | no | filter to specific metric e.g. `iv`, `beta` (ignored when `summary=true`) |

**Summary mode** (`summary=true`) — lightweight poll after `track_symbols`. **REST:** `GET /v1/track?summary=true`

```json
{"data": {"total": 48, "ready": 42, "pending": 5, "failed": 1}}
```

`pending == 0` = all backfill complete (symbols are ready or failed).

**Detail mode** (default) — per-symbol, per-metric breakdown. **REST:** `GET /v1/track`

```json
{
  "data": [
    {
      "symbol": "AAPL",
      "sync_status": "ready",
      "analytics": [
        {"metric": "iv",                    "status": "ready"},
        {"metric": "volatility",            "status": "ready"},
        {"metric": "iv_rv_spread",          "status": "ready"},
        {"metric": "skew",                  "status": "ready"},
        {"metric": "iv_term_structure",     "status": "pending", "note": "waiting for options backfill (options_1m)"},
        {"metric": "earnings_implied_move", "status": "ready"},
        {"metric": "sentiment",             "status": "ready"},
        {"metric": "dividends",             "status": "ready"},
        {"metric": "unusual_volume",        "status": "ready"},
        {"metric": "beta",                  "status": "pending", "note": "requires 252 trading days; 45 available"},
        {"metric": "correlation",           "status": "pending", "note": "requires 252 trading days; 45 available"}
      ]
    }
  ]
}
```

`sync_status` lifecycle: `pending` → `syncing` → `ready` | `failed`. Per-metric `status`: `ready` (queryable), `pending` (backfill running — analytics tool returns 404 until ready), `disabled` (not enabled). `note` explains what data is still missing.

| Metric | Requires to reach `ready` |
|--------|--------------------------|
| `iv`, `volatility`, `iv_rv_spread`, `skew` | `stocks_1d` bars + ATM IV history |
| `iv_term_structure`, `earnings_implied_move` | `options_1m` data with Greeks |
| `sentiment` | daily sentiment + ATM IV history (news from `daily_sentiment`, options from `atm_iv_daily`) |
| `dividends` | Alpha Vantage dividends fetch |
| `unusual_volume` | `stocks_1m` intraday bars |
| `beta`, `correlation` | 252 trading days of `stocks_1d` |

Filters (detail mode): `symbol=AAPL` (single symbol), `status=pending` (any pending metric), `metric=beta&status=pending` (specific metric blocked — fastest diagnostic).

### 7.4 `system_status`

Health check for all AlphaDB components. Call first when a tool group is failing to identify the broken upstream. **REST:** `GET /v1/health`

No parameters.

**Returns:**

```json
{
  "status": "degraded",
  "components": {
    "database":           {"status": "up",   "latency_ms": 5},
    "redis":              {"status": "up",   "latency_ms": 2},
    "polygon_proxy":      {"status": "up",   "latency_ms": 120, "source": "proxy"},
    "tastytrade_proxy":   {"status": "up",   "latency_ms": 85,  "source": "proxy"},
    "schwab_proxy":       {"status": "down", "latency_ms": 0,   "source": "proxy", "error": "connection refused"},
    "alphavantage_proxy": {"status": "up",   "latency_ms": 230, "source": "proxy"},
    "fred_proxy":         {"status": "up",   "latency_ms": 95,  "source": "proxy"},
    "searxng":            {"status": "up",   "latency_ms": 45,  "source": "proxy"}
  },
  "provider": "polygon",
  "current_date": "2025-06-15",
  "as_of_mode": true,
  "as_of_boundary": "2025-06-15",
  "as_of": "2026-02-14T14:30:00Z"
}
```

`status`: `healthy` (all up), `degraded` (proxy/SearXNG down; DB and Redis healthy), `unhealthy` (DB or Redis down). `error` present on a component only when `down`. Unconfigured components omitted entirely. `provider` = active `QUOTE_PROVIDER`.

`current_date`: the effective date — boundary date when as-of mode is active, real Eastern date otherwise. Use this to answer "what day is it?" in both modes. `as_of_mode`: `true` when as-of mode is active, `false` otherwise. `as_of_boundary`: the boundary date (only present when `as_of_mode` is `true`).

| Tool group failing | Check |
|--------------------|-------|
| `bars`, `quotes`, `options_chain`, `options_expirations` | proxy for active `provider` field |
| `earnings`, `dividends`, `company_overview`, `insider_transactions`, `technical_indicator` | `alphavantage_proxy` |
| `fred_series` | `fred_proxy` |
| `market_metrics` | `tastytrade_proxy` |
| `market_hours` | `schwab_proxy` (recent/future) or `calendar` (historical) |
| `search_news`, `search_general` | `searxng` |
| `search_ticker` | `alphavantage_proxy` |
| `iv_metrics`, `beta`, `sentiment`, and all other analytics | `database` |

### 7.5 `restart_server`

Gracefully exits the MCP process. The MCP client (Claude Code) will automatically respawn it, picking up any newly built binary. Use after running `make build-mcp` to reload without restarting the MCP host.

No parameters.

**Returns (200):**

```json
{"status": "restarting"}
```

### 7.6 `trigger_sync`

Trigger a data sync for all tracked symbols. Creates bulk bar sync jobs, computed analytics jobs (IV, Greeks), and insight jobs (sentiment, dividends). Returns 409 if a sync is already running. **REST:** `POST /v1/admin/sync/trigger`

No parameters.

**Returns (202 Accepted):**

```json
{
  "message": "sync triggered",
  "sync_id": 42
}
```

**Returns (409 Conflict — sync already running):**

```json
{
  "error": "sync is already running",
  "sync_id": 41,
  "started_at": "2026-02-23T20:15:00-05:00"
}
```

Use `sync_status` to monitor progress after triggering.

### 7.7 `sync_status`

Get the latest sync status. Returns sync_id, status, timing, symbol count, and summary. **REST:** `GET /v1/admin/sync/latest`

No parameters.

**Returns:**

```json
{
  "data": {
    "sync_id": 42,
    "status": "completed",
    "started_at": "2026-02-23T20:15:00-05:00",
    "finished_at": "2026-02-23T21:03:42-05:00",
    "symbols": 45,
    "summary": {
      "total_jobs": 128,
      "completed": 125,
      "failed": 3
    }
  },
  "meta": {
    "timestamp": "2026-02-23T22:00:00-05:00"
  }
}
```

| Field | Description |
|-------|-------------|
| `sync_id` | Unique identifier for this sync run |
| `status` | `running`, `completed`, or `failed` |
| `started_at` | When the sync started (Eastern RFC3339) |
| `finished_at` | When the sync finished — omitted while `running` |
| `symbols` | Number of tracked symbols in the run |
| `summary` | Job statistics — omitted while `running`, populated on completion/failure |

Returns 404 if no sync has ever run. Poll this after `trigger_sync` to monitor completion.

### 7.8 As-Of Mode (Admin HTTP Endpoints)

As-of mode defines a **boundary date** for deterministic historical replay. The agent must always provide `as_of` on every request — the server rejects requests without it (400) and rejects `as_of` values after the boundary (400). `start` or `end` after the boundary also returns 400. All responses include an `X-As-Of-Boundary` header. There are no MCP tools for as-of mode — it is controlled exclusively via admin HTTP endpoints.

**Authentication:** Requires `ALPHA_AS_OF_KEY` or `ALPHA_ADMIN_KEY`. The `GET` endpoint also accepts `ALPHA_API_KEY` (the MCP server polls it to sync state).

#### `POST /v1/admin/as-of-date` — Enable as-of mode

```bash
curl -X POST http://localhost:8080/v1/admin/as-of-date \
  -H "X-API-Key: $ALPHA_AS_OF_KEY" \
  -H "Content-Type: application/json" \
  -d '{"end": "2025-06-15"}'
```

Body: `{"end": "2025-06-15"}` (YYYY-MM-DD → ET midnight) or `{"end": "2025-06-15T14:30:00-04:00"}` (RFC3339 for intraday).

**Returns (200):**

```json
{"enabled": true, "end": "2025-06-15T00:00:00-04:00"}
```

#### `GET /v1/admin/as-of-date` — Query as of state

```bash
curl http://localhost:8080/v1/admin/as-of-date -H "X-API-Key: $ALPHA_API_KEY"
```

**Returns (200):**

```json
{"enabled": true, "end": "2025-06-15T00:00:00-04:00"}
```

Or when inactive: `{"enabled": false}`

#### `DELETE /v1/admin/as-of-date` — Disable as-of mode

```bash
curl -X DELETE http://localhost:8080/v1/admin/as-of-date -H "X-API-Key: $ALPHA_AS_OF_KEY"
```

**Returns (200):** `{"enabled": false}`

#### Behavior when active

| Scenario | Behavior |
|----------|----------|
| Agent omits `as_of` | 400 — `as_of` is required in as of mode |
| Agent passes `as_of` before or at boundary | Allowed |
| Agent passes `as_of` after boundary | 400 — `as_of` exceeds as of boundary |
| Agent passes `start` after boundary | 400 — `start` exceeds as-of boundary |
| Agent passes `end` after boundary | 400 — `end` exceeds as-of boundary |
| Agent omits `as_of` | 400 — `as_of` is required in as-of mode |
| `system_status` | Returns `current_date` = boundary date, `as_of_mode` = true, `as_of_boundary` = boundary date |
| Trading endpoints (`/v1/trading/*`) | Pricing endpoints require `as_of`; read-only endpoints (`get_account_summary`, `pnl-history`, lists, detail) pass through |
| Live streaming (`/v1/stream/*`) | Blocked — 403 |
| Market movers, status, forex, indices | Blocked — 403 |
| Portfolio greeks, raw query | Blocked — 403 |
| Fundamentals: insiders, indicator, transcript | Blocked — 403 |
| Search: news, general, ticker | Blocked — 403 (all call external APIs) |
| Provider proxies (`/v1/proxy/*`) | Blocked — 403 (Polygon, FRED, tastytrade, Schwab) |
| Raw Alpha Vantage proxy (`/v1/alphavantage/query`) | Blocked — 403 |
| Analytics: `put_call_ratio` | Blocked — 403 (requires live API data not stored in DB) |
| Analytics: `earnings_implied_move` | Allowed — returns pre-computed implied/actual move from `earnings_quarterly` |
| Analytics: `earnings_calendar` | Allowed — reads from `earnings_quarterly` table (no live fallback in as-of mode) |
| Sentiment (via `alpha/analytics?metric=sentiment`) | Allowed — reads pre-computed scores from DB |
| Market hours | Allowed — calendar-based, no external API |

The MCP server polls `GET /v1/admin/as-of-date` every 30 seconds to sync state. After enabling or disabling, allow up to 30s for MCP tools to reflect the change.

---

## Data Quality

Some analytics tools include a `data_quality` object when fallback data sources were used instead of live market quotes. This lets agents know the provenance and limitations of the data, which is critical for cross-ticker comparisons.

```json
{
  "data_quality": {
    "price_source": "close",
    "iv_source": "computed",
    "note": "Greeks computed from daily close prices via Black-Scholes; bid/ask not available"
  }
}
```

| Field | Values | Meaning |
|-------|--------|---------|
| `price_source` | `"quote"` | Options priced from live bid/ask mid (highest quality) |
| | `"close"` | Options priced from daily close (bar data — no bid/ask available) |
| `iv_source` | `"provider"` | Greeks supplied by the data provider (Polygon, tastytrade) |
| | `"computed"` | Greeks computed by AlphaDB via Black-Scholes from close prices |
| | `"n/a"` | IV not applicable or not available |
| `note` | string | Human-readable explanation of the data source and any caveats |

**When `data_quality` is omitted**, the data comes from live quotes with provider-supplied Greeks — the highest quality source. During market hours, live-eligible tools (`iv_term_structure`, `earnings_implied_move`) use Polygon snapshots with real bid/ask and provider Greeks, so `data_quality` is typically omitted.

**Cross-ticker comparison guidance:** When comparing analytics across symbols, check that `data_quality.price_source` and `iv_source` match. Comparing a symbol with `"quote"` pricing against one with `"close"` pricing may produce misleading relative rankings. The `note` field provides context for the agent to communicate data limitations to the user.

**Tools that include `data_quality`:** `iv_term_structure`, `earnings_implied_move`.

---

## Summary

| # | Category | Tools | Count |
|---|----------|-------|-------|
| 1 | Market Data (Polygon/tastytrade/Schwab) | `bars`, `quotes`, `options_chain`, `options_expirations`, `market_metrics`, `market_hours`, `market_hours_range`, `list_indices` | 8 |
| 2 | Fundamentals (Alpha Vantage) | `earnings`, `dividends`, `company_overview`, `insider_transactions`, `earnings_transcript`, `alpha_vantage_query` | 6 |
| 3 | Indicators & Macro (AV + FRED) | `technical_indicator`, `fred_series` | 2 |
| 4 | Analytics (AlphaDB-computed) | `iv_metrics`, `iv_term_structure`, `iv_rv_spread`, `skew`, `earnings_implied_move`, `sentiment`, `volatility`, `beta`, `correlation`, `unusual_volume`, `portfolio_greeks`, `scan_symbols`, `earnings_calendar`, `put_call_ratio` | 14 |
| 5 | Composite | `symbol_context`, `market_context` | 2 |
| 6 | Search (SearXNG + AV) | `search_news`, `search_ticker`, `search_general` | 3 |
| 7 | Admin | `track_status`, `system_status`, `restart_server`, `trigger_sync`, `sync_status` | 5 |
| 8 | Trading / OMS | `create_account`, `update_account`, `list_accounts`, `get_account`, `get_account_summary`, `adjust_cash`, `reset_account`, `take_snapshot`, `get_pnl_history`, `get_strategy_performance`, `simulate_outcome`, `replay_day`, `get_audit_log`, `reconcile_account`, `list_structures`, `get_collateral`, `resolve_expirations`, `validate_order`, `validate_order_batch`, `place_order`, `get_order`, `list_orders`, `count_orders`, `cancel_order`, `sync_order`, `list_trading_positions`, `count_trading_positions`, `get_trading_position`, `update_trading_position`, `close_position`, `get_position_structure`, `get_position_collateral`, `place_spread`, `close_spread`, `get_spread`, `list_spreads`, `cancel_spread`, `sync_spread`, `get_spread_structure`, `create_group`, `list_groups`, `get_group`, `close_group`, `adjust_group`, `execute_roll`, `list_rolls`, `get_roll`, `create_contingent`, `list_contingents`, `cancel_contingent`, `place_bracket`, `get_account_greeks`, `position_alerts`, `risk_dashboard` | 54 |
| 9 | Portfolio | `create_position`, `list_positions`, `record_position_close` | 3 |
| 10 | Agentic (not yet implemented) | `create_insight_job`, `get_job_status`, `get_insight_result` | 3 |
| | **Total** | | **99** |

### Consumer Access Matrix

| Tool | Strategist Daily | Strategist Live | Financial Analyst |
|------|:---:|:---:|:---:|
| `market_context` | First call every session | First call every session | On demand |
| `symbol_context` | Per symbol in watchlist | Per symbol with issues | On demand |
| `bars` | Trend analysis | — | Chart analysis |
| `quotes` | Batch screening (50+ symbols) | Quick price check | On demand |
| `options_chain` | Vol surface analysis | — | Chain exploration |
| `options_expirations` | DTE targeting | — | Chain exploration |
| `iv_metrics` | Deep-dive (beyond symbol_context) | — | On demand |
| `iv_term_structure` | Architect DTE decisions | — | On demand |
| `iv_rv_spread` | Deep-dive (beyond composites) | — | Edge analysis |
| `skew` | Template selection | — | Strike analysis |
| `earnings_implied_move` | Position sizing around events | — | Earnings analysis |
| `scan_symbols` | New-position discovery | — | On demand |
| `earnings_calendar` | Daily morning risk check | Daily risk check | On demand |
| `put_call_ratio` | Sentiment confirmation | Quick sentiment check | On demand |
| `earnings` | Hard filter (<5 days) | — | On demand |
| `dividends` | Hard filter (<3 days) | — | On demand |
| `company_overview` | Discovery evaluation | — | On demand |
| `beta` | Risk sizing | — | On demand |
| `unusual_volume` | Screening, discovery | Quick check | On demand |
| `sentiment` | Screening weight | Quick check | On demand |
| `correlation` | Portfolio diversification | — | Portfolio analysis |
| `portfolio_greeks` | — | — | Risk analysis, what-if |
| `insider_transactions` | — | — | Conviction signal |
| `earnings_transcript` | — | — | Deep analysis |
| `technical_indicator` | Discovery evaluation | — | Setup quality |
| `fred_series` | Macro regime | — | Research |
| `search_news` | Breaking news, events | Breaking news | Research |
| `search_ticker` | Per-symbol sentiment | Quick check | Research |
| `search_general` | — | — | SEC filings, reports |
| `market_metrics` | IV rank validation, liquidity check | Quick IV check | IV/liquidity screening |
| `market_hours` | Session schedule, trading day check | — | On demand |
| `market_hours_range` | Bulk trading days for date range | — | Replay startup |
| `list_indices` | Index symbol reference | — | On demand |
| `volatility` | Full vol profile | — | Vol analysis |
| `track_status` | Poll completion (summary=true) or inspect detail | — | On demand |
| `system_status` | Diagnostics, health check | Connectivity check | Diagnostics |
| `restart_server` | Reload after binary rebuild | — | — |
| `trigger_sync` | Trigger data sync for all tracked symbols | — | On demand |
| `sync_status` | Get latest sync status | — | On demand |
| `list_accounts` | Review accounts, balances | — | Execution |
| `get_account_summary` | Cash, P&L, positions value | — | Execution |
| `validate_order` | Pre-trade risk check | — | Execution |
| `place_order` | Execute trade | — | Execution |
| `list_orders` | Order history | — | Execution |
| `get_order` | Fill details | — | Execution |
| `cancel_order` | Cancel open order | — | Execution |
| `sync_order` | Sync live Schwab status | — | Execution |
| `list_trading_positions` | Open/closed positions | — | Trading |
| `get_trading_position` | Position details + P&L | — | Trading |
| `close_position` | Close position (market order) | — | Trading |
| `update_trading_position` | Annotate exit strategy | Annotate exit strategy | Trading |
| `adjust_cash` | Manual cash sync | — | Execution |
| `reset_account` | Reset paper account | — | Setup |
| `take_snapshot` | Daily P&L snapshot | — | Execution |
| `get_pnl_history` | Performance review | — | Execution |
| `validate_order_batch` | Multi-order pre-check | — | Execution |
| `create_account` | Create paper/live account | — | Setup |
| `update_account` | Update risk limits | — | Setup |
| `place_spread` | Multi-leg atomic spread | — | Execution |
| `close_spread` | Close spread (opposing legs) | — | Execution |
| `get_spread` | Spread + leg details | — | Execution |
| `list_spreads` | Filter spreads | — | Execution |
| `sync_spread` | Sync live spread fills | — | Execution |
| `create_group` | Create strategy group | — | Strategy management |
| `list_groups` | Review open groups | — | Strategy management |
| `get_group` | Group positions summary | — | Strategy management |
| `close_group` | Close all group positions | — | Strategy management |
| `adjust_group` | Adjust group (close + add legs) | — | Strategy management |
| `execute_roll` | Roll position/spread | — | Execution |
| `list_rolls` | Roll history | — | Execution |
| `get_roll` | Roll details | — | Execution |
| `create_contingent` | Manual contingent link | — | Automation |
| `list_contingents` | Contingent status | — | Automation |
| `cancel_contingent` | Cancel pending contingent | — | Automation |
| `get_position_structure` | Structure & risk analysis | — | Risk analysis |
| `get_spread_structure` | Spread risk metrics | — | Risk analysis |
| `list_structures` | All position structures | — | Risk analysis |
| `get_collateral` | Collateral & buying power | — | Risk management |
| `get_position_collateral` | Per-position collateral | — | Risk management |
| `resolve_expirations` | Expire/exercise options | — | Lifecycle |
| `get_audit_log` | Audit trail | — | Compliance |
| `reconcile_account` | Sync broker status | — | Operations |
| `get_strategy_performance` | Win/loss stats | — | Performance review |
| `count_orders` | Order count | — | Execution |
| `count_trading_positions` | Position count | — | Trading |
| `place_bracket` | Entry + TP + SL in one call | — | Execution |
| `get_account_greeks` | Account portfolio Greeks | — | Risk analysis |
| `position_alerts` | Positions near exit thresholds | — | Monitoring |
| `risk_dashboard` | Cross-account exposure | — | Risk management |

---

## §8 Trading / OMS

Order Management System tools for paper and live (Schwab) trading. OMS requires `OMS_DB_URL` to be set. If not configured, the tools return a service-unavailable error. Call `list_accounts` on session start to verify OMS connectivity.

**Paper trading** fills immediately at market price (24/7, including outside market hours). Set `"provider": "paper"` when creating an account. **Live trading** via Schwab requires `SCHWAB_APP_KEY`, `SCHWAB_APP_SECRET`, and `SCHWAB_TOKEN_URL`.

**Multi-account Schwab:** A single Schwab login can access multiple accounts (401k, Roth IRA, brokerage). Create a separate OMS account per Schwab sub-account and set `params.schwab_account_number` to the Schwab account number. If only one Schwab account exists under the login, this param is optional. If multiple exist without the param, the adapter returns an error listing available account numbers.

### 8.1 Account Tools

#### `create_account`
Creates a trading account. Idempotent on `key`.

**Parameters:**
- `key` (required, string) — Unique identifier, e.g. `"paper"`, `"schwab-main"`
- `name` (string) — Human-readable name
- `provider` (string) — `"paper"` (default) or `"schwab"`
- `initial_cash` (number) — Starting cash, default 100000
- `max_order_value` (number) — Max notional per order
- `max_open_positions` (integer) — Max simultaneous open positions
- `max_order_quantity` (integer) — Max shares/contracts per order (fat-finger guard)
- `daily_loss_limit` (number) — Daily drawdown limit (order rejected when exceeded)
- `params` (object) — Provider-specific params. Slippage: `slippage_pct` (fraction of price, single-leg default), `spread_slippage_pct` (fraction of price, flat spread override), `slippage_1leg`..`slippage_4leg` (ORATS: fraction of bid-ask half-spread). Lookup: `slippage_Nleg` → `spread_slippage_pct` → `slippage_pct` → 0. Commission: `commission_per_contract` (default $0.65), `commission_per_share` (default $0). Schwab multi-account: `schwab_account_number` (string) — the Schwab account number to route orders to (required when multiple Schwab accounts exist under the same login).

#### `update_account`
Updates risk limits or params on an existing account.

**Parameters:** same fields as `create_account` (all optional except `key`)

#### `list_accounts`
Returns all accounts with their current configuration and risk limits.

**No parameters.**

#### `get_account`
Returns a single trading account by key — raw record with limits, params, and configuration.

**Parameters:**
- `key` (required, string) — Account key

#### `get_account_summary`
Returns cash balance, open positions value, and daily P&L.

**Parameters:**
- `key` (required, string) — Account key

**Response:** `{cash_balance, open_positions_value, total_value, daily_realized_pnl, daily_unrealized_pnl, open_positions_count, open_spreads_count, collateral_reserved, buying_power}`

#### `adjust_cash`
Adjusts the cash balance of a trading account. Records an audit trail in account params.

**Parameters:**
- `key` (required, string) — Account key
- `amount` (required, number) — Positive adds, negative subtracts
- `reason` (string) — Reason for adjustment (e.g. `"manual trade sync"`, `"dividend"`)

#### `reset_account`
Wipes all orders, fills, and positions for a paper account and restores initial cash. Rejects non-paper accounts.

**Parameters:**
- `key` (required, string) — Account key (must be `provider=paper`)

#### `take_snapshot`
Captures the current account state (cash, positions value, P&L) as a point-in-time snapshot. One snapshot per account per date (upserts on same day).

**Parameters:**
- `key` (required, string) — Account key

**Response:** `{id, account_key, snapshot_date, cash_balance, positions_value, total_value, realized_pnl, unrealized_pnl}`

#### `get_pnl_history`
Returns account snapshots within a date range. Optionally groups realized P&L by `metadata.strategy`.

**Parameters:**
- `key` (required, string) — Account key
- `start` (required, string) — Start date `YYYY-MM-DD`
- `end` (required, string) — End date `YYYY-MM-DD`
- `group_by` (string) — `"strategy"` to include `strategy_breakdown` in response

**Response:** `{snapshots: [...], risk_metrics: {...}, strategy_breakdown?: [{strategy, realized_pnl}]}`

The `risk_metrics` object is always present, computed from the daily `total_value` equity curve:

| Field | Type | Description |
|-------|------|-------------|
| `sharpe_ratio` | number \| null | Annualized Sharpe ratio (null if < 5 data points or zero volatility) |
| `sortino_ratio` | number \| null | Annualized Sortino ratio (null if < 5 data points or no negative returns) |
| `max_drawdown_pct` | number \| null | Peak-to-trough max drawdown as % (negative value) |
| `max_drawdown_duration_days` | number \| null | Longest drawdown period in trading days |
| `current_drawdown_pct` | number \| null | Current distance from equity peak as % |
| `calmar_ratio` | number \| null | Annualized return / abs(max drawdown) (null if no drawdown) |
| `total_return_pct` | number \| null | Total return over period as % |
| `annualized_return_pct` | number \| null | Annualized return (252 trading days) as % |
| `volatility_annual` | number \| null | Annualized volatility as % |
| `win_rate` | number \| null | Fraction of positive-return days (0–1) |
| `profit_factor` | number \| null | Sum of positive returns / sum of negative returns |
| `best_day` | object \| null | `{date, return_pct}` — highest single-day return |
| `worst_day` | object \| null | `{date, return_pct}` — lowest single-day return |
| `risk_free_rate_annual` | number \| null | Risk-free rate used (null = 0 assumed) |
| `num_days` | number | Number of snapshots in the series |
| `warning` | string \| null | Present when < 20 trading days of data |

Metrics require at least 2 snapshots. Sharpe/Sortino require at least 5. Returns are computed from mark-to-market `total_value` (includes unrealized P&L). Annualization uses 252 trading days. Cash deposits/withdrawals via `adjust_cash` are not backed out (simple returns, not TWR).

#### `simulate_outcome`
Walks daily bars forward from each open position's entry date, checks `closing_strategy` exit rules, and optionally closes positions when triggers fire. Supports both equity and spread positions.

**Parameters:**
- `account_key` (required, string) — Account key
- `position_ids` (array of strings) — Optional subset of position UUIDs to evaluate. Omit to evaluate all open positions.
- `as_of` (string) — End date `YYYY-MM-DD` for bar walk. Default: today.
- `apply` (boolean) — If `true`, close triggered positions and update account balances. Default: `false` (read-only preview).

**Closing strategy keys (equity/stock):**
- `stop_loss_price` (number) — Exit when bar low ≤ price (long) or bar high ≥ price (short). Uses bar open on gap-through for realistic fill.
- `take_profit_price` (number) — Exit when bar high ≥ price (long) or bar low ≤ price (short). Uses bar open on gap-through.
- `trailing_stop_pct` (number) — Percent trailing stop from high-water mark. e.g. `5.0` = 5%. Long: triggers when price drops below `HWM × (1 - pct/100)`. Short: triggers when price rises above `LWM × (1 + pct/100)`. Gap-through aware.
- `max_hold_days` (integer) — Exit at bar close after N trading days
- `eod_exit_time` (string) — `"HH:MM"` in Eastern time. Closes position when the configured time is reached. Valid range: `09:30`–`16:00`. Evaluated by `replay_day` (triggers at the exact minute bar) and live auto-exit. Not evaluated by `simulate_outcome` (daily bars lack intraday resolution). Price-based rules (stop loss, take profit) on the same bar take priority.

**Closing strategy keys (spreads and standalone options):**
- `profit_pct` (number) — Credit spreads/short options: exit when cost-to-close < entry credit × (1 - threshold). Debit spreads/long options: exit when value exceeds entry cost by threshold. e.g. `0.50` = take profit at 50%.
- `loss_pct` (number) — Exit when loss ≥ percentage of entry value
- `intraday_profit_pct` (number) — Same math as `profit_pct` but only evaluated during RTH (09:30–16:00 ET). For 0DTE strategies that should only take profits when markets are actually tradeable.
- `max_dollar_loss` (number) — Absolute $ loss guardrail, evaluated continuously per-tick. Fires when unrealized loss ≥ threshold. Use for undefined-risk positions (ratio spreads, backspreads) where a percent-of-entry-credit rule is not meaningful.
- `max_dollar_profit` (number) — Absolute $ profit target, evaluated continuously per-tick. Fires when profit ≥ threshold. Use for ratio spreads with near-zero entry credit where `profit_pct` is meaningless (50% of $0.05 = $0.025). Exit reason: `MAX_PROFIT`.
- `max_loss` (number) — **Pre-trade entry validation cap only.** The order/spread is rejected at submit time if projected max loss exceeds this threshold. NOT evaluated during the trade — use `max_dollar_loss` for continuous evaluation. Transitional fallback: when `max_dollar_loss` is unset, the evaluator reads `max_loss` as the continuous guardrail for deploy safety.
- `dte_exit` (integer) — Exit when DTE < N trading days
- `delta_exit` (number) — Spreads only: exit when |net_delta| ≥ threshold. Per-position normalized (`Σ(bs_delta × side_sign × qty) / totalContracts`), so thresholds are size-invariant. Risk kill switch — fires before P&L% stops.
- `gamma_exit` (number) — Spreads only: exit when |net_gamma| ≥ threshold. Same normalization as `delta_exit`. Risk kill switch — fires before P&L% stops.
- `trailing_stop_pct` (number) — Trailing stop on P&L%. Triggers when P&L% drops below `peak × (1 - pct/100)`. Only activates after a positive P&L peak has been reached.
- `max_hold_days` (integer) — Exit at last bar's close after N trading days (end-of-day fill, consistent with equity)
- `stop_loss_price` (number) — Standalone options only: underlying price level trigger (repriced via Black-Scholes at exit). Ignored for spreads.
- `take_profit_price` (number) — Standalone options only: underlying price level trigger. Ignored for spreads.
- `eod_exit_time` (string) — `"HH:MM"` in Eastern time. Same behavior as equity key: `replay_day` + live auto-exit only.

**Tie-breaking order** when multiple rules fire on the same tick (highest priority first): Expiration > GammaExit > DeltaExit > MaxDollarLoss($) > StopLoss > LossPct > EODExitTime > DTEExit > MaxHold > IntradayProfitPct > MaxDollarProfit($) > TakeProfit > ProfitPct > TrailingStop. Kill switches (gamma, delta, absolute $ loss) fire first so a position exits before the P&L% stop absorbs the full move.

**ATR-based trailing:** To trail by an ATR multiple, compute the percentage at entry: `trailing_stop_pct = (ATR_multiplier × ATR / entry_price) × 100`. e.g. 1.5× ATR of $8 on a $200 entry = `trailing_stop_pct: 6.0`. To delay activation (e.g. activate after day 0), place the order without `trailing_stop_pct` and use `update_trading_position` to add it after the desired holding period.

Spreads and standalone options auto-exit at expiration using intrinsic value. Standalone option positions (call/put without `spread_id`) are repriced via Black-Scholes at each bar.

**Exit reasons:** `EXPIRATION`, `GAMMA_RISK`, `DELTA_BREACH`, `MAX_LOSS` (from `max_dollar_loss`), `STOP_LOSS`, `LOSS_LIMIT`, `EOD_EXIT`, `DTE_EXIT`, `MAX_HOLD`, `INTRADAY_PROFIT`, `MAX_PROFIT` (from `max_dollar_profit`), `TAKE_PROFIT`, `PROFIT_TARGET`, `TRAILING_STOP` (listed in tie-breaker priority order)

**Response:** `{outcomes: [{position_id, spread_id?, symbol, strategy?, instrument_type, exit_date, exit_price, exit_reason, pnl, tc_total, hold_days, legs?}], split_adjustments?, evaluated, skipped, tc_total, errors?, diagnostics?}`

`tc_total` is round-trip transaction costs (entry + exit). Options/spreads: `commission_per_contract × qty × 2` per leg. Equity: `commission_per_share × qty × 2`. Top-level `tc_total` is the aggregate across all outcomes. Gross P&L is in `pnl`; net = `pnl - tc_total`.

Equity and standalone option outcomes return one outcome per position. Spread outcomes return **one outcome per spread** with `instrument_type: "spread"`, net `pnl` across all legs, and a `legs` array with per-leg breakdown: `[{position_id, instrument_type, side, exit_price, pnl}]`. The top-level `pnl` is the sum of all leg P&Ls.

`strategy` is populated from the position's first-class `strategy` column (falls back to `metadata.strategy` for legacy positions).

`diagnostics` is an array of per-position debugging info: `{position_id, symbol, bars_fetched, status}`. Status values: `triggered`, `no_trigger`, `no_bars`, `error`.

Positions without `closing_strategy` are skipped. When `apply=true`, triggered positions are closed: cash is adjusted, position status set to `closed`, and `realized_pnl` is recorded. Default is read-only preview.

#### `replay_day`
Evaluate all open positions against **minute bars** for a single trading day and close triggered positions. Used for replay/backtesting — the agent steps through historical days one at a time.

**Parameters:**
- `account_key` (required, string) — Account key
- `date` (required, string) — Trading date to replay `YYYY-MM-DD`

**Behavior:**
- Walks `stocks_1m` bars for equity positions, `options_1m` bars for options/spreads (minute-by-minute)
- Options use real prices from `options_1m` (mid preferred, close fallback); falls back to stock minute bars + Black-Scholes if option bars unavailable
- Always applies: triggered positions are closed and cash updated (no preview mode)
- Supports the same closing strategy keys as `simulate_outcome`

**Response:** `{date, exits: [{position_id, spread_id?, symbol, strategy?, instrument_type, exit_date, exit_price, exit_reason, pnl, tc_total, hold_days, legs?}], split_adjustments?, entries_today, evaluated, skipped, errors?, diagnostics?}`

`entries_today` counts all positions opened on that date (including any already closed by prior replay days).

**Replay workflow:** Call `replay_day` for each trading day in sequence, then `get_account_summary` to see the resulting portfolio state.

**Split adjustment:** Both `simulate_outcome` and `replay_day` automatically resolve stock splits before evaluation — adjusting quantity, cost basis, strike, and price-based closing strategy fields. Percentage/dollar thresholds are not adjusted. Idempotent. Responses include `split_adjustments` array when splits were applied.

#### `resolve_splits`
Manually trigger split resolution for an account. Automatically called by `simulate_outcome` and `replay_day` — use only for ad-hoc inspection.

**Parameters:**
- `account_key` (required, string) — Account key
- `as_of` (required, string) — Cutoff date `YYYY-MM-DD` — only splits on or before this date are applied

**Response:** `{adjusted, orders_adjusted, adjustments: [{position_id, symbol, instrument_type, split_date, split_from, split_to, old_quantity, new_quantity, old_cost_basis, new_cost_basis, ...}], errors?}`

### 8.2 Order Tools

#### `validate_order`
Validates an order against account risk limits without placing it. Returns 422 with `violations` array if any limit is breached.

#### `validate_order_batch`
Validates multiple orders at once. Returns per-order results (does not short-circuit on first failure).

**Parameters:**
- `orders` (required, array) — Array of order objects (same schema as `place_order`)

**Response:** Array of `{symbol, valid, error?, warnings?}` — one per order.

#### `place_order`
Places a buy or sell order. Paper accounts fill immediately. Schwab orders are submitted and require `sync_order` to check fill status.

**Order schema (shared by `validate_order` and `place_order`):**
- `account_key` (required, string)
- `symbol` (required, string) — Ticker, e.g. `"AAPL"`
- `side` (required, string) — `"buy"` or `"sell"`
- `quantity` (required, integer) — Shares or contracts
- `instrument_type` (string) — `"stock"` (default), `"call"`, `"put"`
- `order_type` (string) — `"market"` (default), `"limit"`, `"stop"`, `"stop_limit"`
- `limit_price` (number) — Required for `limit` and `stop_limit`
- `stop_price` (number) — Required for `stop` and `stop_limit`
- `strike` (number) — Strike price (options only)
- `expiry` (string) — Expiration date `YYYY-MM-DD` (options only)
- `time_in_force` (string) — `"day"` (default), `"gtc"`, `"fok"`, `"ioc"`. Passed to broker for live orders.
- `note` (string) — Trade rationale
- `placed_by` (string) — Agent or user identifier, e.g. `"strategist-agent"`
- `closing_strategy` (object) — Structured exit plan enforced by `simulate_outcome` and `auto-exit`. Equity: `{"stop_loss_price": 140, "take_profit_price": 170, "trailing_stop_pct": 5.0, "max_hold_days": 30}`. Spreads: `{"profit_pct": 0.50, "loss_pct": 1.0, "max_dollar_loss": 500, "max_dollar_profit": 150, "dte_exit": 7, "delta_exit": 0.30, "gamma_exit": 0.05, "trailing_stop_pct": 5.0}`. Time-based: `{"eod_exit_time": "15:30"}`.
- `metadata` (object) — User-controlled structured tags that flow from order to position, e.g. `{"strategy": "iron_condor", "signal_id": "abc123"}`
- `idempotency_key` (string) — Optional dedup key. If a previous order exists with the same `account_key` + `idempotency_key`, returns that order instead of creating a new one.
- `opened_at` (string) — ISO8601 timestamp to override the position's `opened_at`. For backtesting/replay where the simulated time differs from wall clock. When omitted, uses server `NOW()`.
- `group_id` (string) — Optional strategy group ID. Links the order and resulting position to a group.

**Response:** `{id, status, symbol, side, quantity, filled_price, fills, outside_market_hours}`

#### `get_order`
Returns order details and fill records by order ID.

**Parameters:** `id` (required, string)

#### `list_orders`
Lists orders filtered by account, status, symbol, placed_by, note content, or metadata.

**Parameters:** `account_key` (string), `status` (string: `pending|open|filled|partial|cancelled|rejected`), `symbol` (string), `placed_by` (string), `note` (string — case-insensitive substring match), `metadata` (object — JSONB containment filter, e.g. `{"strategy": "momentum"}`)

#### `cancel_order`
Cancels an open order.

**Parameters:** `id` (required, string)

#### `sync_order`
Syncs a live Schwab order to retrieve the latest fill status. No-op for paper orders.

**Parameters:** `id` (required, string)

### 8.3 Position Tools (OMS-derived)

These track positions derived from order fills — distinct from Portfolio §9 which is manual note-taking.

#### `list_trading_positions`
Lists OMS positions (auto-derived from fills) filtered by account, status, instrument type, symbol, strategy, or metadata.

**Parameters:** `account_key` (string), `status` (string: `open|closed`), `instrument_type` (string: `stock|call|put`), `symbol` (string), `strategy` (string — first-class column filter, e.g. `"hunter_momentum"`), `spread_id` (string), `group_id` (string), `metadata` (object — JSONB containment filter), `opened_after` (string: YYYY-MM-DD), `opened_before` (string: YYYY-MM-DD), `closed_after` (string: YYYY-MM-DD), `closed_before` (string: YYYY-MM-DD), `limit` (integer, default 100), `offset` (integer), `sort` (string: opened_at, -opened_at, closed_at, -closed_at, realized_pnl, symbol)

**Response fields per position:** `symbol`, `side`, `strategy`, `quantity`, `avg_cost_basis`, `realized_pnl`, `multiplier`, `metadata`

#### `get_trading_position`
Returns a single OMS position with full fill history.

**Parameters:** `id` (required, string)

#### `update_trading_position`
Updates annotations on an open position. Uses merge semantics — existing keys are preserved, provided keys are added/overwritten.

**Parameters:**
- `id` (required, string) — Position ID
- `closing_strategy` (object) — Exit plan updates to merge
- `params` (object) — Params updates to merge
- `metadata` (object) — Metadata updates to merge

Only `closing_strategy`, `params`, and `metadata` can be updated. Fill-derived fields (quantity, avg_cost_basis, etc.) are immutable.

#### `close_position`
Closes an open position by placing an opposing market order. Paper accounts fill immediately.

**Parameters:** `id` (required, string)

### 8.4 Spread Tools (Multi-Leg)

Multi-leg atomic spread orders. Paper fills immediately at theoretical prices; live (Schwab) uses `ALL_OR_NONE`. Per-leg positions are created with `spread_id` linkage. Limit spreads (`credit`/`debit`) require option pricing to distribute the net price across legs — rejected if pricing is unavailable.

#### `place_spread`
Places a multi-leg spread order (verticals, iron condors, butterflies, ratio spreads).

**Parameters:**
- `account_key` (required, string), `symbol` (required, string — underlying)
- `legs` (required, array of 2-4) — each: `option_type` (`call|put`), `strike` (number), `expiry` (`YYYY-MM-DD`), `side` (`buy|sell`), `quantity` (integer), `position_effect` (`opening|closing`, default `opening`)
- `net_price` (number — positive=credit, negative=debit), `net_price_type` (required: `credit|debit|even|market`)
- `placed_by` (string), `note` (string), `metadata` (object), `closing_strategy` (object), `idempotency_key` (string)
- `opened_at` (string) — ISO8601 timestamp to override position `opened_at` (for backtesting/replay). When omitted, uses server `NOW()`.
- `group_id` (string) — Optional strategy group ID. Links the spread and resulting positions to a group.

**Auto-enrichment:** On fill, AlphaDB auto-populates `entry_context` with market snapshot (dte, underlying_price, iv_rank, vix, vix_term_ratio, net_entry_value). On exit, `exit_context` captures exit_reason, hold_days, realized_pnl, dte_remaining. No client action needed — all data sourced server-side.

#### `close_spread`
Closes a filled spread by placing opposing legs at market price. Adds `closes_spread_id` to metadata. Returns an error if any leg's position is already closed — partial close would create naked exposure.

**Parameters:** `id` (required, string)

#### `get_spread`
Returns spread order with legs and fill details.

**Parameters:** `id` (required, string)

#### `list_spreads`
Lists spreads filtered by account, status, symbol, placed_by, or metadata.

**Parameters:** `account_key` (string), `status` (string: `pending|open|filled|partial|cancelled|rejected`), `symbol` (string), `placed_by` (string), `metadata` (object)

**Response includes `entry_context` and `exit_context`** (JSONB, auto-populated by AlphaDB):
- `entry_context`: `dte`, `net_entry_value`, `underlying_price`, `iv_rank` (0-1 fraction), `vix`, `vix_term_ratio` — captured at fill time.
- `exit_context`: `exit_reason`, `hold_days`, `realized_pnl`, `dte_remaining`, `exit_date`, `underlying_price`, `iv_rank`, `vix` — captured at exit time (replay or auto-exit).
- Fields are omitted when data is unavailable. Client-owned context (regime, strategy) is in `metadata`.

#### `sync_spread`
Syncs a live spread order with broker to retrieve latest fill status. No-op for paper.

**Parameters:** `id` (required, string)

### 8.5 Strategy Group Tools

Groups organize related orders, spreads, and positions into a single logical trade structure (e.g., an iron condor built from two vertical spreads). Orders and spreads can reference a `group_id` to join a group.

#### `create_group`
Creates a strategy group for organizing multi-leg/multi-spread trade structures.

**Parameters:**
- `account_key` (required, string) — Account key
- `name` (required, string) — Descriptive name, e.g. `"SPY iron condor Apr"`
- `symbol` (required, string) — Primary underlying ticker
- `closing_strategy` (object) — Group-level exit plan
- `metadata` (object) — User-controlled tags

#### `list_groups`
Lists strategy groups with optional filtering.

**Parameters:** `account_key` (string), `status` (string: `open|closed|partial|adjusting|rolling`), `symbol` (string), `limit` (integer), `offset` (integer)

#### `get_group`
Returns a strategy group with positions summary including open/closed counts and total realized P&L.

**Parameters:** `id` (required, string)

#### `close_group`
Closes all open positions in a group and marks the group as closed.

**Parameters:** `id` (required, string)

**Response:** `{group, closed_count, errors?}`

#### `adjust_group`
Adjusts a group by closing specified positions and/or adding new orders or spreads. All new legs are automatically assigned to the group.

**Parameters:**
- `id` (required, string) — Group ID
- `close_positions` (array of strings) — Position IDs to close
- `add` (array) — Spread orders to add (same schema as `place_spread`)
- `add_orders` (array) — Single-leg orders to add (same schema as `place_order`)
- `note` (string) — Reason for adjustment

At least one of `close_positions`, `add`, or `add_orders` must be provided.

**Response:** `{group, closed_count, closed_pnl, added_orders?, added_spreads?, errors?}`

### 8.6 Roll Tools

Roll an existing position or spread: atomically close the old and open a new one. Group ID is inherited from the closed position.

#### `execute_roll`
Executes a position roll — close one position/spread and open a replacement.

**Parameters:**
- `account_key` (required, string)
- `position_id` (string) — Position to close (single-leg roll)
- `spread_id` (string) — Spread to close (spread roll)
- `new_order` (object) — New single-leg order to open (same schema as `place_order`)
- `new_spread` (object) — New spread to open (same schema as `place_spread`)

Exactly one of `position_id`/`spread_id` and one of `new_order`/`new_spread` must be set.

**Response:** `{roll_order: {id, realized_pnl, net_debit_credit, status}, closed_pnl, new_position}`

#### `list_rolls`
Lists roll orders with optional filtering.

**Parameters:** `account_key` (string), `group_id` (string), `limit` (integer), `offset` (integer)

#### `get_roll`
Returns a roll order by ID with close/open details.

**Parameters:** `id` (required, string)

### 8.7 Contingent Order Tools (OTO/OCO)

Contingent links fire child orders when a parent fills or is cancelled. Auto-created from `closing_strategy` fields (`take_profit_price`, `stop_loss_price`) when an order fills — TP and SL are linked as OCO (when either fills, the other cancels).

#### `create_contingent`
Creates a contingent link manually.

**Parameters:**
- `account_key` (required, string)
- `trigger_type` (required, string) — `"on_fill"` or `"on_cancel"`
- `parent_order_id` (string) — Parent order (exactly one parent required)
- `parent_spread_id` (string) — Parent spread (exactly one parent required)
- `child_template` (required, object) — Order to place when triggered (same schema as `place_order`)
- `group_id` (string) — Optional strategy group

#### `list_contingents`
Lists contingent links with optional filtering.

**Parameters:** `account_key` (string), `status` (string: `pending|triggered|cancelled|expired`), `group_id` (string), `parent_order_id` (string)

#### `cancel_contingent`
Cancels a pending contingent link.

**Parameters:** `id` (required, string)

### 8.8 Structure Analysis Tools

Analyze the spread type and compute risk metrics for positions. Pure computation — detects vertical credit/debit, iron condor, single-leg, or custom structures.

#### `get_position_structure`
Returns structure analysis for a position's spread (or single-leg analysis if no spread).

**Parameters:** `id` (required, string) — Position ID

**Response:** `{structure_type, is_defined_risk, contracts, width, net_premium, max_gain, max_loss, breakevens, legs}`

Structure types: `vertical_credit`, `vertical_debit`, `iron_condor`, `covered_call`, `straddle`, `strangle`, `calendar`, `diagonal`, `single_leg`, `custom`.

#### `get_spread_structure`
Returns structure analysis for all positions in a spread.

**Parameters:** `id` (required, string) — Spread ID

#### `list_structures`
Returns structure analysis for all open positions in an account.

**Parameters:** `key` (required, string) — Account key

### 8.9 Collateral Tools

Compute margin requirements and buying power. Paper accounts use collateral-aware buying power for pre-trade validation.

#### `get_collateral`
Returns the collateral summary for an account: total reserved, buying power, and per-position/spread requirements.

**Parameters:** `key` (required, string) — Account key

**Response:** `{account_key, total_reserved, current_cash, buying_power, requirements: [{position_ids, spread_id?, symbol, requirement_type, reserved_amount, formula}]}`

Requirement types: `defined_risk` (MaxLoss), `naked_short` (max of 20% notional or premium), `covered_call` (0 — short call covered by stock), `none`.

#### `get_position_collateral`
Returns the collateral requirement for a single position or its spread.

**Parameters:** `id` (required, string) — Position ID

### 8.10 Expiration & Audit Tools

#### `resolve_expirations`
Resolves expired option positions. Paper/sim: OTM expires worthless, ITM triggers exercise/assignment with stock position creation. Live accounts: detection only.

**Parameters:**
- `key` (required, string) — Account key
- `as_of` (required, string) — Date to check expirations `YYYY-MM-DD`

**Response:** `{expired, worthless, assigned, details: [{position_id, symbol, action, cash_delta}], errors?}`

Actions: `expired_worthless`, `exercised` (ITM long), `assigned` (ITM short).

#### `get_audit_log`
Returns the immutable audit trail for an account.

**Parameters:**
- `key` (required, string) — Account key
- `event_type` (string) — Filter: `fill_applied`, `cash_adjustment`, `order_state_change`, `option_expired`, `option_assigned`, `group_created`, `group_closed`, `group_adjusted`, `roll_executed`
- `entity_type` (string) — Filter: `order`, `spread`, `position`, `account`
- `entity_id` (string) — Filter by specific entity
- `limit` (integer), `offset` (integer)

#### `reconcile_account`
Syncs all open orders/spreads with broker and resolves expired options.

**Parameters:** `key` (required, string) — Account key

### 8.11 Bracket Orders

#### `place_bracket`
Places an entry order with take-profit (limit) and stop-loss (stop) contingent orders in one call. TP and SL are automatically created as OTO contingent orders with OCO cancellation (filling one cancels the other).

**Parameters:** All `place_order` parameters plus:
- `take_profit_price` (number) — Limit price for the take-profit child order
- `stop_loss_price` (number) — Price for the stop-loss child order

At least one of `take_profit_price` or `stop_loss_price` is required.

**Response:** `{entry_order, fill?, tp_contingent_id?, sl_contingent_id?, warnings?}`

**Behavior:** Internally calls `place_order` with the closing strategy injected, then reads the auto-created contingent links. Paper accounts fill the entry immediately; contingent orders trigger on fill.

### 8.12 Portfolio Analytics

#### `get_account_greeks`
Returns aggregated Greeks (delta, gamma, theta, vega) across all open option positions for a trading account. Uses Black-Scholes with ATM IV history — works 24/7 without live quotes.

**Parameters:** `account_key` (required, string)

**Response:**
```json
{
  "account_key": "paper",
  "total_delta": -45.2,
  "total_gamma": 1.8,
  "total_theta": -12.5,
  "total_vega": 85.3,
  "positions": [
    {
      "position_id": "...", "symbol": "AAPL", "instrument_type": "put",
      "strike": 180, "expiry": "2026-05-16", "side": "short", "quantity": 2,
      "delta": -0.35, "gamma": 0.02, "theta": -0.05, "vega": 0.30, "iv": 0.28,
      "adj_delta": 70.0, "adj_gamma": -4.0, "adj_theta": 10.0, "adj_vega": -60.0
    }
  ],
  "errors": []
}
```

Note: Adjusted values = raw × quantity × multiplier × side_sign (short negates delta/gamma/vega).

**Difference from `portfolio_greeks` (§4):** The analytics tool operates on any symbols. This tool operates specifically on OMS trading positions for a given account.

#### `position_alerts`
Returns alerts for positions approaching their closing strategy thresholds. Use to monitor which positions need attention.

**Parameters:** `account_key` (required, string)

**Response:**
```json
{
  "account_key": "paper",
  "evaluated_at": "2026-04-02T14:30:00-04:00",
  "evaluated": 12,
  "alerts": [
    {
      "position_id": "...", "symbol": "AAPL",
      "alert_type": "approaching_stop_loss",
      "current_value": 147.5, "threshold_value": 145.0,
      "message": "AAPL at 147.50, stop loss at 145.00 (1.7% away)"
    }
  ]
}
```

Alert types: `approaching_stop_loss` (within 5%), `approaching_take_profit` (within 5%), `approaching_max_hold` (within 2 days), `approaching_dte_exit` (within 2 days), `approaching_eod_exit` (within 15 minutes).

#### `risk_dashboard`
Returns cross-account risk dashboard showing total exposure, Greeks, collateral, and unrealized P&L across all active trading accounts.

**Parameters:** None.

**Response:**
```json
{
  "accounts": [
    {
      "account_key": "paper", "provider": "paper",
      "open_positions": 8, "unrealized_pnl": 1250.50,
      "cash_balance": 95000, "collateral_reserved": 3500,
      "buying_power": 91500,
      "total_delta": -45.2, "total_gamma": 1.8,
      "total_theta": -12.5, "total_vega": 85.3
    }
  ],
  "cross_account_delta": -45.2,
  "total_collateral": 3500,
  "total_buying_power": 91500,
  "total_unrealized_pnl": 1250.50
}
```

Greeks are included when a GreeksCalculator is configured; otherwise they default to 0. Accounts that fail to load positions include an `error` field instead of zeroing out.

---

## §9 Portfolio

Lightweight agent portfolio tracking. These tools persist position notes across sessions using the AlphaDB `positions` table. Unlike OMS Trading positions (§8.3) which are auto-derived from fills, Portfolio positions are manually recorded by the agent.

**Use case:** An agent records a position after placing an order, then reviews open positions at the start of each session to assess current exposure.

#### `create_position`
Records a new position (stock or option leg).

**Parameters:**
- `symbol` (required, string) — Underlying ticker
- `quantity` (required, integer) — Shares or contracts; negative = short
- `cost_basis` (required, number) — Entry price per share (stocks) or per contract (options, pre-multiplier)
- `instrument_type` (string) — `"stock"` (default), `"call"`, `"put"`
- `strike` (number) — Strike price (options only)
- `expiry` (string) — Expiration `YYYY-MM-DD` (options only)
- `open_date` (string) — Trade date `YYYY-MM-DD`, default today
- `note` (string) — Trade rationale

**Response:** `{id, symbol, quantity, cost_basis, instrument_type, open_date, status}`

#### `list_positions`
Lists recorded positions. Filter by status to see open or closed positions.

**Parameters:** `status` (string: `"open"`, `"closed"`, or omit for all)

**Response fields:** `id`, `symbol`, `quantity`, `cost_basis`, `instrument_type`, `strike`, `expiry`, `open_date`, `close_date`, `close_price`, `dollar_value` (open only), `realized_pnl` (closed only)

#### `record_position_close`
Records exit price and date for a portfolio position and calculates realized P&L. Does **not** place a broker order — use `close_position` (§8.3) for that.

**Parameters:**
- `id` (required, string) — Position UUID from `list_positions`
- `close_price` (required, number) — Exit price per share or per contract
- `close_date` (string) — Exit date `YYYY-MM-DD`, default today

---

## §10 Agentic Analytics

**Not yet implemented — all tools return an error.** These tools are reserved for a future agentic analytics platform where agents can create custom insight jobs at runtime (e.g., "correlate AAPL with SPY over 90 days", "compute rolling IV percentile for tech sector ETFs"). Do not attempt to call these tools.

#### `create_insight_job`
Creates a custom analytics job asynchronously.

**Parameters:** `job_type` (required, string), `symbols` (array), `parameters` (object), `description` (string)

#### `get_job_status`
Checks status of an agentic analytics job.

**Parameters:** `job_id` (required, string)

#### `get_insight_result`
Retrieves output of a completed job.

**Parameters:** `job_id` (required, string)
