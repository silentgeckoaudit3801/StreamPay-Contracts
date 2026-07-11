# Linear Vesting Specification

`StreamPay-Contracts` includes linear vesting behavior in the current test and
helper surface. This document records the implemented arithmetic so integrators
and auditors do not have to infer it from tests alone.

## Formula

The helper `compute_linear_vested(total_amount, duration_seconds, elapsed_seconds)` computes:

```text
capped_elapsed = min(elapsed_seconds, duration_seconds)
vested = total_amount * capped_elapsed / duration_seconds
```

The multiplication uses `saturating_mul`; integer division floors intermediate
vesting amounts. Any rounding remainder is released at the end once
`elapsed_seconds >= duration_seconds`.

## Settlement Behavior

A linear vesting stream releases only the newly vested delta on each settlement.
The tests cover:

- Multiple settlements over the same vesting schedule.
- Rounding for a 1,000 unit / 3 second schedule (`333`, `333`, `334`).
- Full unlock after the duration has elapsed.
- A schedule anchor that persists across stop/start boundaries.

## Operational Semantics

- Vesting is anchored to the first start timestamp rather than each later
  resume timestamp.
- The effective vested amount is capped by `total_amount`.
- The same accounting invariant still applies: released funds reduce `balance`
  and must not cause `balance + claimable_balance` to exceed the original
  deposit.

## Integration Guidance

- Treat displayed vesting progress as ledger-time based, not wall-clock based.
- Expect integer truncation before the final settlement releases the remainder.
- Use `withdraw_stream` semantics from `docs/accrual-spec.md` for the final
  token transfer once vested funds become claimable.