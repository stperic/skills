# AlphaDB HTTP API

AlphaDB is an **MCP server** that gives LLM trading agents a unified interface to market data. This document covers the HTTP API.

## Architecture

External clients access data through these paths:

| Path | Description | Source |
|------|-------------|--------|
| **Price Data** (`/v1/bars`, `/v1/quotes`, `/v1/fred`) | Bars, quotes, FRED series — provider-agnostic, normalized | Polygon, Schwab, tastytrade, FRED |
| **Options** (`/v1/options/*`) | Options chain and expirations | Polygon, Schwab, tastytrade |
| **Fundamentals** (`/v1/fundamentals/*`) | Earnings, dividends, company overview, insiders, transcripts, indicators | Alpha Vantage |
| **Market** (`/v1/market/*`) | Market hours, movers, status, forex, metrics, indices | Schwab, tastytrade |
| **Context** (`/v1/context/*`) | Composite symbol and market context (multi-source) | Multiple providers |
| **Search** (`/v1/search/*`) | News, ticker, and general web search | SearXNG |
| **Computed Analytics** (`/v1/alpha/*`) | Derived metrics computed from the database | AlphaDB DB |
| **Streaming (SSE)** (`/v1/stream/*`) | Real-time quotes, trades, Greeks via Server-Sent Events | tastytrade/DXLink or Schwab WebSocket (configurable via `REALTIME_PROVIDER`) |
| **Trading (OMS)** (`/v1/trading/*`) | Order management for paper and live Schwab trading | OMS DB |
| **Admin** (`/v1/admin/*`, mutations) | Symbol tracking, sync, jobs, provider management | AlphaDB internal |
| **Provider Proxy** (`/v1/proxy/*`) | Pass-through to upstream APIs with caching + rate limiting | Polygon, FRED, tastytrade, Schwab |

## Authentication

Process-scoped mutation key plus optional read/OMS tiers, all passed via `X-API-Key` (or `Authorization: Bearer`):

| Key | Env Var | Where set | Gates |
|---|---|---|---|
| **Live Mutation Key** | `ALPHA_LIVE_KEY` | live process only | every mutation on `:8080` (symbols, sync jobs, rates, trading writes against live accounts, …) |
| **Replay Mutation Key** | `ALPHA_REPLAY_KEY` | replay process only | every mutation on `:8081` (`as-of-date` set/clear, trading writes against replay accounts, simulate, replay-day, …) |
| **OMS Key** | `ALPHA_OMS_KEY` | both | Schwab live trading (real-money gate, kept distinct from the mutation key) |
| **API Key** | `ALPHA_API_KEY` | both | read-only `/v1/*` routes; optional (empty = unauthenticated reads allowed) |

**Pick the mutation key by process, not by endpoint.** Live writes go to `:8080` and use `ALPHA_LIVE_KEY`; replay writes go to `:8081` and use `ALPHA_REPLAY_KEY`. A wrong-key call returns `401`; a right-key call to the wrong port returns either `400 WRONG_ROLE` (route exists, account_key in wrong domain) or `404` (route physically absent — only as-of-date hits this path).

### Route auth summary

| Routes | Auth | Methods |
|--------|------|---------|
| `GET /v1/bars`, `GET /v1/quotes`, `GET /v1/fred`, `GET /v1/options/*`, `GET /v1/fundamentals/*` | API Key (or public) | GET |
| `GET /v1/market/*`, `GET /v1/context/*`, `GET /v1/search/*`, `GET /v1/alpha/*`, `GET /v1/proxy/*` | API Key (or public) | GET |
| `GET /v1/stream/*`, `GET /v1/trading/*` | API Key (or public) | GET |
| `GET /v1/schema`, `POST /v1/query` | API Key (or public) | GET, POST |
| `POST/DELETE /v1/symbols/*`, `POST/DELETE /v1/track/batch` | **Live Mutation Key** (`:8080`) | POST, PUT, DELETE |
| `POST/DELETE /v1/jobs/*` | **Live Mutation Key** (`:8080`) | DELETE, POST |
| `POST /v1/rates/*`, `POST/DELETE /v1/alpha/enable|disable` | **Live Mutation Key** (`:8080`) | POST, DELETE |
| `GET/POST /v1/admin/sync/*`, `GET/POST /v1/admin/provider-periods/*` | **Live Mutation Key** (`:8080`) | all |
| `POST/DELETE /v1/admin/as-of-date` (replay process only — `:8081`) | **Replay Mutation Key** | POST, DELETE |
| `GET /v1/admin/as-of-date` (replay process only — `:8081`) | **Replay Mutation Key** or **API Key** | GET |
| `POST /v1/trading/orders`, `POST /v1/trading/spreads`, etc. | Role's mutation key (route by `account_key`) | POST |

---

## Role routing (two-process deployment)

AlphaDB runs as **two processes** on the same host (see `architecture.md` and `oms_replay.md`). The role guard gates writes by **`account_key`**, not by endpoint — every write endpoint (`place_order`, `place_spread`, `close_spread`, `adjust_cash`, `roll_position`, …) exists on both ports.

| Port | Role | Writes allowed for accounts |
|---|---|---|
| `:8080` | `live` | listed in `ALPHADB_LIVE_ACCOUNT_KEYS` (typically `paper`, `schwab-sim`) |
| `:8081` | `replay` | every other account (replay/sim, e.g. `alphaseeker-sim`) |

**Routing rule:** pick the port by the `account_key` in the request. Sending a write for a non-matching account returns `400 WRONG_ROLE` naming the account and the role that owns it.

```
POST /v1/trading/orders         {account_key: "paper", ...}            → :8080  ✓
POST /v1/trading/orders         {account_key: "paper", ...}            → :8081  ✗ WRONG_ROLE
POST /v1/trading/orders         {account_key: "alphaseeker-sim", ...}  → :8081  ✓
POST /v1/trading/spreads        {account_key: "schwab-sim", ...}       → :8080  ✓
POST /v1/trading/close          {account_key: "alphaseeker-sim", ...}  → :8081  ✓
```

**`replay-day` and `simulate-outcome`** exist on both ports but follow the same per-`account_key` rule — they only succeed for accounts in the process's role domain. Call them on the port that owns the account.

**`/v1/admin/as-of-date`** (GET/POST/DELETE) is **physically absent on the live process** — the route group is only registered when `cfg.Role == "replay"`. Calls to `:8080` return `404`, not `400 WRONG_ROLE`. These set the process-global `asOfBoundary` and are the whole reason the replay process exists; route them to `:8081`.

Reads other than as-of-date are not role-gated and work on either port; prefer the port matching the workload to avoid mixing.

## Data Routing

Data source is determined automatically by `as_of`:

| Request | Behavior |
|---------|----------|
| No `as_of` | **Live** — fetches from upstream provider (Schwab, Alpha Vantage, tastytrade, FRED) |
| `as_of=YYYY-MM-DD` | **Historical** — reads from local database (stored bars, options, analytics) |

In **as-of replay mode** (boundary active), `as_of` is required on all date-aware endpoints and must not exceed the boundary date. All data comes from the database — no upstream provider calls.

## Parameter Conventions

Endpoints use either `symbol` (singular) or `symbols` (plural) — the parameter name indicates the response shape:

| Parameter | Accepts | Response Shape |
|-----------|---------|----------------|
| `symbol=AAPL` | Single ticker | Flat entity — fields at top level |
| `symbols=AAPL` | Single ticker (comma-separated) | Keyed map — `{"data": {"AAPL": {...}}}` |
| `symbols=AAPL,MSFT,SPY` | Multiple tickers | Keyed map — `{"data": {"AAPL": {...}, "MSFT": {...}, "SPY": {...}}}` |

**Single-symbol endpoints** (`symbol`): `bars`, `symbol_context`, `search_ticker`, `insider_transactions`, `earnings_transcript`, `technical_indicator`.

**Multi-symbol endpoints** (`symbols`): `quotes`, `earnings`, `dividends`, `company_overview`, `market_metrics`, `stream/*`, `correlation`, `unusual_volume`.

**Analytics endpoints** accept **both** `symbol` and `symbols` — the parameter name controls the response shape. `symbol=AAPL` returns a flat entity; `symbols=AAPL,MSFT` returns a keyed map with per-symbol results. Analytics tools: `iv_metrics`, `iv_term_structure`, `iv_rv_spread`, `skew`, `earnings_implied_move`, `sentiment`, `volatility`, `beta`.

The `underlying` parameter is used only for options-specific endpoints (`/v1/options/chain`, `/v1/options/expirations`) where the ticker is an underlying, not a direct symbol.

## Response Conventions

All endpoints wrap successful responses in a `{"data": <payload>}` envelope. Non-OMS endpoints also include `"meta": {"request_id", "count", "timestamp"}`. Errors use `{"error": {"message": "..."}}`.

**Empty lists are always `[]`, never `null` or omitted.** List endpoints (`list_orders`, `list_trading_positions`, `list_spreads`, `list_accounts`, etc.) guarantee an empty JSON array when no results match — clients can rely on `data` being an array without null-checking.

**202 Accepted** responses include `job_id` for async backfill jobs.

---

## 1. Provider Proxies

Pass-through to upstream APIs with Redis caching and rate limiting. Auth is injected server-side — clients never need API keys.

### Polygon Proxy

```
GET /v1/proxy/polygon/*path
```

Forwards any Polygon API path. The `*path` is appended directly to the Polygon base URL.

```bash
# Stock bars
curl "http://localhost:8080/v1/proxy/polygon/v2/aggs/ticker/AAPL/range/1/day/2025-01-01/2025-12-31"

# Options chain snapshot
curl "http://localhost:8080/v1/proxy/polygon/v3/snapshot/options/AAPL"

# Ticker snapshot (current price)
curl "http://localhost:8080/v1/proxy/polygon/v2/snapshot/locale/us/markets/stocks/tickers/AAPL"

# Ticker details
curl "http://localhost:8080/v1/proxy/polygon/v3/reference/tickers/AAPL"

# Options expirations
curl "http://localhost:8080/v1/proxy/polygon/v3/reference/options/contracts?underlying_ticker=AAPL"
```

### Alpha Vantage Proxy

```
GET /v1/alphavantage/query?function={function}&...
```

Forwards all query parameters to Alpha Vantage. The `function` parameter is required. API key is injected server-side. Rate limited at 75 req/min (Premium plan).

This is a **raw pass-through** to the full Alpha Vantage API — any function listed in the [Alpha Vantage documentation](https://www.alphavantage.co/documentation/) can be called. This includes 100+ functions across stocks, options, fundamentals, forex, crypto, commodities, economic indicators, and technical indicators that are not covered by the dedicated `/v1/fundamentals/*` endpoints.

**Dedicated endpoints** (`/v1/fundamentals/earnings`, `/v1/fundamentals/dividends`, etc.) are preferred when available — they add batch support, `as_of`, DB-first caching, and normalized responses. Use this pass-through for functions without a dedicated endpoint (e.g., `INCOME_STATEMENT`, `BALANCE_SHEET`, `CASH_FLOW`, `HISTORICAL_OPTIONS`, `REAL_GDP`, `CPI`, etc.).

```bash
# Earnings (prefer /v1/fundamentals/earnings for batch + as_of support)
curl "http://localhost:8080/v1/alphavantage/query?function=EARNINGS&symbol=AAPL"

# Company overview
curl "http://localhost:8080/v1/alphavantage/query?function=OVERVIEW&symbol=AAPL"

# Income statement (no dedicated endpoint — use pass-through)
curl "http://localhost:8080/v1/alphavantage/query?function=INCOME_STATEMENT&symbol=AAPL"

# Balance sheet
curl "http://localhost:8080/v1/alphavantage/query?function=BALANCE_SHEET&symbol=AAPL"

# Historical options
curl "http://localhost:8080/v1/alphavantage/query?function=HISTORICAL_OPTIONS&symbol=AAPL&date=2025-06-15"

# Technical indicator
curl "http://localhost:8080/v1/alphavantage/query?function=RSI&symbol=AAPL&interval=daily&time_period=14&series_type=close"

# Economic indicators
curl "http://localhost:8080/v1/alphavantage/query?function=REAL_GDP"
curl "http://localhost:8080/v1/alphavantage/query?function=CPI"
curl "http://localhost:8080/v1/alphavantage/query?function=TREASURY_YIELD&maturity=10year"
```

### FRED Proxy

```
GET /v1/proxy/fred/*path
```

Forwards any FRED API path. The `*path` is appended to the FRED base URL (`https://api.stlouisfed.org/fred`).

```bash
# Treasury yields
curl "http://localhost:8080/v1/proxy/fred/series/observations?series_id=DGS10"

# VIX
curl "http://localhost:8080/v1/proxy/fred/series/observations?series_id=VIXCLS"

# Fed funds rate
curl "http://localhost:8080/v1/proxy/fred/series/observations?series_id=DFF&observation_start=2026-01-01"
```

### tastytrade Proxy

```
GET /v1/proxy/tastytrade/*path
```

Forwards any tastytrade REST API path. Auth uses Bearer token (OAuth2 refresh_token grant, auto-refresh).

```bash
# Market metrics (IV rank, IV percentile, liquidity — tastytrade-only data)
curl "http://localhost:8080/v1/proxy/tastytrade/market-metrics?symbols=AAPL,SPY,TSLA"

# Equity quotes (bid/ask, last, prev-close, volume)
curl "http://localhost:8080/v1/proxy/tastytrade/market-data/by-type?equity=AAPL&equity=SPY"

# Options chain (nested by expiration)
curl "http://localhost:8080/v1/proxy/tastytrade/option-chains/AAPL/nested"
```

**Note:** tastytrade returns all numeric values as strings. The MCP tools parse these automatically.

### Provider Capabilities

| Provider | Data Types | Rate Limits |
|----------|------------|-------------|
| **Polygon.io** (Massive plan) | Historical bars, options chains with Greeks, snapshots, reference data | Unlimited API calls, 10 concurrent |
| **tastytrade** (REST) | Equity quotes, options chains (no Greeks), market metrics (IV rank, IV percentile, liquidity) | OAuth2 Bearer token |
| **tastytrade** (WebSocket) | Real-time streaming quotes, trades, options Greeks via DXLink protocol | OAuth2 Bearer token |
| **Schwab** (REST) | Equity quotes, options chains with full Greeks (delta/gamma/theta/vega/rho), historical bars — proper float64 types | OAuth2 via Cloudflare Worker bot |
| **Schwab** (WebSocket) | Real-time streaming quotes, trades, options Greeks | OAuth2 via Cloudflare Worker bot |
| **Alpha Vantage** | Earnings, dividends, fundamentals, insider transactions, technical indicators, transcripts | 75/min (Premium) |
| **FRED** | Treasury rates, market indices, macro indicators | Unlimited |

AlphaDB handles rate limiting internally. If you receive 429 errors, reduce request frequency.

### Caching

All proxy responses are cached in Redis with configurable TTL (default: 1 minute). On Redis failure, requests fall through to upstream. Cache key: `proxy:{provider}:{hash(path+params)}`.

---

## 2. Market Data Endpoints

Provider-agnostic endpoints that normalize responses across Polygon, tastytrade, and Schwab. Use these instead of the Polygon proxy paths — they return consistent field names regardless of `QUOTE_PROVIDER` and switch providers transparently when `QUOTE_PROVIDER` changes.

### Bars

```
GET /v1/bars?symbol={symbol}&timeframe={timeframe}&start={date}&end={date}&limit={n}
```

| Param | Required | Default |
|-------|----------|---------|
| `symbol` | yes | — |
| `timeframe` | yes | `1m`, `5m`, `15m`, `1h`, `1d` |
| `start` | no | 30 trading days ago |
| `end` | no | today |
| `limit` | no | 500 (max 50000 for tracked symbols from DB, 5000 for untracked via proxy) |

```bash
curl "http://localhost:8080/v1/bars?symbol=SPY&timeframe=1d&start=2026-01-01"
```

```json
{
  "symbol": "SPY",
  "bars": [
    {"t": "2026-01-02T00:00:00Z", "o": 580.10, "h": 584.20, "l": 578.50, "c": 582.30, "v": 68000000, "vw": 581.40, "n": 720000},
    {"t": "2026-01-03T00:00:00Z", "o": 582.30, "h": 585.00, "l": 579.80, "c": 581.50, "v": 55000000, "vw": 582.00, "n": 640000}
  ],
  "count": 2,
  "provider": "polygon"
}
```

Bar fields use compact names: `t` (RFC3339 timestamp), `o` (open), `h` (high), `l` (low), `c` (close), `v` (volume). Optional: `vw` (VWAP, present when available), `n` (trade count, present when non-zero). Index symbols (`$VIX`, `$SPX`, etc.) return OHLC only — no `v`, `vw`, or `n`. `provider` reflects the active `QUOTE_PROVIDER` — no path changes needed when switching providers. When served from local DB (via `as_of`), includes `"timeframe"` field. `bars` uses Polygon (default) or Schwab; tastytrade has no historical bars endpoint.

### Quotes

```
GET /v1/quotes?symbols={SYMBOLS}&as_of={TIMESTAMP}
```

Up to 50 comma-separated symbols. Optional `as_of` (RFC3339 or YYYY-MM-DD) for point-in-time historical quotes from DB.

```bash
curl "http://localhost:8080/v1/quotes?symbols=SPY,AAPL"
```

```json
{
  "data": {
    "SPY":  {"price": 581.50, "change": -0.80, "change_pct": -0.14, "volume": 55000000, "prev_close": 582.30, "high": 585.00, "low": 579.80},
    "AAPL": {"price": 228.40, "change":  1.20, "change_pct":  0.53, "volume": 42000000, "prev_close": 227.20, "high": 229.50, "low": 227.00}
  },
  "as_of": "2026-02-14T14:30:00Z",
  "provider": "polygon"
}
```

### Options Chain

```
GET /v1/options/chain?underlying={SYMBOL}&expiry={YYYY-MM-DD}&type={call|put|all}&min_strike={N}&max_strike={N}&as_of={TIMESTAMP}
```

Optional `as_of` (RFC3339 or YYYY-MM-DD) for point-in-time historical chain from DB.

```bash
curl "http://localhost:8080/v1/options/chain?underlying=SPY&expiry=2026-03-20&type=put"
```

```json
{
  "underlying": "SPY",
  "contracts": [
    {
      "strike": 580.0, "expiry": "2026-03-20", "type": "put",
      "bid": 5.20, "ask": 5.40, "last": 5.30,
      "volume": 12500, "oi": 85000, "iv": 0.18,
      "delta": -0.45, "gamma": 0.012, "theta": -0.08, "vega": 0.15
    }
  ],
  "count": 150,
  "provider": "polygon"
}
```

Up to 13 normalized fields per contract: `strike`, `expiry`, `type`, `bid`, `ask`, `last`, `volume`, `oi`, `iv`, `delta`, `gamma`, `theta`, `vega`. Greeks are from provider-supplied values (Polygon, Schwab). tastytrade chain responses have `delta`/`gamma`/`theta`/`vega` as `null` — tastytrade does not supply Greeks via REST. **Historical responses** (via `as_of`) omit fields with no data: `bid`, `ask`, and `oi` are absent for bars from Polygon flat files (OHLCV only); Greeks fields are absent when IV could not be solved.

### Options Expirations

```
GET /v1/options/expirations?underlying={SYMBOL}&as_of={TIMESTAMP}
```

| Param | Required | Default |
|-------|----------|---------|
| `underlying` | yes | -- |
| `as_of` | no | -- (RFC3339 or YYYY-MM-DD for point-in-time historical expirations from DB) |

```bash
curl "http://localhost:8080/v1/options/expirations?underlying=SPY"
curl "http://localhost:8080/v1/options/expirations?underlying=SPY&as_of=2025-06-15"
```

```json
{
  "underlying": "SPY",
  "expirations": ["2026-02-21", "2026-02-28", "2026-03-20", "2026-06-19"],
  "count": 4,
  "provider": "polygon"
}
```

All dates are ISO strings. Expired expirations are excluded.

### Earnings

```
GET /v1/fundamentals/earnings?symbols={SYMBOLS}
```

Accepts 1 to 50 comma-separated symbols. Returns next earnings date with days_until/timing plus last 4 quarters of EPS data. DB first for tracked symbols, Alpha Vantage fallback for untracked. Optional `as_of` (RFC3339 or YYYY-MM-DD) filters quarters to `reportedDate <= as_of` and adjusts next-earnings logic.

```bash
curl "http://localhost:8080/v1/fundamentals/earnings?symbols=AAPL"
curl "http://localhost:8080/v1/fundamentals/earnings?symbols=AAPL,MSFT,GOOGL"
curl "http://localhost:8080/v1/fundamentals/earnings?symbols=AAPL&as_of=2025-06-15"
```

**Response:** `{"data": {"AAPL": {"next_date": "2026-04-30", "days_until": 70, "timing": "AMC", "quarters": [...], "source": "db"}, ...}}` — per-symbol map. DB-backed symbols have `"source": "db"`, AV fallback has `"source": "alphavantage"`. Unavailable symbols get `{"error": "not tracked"}`.

### Dividends

```
GET /v1/fundamentals/dividends?symbols={SYMBOLS}
```

Accepts 1 to 50 comma-separated symbols. Returns next ex-dividend date, days until, amount, yield, and frequency. DB first for tracked symbols, Alpha Vantage fallback for untracked. Optional `as_of` for historical point-in-time view.

```bash
curl "http://localhost:8080/v1/fundamentals/dividends?symbols=AAPL"
curl "http://localhost:8080/v1/fundamentals/dividends?symbols=AAPL,MSFT,JNJ"
curl "http://localhost:8080/v1/fundamentals/dividends?symbols=AAPL&as_of=2025-06-15"
```

**Response:** `{"data": {"AAPL": {"next_ex_date": "2026-05-09", "days_until": 30, "amount": 0.25, "yield_pct": 0.64, "frequency": "quarterly", "source": "db"}, ...}}` — per-symbol map. Unavailable symbols get `{"error": "not tracked"}`.

### Company Overview

```
GET /v1/fundamentals/company?symbols={SYMBOLS}
```

Returns 10 company fundamentals fields per symbol: `symbol`, `name`, `sector`, `industry`, `market_cap`, `pe_ratio`, `eps`, `revenue`, `revenue_growth_pct`, `description`. Accepts 1–50 comma-separated symbols.

**DB-first with AV fallback.** Tracked symbols are synced nightly from Alpha Vantage and served from the `company_overview` table (30-day refresh cadence). Untracked symbols fall back to a live AV API call. The `source` field indicates `"db"` or `"alphavantage"`.

```bash
curl "http://localhost:8080/v1/fundamentals/company?symbols=AAPL,MSFT,JNJ"
```

**Response:** `{"data": {"AAPL": {"symbol": "AAPL", "name": "Apple Inc", "sector": "TECHNOLOGY", "industry": "CONSUMER ELECTRONICS", "market_cap": 3784127807000, "pe_ratio": 32.59, "eps": 7.9, "revenue": 435617006000, "revenue_growth_pct": 0.157, "description": "...", "source": "db"}, ...}}` — per-symbol map. Unavailable symbols get `{"error": "not available"}`.

### Insider Transactions

```
GET /v1/fundamentals/insiders?symbol={SYMBOL}&days={N}
```

Recent insider buys and sells. Default lookback: 90 days.

```bash
curl "http://localhost:8080/v1/fundamentals/insiders?symbol=AAPL&days=90"
```

### Earnings Transcript

```
GET /v1/fundamentals/transcript?symbol={SYMBOL}&quarter={QUARTER}
```

Earnings call transcript text. Quarter format: `2026Q1`. Omit quarter for latest.

```bash
curl "http://localhost:8080/v1/fundamentals/transcript?symbol=AAPL&quarter=2026Q1"
```

### Market Metrics

```
GET /v1/market/metrics?symbols={SYMBOLS}
```

tastytrade IV rank, IV percentile, and liquidity ratings. Always uses tastytrade regardless of `QUOTE_PROVIDER`.

```bash
curl "http://localhost:8080/v1/market/metrics?symbols=AAPL,SPY"
```

Returns the normalized tastytrade market-metrics payload. The raw proxy path (`/v1/proxy/tastytrade/market-metrics`) also works but returns string-typed numeric fields — tastytrade encodes all numbers as JSON strings.

### Market Hours

```
GET /v1/market/hours?markets={MARKET_TYPE}&date={YYYY-MM-DD}
```

Returns market hours, session schedules, and trading day status. Always includes `is_trading_day` and `is_early_close` at the top level.

**Sources:** Recent/future dates use Schwab API (full session hours). Historical dates (>7 days past) fall back to the `market_holidays` DB table via CalendarService. The `source` field indicates which was used.

| Param | Type | Required | Default |
|-------|------|----------|---------|
| `markets` | string | no | `equity` — one of: `equity`, `option`, `bond`, `future`, `forex` |
| `date` | string | no | Today (Eastern) — format: `YYYY-MM-DD`, any date (past or future) |

```bash
# Equity market hours for today
curl "http://localhost:8080/v1/market/hours"

# Options market hours for a specific date
curl "http://localhost:8080/v1/market/hours?markets=option&date=2026-02-20"

# Check if a historical date was a trading day
curl "http://localhost:8080/v1/market/hours?date=2025-12-25"
```

**Response** (trading day, Schwab source):

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
        "product": "EQ",
        "productName": "equity",
        "isOpen": true,
        "sessionHours": {
          "preMarket": [
            { "start": "2026-02-20T07:00:00-05:00", "end": "2026-02-20T09:30:00-05:00" }
          ],
          "regularMarket": [
            { "start": "2026-02-20T09:30:00-05:00", "end": "2026-02-20T16:00:00-05:00" }
          ],
          "postMarket": [
            { "start": "2026-02-20T16:00:00-05:00", "end": "2026-02-20T20:00:00-05:00" }
          ]
        }
      }
    }
  }
}
```

**Response** (holiday, calendar source):

```json
{
  "date": "2025-12-25",
  "is_trading_day": false,
  "is_early_close": false,
  "source": "calendar"
}
```

**Response** (early close, calendar source):

```json
{
  "date": "2025-11-28",
  "is_trading_day": true,
  "is_early_close": true,
  "early_close_time": "13:00",
  "source": "calendar",
  "sessionHours": {
    "preMarket": [
      { "start": "2025-11-28T07:00:00-05:00", "end": "2025-11-28T09:30:00-05:00" }
    ],
    "regularMarket": [
      { "start": "2025-11-28T09:30:00-05:00", "end": "2025-11-28T13:00:00-05:00" }
    ]
  }
}
```

Calendar-sourced trading days include a synthetic `sessionHours` block matching the Schwab response shape. `preMarket` (7:00–9:30 ET) is always present on trading days. `regularMarket` uses `early_close_time` when applicable (13:00 ET on early close days, 16:00 ET otherwise). `postMarket` (16:00–20:00 ET) is included on normal days but omitted on early close days (no after-hours session).

When the market is closed (weekends, holidays), `isOpen` is `false` and `sessionHours` is omitted. Inner key is always the product code (`EQ`, `OPT`, `BON`, `FUT`, `FOR`) — normalized by AlphaDB.

### Market Hours Range (Bulk)

```
GET /v1/market/hours/range?start={YYYY-MM-DD}&end={YYYY-MM-DD}
```

Returns trading day info for every date in the range (inclusive). Max range: 730 days. Always uses the calendar source (DB-backed).

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `start` | string | yes | Start date `YYYY-MM-DD` |
| `end` | string | yes | End date `YYYY-MM-DD` (must be >= start) |

```bash
# All trading days for 2025
curl "http://localhost:8080/v1/market/hours/range?start=2025-01-01&end=2025-12-31"
```

**Response:**

```json
{
  "start": "2025-01-01",
  "end": "2025-01-03",
  "count": 3,
  "source": "calendar",
  "days": [
    {
      "date": "2025-01-01",
      "is_trading_day": false,
      "is_early_close": false,
      "source": "calendar"
    },
    {
      "date": "2025-01-02",
      "is_trading_day": true,
      "is_early_close": false,
      "source": "calendar",
      "sessionHours": {
        "preMarket": [{ "start": "2025-01-02T07:00:00-05:00", "end": "2025-01-02T09:30:00-05:00" }],
        "regularMarket": [{ "start": "2025-01-02T09:30:00-05:00", "end": "2025-01-02T16:00:00-05:00" }],
        "postMarket": [{ "start": "2025-01-02T16:00:00-05:00", "end": "2025-01-02T20:00:00-05:00" }]
      }
    },
    {
      "date": "2025-01-03",
      "is_trading_day": true,
      "is_early_close": false,
      "source": "calendar",
      "sessionHours": { "..." }
    }
  ]
}
```

Each entry in `days` has the same shape as the single-date `/v1/market/hours` calendar response. Use this instead of looping day-by-day — one call replaces 365 sequential round-trips at replay startup.

---

## 3. Computed Analytics

Unified analytics endpoint serving all computed metrics. These are insights that AlphaDB derives from its database — data that upstream providers don't offer directly.

```
GET /v1/alpha/analytics?metric={name}&symbol={symbol}
GET /v1/alpha/analytics?metric={name}&symbols={symbols}   # Batch: comma-separated, up to 50
GET /v1/alpha/news?symbol={symbol}              # News sentiment (query param)
GET /v1/alpha/scan                               # Cross-symbol screener (see below)
GET /v1/alpha/status                             # Analytics tracking status (see Section 5)
```

When `?metric` is omitted, returns the registry of available metrics with their parameters.

**Batch support:** Analytics metrics accept both `symbol` (single, flat response) and `symbols` (comma-separated up to 50, keyed map response). `symbol=AAPL` returns a flat entity; `symbols=AAPL,MSFT` returns `{"data": {"AAPL": {...}, "MSFT": {...}}, "count": N}`. `symbols=AAPL` (single value via plural param) is normalized to a flat response. The server handles fan-out internally. IV metrics specifically use a single optimized DB query for batch.

All responses include:
- `market_closed: true|false` — whether the market was closed when data was computed

### Available Metrics

| Metric | Description | Required Params | Optional Params |
|--------|-------------|-----------------|-----------------|
| `iv` | IV rank, percentile, current IV, 52-week range | `symbol` or `symbols` | `start`, `end` |
| `volatility` | Historical vs implied volatility with premium analysis | `symbol` or `symbols` | `start`, `end`, `as_of` |
| `iv_rv_spread` | Implied vs realized volatility spread and ratio | `symbol` or `symbols` | |
| `skew` | 25-delta put/call volatility skew | `symbol` or `symbols` | |
| `iv_term_structure` | IV across expirations with contango/backwardation analysis | `symbol` or `symbols` | `min_dte`, `max_dte`, `as_of` |
| `earnings_implied_move` | Options-implied expected move around earnings vs historical | `symbol` or `symbols` | `as_of` (historical: returns pre-computed from DB) |
| `sentiment` | Sentiment dashboard: news + options signals with per-group freshness | `symbol` or `symbols` | `as_of` |
| `earnings` | Next earnings date, days until, timing (AMC/BMO) | `symbol` or `symbols` | |
| `dividends` | Next ex-dividend date, days until, yield | `symbol` or `symbols` | |
| `unusual_volume` | Relative volume (RVOL) for unusual activity detection | `symbol` or `symbols` | `threshold`, `min_dollar_volume` |
| `beta` | Stock beta relative to S&P 500 (SPY) — systematic risk measure | `symbol` or `symbols` | `days` (20-252, default 252), `as_of` |
| `correlation` | Pairwise correlation matrix between symbols (-1 to 1) | `symbols` | `days` (10-252, default 30) |
| `earnings_calendar` | Upcoming earnings across all tracked symbols | — (none) | `days_ahead` (1-90, default 7) |
| `put_call_ratio` | Put/call volume ratio for near-term options (0-45 DTE). Live Polygon data. | `symbol` or `symbols` | — |

### IV Metrics

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=iv&symbol=AAPL"
curl "http://localhost:8080/v1/alpha/analytics?metric=iv&symbols=AAPL,MSFT,SPY"  # batch
```

**Snapshot mode** (default — no `start`/`end`):
```json
{
  "data": {
    "symbol": "AAPL",
    "current_iv": 0.2834,
    "iv_rank": 45.5,
    "iv_percentile": 52.3,
    "iv_52w_high": 0.55,
    "iv_52w_low": 0.18,
    "as_of": "2026-02-12"
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

**Batch mode** (multiple symbols):
```json
{
  "data": {
    "AAPL": {"current_iv": 0.2834, "iv_rank": 45.5, "iv_percentile": 52.3, "iv_52w_high": 0.55, "iv_52w_low": 0.18, "as_of": "2026-02-12"},
    "MSFT": {"current_iv": 0.2100, "iv_rank": 32.1, "iv_percentile": 38.0, "iv_52w_high": 0.45, "iv_52w_low": 0.15, "as_of": "2026-02-12"},
    "SPY": {"current_iv": 0.1580, "iv_rank": 28.0, "iv_percentile": 30.5, "iv_52w_high": 0.35, "iv_52w_low": 0.10, "as_of": "2026-02-12"}
  },
  "count": 3,
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

**Time series mode** (with `start` and/or `end`):
```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=iv&symbol=AAPL&start=2025-01-01"
```

Returns daily IV data with `iv`, `iv_rank`, `iv_percentile`, `hv_10`, `hv_20`, `hv_30`, `iv_hv_spread`, `stock_close`.

### Volatility Analysis

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=volatility&symbol=SPY"
```

```json
{
  "data": {
    "symbol": "SPY",
    "as_of": "2026-02-04",
    "market_context": {
      "underlying_price": 689.17,
      "next_earnings_date": "2026-04-15",
      "days_to_earnings": 62
    },
    "volatility_profile": {
      "historical": {"hv_10": 0.089, "hv_20": 0.112, "hv_30": 0.099, "hv_percentile": 38.5},
      "implied": {"iv_30": 0.178, "iv_rank": 45.2, "iv_percentile": 52.1, "iv_high_52w": 0.45, "iv_low_52w": 0.12},
      "structure": {"skew_25d": 0.031, "term_slope": -0.014, "put_call_iv_ratio": 1.02}
    },
    "premium_analysis": {
      "iv_hv_spread": 0.079,
      "iv_premium_pct": 80.4,
      "recommendation": "sell_premium"
    }
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

Supports three modes:
- **Snapshot** (default): Latest volatility analysis
- **Historical snapshot** (`?as_of=2025-06-01`): Point-in-time analysis
- **Time series** (`?start=...&end=...`): Daily volatility data

### IV/RV Spread

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=iv_rv_spread&symbol=SPY"
```

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
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

### Skew

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=skew&symbol=SPY"
```

```json
{
  "data": {
    "skew_25d": 0.06,
    "put_25d_iv": 0.20,
    "call_25d_iv": 0.14,
    "atm_iv": 0.17,
    "skew_percentile": null,
    "expiry": "2026-03-21",
    "data_quality": {"price_source": "quote"},
    "_missing": ["skew_percentile: requires 52-week skew history"]
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

When `data_quality.price_source` is `"quote"`, per-strike IVs were computed from live Polygon options snapshot (market hours). When `"close"`, they came from DB close-price options chain (after hours or no live data). When per-strike IVs are unavailable entirely (no 25-delta match), `put_25d_iv`, `call_25d_iv`, and `expiry` return `null` with `_missing` notes.

### IV Term Structure

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=iv_term_structure&symbol=SPY&max_dte=60"
```

```json
{
  "data": {
    "structure": "backwardation",
    "near_term_iv": 0.25,
    "near_term_dte": 3,
    "far_term_iv": 0.22,
    "far_term_dte": 10,
    "expirations": [
      {"expiry": "2026-02-07", "dte": 3, "iv": 0.25},
      {"expiry": "2026-02-14", "dte": 10, "iv": 0.22}
    ]
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

`near_term_iv`/`far_term_iv` are convenience fields extracted from the nearest and furthest expirations.

**Historical term structure** — pass `as_of` for a point-in-time snapshot:

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=iv_term_structure&symbol=SPY&as_of=2025-03-03&min_dte=20&max_dte=90"
```

Returns the term structure as it existed on that date, using options data from the DB. Response includes `"as_of": "2025-03-03T00:00:00-05:00"` in the data object.

### Earnings Implied Move

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=earnings_implied_move&symbol=AAPL&history_n=8"
```

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
    "expiry_match": "nearest_before"
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

**Parameters:**
- `history_n` (optional, default `8`, clamped to `[1, 20]`) — number of historical quarters to include in the distribution.
- `as_of` (optional) — when set, switches to historical mode and returns the pre-computed implied move from `earnings_quarterly` for the most recent quarter at-or-before `as_of`. The historical distribution is built from quarters strictly prior to that anchor (no leakage).

**Historical fields:**
- `historical_move_mean_pct` — arithmetic mean of absolute realized moves over the last N quarters.
- `historical_move_median_pct` — median (50th percentile, type-7 interpolation). More robust than mean for skewed distributions.
- `historical_move_p75_pct` — 75th percentile.
- `historical_sample_size` — actual number of usable quarters (≤ `history_n`; quarters with no resolvable move are skipped).
- `implied_vs_historical_ratio` = `implied_move_pct / historical_move_median_pct` — **preferred filter** for earnings-crush strategies. Values `> 1.0` indicate the market is pricing more move than historically realized (positive EV for premium sellers).
- `implied_vs_actual_ratio` and `avg_historical_move_pct` are kept as back-compat aliases (mean-based). New clients should prefer the `_historical_` fields.

Only available in live mode when earnings are ≤ 90 days away.

### Sentiment

Returns a sentiment dashboard with two signal groups: **news** (article-derived QSE scores) and **options** (IV rank, skew, IV-HV spread, put/call IV ratio). Each group carries freshness metadata (`as_of_date`, `stale_days`, `data_quality`).

**Live mode** (no `as_of`): news fetched live from Alpha Vantage, options from live chain + DB. Falls back to DB carry-forward when AV returns 0 articles (common for ETFs).

**Historical mode** (`as_of`): both groups from DB only.

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=sentiment&symbol=AAPL"
curl "http://localhost:8080/v1/alpha/analytics?metric=sentiment&symbol=QQQ&as_of=2026-03-15"
```

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
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

**Freshness tiers:** `fresh` (0-1 trading days), `stale` (2-5), `very_stale` (6-30), `none` (>30 or no data — all value fields are `null`). For ETFs (QQQ, IWM, TLT), news is often stale/none while options signals are fresh — the options group provides the primary sentiment read.

**Partial success:** Returns 200 if at least one group has data. 404 only if both groups are empty.

### Earnings

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=earnings&symbol=AAPL"
```

```json
{
  "data": {
    "symbol": "AAPL",
    "report_date": "2026-04-24",
    "days_until": 68,
    "timing": "AMC",
    "eps_estimate": 1.62,
    "as_of": "2026-02-10"
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

`eps_estimate` is included when available. Returns 404 if no upcoming earnings found.

### Dividends

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=dividends&symbol=AAPL"
```

```json
{
  "data": {
    "symbol": "AAPL",
    "ex_dividend_date": "2026-02-07",
    "days_until": 3,
    "dividend_amount": 0.25,
    "yield_pct": 0.54,
    "as_of": "2026-02-10"
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

`yield_pct` is computed from the last 4 quarterly cash dividends divided by current price. Returns 404 if no dividend data found.

### Unusual Volume

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=unusual_volume&symbols=AAPL,NVDA,SPY"
```

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
  "filters_applied": {"threshold": 2.0, "min_dollar_volume": 5000000},
  "meta": {"timestamp": "..."}
}
```

### Beta

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=beta&symbol=AAPL"
# Historical: regression window ending on a specific date
curl "http://localhost:8080/v1/alpha/analytics?metric=beta&symbol=AAPL&as_of=2026-02-24"
```

```json
{
  "data": {
    "symbol": "AAPL",
    "beta": 1.25,
    "r_squared": 0.72,
    "benchmark": "SPY",
    "period_days": 252,
    "data_points": 252
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

Optional `?days=` parameter for shorter lookback (range: 20-252, default: 252). Beta > 1 = more volatile than the market; beta < 1 = less volatile.

### Correlation

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=correlation&symbols=AAPL,MSFT,NVDA"
```

```json
{
  "data": {
    "symbols": ["AAPL", "MSFT", "NVDA"],
    "period_days": 30,
    "data_points": 30,
    "matrix": {
      "AAPL": {"AAPL": 1.0, "MSFT": 0.82, "NVDA": 0.75},
      "MSFT": {"AAPL": 0.82, "MSFT": 1.0, "NVDA": 0.78},
      "NVDA": {"AAPL": 0.75, "MSFT": 0.78, "NVDA": 1.0}
    },
    "as_of": "2026-02-14"
  },
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

Accepts 2-20 comma-separated symbols. Optional `?days=` for lookback (range: 10-252, default: 30). Values near +1 = move together; near -1 = move opposite; near 0 = uncorrelated.

### Earnings Calendar

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=earnings_calendar&days_ahead=7"
```

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
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

No `symbol` required. Scans all tracked symbols with earnings data in the DB. Optional `?days_ahead=` (default 7, max 90).

### Put/Call Ratio

```bash
curl "http://localhost:8080/v1/alpha/analytics?metric=put_call_ratio&symbol=SPY"
```

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
  "meta": {"timestamp": "..."},
  "market_closed": false
}
```

Live options volume from Polygon snapshot. Covers near-term contracts (0-45 DTE). Ratio > 1.0 = more puts than calls (bearish hedging). Ratio < 0.7 = call-heavy (bullish speculation). `put_call_ratio` is `null` when no call volume is found (market closed or symbol has no near-term options).

### Portfolio Greeks

```
POST /v1/portfolio/greeks
```

Aggregated Greeks across option positions with optional what-if scenario analysis.

```bash
curl -X POST "http://localhost:8080/v1/portfolio/greeks" \
  -H "Content-Type: application/json" \
  -d '{
    "positions": [
      {"symbol": "AAPL", "option_symbol": "O:AAPL250321C00185000", "quantity": 10, "side": "long"},
      {"symbol": "AAPL", "option_symbol": "O:AAPL250321P00180000", "quantity": -5, "side": "short"}
    ]
  }'
```

With what-if (test adding a position without committing):
```bash
curl -X POST "http://localhost:8080/v1/portfolio/greeks" \
  -H "Content-Type: application/json" \
  -d '{
    "positions": [
      {"symbol": "AAPL", "option_symbol": "O:AAPL250321C00185000", "quantity": 10, "side": "long"}
    ],
    "include_what_if": {"option_symbol": "O:AAPL250321P00180000", "quantity": 5}
  }'
```

```json
{
  "data": {
    "as_of": "2026-02-14T15:30:00Z",
    "current_portfolio": {
      "net_delta": 350.5,
      "net_gamma": 12.4,
      "net_theta": -85.2,
      "net_vega": 142.0,
      "delta_dollars": 64000,
      "position_count": 2
    },
    "positions": [
      {
        "option_symbol": "O:AAPL250321C00185000",
        "symbol": "AAPL",
        "quantity": 10,
        "side": "long",
        "delta": 0.55,
        "gamma": 0.012,
        "theta": -0.08,
        "vega": 0.15
      }
    ],
    "warnings": []
  },
  "meta": {"timestamp": "..."}
}
```

This is a POST endpoint (not part of the analytics dispatch) because it requires a structured request body with position arrays.

### Cross-Symbol Screener

```
GET /v1/alpha/scan
```

Filters all tracked symbols by IV rank, days to earnings, RVOL, and IV/RV ratio. Returns matching symbols with their analytics snapshot. All filter params are optional.

| Param | Type | Default |
|-------|------|---------|
| `min_iv_rank` / `max_iv_rank` | float | — |
| `min_days_to_earnings` / `max_days_to_earnings` | int | — |
| `min_days_to_ex_div` | int | — (exclude symbols with ex-div within N days; non-payers pass) |
| `min_rvol` / `max_rvol` | float | — |
| `min_iv_rv_ratio` / `max_iv_rv_ratio` | float | — |
| `min_liquidity` | int | — (1–5 tastytrade liquidity rating; 3 = recommended minimum) |
| `sort_by` | string | — (`iv_rank`, `rvol`, `days_to_earnings`, `iv_rv_ratio`) |
| `order` | string | `desc` (`asc` or `desc`) |

```bash
# Premium-selling screen: high IV, not near earnings, liquid options, sorted by IV rank desc
curl "http://localhost:8080/v1/alpha/scan?min_iv_rank=70&min_days_to_earnings=14&min_iv_rv_ratio=1.2&min_liquidity=3&sort_by=iv_rank"

# Find symbols approaching earnings (for earnings plays)
curl "http://localhost:8080/v1/alpha/scan?max_days_to_earnings=7&sort_by=days_to_earnings&order=asc"

# Full universe snapshot (no filters — returns all tracked symbols with IV rank)
curl "http://localhost:8080/v1/alpha/scan"
```

```json
{
  "data": [
    {"symbol": "NVDA", "iv_rank": 81.0, "current_iv": 0.63, "days_to_earnings": 21, "earnings_timing": "AMC", "iv_rv_ratio": 1.55, "iv_rv_spread": 0.11},
    {"symbol": "TSLA", "iv_rank": 74.2, "current_iv": 0.58, "days_to_earnings": 34, "earnings_timing": "AMC", "iv_rv_ratio": 1.42, "iv_rv_spread": 0.09}
  ],
  "count": 2,
  "scanned": 18,
  "excluded_no_data": 3,
  "excluded": ["SMCI", "GME", "MSTR"],
  "filters": {"min_iv_rank": 70, "min_days_to_earnings": 14, "min_iv_rv_ratio": 1.2, "sort_by": "iv_rank", "order": "desc"},
  "as_of": "2026-02-14T14:30:00Z"
}
```

`excluded_no_data` is only present when > 0 — it counts symbols excluded because analytics data was not available (not yet backfilled), as distinct from symbols that failed the filter threshold. `excluded` lists the actual symbol names (same count as `excluded_no_data`). Metrics not fetched (driven by active filters) are absent from each symbol's result. When no filters are active, only `iv_rank` and `current_iv` are included per symbol.

---

## 4. Real-Time Streaming (SSE)

Server-Sent Events for real-time market data. These are long-lived HTTP connections — the server pushes updates as they arrive from the configured stream provider. Set `REALTIME_PROVIDER=tastytrade` (default, DXLink WebSocket) or `REALTIME_PROVIDER=schwab` (Schwab WebSocket, numeric field IDs mapped to normalized events). **Not available via MCP** — for AI agents, use the `quotes` MCP tool instead.

```
GET /v1/stream/quotes?symbols={SYMBOLS}    # Real-time bid/ask quotes
GET /v1/stream/trades?symbols={SYMBOLS}    # Real-time trades
GET /v1/stream/greeks?symbols={SYMBOLS}    # Real-time options Greeks
GET /v1/stream/all?symbols={SYMBOLS}       # All event types combined
```

```bash
# Stream quotes for multiple symbols
curl -N "http://localhost:8080/v1/stream/quotes?symbols=AAPL,SPY,NVDA"

# Stream options Greeks (use dot-prefix format for both providers)
curl -N "http://localhost:8080/v1/stream/greeks?symbols=.AAPL250321C00185000"

# Stream everything for a symbol
curl -N "http://localhost:8080/v1/stream/all?symbols=AAPL,.AAPL250321C00185000"
```

**Option symbol format:** Both tastytrade and Schwab streaming use dot-prefix notation (`.AAPL250321C00185000`). This is distinct from Polygon's `O:` prefix format used in REST proxy endpoints.

Response format is `text/event-stream`. On connect, a `connected` event fires with `client_id`. Subsequent events are normalized domain events regardless of provider:

```
event: connected
data: {"client_id":"abc123","symbols":["AAPL","SPY"]}

event: event
data: {"type":"quote","symbol":"AAPL","timestamp":"2026-02-14T15:30:01Z","data":{"bid_price":185.20,"ask_price":185.25,"bid_size":300,"ask_size":150,"mid_price":185.225,"spread":0.05}}

event: event
data: {"type":"trade","symbol":"AAPL","timestamp":"2026-02-14T15:30:01Z","data":{"price":185.22,"size":100,"day_volume":42000000}}

event: event
data: {"type":"greeks","symbol":".AAPL250321C00185000","timestamp":"2026-02-14T15:30:01Z","data":{"price":2.45,"iv":0.28,"delta":0.42,"gamma":0.018,"theta":-0.09,"vega":0.14,"rho":0.05}}
```

---

## 5. Symbol Tracking & Management (Live Mutation Key Required)

All mutation endpoints in this section live on `:8080` only and require the **Live Mutation Key** (`ALPHA_LIVE_KEY`). Read-only endpoints (GET) are public.

### Add Symbol

```
POST /v1/symbols                          # Live Mutation Key required
```

```json
{"symbol": "AAPL"}
```

Optional fields:

```json
{
  "symbol": "AAPL",
  "stock_interval": "1m",
  "options_interval": "1d",
  "analytics": ["iv", "sentiment", "dividends"]
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `symbol` | string | required | Ticker symbol |
| `stock_interval` | string | `"1m"` | `"1m"` or `"1d"` — stock bar granularity |
| `options_interval` | string | none | `"1m"` or `"1d"` — options bar granularity. Omit to disable. |
| `analytics` | array of strings | all | Enable only specific metrics |

Analytics that require a specific granularity override the user's choice (e.g., Greeks adds `1d` automatically).

Response (202 Accepted — new symbol):
```json
{
  "symbol": "AAPL",
  "analytics": ["iv", "volatility", "iv_rv_spread", "skew", "iv_term_structure",
                 "earnings_implied_move", "sentiment", "dividends", "unusual_volume",
                 "beta", "correlation"],
  "sync_status": "pending",
  "jobs_count": 4
}
```

Response (200 OK — already tracked):
```json
{
  "symbol": "AAPL",
  "analytics": ["iv", "volatility", "skew", "dividends"],
  "sync_status": "ready",
  "message": "symbol is already tracked; disable first to change analytics configuration"
}
```

### List Symbols

```
GET /v1/symbols                           # Public
GET /v1/symbols/:symbol                   # Public — single symbol detail
GET /v1/symbols/status                    # Public — data status across all symbols
```

### Delete Symbol

```
DELETE /v1/symbols/:symbol                # Live Mutation Key required
```

```bash
curl -X DELETE -H "Authorization: Bearer $ADMIN_KEY" http://localhost:8080/v1/symbols/AAPL
```

Optional query parameter: `?delete_data=true` to also remove stored data.

### Track Status

```
GET /v1/track                             # Public
GET /v1/track?summary=true                # Public — aggregate readiness counts
```

Optional query parameters:

| Param | Description |
|-------|-------------|
| `symbol` | Single symbol detail, e.g. `?symbol=AAPL` |
| `status` | Filter by metric status: `ready`, `pending`, or `disabled` |
| `metric` | Filter to a specific metric, e.g. `?metric=beta&status=pending` |
| `summary` | `true` for aggregate counts only |

**Full response:**
```json
{
  "data": [
    {
      "symbol": "AAPL",
      "sync_status": "ready",
      "analytics": [
        {"metric": "iv",         "status": "ready"},
        {"metric": "volatility", "status": "ready"},
        {"metric": "sentiment",  "status": "pending"},
        {"metric": "beta",       "status": "pending", "note": "requires 252 trading days of price history; 45 days available"},
        {"metric": "dividends",  "status": "ready"}
      ]
    }
  ],
  "meta": {"request_id": "...", "count": 1, "timestamp": "..."}
}
```

**Summary response** (`?summary=true`):
```json
{
  "data": {
    "total": 48,
    "ready": 42,
    "pending": 5,
    "failed": 1
  }
}
```

Metric status values:
- `ready` — data is available and analytics are computed
- `pending` — sync jobs are still running; `note` field explains what's needed
- `disabled` — feature not enabled for this symbol

### Batch Track / Untrack

```
POST   /v1/track/batch                    # Live Mutation Key required
DELETE /v1/track/batch                    # Live Mutation Key required
```

**Batch track body:**
```json
{"symbols": ["AAPL", "NVDA", "TSLA", "SPY", "QQQ"]}
```

Optional interval parameters:

```json
{
  "symbols": ["AAPL", "NVDA"],
  "stock_interval": "1d",
  "options_interval": "1d"
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `symbols` | array of strings | required | 1–1000; deduplicated and uppercased |
| `stock_interval` | string | `"1m"` | `"1m"` or `"1d"` — stock bar granularity |
| `options_interval` | string | none | `"1m"` or `"1d"` — options bar granularity. Omit to disable options collection. |

Response:
```json
{
  "results": [
    {"symbol": "AAPL",    "status": "already_tracked"},
    {"symbol": "NVDA",    "status": "accepted", "sync_status": "pending", "jobs_count": 8},
    {"symbol": "INVALID", "status": "error",    "error": "symbol not found"}
  ],
  "accepted": 1,
  "already_tracked": 1,
  "failed": 1
}
```

Per-symbol `status` values: `accepted` (new, jobs started), `already_tracked`, or `error`.

### Analytics Enable / Disable

```
POST   /v1/alpha/enable                   # Live Mutation Key required
DELETE /v1/alpha/disable/:symbol          # Live Mutation Key required
GET    /v1/alpha/status                   # Public
```

Enable/disable analytics tracking for symbols. `GET /v1/alpha/status` returns current analytics tracking state.

---

## 6. SQL Query

Read-only SQL access to the database for custom analysis. Both endpoints are **public** — no admin key required. Queries are restricted to `SELECT` and `WITH` (CTE) statements only.

### Get Schema

```bash
# All tables
curl http://localhost:8080/v1/schema

# Single table
curl "http://localhost:8080/v1/schema?table=stocks_1d"
```

### Execute Query

```bash
curl -X POST http://localhost:8080/v1/query \
  -H "Content-Type: application/json" \
  -d '{"query": "SELECT symbol, time, close FROM stocks_1d WHERE symbol = '\''AAPL'\'' ORDER BY time DESC LIMIT 10"}'
```

Restrictions: Only `SELECT` and `WITH` (CTE) statements. Max 10,000 rows.

---

## 7. Admin Endpoints (Live Mutation Key Required — :8080)

All endpoints in this section run on `:8080` only and require the **Live Mutation Key** (`ALPHA_LIVE_KEY`).

### Data Sync

```
POST /v1/admin/sync/trigger               # Trigger nightly sync
POST /v1/admin/sync/s3-download            # Trigger S3 flat file download
GET  /v1/admin/sync/status                 # Sync status
GET  /v1/admin/sync/latest                 # Latest nightly sync status
GET  /v1/admin/sync/summary                # Sync job summary
```

**Trigger sync** starts a data sync for all tracked symbols: bulk bar sync, computed analytics (IV, Greeks), and insight jobs (sentiment, earnings, fundamentals). Returns 409 if a sync is already running.

```bash
curl -X POST -H "Authorization: Bearer $ADMIN_KEY" http://localhost:8080/v1/admin/sync/trigger
```

Response (202 Accepted):
```json
{
  "message": "sync triggered",
  "sync_id": 42
}
```

Response (409 Conflict — sync already running):
```json
{
  "error": "sync is already running",
  "sync_id": 41,
  "started_at": "2026-02-23T20:15:00-05:00"
}
```

**Latest sync status:**

```bash
curl -H "Authorization: Bearer $ADMIN_KEY" http://localhost:8080/v1/admin/sync/latest
```

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

Returns 404 if no sync has ever run. The sync also runs automatically at the scheduled daily time.

### Provider Period Management

```
GET  /v1/admin/provider-periods            # List all provider periods
GET  /v1/admin/provider-periods/active     # Current active provider period
POST /v1/admin/provider-periods/switch     # Switch active provider
```

Controls which upstream provider is used for market data (Polygon, Schwab, tastytrade).

### Jobs (Admin Mutations)

```
DELETE /v1/jobs/:id                        # Cancel a job
POST   /v1/jobs/retry                      # Retry all failed jobs
POST   /v1/jobs/:id/retry                  # Retry a specific job
```

Read-only job endpoints (`GET /v1/jobs`, `GET /v1/jobs/:id`, `GET /v1/jobs/summary`) are public.

### Rates

```
POST /v1/rates/fetch                       # Fetch latest treasury rates
POST /v1/rates/backfill                    # Backfill historical rates
```

`GET /v1/rates` is public.

### As-Of Mode

Simulates a specific point-in-time for all market data queries. Useful for backtesting and replay.

```
POST   /v1/admin/as-of-date                 # Replay Mutation Key — :8081 only
GET    /v1/admin/as-of-date                 # Replay Mutation Key or API Key — :8081 only
DELETE /v1/admin/as-of-date                 # Replay Mutation Key — :8081 only
```

**Set body:** `{"end": "2026-01-15"}` — YYYY-MM-DD or RFC3339 timestamp. All subsequent queries act as if the current date is the specified date.

**Clear:** Removes the as-of boundary — queries return live data again.

**Behavior when active:**
- Date-aware endpoints (bars, quotes, options chain, etc.) require `as_of ≤ boundary`; `start` or `end` after boundary returns 400.
- All data comes from the local DB — no upstream provider calls during replay.
- Live-only endpoints return 403: streaming, movers, market status/forex/indices, insiders, indicator, transcript, metrics, portfolio greeks, raw query.
- All external API calls are blocked: search (news/general/ticker), provider proxies (Polygon, FRED, tastytrade, Schwab), raw Alpha Vantage proxy.
- Live-only analytics metrics return 403: `put_call_ratio` (requires live API data not stored in DB). `earnings_implied_move` is allowed (returns pre-computed implied/actual move from `earnings_quarterly`). `earnings_calendar` is also allowed (reads from `earnings_quarterly` table, no live fallback in as-of mode).
- Trading endpoints require `as_of` for pricing — **except** read-only endpoints (`get_account_summary`, `pnl-history`, list endpoints, position detail) which pass through without requiring `as_of`.
- Pass-through: admin, CRUD, symbol management, job management, market hours (calendar-based).

### Validation

```
GET /v1/admin/validate                     # Run validation checks
```

---

## 8. System & Health

| Endpoint | Description |
|----------|-------------|
| `/health` | Liveness check |
| `/ready` | Readiness check (DB, Redis, providers) |
| `/metrics` | Prometheus metrics |
| `/openapi.json` | OpenAPI spec |

---

## Error Handling

```json
{
  "error": {
    "code": "SYMBOL_NOT_TRACKED",
    "message": "Symbol UNKNOWN is not tracked. Use POST /v1/track to add it.",
    "retry_after": 30
  }
}
```

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `SYMBOL_NOT_TRACKED` | 404 | Symbol not in watchlist |
| `FETCH_DISABLED` | 503 | Gateway in read-only mode |
| `QUERY_FORBIDDEN` | 400 | SQL query contains forbidden statements |

---

## Calculation Constants

### Volatility & Returns

| Constant | Value | Description |
|----------|-------|-------------|
| Trading Days/Year | 252 | Annualizing volatility and returns |
| Default Timezone | America/New_York | All market times |
| HV Formula | `std(log_returns) * sqrt(252)` | Annualized historical volatility |
| IV Premium: Sell | +25% | IV premium above HV triggers "sell_premium" |
| IV Premium: Buy | -10% | IV discount below HV triggers "buy_premium" |

### Beta & Correlation

| Constant | Value | Description |
|----------|-------|-------------|
| Beta Default Period | 252 days | Standard 1-year rolling beta |
| Correlation Default | 30 days | Default correlation window |
| Max Symbols (Correlation) | 20 | Maximum in correlation matrix |
| Min Data Points | 10 | Minimum aligned return pairs |

### Unusual Volume

| Constant | Value | Description |
|----------|-------|-------------|
| Default Threshold | 2.0x | Volume ratio to trigger alert |
| Default Lookback | 60 min | 1-hour window |
| Method | Time-of-day normalized | Compares to same time-of-day historical average |

---

## 9. Trading (OMS)

Order management for paper trading (immediate synthetic fills) and live Schwab trading. All data lives in the **OMS database** (`OMS_DB_URL`) — isolated from the main analytics DB. Paper trading fills 24/7 at last market price with optional slippage. A default `paper` account is seeded on first start.

**Requires `OMS_DB_URL`** — gateway logs a warning and disables these endpoints if not set.

### Accounts

```
POST   /v1/trading/accounts                   Create account (201)
GET    /v1/trading/accounts                   List all accounts
GET    /v1/trading/accounts/:key              Get account
PATCH  /v1/trading/accounts/:key              Update risk limits or params
GET    /v1/trading/accounts/:key/summary      Computed summary (cash, P&L, positions value)
POST   /v1/trading/accounts/:key/reset        Reset account — wipe all orders, fills, positions (paper only)
POST   /v1/trading/accounts/:key/adjust-cash  Adjust cash balance with reason
POST   /v1/trading/accounts/:key/snapshot     Take P&L snapshot (201)
GET    /v1/trading/accounts/:key/pnl-history  P&L history (?start=YYYY-MM-DD&end=YYYY-MM-DD&group_by=strategy)
POST   /v1/trading/accounts/:key/simulate-outcome  Forward-evaluate positions against daily bars
POST   /v1/trading/accounts/:key/simulate-outcome/replay-day  Replay one day using minute bars (always applies)
POST   /v1/trading/accounts/:key/resolve-splits  Adjust positions/orders for stock splits up to as_of date
```

**Create body:** `key` (required), `name`, `provider` (`paper`|`schwab`|`schwab-sim`|`alpaca`, default `paper`), `currency` (default `USD`), `initial_cash` (default 100000), `max_order_value`, `max_position_value`, `max_open_positions`, `max_order_quantity`, `daily_loss_limit`, `params` (e.g. `{"slippage_pct": 0.001}`).

**Multi-account Schwab:** A single Schwab login can access multiple accounts (401k, Roth IRA, brokerage, etc.). Create a separate OMS account for each and set `params.schwab_account_number` to the Schwab account number (visible on schwab.com). The adapter matches this against the Schwab API to route orders to the correct account. If only one Schwab account exists, the param is optional (auto-selected). If multiple exist and the param is missing, order placement returns an error listing available account numbers.

**Summary response:**
```json
{
  "account_key": "paper",
  "provider": "paper",
  "cash_balance": 98500.00,
  "open_positions_value": 1600.00,
  "total_value": 100100.00,
  "daily_realized_pnl": 100.00,
  "daily_unrealized_pnl": 50.00,
  "open_positions_count": 1,
  "collateral_reserved": 350.00,
  "buying_power": 98150.00,
  "as_of": "2026-02-18T14:30:00Z"
}
```

**Adjust cash body:** `amount` (required, positive to add, negative to withdraw), `reason` (optional memo). Adjustments are logged in the account's `params.cash_adjustments` array.

**Reset:** Paper accounts only — returns 422 for live accounts. Deletes all orders, fills, and positions; resets cash to `initial_cash`.

**Snapshot response:**
```json
{
  "id": "...",
  "account_key": "paper",
  "snapshot_date": "2026-02-23",
  "cash_balance": 98500.00,
  "positions_value": 1600.00,
  "total_value": 100100.00,
  "realized_pnl": 250.00,
  "unrealized_pnl": 50.00,
  "metadata": {},
  "created_at": "2026-02-23T21:00:00Z"
}
```

- `realized_pnl` — sum of all closed positions' realized P&L
- `unrealized_pnl` — computed at snapshot time from open positions at current market prices
- One snapshot per account per date (`UNIQUE(account_key, snapshot_date)`)

**P&L history:** Returns `snapshots` array within date range. Add `?group_by=strategy` to include a `strategy_breakdown` with per-strategy P&L attribution (derived from `metadata.strategy` on orders/positions).

**Strategy performance:** `GET /v1/trading/accounts/:key/strategy-performance?strategy=X&days=7` — server-side aggregation of closed positions for a strategy. Returns `{strategy, trade_count, win_count, loss_count, win_rate, total_pnl, avg_win, avg_loss, profit_factor, consecutive_losses, max_consecutive_losses, last_trades}`. `days` is optional (0 or omit = all time). Uses `metadata.strategy` on positions for filtering.

### Orders

```
POST   /v1/trading/orders/validate         Validate without placing (200 pass | 422 limit details)
POST   /v1/trading/orders/validate-batch   Validate multiple orders (200, per-order results)
POST   /v1/trading/orders                  Place order (201)
GET    /v1/trading/orders                  List (filters below)
GET    /v1/trading/orders/count            Count matching orders (same filters, returns {count: N})
GET    /v1/trading/orders/:id              Get order + fills
DELETE /v1/trading/orders/:id              Cancel (200 on success, 422 if not cancellable)
POST   /v1/trading/orders/:id/sync         Sync live Schwab order status + fills
```

**List filters:** `?account_key=`, `?status=` (pending|open|filled|partial|cancelled|rejected), `?symbol=`, `?placed_by=`, `?note=` (substring match), `?metadata.key=value` (JSONB containment — e.g. `?metadata.strategy=momentum`), `?submitted_after=YYYY-MM-DD`, `?submitted_before=YYYY-MM-DD`, `?limit=N` (default 100), `?offset=N`, `?sort=` (submitted_at, -submitted_at, updated_at, symbol, status).

**Place/validate body:**

| Field | Type | Description |
|-------|------|-------------|
| `account_key` | string | Required. e.g. `"paper"` |
| `symbol` | string | Required. e.g. `"AAPL"` |
| `side` | string | Required. `"buy"` or `"sell"` |
| `quantity` | integer | Required. Shares or contracts |
| `instrument_type` | string | `"stock"` (default), `"call"`, `"put"` |
| `order_type` | string | `"market"` (default), `"limit"`, `"stop"`, `"stop_limit"` |
| `limit_price` | number | Required for limit/stop_limit |
| `stop_price` | number | Required for stop/stop_limit |
| `strike` | number | Required for options |
| `expiry` | string | Required for options. `YYYY-MM-DD` |
| `time_in_force` | string | `"day"` (default), `"gtc"`, `"fok"`, `"ioc"`. Passed to broker for live orders. |
| `note` | string | Optional memo |
| `placed_by` | string | Who/what placed the order, e.g. `"strategist-agent"` or `"eric"` |
| `closing_strategy` | object | Structured exit plan enforced by `simulate-outcome` and `auto-exit`. Equity: `{"stop_loss_price": 140, "take_profit_price": 170, "trailing_stop_pct": 5.0, "max_hold_days": 30}`. Spreads/Options: `{"profit_pct": 0.50, "loss_pct": 1.5, "max_dollar_loss": 500, "max_dollar_profit": 150, "dte_exit": 7, "delta_exit": 0.30, "gamma_exit": 0.05, "trailing_stop_pct": 5.0, "max_hold_days": 30}`. Time-based: `{"eod_exit_time": "15:30"}`. |
| `params` | object | Arbitrary order parameters |
| `metadata` | object | User-controlled tags (GIN-indexed). Flows from order to position on fill. e.g. `{"strategy": "momentum", "signal_id": "abc123"}` |
| `idempotency_key` | string | Optional dedup key. Unique per account. If a previous order exists with the same key, returns that order instead of creating a new one. |
| `opened_at` | string | ISO8601 timestamp to override position `opened_at`. For backtesting/replay where simulated time differs from wall clock. When omitted, uses `NOW()`. |

**Validate batch:** POST array of order bodies. Returns per-order `{valid, warnings}` results.

**Metadata & strategy flow:** `order.metadata` is copied to the position when the order fills and opens a new position. Subsequent fills on the same position do not overwrite metadata (unless reopening a closed position, which refreshes all ownership fields). The `strategy` value is extracted from `metadata.strategy` at fill time and stored as a first-class column on `trading_positions` — it is part of the unique constraint, so orders from different strategies (e.g. `hunter_momentum` vs `hunter_vwap_reversion`) create independent positions even for the same symbol and side. Fills have a separate `params` field for transaction-specific audit data.

**closing_strategy semantics:** The OMS stores closing strategy as structured JSON. The `simulate-outcome` endpoint enforces these rules by walking daily bars forward from the position's open date; the `simulate-outcome/replay-day` endpoint does the same at minute-bar granularity for a single day. Stop-loss triggers use **gap-through pricing** — if the bar opens past the stop level, the fill price is the open (not the stop price), reflecting realistic slippage on gap events.

**Equity/stock keys:** `stop_loss_price` (absolute price — gap-through aware), `take_profit_price` (absolute price), `trailing_stop_pct` (percent trailing stop from high-water mark), `max_hold_days` (trading days), `eod_exit_time` (`"HH:MM"` ET — see below).

**Spreads and standalone options keys:** `profit_pct` (fraction of entry credit for credit spreads; fraction of entry debit for debit spreads/long options), `loss_pct` (fraction of entry credit/debit), `intraday_profit_pct` (same math as `profit_pct` but RTH-gated — only evaluated during 09:30–16:00 ET, for 0DTE strategies), `max_dollar_loss` (absolute $ loss guardrail, evaluated continuously per-tick — fires when unrealized loss ≥ threshold), `max_dollar_profit` (absolute $ profit target, evaluated continuously per-tick — fires when profit ≥ threshold; useful for ratio spreads with near-zero entry credit where `profit_pct` is meaningless), `dte_exit` (close when DTE ≤ threshold), `delta_exit` (|net_delta| threshold for spreads — risk kill switch), `gamma_exit` (|net_gamma| threshold for spreads — risk kill switch), `trailing_stop_pct` (trailing stop on P&L%), `max_hold_days` (trading days from open — exits at last bar's close, consistent with equity), `eod_exit_time` (`"HH:MM"` ET — see below). `max_loss` is a **pre-trade entry validation cap** — the order/spread is rejected at submit time if projected max loss exceeds the threshold; it is NOT evaluated during the trade (use `max_dollar_loss` for continuous evaluation). Transitional fallback: when `max_dollar_loss` is unset, the evaluator reads `max_loss` as the continuous guardrail for deploy safety. Standalone options (positions without a spread) are repriced daily via Black-Scholes using the underlying bar's close. `stop_loss_price` and `take_profit_price` are supported on standalone options as **underlying price levels** (gap-through aware). These fields are ignored for spreads — use `profit_pct`/`loss_pct` instead.

**Delta/gamma exit (spreads only):** `delta_exit` and `gamma_exit` are absolute thresholds on per-position normalized |net_delta| and |net_gamma|. Net Greeks are computed as `Σ(bs_greek × side_sign × quantity) / totalContracts` where `side_sign` = +1 long / −1 short. Per-position normalization makes thresholds size-invariant — a 1-contract and 10-contract iron condor use the same `delta_exit` value. Greeks are computed from Black-Scholes using cached daily ATM IV in replay and live ATM IV in auto-exit. Known drift: replay uses ATM IV for every leg, so OTM gamma is systematically overstated vs market-implied skew — tune thresholds against replay numbers, not live desk Greeks.

**Tie-breaking order:** When multiple exit rules fire on the same tick, the highest-priority rule wins: Expiration > GammaExit > DeltaExit > MaxDollarLoss($) > StopLoss > LossPct > EODExitTime > DTEExit > MaxHold > IntradayProfitPct > MaxDollarProfit($) > TakeProfit > ProfitPct > TrailingStop. Kill switches (gamma, delta, absolute $ loss) fire first so a position exits before the P&L % stop absorbs the full move.

**Trailing stop behavior:** `trailing_stop_pct` is a percent value (e.g. `5.0` = 5%). For equities, it tracks the high-water mark (HWM) of the underlying price and triggers when price drops below `HWM × (1 - pct/100)` for long positions, or rises above the low-water mark `LWM × (1 + pct/100)` for short positions. Gap-through aware — if the bar opens past the trail level, the fill price is the open. For options and spreads, trailing stop tracks peak P&L% and triggers when P&L% drops below `peak × (1 - pct/100)`. To use ATR-based trailing, compute the percentage at entry: `trailing_stop_pct = (ATR_multiplier × ATR / entry_price) × 100`. To delay activation (e.g. activate after day 0), place the order without `trailing_stop_pct` and `PATCH` the position to add it after the desired holding period.

**EOD exit time:** `eod_exit_time` is a `"HH:MM"` string in Eastern time (e.g. `"15:30"`). The position is closed with exit reason `EOD_EXIT` when the configured time is reached. Valid range: `09:30`–`16:00`. Works for equity, option, and spread positions. Evaluated in two modes:
- **Auto-exit (live/paper):** closes at wall-clock time. Checked after stop-loss/take-profit so price-based rules take priority.
- **replay-day:** closes at the minute bar whose timestamp reaches the configured time, with exact intraday pricing. Price-based rules on the same bar take priority.
Not evaluated by `simulate-outcome` (daily bars lack intraday resolution — use `replay-day` for intraday exit simulation). The position monitor generates an `approaching_eod_exit` alert within 15 minutes of the exit time.

**Simulate-outcome body:**
```json
{
  "position_ids": ["uuid-1", "uuid-2"],
  "as_of": "2026-03-01",
  "apply": false
}
```
All fields optional. Omit `position_ids` to evaluate all open positions. Default is read-only preview (`apply=false`). Set `apply=true` to close triggered positions and update account balances. Response: `{outcomes: [...], split_adjustments?, evaluated, skipped, tc_total, errors?, diagnostics?}`. Each outcome: `{position_id, spread_id?, symbol, strategy?, instrument_type, exit_date, exit_price, exit_reason, pnl, tc_total, hold_days, legs?}`. `tc_total` is the round-trip transaction cost (entry + exit commissions): `commission_per_contract × qty × 2` for options, `commission_per_share × qty × 2` for equity. Gross P&L is in `pnl`; net = `pnl - tc_total`. Equity/standalone option outcomes are one per position. Spread outcomes return **one per spread** with `instrument_type: "spread"`, net `pnl`, and `legs` array: `[{position_id, instrument_type, side, exit_price, pnl}]`. `strategy` is populated from the position's first-class `strategy` column (falls back to `metadata.strategy` for legacy positions).

**Replay-day body (minute-bar evaluation):**
```json
{
  "date": "2026-03-10"
}
```
`date` is required (YYYY-MM-DD). Walks `stocks_1m` bars for equities and `options_1m` bars for options/spreads for that single day. Always applies — triggered positions are closed and cash updated. Response: `{date, exits: [...], split_adjustments?, entries_today, evaluated, skipped, errors?, diagnostics?}`. `entries_today` counts all positions opened on that date (including any already closed by prior replay days). Each exit has the same shape as `simulate-outcome` outcomes. Options use real prices from `options_1m` (mid preferred, close fallback); if option minute bars are unavailable, falls back to stock minute bars + Black-Scholes repricing.

For replay/backtesting, the agent calls `replay_day` once per trading day to advance time, then calls `get_account_summary` to see the resulting state.

**Determinism guarantee:** For bitwise-reproducible replay across runs, the caller must set the server's simulated clock via `POST /v1/admin/as-of-date` before invoking `replay-day`. Under that precondition, back-to-back replays over the same date range with the same inputs produce identical fill prices, exit timestamps, exit reasons, and per-day P&L — same order, same values. UUIDs (`spread_id`, fill IDs) are exempt from this guarantee and should be stripped before hashing trade logs. Without `asOfBoundary` set, replay still functions but is not cross-run deterministic.

**Corporate action (split) adjustment:** Both `simulate-outcome` and `replay-day` automatically resolve stock splits before evaluation. Adjusted fields: quantity, cost basis, strike (options), `stop_loss_price`, `take_profit_price`, HWM, and pending order prices. Not adjusted: `profit_pct`, `loss_pct`, `intraday_profit_pct`, `trailing_stop_pct`, `max_loss`, `max_dollar_loss`, `max_hold_days`, `dte_exit`, `delta_exit`, `gamma_exit`, `eod_exit_time`. Idempotent. Responses include a `split_adjustments` array (omitted when empty) with per-position old/new values.

**Resolve-splits (manual):** `POST /v1/trading/accounts/:key/resolve-splits` — Body: `{"as_of": "YYYY-MM-DD"}`. Response: `{adjusted, orders_adjusted, adjustments: [...], errors?}`. Called automatically by simulate-outcome/replay-day; this endpoint is for ad-hoc use.

Paper fills include `"outside_market_hours": true` in params when filled outside 9:30-16:00 ET.

### Positions

```
GET    /v1/trading/positions             List (filters below)
GET    /v1/trading/positions/count       Count matching positions (same filters, returns {count: N})
GET    /v1/trading/positions/:id         Get position
PATCH  /v1/trading/positions/:id         Update annotations (closing_strategy, params, metadata — merge semantics)
POST   /v1/trading/positions/:id/close   Close (places opposing market order, 409 if not open)
```

**List filters:** `?account_key=`, `?status=` (open|closed), `?instrument_type=` (stock|call|put), `?symbol=`, `?strategy=` (first-class column, e.g. `hunter_momentum`), `?metadata.key=value` (JSONB containment), `?opened_after=YYYY-MM-DD`, `?opened_before=YYYY-MM-DD`, `?closed_after=YYYY-MM-DD`, `?closed_before=YYYY-MM-DD`, `?limit=N` (default 100), `?offset=N`, `?sort=` (opened_at, -opened_at, closed_at, -closed_at, realized_pnl, symbol; prefix `-` for DESC).

**Update position body:** `closing_strategy`, `params`, `metadata` — all optional objects with merge semantics (new keys are added, existing keys are overwritten, omitted keys are preserved).

Positions are **automatically created from fills** — never entered manually. Each position tracks `strategy` (extracted from `metadata.strategy` at fill time), `avg_cost_basis`, `realized_pnl`, `cumulative_commissions`, `position_side` (`long`|`short`), and `multiplier` (1 for stocks, 100 for options). The unique constraint is `(account_key, symbol, instrument_type, strike, expiry, position_side, strategy)` — different strategies hold independent positions.

### Spreads (Multi-Leg Orders)

```
POST   /v1/trading/spreads                Place spread (201)
GET    /v1/trading/spreads                List (filters below)
GET    /v1/trading/spreads/:id            Get spread + legs
DELETE /v1/trading/spreads/:id            Cancel (200 on success)
POST   /v1/trading/spreads/:id/close      Close spread (opposing legs at market)
POST   /v1/trading/spreads/:id/sync       Sync live spread status + fills
```

**List filters:** `?account_key=`, `?status=` (pending|open|filled|partial|cancelled|rejected), `?symbol=`, `?placed_by=`, `?metadata.key=value`, `?submitted_after=YYYY-MM-DD`, `?submitted_before=YYYY-MM-DD`, `?limit=N` (default 100), `?offset=N`, `?sort=` (submitted_at, -submitted_at, updated_at, symbol, status).

**Place body:**

| Field | Type | Description |
|-------|------|-------------|
| `account_key` | string | Required. e.g. `"paper"` |
| `symbol` | string | Required. Underlying ticker, e.g. `"SPY"` |
| `legs` | array | Required. 2-4 option legs (see below) |
| `net_price` | number | Positive = credit, negative = debit. Omit for market. |
| `net_price_type` | string | Required. `"credit"`, `"debit"`, `"even"`, `"market"` |
| `atomicity` | string | Always `"all_or_none"` (default, only supported value) |
| `placed_by` | string | Who/what placed the spread |
| `closing_strategy` | object | Structured exit plan |
| `note` | string | Optional memo |
| `metadata` | object | User-controlled tags |
| `idempotency_key` | string | Optional dedup key |
| `opened_at` | string | ISO8601 timestamp to override position `opened_at` (backtesting/replay). Omit for `NOW()`. |

**Leg schema:**

| Field | Type | Description |
|-------|------|-------------|
| `option_type` | string | Required. `"call"` or `"put"` |
| `strike` | number | Required. Strike price |
| `expiry` | string | Required. `YYYY-MM-DD` |
| `side` | string | Required. `"buy"` or `"sell"` |
| `quantity` | integer | Required. Number of contracts |
| `position_effect` | string | `"opening"` (default) or `"closing"` |

**Leg response fields** (returned after fill): `fill_price`, `fill_qty`, `commission_amount`.

**Net price mapping to Schwab:**

| `net_price_type` | `net_price` | Schwab `orderType` | Schwab `price` |
|---|---|---|---|
| `credit` | 1.85 (positive) | `NET_CREDIT` | 1.85 |
| `debit` | -2.50 (negative) | `NET_DEBIT` | 2.50 (abs) |
| `even` | 0 or nil | `NET_ZERO` | 0 |
| `market` | nil | `MARKET` | omitted |

**Paper mode:** All legs fill synchronously at option mid price (bid/ask when available, Black-Scholes fallback). Individual leg positions are created with `spread_id` linkage. Limit spreads (`net_price_type: "credit"` or `"debit"`) require option pricing to distribute the net price across legs — rejected if the pricer is unavailable or theoretical pricing diverges significantly from the limit price.

**Slippage & commission params (account.params):**

| Key | Description | Example |
|-----|-------------|---------|
| `slippage_pct` | Default slippage for single-leg orders (fraction of price) | `0.001` (0.1%) |
| `spread_slippage_pct` | Flat spread slippage, fraction of price (overrides `slippage_pct`) | `0.02` (2%) |
| `slippage_1leg`..`slippage_4leg` | ORATS width-based: fraction of bid-ask half-spread | `0.75`, `0.68`, `0.60`, `0.56` |
| `commission_per_contract` | Per-contract commission for options trades | `0.65` ($0.65) |
| `commission_per_share` | Per-share commission for equity trades | `0` ($0 default) |

**Slippage lookup:** `slippage_Nleg` → `spread_slippage_pct` → `slippage_pct` → 0. For 5+ leg spreads, the per-leg-count lookup is skipped. `slippage_Nleg` values are **width-based** (applied as `mid ± halfSpread × fraction`). `spread_slippage_pct` and `slippage_pct` are **price-based** (`price × (1 ± fraction)`).

**Live mode (Schwab):** Submitted as a single multi-leg order with `complexOrderStrategyType: "CUSTOM"`, `specialInstruction: "ALL_OR_NONE"`. Poll `/sync` to retrieve fills.

**Entry/exit context (auto-enrichment):**

Spreads have two JSONB fields auto-populated by AlphaDB — no client action required:

| Field | When | Contents |
|-------|------|----------|
| `entry_context` | At fill time | `dte`, `net_entry_value`, `underlying_price`, `iv_rank` (0-1), `vix`, `vix_term_ratio` |
| `exit_context` | At exit time (replay or auto-exit) | `exit_reason`, `hold_days`, `realized_pnl`, `dte_remaining`, `exit_date`, `underlying_price`, `iv_rank`, `vix` |

All market data fields are sourced server-side from AlphaDB's own data (IV metrics, price bars, VIX/VIX3M quotes). Fields are omitted when data is unavailable (best-effort). Client-owned context (regime, strategy, signal_id) stays in `metadata`.

Example `entry_context` on a filled spread:
```json
{
  "entry_context": {
    "dte": 57,
    "net_entry_value": 150.00,
    "underlying_price": 400.57,
    "iv_rank": 0.287,
    "vix": 22.8,
    "vix_term_ratio": 0.995
  }
}
```

**Close spread:** `POST /v1/trading/spreads/:id/close` generates opposing legs (buy→sell, sell→buy) with `position_effect: "closing"` and submits as a new market-priced spread. Metadata includes `closes_spread_id` referencing the original. Returns an error if any leg's position is already closed — partial close would create naked exposure.

**Example — 1-1-2 put ratio spread:**
```json
{
  "account_key": "paper",
  "symbol": "SPY",
  "legs": [
    {"option_type": "put", "strike": 560, "expiry": "2026-03-20", "side": "buy", "quantity": 1},
    {"option_type": "put", "strike": 555, "expiry": "2026-03-20", "side": "sell", "quantity": 2}
  ],
  "net_price": 0.85,
  "net_price_type": "credit",
  "placed_by": "strategist-agent",
  "metadata": {"strategy": "put_ratio_spread"}
}
```

### Strategy Groups

Groups organize related orders, spreads, and positions into a single logical trade structure (e.g., an iron condor built from two vertical spreads).

```
POST   /v1/trading/groups                  Create group (201)
GET    /v1/trading/groups                  List (?account_key=&status=&symbol=)
GET    /v1/trading/groups/:id              Get group with positions summary
POST   /v1/trading/groups/:id/close        Close all open positions in group
POST   /v1/trading/groups/:id/adjust       Adjust group — close positions + add new legs
```

**Create body:** `account_key` (required), `name` (required, e.g. `"SPY iron condor Apr"`), `symbol` (required), `closing_strategy` (object), `metadata` (object).

**Group statuses:** `open`, `closed`, `partial` (some legs failed), `adjusting` (adjustment in progress), `rolling` (roll in progress).

**Group summary response:**
```json
{
  "group": {"id": "...", "name": "SPY iron condor Apr", "symbol": "SPY", "status": "open", ...},
  "positions": [...],
  "open_positions": 4,
  "closed_positions": 0,
  "total_realized_pnl": 0
}
```

**Adjust body:**
```json
{
  "close_positions": ["uuid-1", "uuid-2"],
  "add": [{"account_key": "paper", "symbol": "SPY", "legs": [...], "net_price_type": "credit"}],
  "add_orders": [{"account_key": "paper", "symbol": "SPY", "side": "buy", "quantity": 10}],
  "note": "widening the put spread"
}
```

All fields optional but at least one of `close_positions`, `add`, or `add_orders` must be provided. `add` entries use the same schema as the spread place endpoint. `add_orders` entries use the same schema as the order place endpoint. Group ID is automatically injected into all added orders/spreads.

**Linking to groups:** Pass `group_id` in `place_order` or `place_spread` request bodies. Orders, spreads, and positions all carry `group_id`. Filter by `?group_id=` on list endpoints.

### Rolling

Roll an existing position or spread: close the old one and open a new one atomically. For paper accounts, both sides execute in a single operation. Group ID is inherited from the closed position.

```
POST   /v1/trading/rolls                   Execute roll (201)
GET    /v1/trading/rolls                   List (?account_key=&group_id=)
GET    /v1/trading/rolls/:id               Get roll details
```

**Roll body:**
```json
{
  "account_key": "paper",
  "position_id": "uuid-of-position-to-close",
  "new_order": {"account_key": "paper", "symbol": "SPY", "side": "buy", "quantity": 10, ...}
}
```

Use `position_id` + `new_order` for single-leg rolls. Use `spread_id` + `new_spread` for spread rolls. Exactly one close target and one open target must be specified.

**Roll response:**
```json
{
  "roll_order": {"id": "...", "realized_pnl": 150.00, "net_debit_credit": -50.00, "status": "completed"},
  "closed_pnl": 150.00,
  "new_position": {"id": "...", "status": "filled", ...}
}
```

### Contingent Orders (OTO/OCO)

Contingent links fire child orders when a parent order fills or is cancelled. Auto-created from `closing_strategy` when an order with `take_profit_price` and/or `stop_loss_price` fills (creates OCO pair).

```
POST   /v1/trading/contingents             Create contingent link (201)
GET    /v1/trading/contingents             List (?account_key=&status=&group_id=)
DELETE /v1/trading/contingents/:id         Cancel pending contingent
```

**Create body:**
```json
{
  "account_key": "paper",
  "trigger_type": "on_fill",
  "parent_order_id": "uuid-of-parent",
  "child_template": {
    "symbol": "AAPL", "side": "sell", "quantity": 10,
    "order_type": "limit", "limit_price": 170
  }
}
```

**Trigger types:** `on_fill` (parent fills → fire child), `on_cancel` (parent cancelled → fire child).

**Statuses:** `pending`, `triggered`, `cancelled`, `expired`.

**OCO behavior:** When a closing strategy creates both TP and SL contingents, they are linked as OCO — when either child fills, the other is automatically cancelled.

### Audit Log

Immutable record of all OMS state changes: fills, cash adjustments, order state changes, reconciliation, contingent triggers, expirations, assignments, group events.

```
GET  /v1/trading/accounts/:key/audit-log   List audit events
```

**Query params:** `?event_type=` (fill_applied, cash_adjustment, order_state_change, reconciliation_sync, order_expired, option_expired, option_assigned, group_created, group_closed, group_adjusted, roll_executed, contingent_created, contingent_triggered, contingent_cancelled), `?entity_type=` (order, spread, position, account), `?entity_id=`, `?limit=`, `?offset=`.

### Reconciliation

Syncs all open orders and spreads with their broker. For live accounts, retrieves latest fill status from Schwab/Alpaca. Also resolves expired options automatically.

```
POST  /v1/trading/accounts/:key/reconcile   Trigger reconciliation
```

Runs automatically on a configurable interval when reconciliation is enabled.

### Position Structure Analysis

Analyzes the spread type and computes risk metrics for positions. Pure computation — no I/O beyond position lookup.

```
GET  /v1/trading/positions/:id/structure    Analyze position's spread structure
GET  /v1/trading/spreads/:id/structure      Analyze spread structure
GET  /v1/trading/accounts/:key/structures   Analyze all open positions
```

**Response:**
```json
{
  "structure_type": "vertical_credit",
  "is_defined_risk": true,
  "contracts": 1,
  "width": 5.0,
  "net_premium": 3.0,
  "max_gain": 300.0,
  "max_loss": 200.0,
  "breakevens": [417.0],
  "legs": [
    {"position_id": "...", "role": "short_body", "instrument_type": "put", "strike": 420, "side": "short", "quantity": 1, "avg_cost_basis": 5.0},
    {"position_id": "...", "role": "long_wing", "instrument_type": "put", "strike": 415, "side": "long", "quantity": 1, "avg_cost_basis": 2.0}
  ]
}
```

**Structure types:** `vertical_credit`, `vertical_debit`, `iron_condor`, `covered_call`, `straddle`, `strangle`, `calendar`, `diagonal`, `single_leg`, `custom` (ratio spreads, partially closed, or unrecognized).

### Collateral Tracking

Computes margin requirements for short options and defined-risk spreads. Paper accounts use collateral-aware buying power for pre-trade validation.

```
GET  /v1/trading/accounts/:key/collateral   Account collateral summary
GET  /v1/trading/positions/:id/collateral   Position/spread collateral requirement
```

**Account collateral response:**
```json
{
  "account_key": "paper",
  "total_reserved": 9350.00,
  "current_cash": 100000.00,
  "buying_power": 90650.00,
  "requirements": [
    {"position_ids": ["..."], "spread_id": "...", "symbol": "SPY", "requirement_type": "defined_risk", "reserved_amount": 350.00, "formula": "max_loss"},
    {"position_ids": ["..."], "symbol": "SPY", "requirement_type": "naked_short", "reserved_amount": 9000.00, "formula": "max(20%_notional, premium)"}
  ]
}
```

**Collateral types:** `defined_risk` (MaxLoss from structure analysis), `naked_short` (simplified Reg-T: max of 20% notional or premium), `covered_call` (0 — short call covered by long stock), `none` (long options, long stock).

`get_account_summary` also includes `collateral_reserved` and `buying_power` fields.

### Expiration Handling

Resolves expired option positions. Paper/sim accounts: OTM expires worthless, ITM triggers exercise (long) or assignment (short) with automatic stock position creation. Live accounts: detection only (broker handles resolution).

```
POST  /v1/trading/accounts/:key/resolve-expirations   Resolve expired options
```

**Body:** `{"as_of": "2026-04-02"}` — date to check expirations against (typically today).

**Response:**
```json
{
  "expired": 2,
  "worthless": 1,
  "assigned": 1,
  "details": [
    {"position_id": "...", "symbol": "SPY", "action": "expired_worthless", "cash_delta": 0},
    {"position_id": "...", "symbol": "SPY", "action": "assigned", "cash_delta": -43000}
  ]
}
```

**Actions:** `expired_worthless` (OTM or pin risk), `exercised` (ITM long option), `assigned` (ITM short option). Pin risk (price == strike) is treated as OTM.

**ITM assignment** atomically: closes the option at intrinsic value and creates a stock position at the strike price. Cash is adjusted for both the option settlement and the stock acquisition/delivery.

### Bracket Orders

Places an entry order with take-profit and stop-loss contingent orders in one call.

```
POST  /v1/trading/brackets   Place bracket order (201)
```

**Body:** All `place_order` fields plus `take_profit_price` (number) and `stop_loss_price` (number). At least one is required.

```json
{
  "account_key": "paper",
  "symbol": "AAPL", "side": "buy", "quantity": 10,
  "take_profit_price": 170.00,
  "stop_loss_price": 140.00
}
```

**Response:** `{entry_order, fill, tp_contingent_id, sl_contingent_id, warnings}`. TP and SL are created as OTO contingent links with OCO cancellation — filling one automatically cancels the other.

### Portfolio Greeks (OMS)

Aggregated Greeks across all open option positions for a trading account.

```
GET  /v1/trading/accounts/:key/greeks   Account portfolio Greeks
```

**Response:** Per-position raw and position-adjusted Greeks (delta × qty × multiplier × side_sign), plus totals. Uses Black-Scholes with ATM IV history — works 24/7 without live quotes. Returns partial results when individual positions fail to price.

### Position Alerts

Monitors positions approaching their closing strategy thresholds.

```
GET  /v1/trading/accounts/:key/position-alerts   Position alerts
```

**Response:** `{account_key, evaluated_at, evaluated, alerts: [{position_id, symbol, alert_type, current_value, threshold_value, message}]}`.

Alert types: `approaching_stop_loss` (within 5%), `approaching_take_profit` (within 5%), `approaching_max_hold` (within 2 days), `approaching_dte_exit` (within 2 days), `approaching_eod_exit` (within 15 minutes).

### Risk Dashboard

Cross-account exposure view aggregating all active trading accounts.

```
GET  /v1/trading/risk-dashboard   Cross-account risk dashboard
```

**Response:** Per-account exposure (positions, unrealized P&L, cash, collateral, buying power, Greeks) plus cross-account totals. Greeks included when GreeksCalculator is configured. Gracefully degrades per-account on errors.

