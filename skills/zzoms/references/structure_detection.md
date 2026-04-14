# Spread Structure Detection

The OMS needs to recognize multi-leg structures so exit rules, risk dashboards, and roll logic can reason about them as units rather than loose legs. The detection pattern is a **fixed-order chain, first match wins** — not a scoring function.

## The chain

```
vertical_credit
  → vertical_debit
  → iron_condor
  → covered_call
  → straddle
  → strangle
  → calendar
  → diagonal
  → single_leg
  → custom
```

Each detector is a **pure function** over a leg set:

```
detect(legs []Leg) (structure Structure, ok bool)
```

It either claims the group (`ok = true`) or declines. The chain stops at the first detector that claims.

## Why fixed order, not scoring

Scoring functions ("which structure best matches these legs?") are tempting but fatal:

- **Ambiguity.** A 4-leg short strangle + 2-leg long hedge can score as iron condor or custom; a scoring function picks one arbitrarily per run.
- **Non-determinism.** Floating-point score ties break differently under different compiler optimizations.
- **Untestable.** You can't write "assert this leg set detects as strangle" without the entire scoring model.
- **Silent drift.** Adding a new structure accidentally changes scores for existing structures.

A fixed-order chain is deterministic, unit-testable (each detector gets its own table-driven tests), and safe to extend (insert at the correct precedence).

## Detector precedence rationale

Why the order above:

1. **`vertical_credit` before `vertical_debit`.** Same leg shape, distinguished by net debit/credit. Credit verticals are more common in retail premium-selling strategies; the earlier match optimizes for the common case.
2. **`iron_condor` before `covered_call`.** IC is a more specific structure (4 legs with strict strike ordering); it must match before fallbacks can claim it.
3. **`covered_call` before `straddle`/`strangle`.** A covered call (long equity + short call) can look like a short single-leg call if the equity leg isn't considered. Precedence ensures the combined structure is detected first.
4. **`straddle` before `strangle`.** A straddle is a degenerate strangle (same strike). Detect the specific case first.
5. **`calendar` before `diagonal`.** A calendar (same strike, different expiry) is a degenerate diagonal. Detect specific first.
6. **`single_leg` before `custom`.** Single-leg is a well-defined fallback; custom is the "I don't know what this is" bucket.

## Detector skeletons

Each detector follows the same shape — claim or decline based on leg count, instrument type, and strike/expiry relationships.

**Vertical credit (2-leg):**
```
requires: exactly 2 legs, both options, same underlying, same expiry,
          opposite sides (one long one short),
          same type (both calls or both puts),
          net credit (short leg more expensive than long leg)
```

**Iron condor (4-leg):**
```
requires: exactly 4 legs, all options, same underlying, same expiry,
          2 calls + 2 puts,
          each type has one long and one short,
          short strikes between long strikes (strict strike ordering),
          net credit
```

**Covered call (2-leg):**
```
requires: exactly 2 legs,
          one equity long (qty == 100 * option_qty),
          one call short
```

**Straddle / strangle (2-leg):**
```
straddle: 2 options, same underlying, same expiry, same strike, one call one put, same side
strangle: 2 options, same underlying, same expiry, different strikes (typically call strike > put strike), one call one put, same side
```

**Calendar / diagonal (2-leg):**
```
calendar: 2 options, same underlying, same strike, different expiries, same type
diagonal: 2 options, same underlying, different strikes AND different expiries, same type
```

**Single-leg:** exactly 1 option or 1 equity leg. Always claims if leg count is 1.

**Custom:** always claims. Last in the chain — never fails.

## Adding a new structure

1. Write the detector as a pure function.
2. Decide its precedence: more specific structures come earlier.
3. Insert it into the chain at that position (not append).
4. Add table-driven tests for positive cases and adjacent-structure negatives (e.g. "is this a butterfly or an iron condor?").
5. Update any downstream dashboards that enumerate structures.

## Testing

Each detector gets a table-driven test with:

- **Positive cases:** canonical leg sets that should match.
- **Negative cases:** leg sets that look similar but should *not* match (wrong expiry, wrong side, wrong strike ordering).
- **Adjacency cases:** leg sets that should be claimed by a different (earlier or later) detector, asserting that this detector declines.
- **Edge cases:** single leg, zero legs, mixed underlyings, missing quantities.

The adjacency tests are the most valuable — they catch chain-order bugs when a new structure is inserted in the wrong position.
