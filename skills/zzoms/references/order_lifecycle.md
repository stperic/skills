# Order Lifecycle — State Machine, Validation, Fills

The order state machine is the backbone of the OMS. Every bug in the lifecycle either adds an illegal transition, loses a transition, or double-fires one. Keep the machine strict and the transitions idempotent.

## Canonical state machine

```
                ┌──────────┐
                │   NEW    │  (created by OrderService, not yet sent to broker)
                └────┬─────┘
                     ▼
              ┌──────────────┐
              │  SUBMITTING  │  (ack pending from broker / paper engine)
              └──────┬───────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌──────────┐
   │SUBMITTED│  │REJECTED │  │ CANCELLED│
   └────┬────┘  └─────────┘  └──────────┘
        │
        ├──────────────┬──────────────┐
        ▼              ▼              ▼
 ┌────────────┐  ┌──────────┐  ┌──────────┐
 │PARTIALLY_  │  │  FILLED  │  │ EXPIRED  │
 │  FILLED    │  └──────────┘  └──────────┘
 └─────┬──────┘
       │
       ▼
 ┌──────────┐
 │  FILLED  │
 └──────────┘
```

Terminal states: `FILLED`, `REJECTED`, `CANCELLED`, `EXPIRED`. Once terminal, never resurrect — a duplicate fill against a terminal order is either a bug or a duplicate message from the broker that must be deduplicated.

## The SUBMITTING state is load-bearing

Most broker APIs are asynchronous: you send an order, receive an ack, and later receive an exec report. If the OMS writes `SUBMITTED` immediately after calling the broker SDK, a crash between the send and the ack leaves the order in an ambiguous state — did the broker receive it?

The `SUBMITTING` state carves out that ambiguity:

- **Persist `SUBMITTING` before the broker call.** The write happens in the same transaction as the order-row insert.
- **Only transition to `SUBMITTED` on confirmed ack.** If the ack arrives, advance. If it never arrives, recon will resolve.
- **On restart, every `SUBMITTING` order is inspected.** Query the broker by client order id. If the broker has it, advance to `SUBMITTED` (or terminal state). If not, re-submit with the same client order id — the broker's idempotency key rejects duplicates.

Without `SUBMITTING`, a restart can either lose orders (OMS thinks it never sent) or double-send them (OMS retries a successful send).

## Idempotency by client order id

Every order carries a client-generated idempotency key (UUID, ULID, or deterministic hash of `(strategy, symbol, timestamp_bucket, legs)`). This key:

1. Goes to the broker as the client order id so duplicate submissions resolve to the same broker order.
2. Is the unique constraint in the OMS orders table: `UNIQUE (account_key, client_order_id)`.
3. Is how reconnect/resubmit distinguishes "is this a retry of a known order" from "is this a new order".

**Never use the broker-assigned order id as the primary key.** It doesn't exist until after the broker accepts the order.

## Validation gates

Order submission runs through validation before transitioning to `SUBMITTING`. Gates are ordered cheapest first so the common failures are rejected without external calls:

1. **Structural validity** — leg count matches structure, strikes monotone where required, expiry in future, quantity > 0.
2. **Instrument resolution** — can we resolve the leg to a tradable instrument?
3. **Entry guardrails** — `max_loss` pre-trade cap (see `risk_controls.md`).
4. **Collateral / buying power** — does the account have the margin/cash for this structure?
5. **Broker-level checks** — day-trade count, account restrictions, halted symbols.

Validation failures write `REJECTED` with a reason code. Never silently drop orders; every rejection is a row in the orders table so backfill and audit work.

## Fill pipeline

```
fill arrives (broker exec report OR paper engine tick)
    │
    ▼
FillRepo.Insert (idempotent on broker_exec_id — see broker_integration.md)
    │
    ▼
OrderService.ApplyFill
    ├─ update order state (PARTIAL → FILLED when qty reached)
    ├─ update trading_positions (see positions.md)
    ├─ compute realized P&L if closing/reducing
    └─ emit position-updated event for downstream (portfolio, monitor)
    │
    ▼
PositionMonitorService sees the updated position on its next tick
```

Key invariants:

- **Fills are append-only.** A correction is another fill, not an update. `FillRepo.Insert` is idempotent on the broker's stable exec id (`broker_exec_id` — FIX `ExecID` tag 17 or the REST equivalent). See `broker_integration.md` for the rationale; using `(order_id, fill_seq)` instead is fragile because some brokers reset `fill_seq` on reconnect.
- **ApplyFill is transactional.** Order state, position row, and realized P&L are updated in a single tx. Partial success is never visible.
- **Order of fills matters for cost basis.** Sort by `(fill_time ASC, broker_exec_id ASC)` deterministically before applying — `broker_exec_id` is the stable tiebreaker. Parallel fill application corrupts cost basis.

## Cancels and amendments

- **Cancel** is a distinct state transition. The cancel request itself goes through `SUBMITTING_CANCEL → CANCEL_SUBMITTED → CANCELLED`. A cancel can race with a fill — if the broker fills before processing the cancel, the order goes to `FILLED`, not `CANCELLED`. The OMS must accept the broker's answer, not force its own.

- **Amend is a lie at most brokers.** Implemented as cancel+new under the hood. Design the OMS accordingly: an "amend" creates a new order row with a new client order id, links back to the original via `replaces_order_id`, and does *not* assume priority or queue position is preserved. If the underlying broker does atomic amendment (rare), the OMS can optimize, but the default model is cancel+new.

## Expiration

Orders with a time-in-force (DAY, GTC, IOC, FOK) have an implicit expiry. The OMS runs a scheduled sweep that transitions expired orders to `EXPIRED`:

- DAY orders expire at session close in the venue's timezone (never UTC, never the OMS host's timezone).
- GTC orders expire at the broker's GTC cutoff (typically 60-180 days; varies).
- IOC/FOK expire at submission time; the broker decides.

**Watch for replay:** during replay, the expiry sweep must use the as-of clock, not wall clock. Otherwise every open order from a historical day is immediately marked expired. This is a recurring bug — gate the sweep on "is replay active?" (see `replay_determinism.md`).

## Reconnect and restart

On OMS restart:

1. Load all non-terminal orders (`NEW`, `SUBMITTING`, `SUBMITTED`, `PARTIALLY_FILLED`).
2. For each, query the broker by client order id.
3. Reconcile: broker state wins. If the broker says `FILLED`, fetch the exec reports and replay them through `ApplyFill`.
4. Background loops (`PositionMonitorService`, `ReconciliationService`) start only after this pass completes.

Skipping step 3 is how "phantom orders" appear — the OMS thinks an order is open, the broker thinks it's filled, and the position book is wrong.
