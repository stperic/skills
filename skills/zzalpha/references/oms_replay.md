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

## Two-process role boundary (live + replay)

The `asOfBoundary` is a process-global. Rather than refactor every `timeutil.Now()` caller to take a per-request clock, AlphaDB runs **two `alphaserver` processes** on the same host sharing one OMS database. Isolation is by `account_key` at the service layer.

| Process | Port | `ALPHADB_ROLE` | Allowed accounts (writes) | Background loops |
|---|---|---|---|---|
| `alphaserver-live` | `8080` | `live` | `ALPHADB_LIVE_ACCOUNT_KEYS` (typically `paper,schwab-sim`) | yes — auto-exit, loss monitor, watchdog, nightly scheduler, reconciliation |
| `alphaserver-replay` | `8081` | `replay` | every account **not** in `LIVE_ACCOUNT_KEYS` (replay/sim accounts) | **no** — silent at startup |

### Substrate

- `internal/oms/domain/role.go` — `Role` enum (`RoleLive` / `RoleReplay`), `RoleGuard.AllowAccount` / `FilterAccounts` / `LiveAccountKeys`, `ErrAccountNotInRole` sentinel.
- `internal/config/config.go` — `RoleConfig{Mode, LiveAccountKeys}`; `Validate` rejects unknown mode and empty list in live.
- `internal/oms/module.go` — `oms.Config.Role *RoleGuard` is required at `New`; the module refuses to boot with a nil guard so production cannot accidentally default-allow.

### Service-layer enforcement

All 12 OMS services accept `WithRoleGuard`. Every write method (`PlaceOrder`, `PlaceSpread`, `CancelOrder/Spread`, `ClosePosition`, `ReplayDay`, `SimulateOutcome`, `ResolveExpirations/Splits`, `ReconcileAccount`, `AdjustCash`, `ResetAccount`, `RollPosition`, …) calls `g.AllowAccount(key)` before mutating. Background loops only attach when `Mode == "live"`.

### HTTP edge

Two different rejection mechanisms, by design:

- **Trading writes** (`place_order`, `place_spread`, `close_spread`, roll, cancel, adjust-cash, reconcile, replay-day, simulate-outcome, …) exist on both ports. The service-layer `g.AllowAccount(key)` check returns `ErrAccountNotInRole`, translated by `internal/oms/http/handler.go` to **`400 WRONG_ROLE`** with the account key and the role that owns it.
- **`/v1/admin/as-of-date`** (GET/POST/DELETE) is **physically absent** on the live process — `server.go` only registers the route group when `cfg.Role == "replay"`. Calls to `:8080` get a stock `404`, no handler invocation. This is deliberate: there is no account_key to gate by, and "404 because the route doesn't exist" is harder to undo in a refactor than a role-check middleware that someone could forget to wire.

Reads other than as-of-date are not role-gated and work on either port.

**Routing summary for clients:**
- Trading writes: route by `account_key`. `paper` / `schwab-sim` (anything in `LIVE_ACCOUNT_KEYS`) → `:8080`. Other accounts → `:8081`. `replay-day` / `simulate-outcome` follow the same account rule.
- As-of-date set/clear/read: `:8081` only. `:8080` returns 404.
- Reads (health, market data, account summary, positions list) work on either port; prefer the port matching the workload to avoid mixing.

### Deployment

`scripts/setup-lxc.sh` installs both systemd units (`alphaserver-live.service` + `alphaserver-replay.service`) and writes three env files: shared `/opt/alphadb/.env`, `live.env` (sets `ALPHADB_ROLE=live`, `ALPHADB_LIVE_ACCOUNT_KEYS`, `HTTP_PORT=8080`), and `replay.env` (`ALPHADB_ROLE=replay`, `HTTP_PORT=8081`). `make lxc` cycles both; `make lxc-live` / `make lxc-replay` cycle one side only.

### What this does NOT solve

A single process is still single-clock — running two concurrent replays in `alphaserver-replay` would conflict on `asOfBoundary`. The expected pattern is one replay session at a time per replay process. If concurrent replays become a requirement, the original per-context-clock refactor (deferred in two cold reviews as too invasive) is the real fix.

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
