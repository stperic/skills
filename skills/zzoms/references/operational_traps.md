# Operational Traps — Bugs That Keep Coming Back

This file is a catalog of the bugs that recur in OMS codebases. Each one has been fixed and re-introduced many times in production systems. Call it a pre-mortem: when you see one of these shapes, slow down.

## 1. "Amend" preserves queue priority

**Shape:** a developer assumes `AmendOrder(id, new_strike)` atomically changes the strike while keeping the broker's order id and queue priority.

**Reality:** most broker APIs implement amend as cancel+new under the hood. The "amended" order has a fresh queue position, a new broker id, and any partial fill on the original is final.

**Fix:** treat amend as cancel+new at the OMS level too. The new order row has its own client order id and a `replaces_order_id` link. Never assume fills on the old order also belong to the new one.

## 2. Fill ordering under reconnect

**Shape:** broker stream drops, OMS reconnects, re-fetches missed exec reports. The reports arrive out of order relative to the original stream.

**Reality:** applying fills out of order corrupts cost basis. A sequence "buy 1 @ 100, sell 1 @ 110" produces different P&L than "sell 1 @ 110, buy 1 @ 100" even though both end with quantity 0.

**Fix:** never apply fills in arrival order. Sort by `(fill_time, broker_exec_id)` before `ApplyFill`, using the broker's stable exec id (FIX `ExecID` tag 17 or the REST equivalent) as the tiebreaker. A per-broker `exec_seq` number is tempting but not reliable — some brokers reset it on reconnect, so an exec_seq-based sort silently reorders fills across a reconnect boundary. The fill repo's insert is idempotent on `broker_exec_id`, but the *application* to positions must be sequential and ordered.

## 3. DST and timezone bugs in expiry

**Shape:** order expiry checked with UTC midnight or wall-clock `time.Now()`.

**Reality:** DST shifts move the US market close by an hour from UTC's view twice a year. A DAY order expiring at 16:00 ET shifts between 20:00 UTC and 21:00 UTC. An expiry check that uses `now.Hour() == 20` silently breaks for six months.

**Fix:** always compare in Eastern time for market events. Use a `timeutil.StartOfDayEastern` helper, never `t.Truncate(24*time.Hour)`. When stepping dates, use `t.AddDate(0,0,±1)`, never `t.Add(±24*time.Hour)` (DST days are 23h or 25h).

## 4. Premature expiration resolution during replay

**Shape:** replay runs on day D. Positions that expire in the replay window get resolved based on *today's* wall-clock date instead of the as-of date, so every position that expired any time before today shows as expired at the start of the replay.

**Reality:** the expiry check uses `time.Now()` instead of `Clock.Now()`. This is the single most common replay-determinism bug because it's easy to miss in code review — the function name (`IsExpired`) doesn't hint at a clock dependency.

**Fix:** grep for `time.Now()` anywhere on the replay path. Replace with injected clock. Gate background loops behind `IsReplayActive` so even loops that aren't explicitly part of the replay path don't advance state using wall clock.

## 5. Global as-of boundary blocks live trading during replay

**Shape:** most OMS implementations store the as-of boundary as a process-global. A human runs a replay; background loops gate and stop ticking. Meanwhile, live accounts in the same process also stop receiving tick-driven updates.

**Reality:** this is not a bug, exactly — it's an architectural constraint. Replays are serial, and live trading is paused during a replay. But it's routinely forgotten and causes "why didn't my stop fire last night during that replay run?" surprises.

**Fix:** document the constraint, run replays off-hours, and plan the per-context-clock refactor if your team needs concurrent replay + live.

## 6. Silent zeros on Greeks failures

**Shape:** a per-leg Greeks computation fails (missing IV, stale quote). The code returns `LegQuote{Delta: 0, Gamma: 0}` and continues.

**Reality:** a zero-delta position looks fine to the exit evaluator. The delta kill switch never fires. The position sits open through a market move that should have closed it.

**Fix:** Greeks failures return errors. The evaluator treats "Greeks not available" as "rule not applicable this tick" (not "rule passed"). Warn-once per position per day to prevent log floods. See `greeks_plumbing.md`.

## 7. `max_loss` evaluated during the trade

**Shape:** a developer wires `max_loss` into both the entry validator and the position monitor, assuming it means "kill the trade when unrealized loss exceeds this."

**Reality:** `max_loss` is a **pre-trade theoretical cap**. It's the width-minus-credit of a credit spread, computed at entry, compared to the threshold, used to reject the entry. It is not an unrealized-loss kill.

**Fix:** `max_loss` is pre-trade only. Continuous dollar-loss kill is `max_dollar_loss`. Two separate fields, two separate semantics. See `risk_controls.md` and `exit_rules.md`.

## 8. Appending a new exit rule to the end of the priority list

**Shape:** a developer adds a new rule (say, `intraday_profit_pct`) and registers it at the bottom of the tie-breaking list.

**Reality:** the new rule is now lower priority than `profit_pct` and `trailing_stop`, so it can never win a tie. The rule appears configured but never fires in any scenario where another rule also fires. Months pass before anyone notices.

**Fix:** the priority list is fixed. Inserting a new rule requires deciding where in the priority order it belongs and putting it there explicitly. Adding a new rule always involves rethinking tie-breaking adjacency — there's no "safe" append.

## 9. UUID tiebreaker on position sort

**Shape:** a developer adds `id ASC` to a position query's `ORDER BY` to "make results deterministic."

**Reality:** UUIDs are per-run random (UUIDv4) or time-based but still per-insert unique (UUIDv7). Either way, they make sort order non-reproducible across replay runs. Two positions with identical natural keys sort differently each run, and replay determinism breaks.

**Fix:** use the natural key as the tiebreaker. See `replay_determinism.md`. If two rows truly have identical natural keys, your unique constraint is wrong.

## 10. Map iteration in the replay path

**Shape:** a developer writes `for k, v := range positionGroups { ... }` somewhere on the replay path.

**Reality:** Go map iteration order is randomized per process. Two runs produce different iteration orders, which produce different decisions, which produce different output.

**Fix:** the grouping function returns `[]Group`, not `map[K]Group`. If a map is used internally, the output is sorted deterministically before return. Add a CI check that greps for `for k := range` on replay paths.

## 11. Administrative bulk ops inside a business transaction

**Shape:** a developer wraps `ResetAccount` inside a `RunTxCtx` closure "for atomicity."

**Reality:** `ResetAccount` touches ~10 tables. The outer tx holds row locks across all of them for the duration. Other transactions wait. Autovacuum can't process dead rows. The database grinds.

**Fix:** admin bulk ops refuse to run inside a tx. The repo method type-switches and returns `ErrResetAccountNested` if called on a tx-scoped copy. See `repository_patterns.md`.

## 12. Broker credentials required for paper trading

**Shape:** credential gating is at the request level ("if request is a trading request, require broker credentials"). Paper accounts start demanding live broker keys.

**Reality:** the gate should be per-account, not per-request. Paper accounts have `provider = paper` and route to the paper engine, which never needs broker credentials.

**Fix:** the credential check sits at the point the broker adapter is actually called, not at the request boundary. Paper orders never reach the adapter, so they never trigger the check. See `broker_integration.md`.

## 13. Exit rule delta/gamma thresholds that don't scale

**Shape:** a developer sets `delta_exit = 100` on a 1-contract iron condor. It works. Then the strategy scales to 10-contract ICs and the rule fires immediately on entry.

**Reality:** without normalization, net delta scales with position size. A 10-contract IC has 10× the raw delta of a 1-contract IC. A threshold calibrated for 1 contract is meaningless at 10.

**Fix:** normalize by total contracts so a 1-contract and 10-contract IC use the same threshold. Canonical formula and rationale live in `exit_rules.md`. Note that this is a **per-contract direction-weighted** normalization — institutional desks often use **dollar delta** (`delta × multiplier × spot`) instead. Both are valid; pick one and stay consistent.

## 14. Exit + close double-fire

**Shape:** an exit rule fires on tick N. The close order is submitted. On tick N+1, before the close fills, the same exit rule is still true — and fires again, submitting a second close order.

**Reality:** the position appears to have a pending close order, but the rule engine doesn't check. Two close orders hit the broker; one over-closes the position (goes negative) or double-fills.

**Fix:** when an exit rule fires and submits a close, mark the position as `exit_pending` until the close fills or fails. The rule engine skips positions in `exit_pending`.

## 15. The intraday RTH bar inclusive-upper-bound bug

**Shape:** `intraday_profit_pct` fires on the 16:00 bar and closes the position at the first after-hours print (usually the worst quote of the day).

**Reality:** minute bars are left-labeled. A bar stamped `16:00` is the first bar *after* the 16:00 close, not the last bar of RTH. Including 16:00 in RTH includes the first after-hours print.

**Fix:** RTH check uses an exclusive upper bound: `[09:30, 16:00)`. See `exit_rules.md`.

## 16. Stale deployed binary makes fixes invisible

**Shape:** a bug is fixed, CI passes, the binary is built and deployed. The bug still shows up in production.

**Reality:** the running process is mapped to the old binary in memory. The new file is on disk, but the service wasn't restarted. `ls -la /proc/$(pgrep server)/exe` shows the discrepancy.

**Fix:** verify before trusting "my fix is live." See `replay_determinism.md` for the `grep -a` and `/proc/$/exe` technique, and the version-tag sanity check for handshake/log output.

## 17. Pending orders that don't replay

**Shape:** replay skips open orders that were `SUBMITTED` but not `FILLED` at the start of the replay window. The replay runs as if those orders didn't exist, while in reality they were alive and could have filled during the window.

**Reality:** replay must include pending orders — option limit orders, pending spread orders, two-phase stop-limit orders — not just filled positions.

**Fix:** at the start of a replay window, include all non-terminal orders and evaluate them against the replay's price path. Track open gaps in a known-issues list; this is usually a "partially implemented" area.

---

When a new bug reaches production, check this list first. If it matches one of these shapes, the same class of bug has been fixed before — review the previous fix and understand why it didn't prevent the recurrence before patching.
