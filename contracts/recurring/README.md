# sororail-recurring

## Purpose

Recurring is pull-based authorization for subscriptions. The payer authorizes a
cap and a cadence; the payee pulls within it. One authorization is stored per
deployed instance.

The contract never holds funds. The payer grants it a token allowance with
`approve`, and each `charge` moves money directly from payer to payee via
`transfer_from`. Revoking that allowance stops charges immediately, with or
without cancelling here.

**Skipped periods are forfeited, not banked.** Each charge schedules the next at
`now + period_seconds`, so a payee who forgets to charge for three months cannot
then take three payments at once. This is a deliberate consumer-protection
choice.

## Entry points

| Entry point | Caller | Effect |
| --- | --- | --- |
| `authorize` | payer | Records the cap and cadence. Requires payer auth. |
| `charge` | payee | Pulls one period's amount from payer to payee via `transfer_from`. |
| `cancel` | payer or payee | Stops future charges. Revoking the token allowance also stops them. |
| `next_chargeable_at`, `is_chargeable`, `remaining_periods` | anyone | Reads the schedule and what is still allowed. |
| `bump`, `get` | anyone | Extends TTL; reads the full authorization record. |

## Example flow

```text
1. The payer approves the recurring contract address on the token for
   amount_per_period * number_of_periods.
2. client.authorize(&payer, &payee, &token, &amount_per_period,
                    &period_seconds, &Some(max_periods))
3. Once per period, client.charge(&payee) pulls one payment. Before the next
   due time it returns RecurringPeriodNotElapsed; after max_periods it returns
   RecurringExhausted.
4. Either party may call client.cancel(&caller, &caller). The payer can also
   revoke the token allowance directly, which stops charges immediately.
```

Full CLI commands are in
[docs/CONTRACT_INVOKE.md](../../docs/CONTRACT_INVOKE.md), and the token movement
diagram is in [docs/fund-flows.md](../../docs/fund-flows.md).

## Errors

Failures use the shared `Error` enum in
[contracts/common/src/errors.rs](../common/src/errors.rs), re-exported as
`sororail_recurring::Error`. Recurring owns codes 80–99; the canonical table is
[docs/ERRORS.md](../../docs/ERRORS.md).

| Variant | Raised by |
| --- | --- |
| `AlreadyInitialized` | `authorize` on an initialized instance |
| `NotInitialized` | any entry point before `authorize` |
| `IdenticalParties` | `authorize` with `payer == payee` |
| `Unauthorized` | `cancel` by someone who is neither party |
| `InvalidAmount` | non-positive `amount_per_period`, or `max_periods == Some(0)` |
| `InvalidDuration` | zero `period_seconds` |
| `InvalidTimeRange` | `authorize` or `charge` scheduling the next charge past `u64::MAX` |
| `Overflow` | `periods_charged` exceeding `u32` |
| `RecurringCancelled` | `charge` or `cancel` after cancellation |
| `RecurringPeriodNotElapsed` | `charge` before `next_chargeable_at` |
| `RecurringExhausted` | `charge` once `max_periods` is reached |
