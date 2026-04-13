# OMS — Replay Determinism & As-Of Date

`SimulationService.ReplayDay` replays a trading day deterministically against historical data. It is **cross-run bitwise-reproducible** under an active `timeutil.asOfBoundary`. This file is the invariant list — read it before touching the replay path.

## Setting the as-of boundary

```
POST /v1/admin/as-of-date    { "date": "2026-02-14" }
DELETE /v1/admin/as-of-date  (clear)
GET  /v1/admin/as-of-date
```

Auth: Admin Key or As-Of Key for set/clear; API Key also accepted for GET.

## Determinism contract

The replay determinism contract rests on three load-bearing invariants. Violating any of them causes non-reproducible runs.

### 1. Ordered spread grouping

`domain.GroupPositionsBySpread` returns a **`[]SpreadGroup` slice** — never a map. **Never reintroduce map iteration here.** Go map iteration order is randomized per process, so any map-based grouping at this layer corrupts determinism.

### 2. Stable natural-key tiebreaker on position repo default sort

Default sort is `opened_at DESC` plus the `idx_trading_positions_unique` columns (`account_key, symbol, instrument_type, strike, expiry, position_side, strategy`). **Never add a UUID tiebreaker like `id ASC`** — UUIDs are per-run random. The natural key is what makes the sort stable across runs.

### 3. Background loops skip ticks during replay

Auto-exit, loss monitor, and other background loops must skip their ticks while `asOfBoundary` is set. This is enforced via `background_loop.go:isReplayActive`. If a background loop ticks during replay, it races with the deterministic replay clock and the run becomes non-reproducible.

## The full invariant list

Lives in the doc comment above `SimulationService.ReplayDay`. Read it before touching the replay path. The three points above are the ones that break most often; the doc comment contains the long tail.

## Global `asOfBoundary` isolation — open follow-up

The current `asOfBoundary` is a process-global. This works when the server runs replays serially, but a concurrent "replay + live" mixed mode would conflict. Tracked in project memory (`project_timezone_review_followups.md`).

## Premature expiration resolution — open follow-up

Positions that expire inside the replay window can be resolved eagerly based on today's date rather than the as-of date. Tracked in the same follow-up list.

## Pending order replay gaps

Four known gaps from replay fixes:

1. Option limit orders
2. Pending spreads
3. StopLimit 2-phase
4. Exit + close double-fire

See project memory `project_pending_order_gaps.md`.

## Replay vs live Greeks drift

Replay uses cached daily ATM IV for every leg, so OTM gamma is systematically overstated vs market-implied skew. Tune `delta_exit` / `gamma_exit` thresholds against replay numbers, not live desk Greeks. See `oms_exit_rules.md`.

## Verifying a deployed binary is actually running the replay fix

Replay determinism fixes can be undone by a stale deployed binary. Before trusting "my fix is live," confirm the running process is mapped to the freshly deployed file. See `operations.md` for the `grep -a` + `/proc/$(pgrep alphaserver)/exe` technique.
