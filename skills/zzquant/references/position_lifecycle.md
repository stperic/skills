# Position Lifecycle: OMS, Fills, Reconciliation

## Scope

`execution.md` covers execution *algorithms* — TWAP, VWAP, IS, microstructure. This file covers what happens *after* the order leaves your process: how to track positions, handle fills, reconcile with the broker, detect phantom state, and survive partial fills, reconnects, and corporate actions. Most production trading bugs live here, not in the strategy code.

## Why an OMS is non-negotiable

Even a one-person quant shop needs an Order Management System. The job: be the **single source of truth** for what the account actually holds. Without it, every component of the system has its own opinion about positions and P&L, and in any non-trivial flow they will disagree.

The OMS's job is not to be smart. Its job is to be correct. Accuracy > features.

## What the OMS tracks

1. **Orders** — every order submitted, with status, timestamps, and fills
2. **Positions** — current holdings by symbol (with sign, quantity, average price)
3. **Fills** — individual trade executions
4. **Cash** — cash balance and changes
5. **Margin / BP** — available margin, buying power, maintenance requirements
6. **Account metadata** — equity, NAV, daily P&L

Strategy code should not track positions locally. It asks the OMS.

## The lifecycle

```
Strategy → Signal → Order Request → Risk Check → Broker → Acknowledgment
                                                      ↓
                                                    Fill(s) → OMS Update → Strategy Read
                                                      ↓
                                                    Reconciliation
```

Each arrow is a failure point. Each failure point needs defenses.

### 1. Signal → order request
Strategy emits an intent: "buy 10 SPY at 450 limit". This is a request, not a fact. It has not happened yet.

### 2. Risk check
Before the order leaves the process:
- BP check: do we have the capital?
- Position check: does this violate position caps?
- Exposure check: does this violate factor/sector caps?
- Event check: are we in a halt window?
- Strategy check: is this strategy halted?

Risk checks happen in the OMS / risk layer, not in the strategy. Strategies should not be trusted with their own risk enforcement — it's too easy to forget a check.

### 3. Broker submission
Order is sent to the broker. Response is one of:
- **Acknowledged** — broker has the order
- **Rejected** — broker refused (reason: insufficient funds, bad symbol, market closed, bad price, ...)
- **Timed out** — no response

Critical: **ack ≠ filled**. The order exists but has not traded yet.

### 4. Fill(s)
A single order can produce multiple fills (partial fills). Fills arrive asynchronously, possibly out of order, possibly after the order is "complete".

Each fill updates:
- Position quantity
- Average price (weighted)
- Cash (for the fill amount + commission)
- Realized P&L (if closing)

The OMS must handle:
- **Multiple partial fills** — aggregate correctly
- **Overfills** — broker fills more than you asked for (rare but happens; typically requires manual resolution)
- **Underfills** — order expires or is canceled with only part done
- **Out-of-order fills** — sequence number to detect

### 5. OMS update
The OMS reconciles the fill with its internal state. Two sources of truth must stay aligned:
- The **fill stream** (what the broker tells you happened)
- The **position snapshot** (what the broker reports you hold)

If they diverge, reconciliation logic fires.

### 6. Reconciliation
Periodically (on-startup, on-reconnect, on-demand), the OMS fetches the authoritative position snapshot from the broker and compares to its own state.

- **Match** → all good
- **Mismatch** → alert, investigate, halt new orders until resolved

Reconciliation cadence:
- On startup: always
- On reconnect: always
- Periodically intraday: every N minutes
- On critical events: before daily close, before overnight hold

## Fill handling details

### Average price math
```python
def update_avg_price(existing_qty, existing_avg, fill_qty, fill_price):
    new_qty = existing_qty + fill_qty
    if new_qty == 0:
        return 0, 0  # flat
    if existing_qty * fill_qty > 0:  # same side: averaging in
        total_cost = existing_qty * existing_avg + fill_qty * fill_price
        return new_qty, total_cost / new_qty
    else:  # opposite side: realizing PnL
        return new_qty, existing_avg  # avg stays the same on the remaining
```
(Simplified — real implementations handle sign conventions and fees explicitly.)

### Realized vs unrealized P&L
- **Unrealized**: `(mark_price − avg_price) × quantity` for open position
- **Realized**: accumulated P&L from closed quantities
- Total P&L = realized + unrealized

Never mix them in allocation logic; they have different certainty.

### Commissions and fees
- Charged per trade, per contract, or per share
- Must hit cash and realized P&L at fill time, not at end-of-day
- Separate field in the fill record for audit

### Multi-leg orders (options)
- A vertical spread, iron condor, etc. is conceptually one order with multiple legs
- Legs may fill independently or together, depending on broker
- Partial leg fills create legging risk — one leg filled, other not → unintended naked position
- OMS should track the parent order as well as leg fills, and handle legging exceptions

## Position reconciliation

### The reconciliation query
Ask the broker: "what positions do I hold right now?" Compare against OMS state.

For each symbol:
- **Quantity match**: exact
- **Average price match**: approximate (broker may compute differently; store both)
- **Cash balance match**: exact, if broker reports it

Any divergence > tolerance → alert.

### Common causes of divergence

1. **Fill not received**: broker executed but the fill message never arrived (network drop, broker bug)
2. **Duplicate fill**: same fill received twice; OMS double-counted
3. **Corporate action**: split / dividend / spinoff happened, broker applied it, OMS didn't
4. **Manual intervention**: someone closed a position outside the system
5. **Symbol change**: ticker changed, broker knows, OMS has old symbol
6. **Account-level reconciliation**: broker sweeps to cash, OMS didn't update

### Reconciliation protocol

On mismatch:
1. **Halt new orders** — no new entries until resolved
2. **Log the discrepancy** — full state diff
3. **Attempt automated resolution**: re-fetch fills, check for missed messages
4. **Alert** — if not resolved in N seconds, page on-call
5. **Manual override** — require explicit human action to resume

Never silently "fix" the position to match the broker. You lose the audit trail and mask a bug.

## The reconnect problem

Systems disconnect from brokers. When they reconnect, they must re-establish state without:
- Losing fills that happened during the disconnect
- Double-counting fills that are re-sent on reconnect
- Acting on stale state

### Reconnect protocol
1. **Freeze new order submission**
2. **Fetch authoritative position snapshot**
3. **Fetch order history since last known timestamp**
4. **Replay missing fills into OMS state**
5. **Reconcile**
6. **Resume order submission**

Broker APIs vary in how they support this. Some use sequence numbers; some use timestamps; some require a "replay from" query. Know yours.

## Startup protocol

Every time the system starts, it does NOT start fresh. The broker has state from yesterday:

1. **Fetch current positions from broker** — this is ground truth
2. **Load OMS state from disk** — what we *thought* we had
3. **Reconcile** — if they match, proceed; if not, halt and investigate
4. **Load order history if reconciliation needs it**
5. **Unblock trading only after reconciliation passes**

Skipping startup reconciliation is how phantom positions happen.

## Partial fill handling

A limit order for 100 shares might fill as 40 + 30 + 30 across 5 minutes. For the strategy, this matters:

- Is the fill "good enough" to consider the entry complete?
- Does the remaining quantity need to be chased with a market order?
- Does the partial fill trigger an exit for the filled portion?

Common patterns:
- **Fill-or-kill (FOK)**: either fills immediately in full or cancels — no partial fills at all
- **All-or-none (AON)**: can wait, but must fill fully when it does
- **Minimum quantity**: fill at least N, else cancel
- **Simple limit**: accept any partial fill

Default for short-horizon strategies: expect partial fills, handle them gracefully.

## Cancel/replace semantics

Modifying an open order is actually two operations at the broker level: cancel the old, submit a new. Race conditions:

- Old order fills between cancel-send and cancel-ack
- New order submitted while old is still active — two orders simultaneously live
- Cancel fails (order already filled) but new order is submitted — unintended additional position

Safer approaches:
- **Cancel-first-then-wait-then-replace**: serializes at the cost of latency
- **Atomic cancel/replace** if broker supports it (FIX order_cancel_replace)
- **Reject cancel/replace if ambiguous state** — let strategy code handle the edge case

## Idempotency

Network retries, reconnects, and duplicate messages are facts of life. Operations must be idempotent:

- **Order submission**: use client-side order IDs; broker rejects duplicate IDs
- **Fill processing**: use broker's fill ID; skip if already processed
- **State updates**: derive from an authoritative event stream (fills + corporate actions), never from direct mutations

The golden rule: **any operation can be replayed without corruption**.

## Corporate action handling

Splits, dividends, spinoffs, mergers happen overnight. The OMS must:

1. **Detect the action** — from a corporate actions feed or broker notifications
2. **Adjust positions** — split 2:1 → quantity doubles, cost basis halves
3. **Reconcile with broker post-adjustment** — confirm the broker applied the same adjustment
4. **Update historical data** — any strategy that queries history must see the adjusted series

Options corporate actions are trickier: strike adjustments, symbol changes (AAPL1 temporary symbol after a split), non-standard deliverables. OCC publishes the official adjustments.

## Audit log

Every OMS action writes to an audit log that can be replayed to reconstruct state. Minimum:

```
timestamp | action_type | order_id | symbol | side | qty | price | status | reason
```

At any point in time, the OMS state = replay of the audit log up to that time. This is what makes debugging tractable.

## Risk controls at the OMS layer

1. **Order ID collision check** — never accept two orders with the same client-ID
2. **Position cap enforcement** — OMS rejects orders that would breach caps; strategy is not trusted
3. **BP enforcement** — OMS rejects orders that exceed available buying power
4. **Symbol whitelist** — orders on unknown symbols rejected (prevents typo fat-fingers)
5. **Rate limit** — max orders per minute per symbol/account
6. **Cancel-on-disconnect** — broker-side and client-side dead-man switches
7. **Kill switch** — single flag that blocks all new order submission

## The three-layer architecture

A robust quant trading stack has three layers with clear separation:

1. **Strategy layer** — generates intents ("I want to buy 10 SPY at 450")
2. **Risk / OMS layer** — validates intents, submits to broker, tracks state
3. **Execution layer** — decides how to submit (algo, venue, timing)

Strategies must not touch the broker directly. Execution must not override risk. Clean separation makes the system testable, auditable, and survivable.

## References

- Johnson, Kwong — *Algorithmic Trading & DMA* (OMS fundamentals, FIX protocol)
- Narang — *Inside the Black Box* (quant trading operations overview)
- FIX Protocol specification — https://www.fixtrading.org/ (order lifecycle, message types)
- Harris — *Trading and Exchanges* (microstructure, order lifecycle)
- CME Group, NYSE API docs — exchange-specific order handling
- Interactive Brokers API reference — representative retail broker API with reconnection semantics
