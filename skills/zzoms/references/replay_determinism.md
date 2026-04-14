# Replay Determinism — Contract and Invariants

Replay runs the OMS against a historical trading day under an **as-of clock** that pins the "current time" to a chosen past moment. The contract: replay must be **cross-run bitwise-reproducible**. Two runs of the same day with the same inputs must produce byte-identical output (fills, positions, P&L, exit decisions).

This is load-bearing. A strategy tuned against a non-deterministic replay is tuning against noise.

## The as-of boundary

The as-of boundary is a process-level setting that pins the clock. Expose it via whatever control surface fits the project — HTTP, CLI, RPC, a feature flag. An HTTP shape might look like:

```
POST   /admin/as-of-date    { "date": "2026-02-14" }
DELETE /admin/as-of-date
GET    /admin/as-of-date
```

The shape is illustrative; the invariant is "there is one place a human sets the boundary, one place the clock reads it, and no other code path advances time." While set:

- `Clock.Now()` returns the as-of time, not wall clock.
- All timestamp-sensitive code (order expiry, EOD exit, max_hold_days, DTE, TTE, Greeks) reads from the injected clock.
- Background loops (position monitor, auto-exit, loss monitor, schedulers) **skip their ticks**. See invariant 3 below.
- The replay driver owns tick advancement — it's the only thing moving the clock forward.

When the boundary is cleared, the clock returns to wall-clock mode and background loops resume.

## The three invariants

All three must hold. Any violation causes non-reproducible runs.

### 1. Ordered spread grouping — slice, not map

`GroupPositionsBySpread` returns `[]SpreadGroup` — an **ordered slice**, never a map.

Go map iteration is randomized per process. A grouping step that returns `map[string][]Position` and iterates it produces a different group order every run. Downstream code that processes groups in iteration order then makes different decisions on each run.

**Never reintroduce map iteration at this layer.** Even a `map` used internally during grouping is dangerous — if the output is converted to a slice via `for k := range m`, the slice order is randomized.

The grouping function builds a slice directly, sorts by a stable key (typically `underlying, expiry, strategy`), and returns it. Callers iterate in slice order.

### 2. Stable natural-key tiebreaker on position sort

The default sort for position queries is:

```
ORDER BY opened_at DESC,
         account_key, symbol, instrument_type, strike, expiry, position_side, strategy
```

The natural-key columns are the unique constraint on `trading_positions` (minus `opened_at`).

**Never add `id ASC` (or any UUID column) as a tiebreaker.** UUIDs are per-run random — a UUID tiebreaker is equivalent to "randomly reorder ties." Two positions with identical `opened_at` and identical natural keys are, by construction, the same logical position and there's nothing to tiebreak.

If two positions share `opened_at` and differ only in a UUID, your unique constraint is wrong.

### 3. Background loops skip ticks during replay

Auto-exit, loss monitor, position monitor, nightly schedulers, and any other time-driven loop **must skip their ticks** while the as-of boundary is set. Enforced via a central check:

```
func (l *BackgroundLoop) tick() {
    if clock.IsReplayActive() {
        return  // replay driver owns advancement
    }
    // normal tick logic
}
```

Rationale: background loops ticking during replay race with the deterministic replay clock. They may read positions mid-replay-tick, evaluate rules with half-updated state, or advance the clock themselves via `time.Now()`. The replay driver must be the sole source of forward motion.

**Every new background loop must wire the gate.** A loop added without the gate will silently break replay for any code path that touches its data.

## The long tail

The three invariants above are the ones that break most often. The doc comment on the replay entry point (`SimulationService.ReplayDay`) carries the full list:

- No `time.Now()` anywhere on the replay path — always `Clock.Now()`.
- No randomness without a seeded RNG; seeds derived from the as-of date for reproducibility.
- No `map` iteration for any output that's consumed in order.
- Fill application ordered by `(fill_time, fill_seq)` deterministically.
- IV cache, dividend yield cache, and quote cache are per-day and read-only during replay.
- No database writes to tables outside the OMS DB (the main DB might be in a different state during replay).
- No network calls (quotes, Greeks, recon) during replay — all inputs come from caches or injected test doubles.

Read the doc comment before touching the replay path. Add to it when a new invariant is discovered during a bug hunt.

## Premature expiration resolution — a recurring bug

Positions that expire inside the replay window can be resolved eagerly based on today's wall-clock date rather than the as-of date. Symptom: "a position that was alive at 10:00 ET on Feb 14 2026 shows as expired because the replay is running on Apr 14 2026."

The fix is always the same: the expiration check must use `Clock.Now()`, not `time.Now()`. Audit every expiration-related call site when this bug is reported.

## Global as-of boundary — open architectural concern

Most implementations store the as-of boundary as a process-global. This works when the server runs replays serially, but a concurrent "replay + live" mixed mode (e.g. one account in replay while another trades live) conflicts — the global clock can't be both.

The forward path is per-context clocks: inject a `Clock` into each service and thread it through via `context.Context`. The global flag becomes a default for the live path; replay paths override via context.

This is a larger refactor than most teams want to undertake in a bug fix. Until it happens, document the constraint: replays are serial, and live trading is paused during a replay run.

## Verifying a deployed binary is running the fix

Applies to any OMS fix, not just replay fixes — included here because replay regressions are one of the most common times it bites. See `observability.md` for the full deployment verification recipe.

## Testing replay determinism

The canonical test: run the same day twice, compare output byte-for-byte.

```
run1 := replayDay("2026-02-14", seed=42)
run2 := replayDay("2026-02-14", seed=42)
assertBytesEqual(run1.fills, run2.fills)
assertBytesEqual(run1.positions, run2.positions)
assertBytesEqual(run1.exitDecisions, run2.exitDecisions)
```

Runs must agree on every field of every row, in every order. A diff of a single UUID tiebreaker is enough to fail the test — and should, because it means invariant 2 is broken.

Run this test in CI on every commit that touches the OMS. The failure mode is usually subtle (a new map iteration, a new UUID column, a new background loop without the gate), and CI is the only reliable way to catch it before it reaches production.
