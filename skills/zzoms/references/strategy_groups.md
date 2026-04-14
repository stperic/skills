# Strategy Groups — Multi-Position Tracking, Rolls, Adjustments

Strategy groups track multiple positions as a single logical unit — a short strangle with two legs, an iron condor rolled from one expiry to the next, a covered call laddered across strikes. Without grouping, a user sees 4 rows for an iron condor and has to mentally reassemble the structure every time.

## The group concept

A group is a named, persisted collection of positions with:

- **A strategy label** (e.g. "spy_strangle_0dte", "qqq_ic_weekly").
- **A set of member positions** — each row in `trading_positions` belongs to at most one group via `strategy_group_id`.
- **A lifecycle state** (`OPEN`, `ROLLING`, `CLOSED`).
- **An aggregate P&L** — sum of member position P&Ls, plus any realized P&L from closed members.
- **An aggregate risk view** — net delta, net gamma, combined max loss.

Groups are created when a structure is entered and dissolved when the last member closes. A roll creates a new generation of the same group rather than dissolving and recreating.

## Why groups matter operationally

Without groups:

- **Exit rules fire per-leg.** A delta_exit on one leg of an iron condor closes that leg only, leaving the other three legs as a broken structure with different risk than intended.
- **P&L is per-leg.** The user has to sum four rows to know the IC's P&L.
- **Rolls are ambiguous.** Closing one expiry's IC and opening the next expiry's IC looks like two unrelated events unless they're linked.

With groups:

- **Exit rules fire on the group.** The group is the unit of decision-making for closing strategy rules that operate on net P&L, net Greeks, or time-to-expiry.
- **P&L and risk roll up.**
- **Rolls are explicit.** A roll is a single operation that atomically closes the old group members and opens the new ones under the same group id.

## Group operations

Canonical operations:

- **`CreateGroup`** — create an empty group with a strategy label.
- **`AddPosition`** — add a position to a group. The position must not already belong to another group.
- **`RemovePosition`** — remove a position (typically when a single leg closes without closing the whole structure).
- **`RollGroup`** — atomically close existing members and open new members under the same group id. New members inherit the group's strategy label and closing strategy spec.
- **`AdjustGroup`** — change the set of members without the "close and reopen" semantics of a roll. Used for adding a protective wing to convert a strangle into an iron condor, or rolling a single leg to a different strike.
- **`CloseGroup`** — close all open members and mark the group `CLOSED`.

## Cross-repo atomicity

Group operations touch multiple repos atomically:

- **`groups`** — the group row itself.
- **`trading_positions`** — member positions.
- **`orders`** / **`fills`** — the order and fill records for the close/open.
- **`accounts`** — collateral and realized P&L updates.

A partial failure mid-operation is catastrophic: a roll that closes the old IC but fails to open the new one leaves the account naked. The pattern to prevent this is a **transaction runner**:

```
func (g *GroupService) adjustGroupAtomic(ctx context.Context, groupID ID, fn func(ctx) error) error {
    return g.txRunner.RunTxCtx(ctx, func(ctx context.Context) error {
        // ctx now carries a tx; repos inside the closure pick it up via ConnFromContext
        return fn(ctx)
    })
}
```

Inside the closure, every repo call on an injected top-level repo uses the tx because the repo's `conn(ctx)` helper reads from the context. See `repository_patterns.md` for the full pattern.

**Important constraint:** administrative bulk operations (like `ResetAccount`) reject nested tx execution. If `adjustGroupAtomic` tried to wrap a `ResetAccount` call, it would fail with `ErrResetAccountNested`. This is intentional — bulk ops hold locks across ~10 tables and should never run inside a business transaction. Group operations must not compose with bulk admin ops.

## Rolls

A roll is the most complex group operation because it spans two expiries and must leave the account in a known-good state even if the broker rejects the new order.

Ordering matters:

```
1. Validate the new leg set (pre-trade checks, collateral with the new risk profile)
2. Reserve collateral for the new legs
3. Close the old legs (submit close orders, wait for fills)
4. Open the new legs (submit open orders, wait for fills)
5. Release collateral for the old legs
6. Commit the group record transition
```

Step 2 is the safety net: by reserving collateral before closing anything, the roll fails fast if the account can't support the new position. Without the reservation, an account at the buying-power limit would close the old IC, try to open the new one, fail, and end up unhedged.

**Step 3 and 4 cannot be a single order submission at most brokers.** Multi-leg spreads are broker-specific; some brokers support native spread orders across expiries, many don't. The OMS should support both:

- **Native spread roll** where the broker supports it (one order, atomic at the broker).
- **Serial roll** where it doesn't (two orders, best-effort atomic at the OMS — with retry and rollback on failure).

## Adjustments

An adjustment modifies a group without the close-and-reopen pattern of a roll. Examples:

- Add a long put wing to a short strangle → converts to iron condor.
- Roll a single leg to a different strike (credit collection).
- Scale the group up or down (add or remove quantity).

Adjustments are more dangerous than rolls because the intermediate state (old leg still open, new leg open, net risk undefined) can be catastrophic if evaluated by an exit rule. The pattern:

1. Pause exit-rule evaluation for the group during the adjustment.
2. Execute the adjustment in a single atomic transaction wrapper.
3. Resume exit-rule evaluation.

The pause is implemented as a `group.frozen = true` flag that the position monitor respects. The frozen state must have a timeout so a crashed adjustment doesn't leave the group permanently frozen.

## Group closing strategy

A group carries its own closing strategy spec that overrides per-position specs. Rationale:

- The risk that matters is the group's risk, not each leg's.
- Per-leg exit rules fire on leg prices, which is misleading for multi-leg structures.
- Group-level rules can use group-level concepts (e.g. "close if net credit received < 0.50× entry credit").

When a group has a closing strategy, member positions inherit it (their per-position specs are ignored). Exit rule evaluation runs against the group context. When the group closes, all members close.

## Strategy label overloading

The `strategy` field serves two purposes:

1. **Cost basis segregation** — see `positions.md`. Two strategies running the same underlying have distinct positions.
2. **Group identification** — all legs of an iron condor share the same strategy label and are candidates for grouping.

This double duty is fine as long as the strategy label is stable and unique per logical strategy. Avoid using it as a free-form "notes" field; it's load-bearing.
