# Reconciliation — Nightly Broker ↔ OMS Comparison

Reconciliation is the safety net that catches every fill, order, and position the OMS missed. It runs nightly (and on-demand) against live-broker accounts and compares the broker's state of record to the OMS's state of record. Drift is an alert.

## Why it's load-bearing

Real OMS systems miss events. Causes:

- **Stream reconnects drop exec reports.** Even with sequence-number replay, edge cases exist.
- **External orders placed via the broker's own UI never flow through the OMS.** See `broker_integration.md`.
- **Crashes during `ApplyFill`.** A crash mid-tx might leave the orders table consistent but the positions table stale.
- **Broker-initiated events.** Early assignment, pin risk, expiration settlement, corporate actions — all can change broker state without a corresponding OMS-originated event.

Reconciliation is how you find out. Without it, drift accumulates silently and the OMS position book diverges from reality over weeks.

## What gets reconciled

Three views, in order of increasing expense:

1. **Position reconciliation** — cheapest. Compare `trading_positions.quantity` to broker's position quantity for each `(symbol, instrument_type, strike, expiry, side)`. Mismatches are the highest-priority alerts because they mean the OMS risk dashboard is wrong.
2. **Fill reconciliation** — medium. Compare `fills` rows to broker's exec reports for the day. Missing fills in the OMS mean a stream gap or crash. Missing fills on the broker side mean a bug (OMS thinks it filled, broker disagrees — this is rare but serious).
3. **Order state reconciliation** — most detailed. Compare order states (`SUBMITTED`, `FILLED`, `CANCELLED`) against broker order states. Mismatches often indicate stuck `SUBMITTING` rows.

Most teams run all three nightly; some run only positions during the trading day (quick sanity check) and the full set overnight.

## The reconciliation pipeline

```
1. Snapshot broker state:
    - list orders since last recon
    - list positions
    - list fills since last recon
2. Snapshot OMS state (same scope) in a read-only tx
3. Diff:
    - orders in broker not in OMS → external order (see broker_integration.md)
    - orders in OMS not in broker → stale/phantom order (stuck SUBMITTING, orphan)
    - fills in broker not in OMS → missed fill; replay through ApplyFill
    - positions mismatching quantity → investigate (could be missed fill, corporate action, early assignment)
4. Remediate or alert (see below)
5. Persist a recon report with drift counts and actions taken
```

## Remediation categories

Each drift type has a default action:

| Drift | Default action | Alert severity |
|---|---|---|
| External order | Record with `source='external'`, log, no alert unless flagged for alerting | INFO |
| Missed fill (broker has it, OMS doesn't) | Insert fill via idempotent `ApplyFill`, log | WARN |
| Phantom order (OMS has it, broker doesn't) | Transition to `CANCELLED` or `REJECTED`, log | WARN |
| Position quantity mismatch with no explaining fill | **Block trading on the account**, alert oncall | CRITICAL |
| Unexplained realized P&L drift | Alert; do not auto-remediate | CRITICAL |

**Critical alerts block trading.** A position mismatch with no explaining event means the OMS's risk view of the account is wrong. Continuing to trade against a wrong risk view is how small drifts become large losses. The unblock is manual after a human investigates.

## Idempotency

Reconciliation must be safe to re-run. Applying a missed fill twice is a corruption; detecting the same drift twice and alerting twice is noise. The pattern:

- **Fills** dedupe on `(broker_order_id, exec_seq)`. Replaying a missed fill is an idempotent insert.
- **Drift records** dedupe on `(recon_run_id, drift_type, entity_id)`. A drift detected in two consecutive runs appears once per run but only alerts on the first detection (subsequent detections are "still drifting," with a different alert policy).
- **Remediation actions** are logged to an audit table with the drift id, so a re-run can tell "I already handled this one."

## Nightly scheduling

Reconciliation runs **after the broker's end-of-day settlement**, not just after market close. Brokers post late fills, apply corporate actions, and settle expirations in the hours after close. A 16:30 recon run misses most of the day's cleanup.

**The settlement-posting time is broker-specific and must be verified empirically**, not taken from docs. IBKR posts late into the evening; some retail brokers don't settle until the following morning. Check when your broker's ledger is actually stable by running a recon at several times and observing when the drift count stops changing.

Typical schedule shape (adjust the nightly time to your broker):

- **Live recon (light):** every 30 min during market hours. Position-only. Detects drift fast for critical alerting.
- **Nightly recon (heavy):** after the broker's observed settlement posting time. Full three-view comparison, with remediation.
- **Weekly recon (deep):** weekend, off-hours. Includes historical drift analysis and trend reporting.

During an active replay, all recon jobs are skipped (see `replay_determinism.md` — background loops are gated during replay).

## The submitting-stuck detector

A sub-check of reconciliation: any order in `SUBMITTING` state for longer than a threshold (e.g. 2 minutes) is stuck. Causes:

- OMS crashed between sending and receiving ack.
- Broker never acked (broker outage).
- Ack arrived but OMS failed to persist the transition.

The detector queries the broker by client order id and reconciles:

```
for each stuck SUBMITTING order:
    if broker has it:
        advance to the broker's state (SUBMITTED / FILLED / REJECTED / CANCELLED)
    else:
        resubmit with the same client order id  (broker idempotency rejects duplicates)
```

Run the detector on a short interval (e.g. every minute). Stuck orders are a leading indicator of broker trouble.

## Corporate action drift

Corporate actions (splits, dividends, mergers, name changes) cause legitimate position drift. A 2:1 split doubles quantity and halves strike on every affected option. The recon must:

1. Apply the corporate action to the OMS side using the `CorporateActionChecker` interface.
2. Compare post-adjustment OMS state to broker state.
3. Alert on residual drift.

If the corporate action is skipped on the OMS side, the recon will see the full drift and may alert critically — which is correct but noisy. Apply corporate actions before reconciliation, not after.

## Reconciliation vs. audit

Reconciliation is about finding drift **now**. Audit is about reconstructing history **later**. They share data sources but have different retention needs:

- Reconciliation needs today's broker state and yesterday's OMS state.
- Audit needs every fill, every order state transition, and every recon report, for 7+ years (Section 17a-4 and similar rules).

Design the fills and orders tables for both: append-only, immutable once written, with the client order id and exec seq as stable identifiers.
