# sororail-batch-payout

## Purpose

Batch payout routes one transaction to many recipients: the payroll primitive.
It holds no storage and no funds — there is no `init`, and therefore no
`storage.rs` in this crate, unlike its siblings. Each call pulls straight from
the funder to each recipient and leaves nothing behind, so there is no position
to archive and no TTL to extend.

**All-or-nothing.** A single failed transfer aborts the whole invocation and
the host reverts every transfer that preceded it. Nobody is paid unless
everybody is. The contract relies on that host guarantee rather than attempting
to unwind by hand, and deliberately never catches a transfer error. Batches are
capped at `MAX_RECIPIENTS` (40) so the check fits the wasm budget.

## Entry points

| Entry point | Caller | Effect |
| --- | --- | --- |
| `execute` | funder | Pays a list of `Payment { recipient, amount }` entries atomically. |
| `execute_equal` | funder | Pays the same `amount_each` to every recipient atomically. |
| `preview` | anyone | Validates a batch and returns its total without transferring. |
| `max_recipients` | anyone | Returns the compile-time recipient cap. |

## Example flow

```text
1. client.preview(&payments) validates size, amounts and total without moving
   funds.
2. client.execute(&funder, &token, &payments) requires the funder's signature
   and transfers each payment in order.
   Or: client.execute_equal(&funder, &token, &recipients, &amount_each).
3. If any transfer fails (for example, insufficient balance), the host reverts
   the whole call: no recipient keeps a payment and no receipt is emitted.
```

Full CLI commands are in
[docs/CONTRACT_INVOKE.md](../../docs/CONTRACT_INVOKE.md), and the token movement
diagram is in [docs/fund-flows.md](../../docs/fund-flows.md).

## Errors

Failures use the shared `Error` enum in
[contracts/common/src/errors.rs](../common/src/errors.rs), re-exported as
`sororail_batch_payout::Error`. Batch payout owns codes 100–119; the canonical
table is [docs/ERRORS.md](../../docs/ERRORS.md).

| Variant | Raised by |
| --- | --- |
| `BatchEmpty` | a batch with no recipients |
| `BatchTooLarge` | more recipients than `MAX_RECIPIENTS` |
| `InvalidAmount` | any non-positive amount in the batch |
| `IdenticalParties` | a recipient that is this contract's own address |
| `Overflow` | the batch total exceeding `i128` |

Token-level failures (for example, insufficient balance) are host errors from
the token contract and revert the entire batch; they are not converted into a
typed `Error` variant.
