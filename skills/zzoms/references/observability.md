# Observability — Alerts, Metrics, and Runbooks

The OMS is the place where silent failures cost the most. The goal of observability is to make every failure mode either loud or detectable within minutes, not hours.

## Alert taxonomy

Alerts are tiered by blast radius:

| Tier | Response | Examples |
|---|---|---|
| **CRITICAL** | Page oncall immediately | Broker circuit open > 5 min, reconciliation drift on positions, account-level kill switch fired, replay test failed in CI on main branch |
| **WARN** | Daily review, possibly automated remediation | Stuck SUBMITTING orders, missed fill replayed, single-symbol Greeks failing for a day, external order detected (if alerting enabled) |
| **INFO** | Log, weekly review | External order detected (default), corporate action applied, recon run completed with N drifts |
| **DEBUG** | Log only, no review | Per-tick exit rule decisions, cache hits, routine loop ticks |

**Critical alerts must block trading or page someone — no exceptions.** An alert that nobody answers is worse than no alert, because it trains the team to ignore the channel.

## Submitting-stuck detector

Any order in `SUBMITTING` state for longer than ~2 minutes is a stuck order. See `order_lifecycle.md` and `reconciliation.md` for the detector and remediation. Metric:

```
oms.orders.submitting.age_seconds (histogram)
```

Alert when the 99th percentile exceeds 120 seconds for 5 minutes. The alert message should include the affected account keys and order ids.

## Circuit breaker state

The broker circuit breaker state must be observable:

```
oms.broker.circuit.state (gauge: 0=CLOSED, 1=HALF_OPEN, 2=OPEN)
oms.broker.circuit.failures_total (counter)
oms.broker.circuit.trips_total (counter)
oms.broker.circuit.open_duration_seconds (gauge)
```

Alert on state = 2 (OPEN) for longer than a trip-window threshold. A flapping circuit (many trips in a short window) is also alert-worthy.

## Reconciliation drift

Every recon run emits:

```
oms.recon.drift_count{type=position|fill|order, severity=critical|warn|info} (gauge)
oms.recon.run_duration_seconds (histogram)
oms.recon.last_success_timestamp (gauge)
```

Alerts:

- **Position drift > 0:** CRITICAL. Auto-block trading on affected accounts.
- **Fill drift > 0:** WARN (auto-remediated, but monitor for trends).
- **Recon run hasn't completed in > 25 hours:** CRITICAL. The safety net is down.

## Exit rule metrics

Every exit rule firing produces:

```
oms.exit_rules.fired_total{rule=stop_loss|delta_exit|...|expiration, account_key=...} (counter)
oms.exit_rules.evaluation_duration_seconds (histogram, per kit)
oms.exit_rules.needs_greeks_count (gauge) -- how many positions are paying the Greeks hot-path cost
```

Monitor the `needs_greeks_count` gauge — if it grows unexpectedly, the hot path is doing more work than intended and the gate logic may be broken.

Monitor tie-breaking: if `trailing_stop` never fires but `loss_pct` fires constantly, either the trailing stop thresholds are wrong or the priority order is wrong. The aggregate firing distribution over a month is a good review artifact.

## Position monitor loop metrics

```
oms.monitor.tick_duration_seconds (histogram)
oms.monitor.positions_evaluated (counter)
oms.monitor.greeks_failures_total (counter)
oms.monitor.ticks_skipped_replay_active (counter)  -- should be 0 outside replay
```

The `ticks_skipped_replay_active` metric is a smoke test: if it's non-zero outside a replay window, something left the as-of boundary set and background loops are silently skipping work.

## Replay determinism tests in CI

The canonical replay test (see `replay_determinism.md`) runs on every commit that touches the OMS. When it fails:

- **Fail the PR.** No merge until the test passes.
- **Post the diff to the alert channel.** The diff reveals which field drifted, which usually points to the bug category (UUID tiebreaker, map iteration, unguarded loop).
- **Include a one-line triage guide** in the alert message pointing at `replay_determinism.md`.

This test is the only mechanism that reliably catches determinism regressions. Without it, the bug reaches production and shows up as "my backtest numbers changed since Monday."

## Tracing

Every order submission carries a trace ID that threads through:

- HTTP/RPC handler
- Validation pipeline
- Broker adapter call
- Fill pipeline on exec report
- Position update
- Exit rule evaluation (if fires)

The trace lets you answer "what happened to this order?" in one query. Without it, the answer requires correlating log lines across services.

## Structured logging

Every log line has:

- `trace_id` for correlation.
- `account_key` for account-scoped filtering.
- `order_id` or `position_id` where applicable.
- `event` — a stable machine-readable tag (`order.submitted`, `fill.applied`, `exit_rule.fired`).
- `reason` — free-form human explanation.

Do not log the full order or position object by default. Logs are sampled and stored; dumping a 5KB object per tick eats the log budget fast. Log the ids and query the DB for details during investigation.

## Notification routing

Alert routing depends on team size and rotation model:

- **Solo / small-team:** lightweight push services (Pushover, Telegram bots, Slack webhooks) for everything, with CRITICAL alerts on a high-priority channel that bypasses notification-silencing.
- **Team with oncall rotation:** PagerDuty or Opsgenie for CRITICAL (with an escalation policy), Slack for WARN, log-only for INFO.
- **Regulated environment:** audit-logged notification delivery for anything that blocks trading or touches a live account.

Typical mapping:

- **CRITICAL:** paging service with a runbook link in the payload.
- **WARN:** team chat channel (batch if multiple alerts fire in a short window to avoid alarm fatigue).
- **INFO:** separate chat channel or log-only.

Notifiers are injected via the `Notifier` interface (see `architecture.md`). A test notifier in dev logs to stdout; production wires the real one. Never hardcode the notifier choice — the interface is what lets you swap providers without touching alert logic.

## Runbooks

Every CRITICAL alert must have a runbook. Runbook structure:

1. **What the alert means.** One sentence.
2. **What state to check.** The exact queries or commands.
3. **What actions are safe.** "You can restart the recon service. Do not manually edit `trading_positions`."
4. **Escalation.** Who to call if the situation isn't covered by the runbook.

Runbooks live in-repo next to the code they cover. A runbook that references a function that got renamed is a runbook that will fail in production.

## Deployment verification — is the running binary actually the fix?

A recurring class of "the fix didn't work" reports comes from a stale deployed binary: the file on disk is updated but the running process is still mapped to the previous version in memory. Before trusting "my fix is live," verify the process itself.

1. **Check the mapped binary vs the file on disk.** On Linux:
   ```
   ls -la /proc/$(pgrep <server>)/exe
   stat /opt/app/<server>
   ```
   `/proc/<pid>/exe` is a symlink to the file the process was launched from. If the file on disk was replaced since launch, the symlink resolves to the old inode (shown as `(deleted)` in `ls -la`). That means the running process is still executing the old binary. Restart is required.

2. **Grep the ELF for source-level markers.** For unstripped Go binaries, `grep -a` on the binary finds identifiers directly — no `go version -m` needed:
   ```
   grep -aoE "MyReplayFix|NewExitRule|ExpectedSymbol" /opt/app/<server> | sort -u
   ```
   If the grep doesn't find your new identifier, the binary on disk doesn't contain the fix and the build step was skipped or pointed at the wrong commit.

3. **Check the version tag in handshake or log output.** If the service reports a version tag (commit hash, build timestamp) on startup or client handshake, confirm it advances on every deploy. A tag that's stale or frozen across restarts means downstream clients are trusting a version number that's lying about what's actually running.

The verification steps are cheap and deterministic. Run them before debugging a fix that "didn't land." In production, a CI check that asserts the commit hash embedded in the binary matches the deployed-version label catches this class of problem automatically.

## Post-incident review

Every incident produces:

- A timeline reconstructed from logs, traces, and metrics.
- A root cause (not just a trigger — "why did this condition exist?").
- A test that would have caught it (add it to CI).
- An updated runbook or alert threshold if the response was slow or wrong.

The test is the deliverable. A post-incident review that doesn't add a test to CI is a review that will replay the same incident in six months.
