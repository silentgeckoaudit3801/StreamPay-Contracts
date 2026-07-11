# Error & Panic Reference

`streampay-contracts` currently signals failures with `panic!` messages. This
is the canonical integration-facing list for the current `src/lib.rs` and
`src/stream.rs` surface.

| Panic message | Triggered by | Root cause / integrator action |
| --- | --- | --- |
| `stream not found` | Any read/update of an unknown id | Use a valid, non-archived stream id. |
| `stream id overflow` | `create_stream` | The 1-based stream counter is exhausted. |
| `rate and balance must be positive` | `create_stream` | `rate_per_second <= 0` or `initial_balance <= 0`. |
| `memo exceeds 32 chars` | Legacy memo-bearing create path | Keep memo bytes within the contract limit. |
| `end_time must be in the future` | `start_stream` | A bounded stream cannot start after its end timestamp. |
| `stream already active` | `start_stream` | The stream is already running. |
| `stream not active` | `stop_stream` | The stream is already inactive. |
| `batch too large` | `batch_settle` | More than `MAX_BATCH_SETTLE_SIZE` (25) ids were supplied. |
| `cannot cancel inactive stream` | `cancel_stream` | Cancellation only applies to active streams. |
| `cannot pause inactive stream` | `pause_stream` | Pause only applies to active streams. |
| `stream already paused` | `pause_stream` | `paused_at` is already set. |
| `cannot resume inactive stream` | `resume_stream` | Resume only applies to active streams that are paused. |
| `stream is not paused` | `resume_stream` | `paused_at == 0`; there is no paused window to resume. |
| `cannot archive active stream` | `archive_stream` | Stop/cancel and finish settlement before archive. |
| `cannot archive stream with unsettled balance` | `archive_stream` | `balance != 0`; unaccrued escrow remains. |
| `cannot archive stream with unclaimed balance` | `archive_stream` | Recipient must withdraw `claimable_balance` first. |
| `rate must be positive` | `update_rate` | New rate must be greater than zero. |
| `rate increase exceeds 10% limit` | `update_rate` | Increases are capped at 110% of the current rate. |
| `expected linear vesting mode` | Test-only assertion | A vesting test observed a non-vesting mode. |

## Notes

- `stop_stream` is payer-only in the implemented surface; older recipient-stop documentation no longer matches `src/lib.rs`.
- `withdraw_stream` is recipient-authenticated and transfers all current
  `claimable_balance` after implicit settlement.
- A v1.0 contract-error enum is still the recommended future integration shape;
  until then, front-ends should map these panic strings to stable user messages.