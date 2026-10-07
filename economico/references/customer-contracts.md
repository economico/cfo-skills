# Customer contracts

A customer contract describes the rights the customer receives under one agreement. Its price,
billing and recognition do not depend on which account collects the money. Follow
[modeling products](modeling-products.md) for the rights and template, and
[payments](payments.md) for collection and refunds.

## Establish and accept

Create/reuse the customer party and preserve the signed order form or evidenced checkout and
accepted terms. Bind the template to that customer with their actual terms and ledger IDs.
Several agreements with the same customer are valid. Date `contracts.create` at the agreement's
start, not at today's backfill date. A wrongly dated unused draft can be discarded and recreated;
accepted history is preserved.

Use `contracts.accept` with `contractId`, current `expectedDocumentId`, expected draft status,
acceptance instant, stable source identity and acceptance evidence. Do not record an acceptance
activity. Inspect the resulting contract's parties, rights, terms and status.

## Record independently timed facts

A provided service right uses `economics.direction: "provide"`. For one obligation or period:

- Record `billing` when the charge is invoiced. Include the evidenced `dueDate` when known.
  Advance billing creates a receivable and deferred value, not earned revenue.
- Record `delivery` or `usage` from fulfillment evidence. Record `recognition` from that
  entitlement or evidenced ratable coverage. Recognition before billing accrues unbilled value;
  later billing clears it. Payment timing does not change revenue timing.
- Read `activity_claims` and collect through `payments.record`, using the exact claim ID and
  component. Partial collections leave an outstanding amount. Unknown matches remain explicit
  unapplied company-bank/wallet funds and can be allocated later.

Use the same right and obligation/period identity across the phases, with distinct source and
occurrence keys. Backfill actual dates and prove each result; do not assume the historical timer
or `activity_plan` has generated the new model's phases. Read actual occurrences and statements.

For receipts, `receipts.record` may atomically create a new billing claim and fully settle it at
that same instant. It does not prove delivery or accelerate recognition. Processor services
belong to their own agreement; use supported provider processing orchestration once, never a
second fee activity on the customer's contract.

## Refund terms before acceptance

When the supplied agreement promises an unearned-service refund, declare that consequence
in the template **before creating and accepting the contract**. Acceptance freezes the terms;
`contracts.cancel` does not infer a refund from a source document or a free-text explanation.
For an evidenced, untaxed unearned amount, the template consequence has this shape (use the
actual right key and monetary alias):

```json
{
  "key": "unearned-refund",
  "trigger": { "type": "transition", "action": "cancel" },
  "sourceRights": ["access"],
  "evidence": ["acceptance"],
  "rule": { "type": "refund_unearned", "amount": "fact:unearned_refund", "monetaryLedger": "money" }
}
```

Declare `unearned_refund` as an integer monetary fact on that right. At cancellation, supply
its evidenced amount in that right's facts and the required acceptance evidence. Inspect the
resulting refund claim before allocating the separate outgoing payment to it. For taxed refunds
use the bill-linked refund rules in [vendor contracts](vendor-contracts.md); do not substitute
a bare untaxed amount. Missing terms on an already accepted agreement are a real modeling gap,
not grounds to invent a journal or declare an unapplied payment settled.

## Changes, cancellation and proof

Use `contracts.amend` for evidenced prospective service-term changes, such as price or seat
quantity. Supply the current document ID, effective instant, changed right terms and acceptance
evidence. Scheduled changes must land at exact billing and service boundaries; existing
obligations, calendars, currency and consequence-bearing rights require explicit adjustment
semantics. The new dated version preserves the same right identity and earlier prices. A changed
right needs the agreement’s actual new definition and evidence, not a price-only amendment.
The [seat expansion recipe](recipes/customer-per-seat-subscription.json) proves July service
recorded after an August amendment still uses July’s price. A period is priced by the version in
force when it began, so July billed in arrears on August 1 keeps July’s price even when the
amendment takes effect on August 1.

`contracts.cancel` or `contracts.terminate` records the evidenced lifecycle decision and any
supported consequences. Outstanding receivables remain collectable. An evidenced refund
obligation is separate from its later outgoing payment. Do not infer either the refund amount
or its payment from cancellation alone.

For a factual service correction, use `contracts.reverse_economics` with the original
occurrence and correction evidence, then record replacement phases with fresh source identities.
Preserve the source's effective date; a date alone does not authorize inventing an intraday time.
When a time is required but absent, use the start of that date and state the convention. Keep an
explicitly supplied instant unchanged. Read the [service correction boundaries](vendor-contracts.md)
before reversing dependent or already settled obligations.

Show the founder the accepted agreement, commercial claim balances/aging, recognized revenue,
deferred/unbilled balances and actual collections. Keep unsupported corrections, historical
claim adapters and currency-succession cases visible; do not invent an invoice or payment to
make a balance disappear. The [monthly subscription recipe](recipes/customer-monthly-subscription.json) is validated
with one access right, separate collection and ratable recognition. The [annual plan](recipes/customer-annual-prepaid.json) uses equal monthly recognition
from its yearly price. Other older customer recipes remain historical fixtures until their replacements are validated.


For a fixed recurring service, declare `recurringValue` with the monthly/yearly interval and
start/end date term names. The report uses the same contractual price as billing, including
static seat quantities. Do not declare observed usage as fixed MRR or invent an amount for an
unknown price. Prove month-end MRR/ARR as well as revenue and collection; cancellation changes
later run rate while preserving earlier snapshots.

For equal calendar-period recognition, set `economics.recognition.allocation: "equal_periods"`
on ratable recognition. Its service periods must exactly partition each billing period.
Omitting this declaration allocates by elapsed time; use the agreement’s stated basis.

Term changes are also refused for rights with existing unscheduled obligations; preserve their
original terms until an explicit adjustment path can carry each unfinished obligation safely.

The [consulting recipe](recipes/customer-consulting-hourly-milestone.json) keeps measured hours
and the fixed milestone as two rights. Recognize accepted work, bill the same obligation and
collect its identified claim independently. `revenue_summary` reconciles billed amounts,
attributed collections and earned revenue; unmatched independent receipts are excluded until
assigned to customer claims. Matching preserves the original cash date.
