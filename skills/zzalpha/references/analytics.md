# Computed Analytics (`/v1/alpha/*`)

AlphaDB's reason for existing: derived metrics that upstream providers don't offer. All computed from stored daily IV/options data + live quotes. Served under `/v1/alpha/*`.

For the full HTTP request/response shapes see `http_api.md`. For the MCP tool schemas see `mcp_tools.md`. This file is the **methodology** reference — what each metric actually computes.

## IV metrics

- **IV rank** — `(current_atm_iv − min_252d) / (max_252d − min_252d)`, scaled 0–100. One-year window.
- **IV percentile** — fraction of the past 252 trading days where ATM IV was below today's. Also 0–100.
- Both are computed from the `iv_daily_*` tables populated by the `iv_daily:*` nightly jobs.

Batch-enabled: a comma-separated `symbols` param returns `{"data": {"SYM1": {...}, ...}}`. Single symbol returns a flat response. `iv_metrics` batch is optimized with a single DB query across all requested symbols.

## IV term structure

Per-expiration ATM IV across the chain, returned as an ordered list. Used to detect contango/backwardation and event-driven IV humps. Live computation against today's options chain.

## IV/RV spread

`atm_iv − realized_vol_20d`. The 20-day realized vol is computed from close-to-close log returns. Positive = IV richer than realized; negative = cheap options.

## Skew (25-delta risk reversal)

**Standard 25Δ risk reversal:** `iv(25Δ call) − iv(25Δ put)`.

Negative values = put skew (puts richer), which is the normal state for equity indices. Positive values = call skew, which shows up in commodity-linked names and during short squeezes.

Commit `dab74b2` fixed `skew_25d` to the **standard** risk reversal definition. Do not reintroduce non-standard sign conventions.

## Earnings implied move

Straddle-priced implied move around the next earnings date, as a percentage of underlying. Computed from the front-month ATM straddle ÷ spot, adjusted for time-to-earnings.

## Sentiment

Daily sentiment score per symbol, computed from news/social sources by the `sentiment:*` nightly jobs (rate-limited queue, no DAG dependencies). Stored in the `sentiment_daily` table.

The sentiment dashboard was added in commit `dab74b2`.

## Volatility

Rolling realized volatility across multiple windows (typically 10d, 20d, 60d, 252d). Close-to-close log returns, annualized.

## Beta

Rolling beta vs SPY (or a configurable benchmark) computed from daily log returns. Window is configurable; default 60d.

## Correlation

Pairwise rolling correlation across a set of symbols. Used by the financial analyst agent to detect regime shifts and basket construction.

## Mean reversion

AR(1) half-life, Augmented Dickey-Fuller p-value (constant model, AIC-selected lag), and z-score on `log(close)` at `as_of`. Computed in `internal/stats` with gonum-backed OLS; ADF p-value interpolated from MacKinnon (1996) asymptotic τ_c surface, log-p interpolation in the left tail. Batch-native — up to 1000 symbols per call, no single-symbol mode. Used by AlphaSeeker's Phase B mean-reversion strategy gate (`half_life ∈ [1,30] AND adf_p_value < 0.01 AND |z_score| ≥ 2.0`).

This endpoint is the canonical implementation; clients consume, they don't reimplement. See `architecture.md` "AlphaDB owns analytics, clients consume."

## Batch mode convention

All single-symbol analytics tools accept comma-separated `symbols` for batch mode (up to 50). Failed symbols in a batch get `{"error": "..."}` entries instead of a data object — partial results don't fail the whole request.

Batch-enabled tools: `iv_metrics`, `iv_term_structure`, `iv_rv_spread`, `skew`, `earnings_implied_move`, `sentiment`, `volatility`, `beta`. Batch-native (no single-symbol mode): `mean_reversion` (up to 1000 symbols).

## Data freshness gates

Earnings has **two freshness gates**: dispatch reads `sync_tracking`, worker reads `earnings_quarterly.updated_at`. Roll back both for an Alpha Vantage refetch; only `sync_tracking` for a local-only recompute. See project memory `reference_sync_gates.md`.

## Earnings timing sources

Two Alpha Vantage endpoints carry BMO/AMC timing:

- `EARNINGS_CALENDAR` — sparse, ~37% coverage
- `EARNINGS` historical — ~100% coverage

The adapter must read **both**. Per-quarter timing handles historical BMO↔AMC switches correctly. Commit `3f934fc` fixed the adapter to actually read the historical endpoint, taking coverage from 37% to 99%. See `reference_timing_limitations.md`.
