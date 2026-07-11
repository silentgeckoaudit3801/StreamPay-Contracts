# StreamPay - Accrual and Settlement Specification

**Document:** `docs/accrual-spec.md`
**Contract:** `StreamPay-Contracts` (Soroban / Rust)
**Version:** 0.2.0 (`VERSION = 2_000`)
**Status:** Normative for the implemented `src/lib.rs` public surface

---

## 1. Overview

A StreamPay stream escrows SEP-41-compatible tokens from `payer` to the contract
at creation time. While the stream is active, value accrues toward `recipient`
at `rate_per_second`, bounded by the remaining escrowed `balance`.

Settlement is now an accounting step, not the final token transfer:

```text
settle_stream  -> balance -= accrued, claimable_balance += accrued
withdraw_stream -> transfer claimable_balance from contract to recipient
```

This replaces the older v0.1 text that described no token transfers, no pause
surface, and no claimable pool.

---

## 2. Stream State

`StreamInfo` stores the operational state for one stream:

| Field | Meaning |
| --- | --- |
| `payer` | Account that creates, funds, starts, stops, pauses, resumes, cancels, archives, and updates rate. |
| `recipient` | Account allowed to withdraw claimable tokens. |
| `token` | SEP-41-compatible token contract used for escrow and withdrawal. |
| `rate_per_second` | Token units accrued per second; must be positive. |
| `balance` | Escrowed but not-yet-accrued balance. |
| `claimable_balance` | Accrued balance owed to recipient but not yet transferred. |
| `start_time` | Timestamp for the current unsettled accrual window. |
| `end_time` | Timestamp recorded when the stream is stopped/cancelled. |
| `is_active` | Whether the stream is running or logically paused. |

The core accounting invariant is:

```text
balance + claimable_balance <= initial_deposit
```

`create_stream` transfers `initial_balance` from the payer into the contract
before the stream is stored.

---

## 3. Linear Accrual Formula

For a running stream with `start_time = T0` and current ledger timestamp `now`:

```text
elapsed = now - start_time
raw_accrued = elapsed * rate_per_second
accrued = min(raw_accrued, balance)
```

The implementation uses saturating arithmetic:

```rust
let amount = (elapsed as i128)
    .saturating_mul(info.rate_per_second)
    .min(info.balance);
```

`rate_per_second` and `initial_balance` are required to be positive at creation.
The `.min(info.balance)` cap prevents over-accrual even if elapsed time or rate
is extremely large.

---

## 4. Settlement

`settle_stream(stream_id)` is permissionless. If the stream cannot accrue, it
returns `0`. Otherwise it:

1. Computes `accrued` for the elapsed window.
2. Decreases `balance` by `accrued`.
3. Increases `claimable_balance` by `accrued`.
4. Advances `start_time` to the settlement boundary.
5. Returns `accrued`.

`batch_settle` applies the same semantics to up to `MAX_BATCH_SETTLE_SIZE` (25)
stream ids and is all-or-nothing if an item panics.

---

## 5. Withdrawals

`withdraw_stream(stream_id)` requires recipient auth. It implicitly settles any
outstanding accrual before transferring tokens, then transfers the whole
`claimable_balance` from the contract to the recipient and resets
`claimable_balance` to `0`.

For an inactive stream, withdrawal settles the final window from `start_time` to
`end_time` so a payer cannot stop a stream and strand earned tokens.

Calling `withdraw_stream` with no claimable amount is idempotent and returns `0`.

---

## 6. Stop, Pause, Resume, Cancel

| Entry point | Effect |
| --- | --- |
| `stop_stream` | Payer-auth. Marks the stream inactive and records `end_time`. Final earned value is handled by withdrawal. |
| `pause_stream` | Payer-auth. Settles accrued value up to pause, keeps the stream logically active, and stores `paused_at`. |
| `resume_stream` | Payer-auth. Requires `paused_at != 0`, clears it, and restarts accrual from the current ledger timestamp. |
| `cancel_stream` | Payer-auth. Requires active stream, settles accrued value, marks inactive, and records `end_time`. |

Pause/resume is distinct from stop: pause is resumable; stop is the inactive
terminal state before final withdrawal/archive.

---

## 7. Archival

`archive_stream` requires payer auth and only succeeds when:

- `is_active == false`
- `balance == 0`
- `claimable_balance == 0`

The `claimable_balance` guard ensures the recipient has withdrawn earned tokens
before the stream record is removed.

---

## 8. Worked Example

```text
initial_balance = 1_000
rate_per_second = 10
start_time = 100
now = 150

elapsed = 50
accrued = min(50 * 10, 1_000) = 500
balance' = 500
claimable_balance' = 500
start_time' = 150
```

If the recipient immediately calls `withdraw_stream`, `500` token units are
transferred from the contract to the recipient and `claimable_balance` returns
to `0`.

---

## 9. Related Specifications

- `docs/vesting.md` documents the linear vesting helper and tests.
- `docs/error-codes.md` lists the current panic strings emitted by the contract.
- `docs/pause-resume.md` covers the pause/resume lifecycle in more depth.