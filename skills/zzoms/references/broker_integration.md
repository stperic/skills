# Broker Integration — Auth, Circuit Breakers, External Orders

Live broker integration is where the OMS meets the messy outside world. Broker APIs are asynchronous, flaky, rate-limited, and full of quirks. The patterns below are what every production OMS converges to, regardless of which broker.

## The broker adapter interface

A broker adapter is injected into the OMS via a narrow interface:

```
type BrokerAdapter interface {
    SubmitOrder(ctx, order) (brokerOrderID, error)
    CancelOrder(ctx, brokerOrderID) error
    GetOrder(ctx, brokerOrderID) (OrderState, error)
    ListOrders(ctx, accountID, since) ([]OrderState, error)
    ListPositions(ctx, accountID) ([]Position, error)
    StreamExecReports(ctx, accountID) (<-chan ExecReport, error)
}
```

Concrete adapters (per broker) implement this interface. The OMS depends on the interface, not on any specific broker SDK. Broker-specific quirks are encapsulated inside the adapter.

## Token refresh and auth management

Most brokers use OAuth or similar short-lived bearer tokens (30-60 minute TTL). The auth manager pattern:

```
type AuthManager struct {
    fetchToken  func(ctx) (Token, error)  // fetches from broker or upstream auth service
    cache       atomic cache
    refreshLock mutex
}

func (a *AuthManager) GetToken(ctx) (string, error) {
    // fast path: cached and not expired
    if tok := a.cache.Load(); tok != nil && !tok.ExpiringSoon() {
        return tok.String, nil
    }
    // slow path: refresh under lock
    a.refreshLock.Lock()
    defer a.refreshLock.Unlock()
    // double-check under lock
    if tok := a.cache.Load(); tok != nil && !tok.ExpiringSoon() {
        return tok.String, nil
    }
    tok, err := a.fetchToken(ctx)
    if err != nil {
        return "", err
    }
    a.cache.Store(tok)
    return tok.String, nil
}
```

Key points:

- **Fast path is lock-free.** Token reads are on every API call; a mutex there is a throughput bottleneck.
- **Refresh is serialized.** Only one refresh in flight at a time — the mutex + double-check prevents a thundering herd on expiry.
- **`ExpiringSoon` buffer.** Refresh at ~80% of TTL, not at expiry. This absorbs clock skew and network latency.
- **Fetching can go through an external service.** Some brokers require a browser-based OAuth flow; a separate auth service (e.g. a Cloudflare Worker) handles the browser flow and exposes a machine-readable endpoint the OMS polls.

## Credential gating is per-account

The check for "does this call need broker credentials?" is **per-account**, keyed off the account's provider:

```
switch account.Provider {
case "paper", "sim":
    // no broker credentials needed
case "broker_live":
    require(brokerAuthManager != nil, "broker credentials required for live account")
}
```

**Never gate broker credentials at the request level or globally.** A global gate means paper orders start demanding live broker keys, which is a common source of "why can't I paper trade on my laptop?" bugs. The gate is always at the point the broker adapter is actually called.

## Circuit breaker

The broker adapter sits behind a circuit breaker that trips on sustained failures:

```
states: CLOSED → OPEN → HALF_OPEN → CLOSED
```

- **CLOSED:** normal operation. Count failures in a rolling window.
- **OPEN:** too many failures. Fail fast without calling the broker. All submissions are rejected with `BROKER_CIRCUIT_OPEN`.
- **HALF_OPEN:** after a cool-down, allow one probe call. If it succeeds, close. If it fails, re-open with an exponential cool-down.

Thresholds are a tradeoff: trip-happy protects capital and avoids wasted calls, trip-shy avoids false outages and keeps the desk trading through transient broker noise. Starting points to **tune per broker** (these are illustrative, not prescriptive):

- **Trip:** e.g. 5 consecutive failures, or 20% failure rate in the last 100 calls. Note that a 5-consecutive rule will trip on a burst of 5 rate-limit (429) responses during a normal throughput spike — combine with rate-limit aware classification so 429s don't count toward the failure count (retry with backoff instead).
- **Cool-down:** e.g. 30 seconds initial, doubling up to a ceiling.
- **Probe:** single call in HALF_OPEN.

Observe the circuit's behavior in paper and staging for at least a week before committing to a set of thresholds for live. A flapping circuit is worse than no circuit.

**Critical:** the circuit breaker state must be observable. A stuck-open circuit is a silent outage. See `observability.md` for the alerting pattern.

Orders rejected due to open circuit go to `REJECTED` state with reason `BROKER_CIRCUIT_OPEN`. They do *not* sit in `SUBMITTING` — that would inflate the SUBMITTING backlog and hide the problem.

## External order detection

Users may place orders through the broker's own web UI or mobile app. These orders arrive in the broker's order book but never went through the OMS. The OMS must detect and record them:

1. **During exec report streaming:** every exec report includes the broker's client order id. If the OMS doesn't recognize the client order id (not in its orders table), treat it as external.
2. **During reconciliation:** the nightly recon pulls the broker's full order list and compares against the OMS. Unknown orders are external.

Handling external orders:

- **Record them.** Insert into `orders` with `source = 'external'` so downstream reports (P&L, position tracking) see them.
- **Derive positions from their fills.** Same fill pipeline as OMS-originated orders. `ApplyFill` doesn't care where the fill came from.
- **Flag them.** Surface "external orders detected" in the operations dashboard. Some teams want to alert; others (more common) just log and review daily.
- **Do not try to manage them via OMS exit rules by default.** The user didn't opt them into a closing strategy. An opt-in mechanism (e.g. a UI action that attaches a closing strategy to an external order) is the right pattern if the team wants it.

## Idempotency across reconnects

The broker adapter must be idempotent on reconnect. Two cases:

1. **OMS restart while broker stream is live.** On restart, the OMS loads all non-terminal orders (see `order_lifecycle.md`) and queries the broker by client order id to reconcile state before reopening the stream.
2. **Broker stream reconnect.** Some broker APIs drop the stream briefly. On reconnect, the OMS must re-request any exec reports missed during the gap (most brokers support a `since_seq_num` parameter). Dedupe by the broker's **stable exec-unique id** — in FIX, `ExecID` (tag 17), which is guaranteed unique per execution within a day by the venue. REST brokers typically surface an equivalent `execution_id` field.

The fill repo's unique constraint is on `(broker_exec_id)`, not on `(order_id, fill_seq)`. Rationale:

- `broker_exec_id` is the broker's own stable identifier for each execution; the broker guarantees uniqueness.
- `(order_id, fill_seq)` is fragile: `order_id` is the OMS-side id, and correlating OMS ids to broker exec reports on reconnect is a join, not a constraint hit. Worse, some brokers reset `fill_seq` on reconnect.
- A duplicate insert on `broker_exec_id` fails harmlessly with a unique-constraint violation — that is the entire idempotency mechanism.

Also persist the (OMS) `order_id` and a derived local `fill_seq` for ordering and display, but the **uniqueness constraint is on the broker's exec id**.

## Rate limiting

Brokers rate-limit API calls (typically 100-200 req/sec for REST, fewer for order submission). The adapter:

- **Tracks quota per endpoint.** Order submission, order status, market data all have separate quotas.
- **Backs off on 429.** Exponential backoff with jitter.
- **Shares quota across the OMS.** A single token-bucket limiter inside the adapter, not per-caller.
- **Doesn't queue submissions.** Quota exhaustion rejects the submission immediately — a queued submission creates the exact "is this submitted or not?" ambiguity the `SUBMITTING` state is designed to avoid.

## Session management

Some brokers require an explicit session start (e.g. after a nightly disconnect window). The OMS must:

- **Start the session before the trading day opens.** Driven by a scheduled job, not by the first order.
- **Fail loudly if session start fails.** A failed session start means no trading today — this is a page-oncall alert, not a retry-silently condition.
- **Tear down cleanly at session end.** Cancel open GTC orders that the broker won't carry across the session break (broker-specific; check the docs).

## Browser-flow OAuth via an external auth service

Some brokers require browser-based OAuth that can't be automated from a headless server. The workaround pattern:

1. A small external service (e.g. a Cloudflare Worker) hosts the OAuth callback.
2. A human completes the browser flow once; the worker caches the resulting refresh token.
3. The OMS fetches access tokens from the worker on demand, using a worker-specific API key.
4. The worker refreshes the refresh token on a schedule, before it expires.

This decouples the OMS from the browser flow and avoids the "token expired at 2 AM and oncall had to do a browser dance" problem.

## What the adapter should *not* do

- **Exit rule evaluation.** That's OMS concern, not broker concern.
- **Position bookkeeping.** The OMS derives positions from fills; the adapter just passes fills through.
- **Retry business logic.** The adapter retries transport failures (timeout, 5xx); it does not retry business rejections (insufficient buying power, halted symbol) — those must surface to the OMS.
- **Caching of reference data.** Symbol metadata, contract specs, etc. live in the market-data layer, not the broker adapter.
