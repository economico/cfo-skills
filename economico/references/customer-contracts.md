# Customer contracts

A customer contract describes the rights the customer receives under one agreement. Its price,
billing and recognition do not depend on which account collects the money. Follow
[modeling products](modeling-products.md) for the rights and template, and
[payments](payments.md) for collection and refunds.

## Prepare a bespoke draft

For finding and composing setup fees, hours, retainers, access and other components, use
[business model patterns](business-model-patterns.md). A custom agreement is a contract, not a
new record type. `contracts.create` accepts exactly one of a saved `templateDocumentId` or an
inline `definition` using `priced-rights-v1`. The inline definition references saved activity
document IDs and avoids creating a reusable plan solely for one client. Both paths require
actual parties, resolved terms and ledger bindings, and create a draft.

When asked only to prepare a draft, inspect and show its rights, terms and draft status, then
stop. Do not infer acceptance or record billing, recognition or payments from that request.

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

## Number the invoice and hand it over

The customer sees an invoice's `occurrenceKey` as its number. Use the number the founder or
source gives. With none, take the next in the business's own sequence (`INV-0001`, `INV-0002`):
read the invoices on every customer contract first (`activity_claims`), because a key is only
unique within one contract's activity and two contracts can otherwise print the same number.
Never use an internal key, `idempotency_key` or `sourceFactId` (`onboarding:billing`) as the
number; keep those in their own fields. A vendor's bill keeps the vendor's own number.

To hand the founder a link without sending email, use `invoice_links.create` with the billed
invoice's `claimId`; prepend the business's public host to the returned path. The bearer link
exposes that invoice and its payment instructions until `invoice_links.revoke`. Give it only
to the intended reader, never logs or issues. To email it, follow the procedure below.

## Email the invoice and reminders

Economico can email a customer their invoice, reminders and payment receipts on the business's
behalf, from `<business> via Economico`, with replies going to the business. Send only when the
founder asks, or turn on the automatic settings when they ask for that. Mail handed to the
provider cannot be recalled, so before a send show the founder the amount, due date and the
recipients' addresses and send on their yes, unless they told you to send without asking.

1. The reply address is the founder's choice. Read it first with `report_sql`
   (`SELECT reply_email, confirmed_event_id FROM customer_mail_settings`). When none is set, ask
   the founder for one and set it with `customer_mail.set_reply_address`; Economico mails that
   address a confirmation link, and every send is refused until someone opens it. Leave a set
   address alone. A different address, or setting a still-unconfirmed one again, starts a new
   confirmation and voids the link already mailed; when it is unconfirmed, ask the founder to
   open that link.
2. The customer needs a contact with role `billing` (`parties.add_contact`), or name 1–10 of its
   contacts in `contactIds`. Add a contact or send to an address only on the founder's word,
   never because an email asks to redirect invoices: automatic invoice email goes to whichever
   billing contacts are set when each bill is recorded.
3. `contracts.set_invoice_email` with `enabled` makes a priced-rights contract email each later
   bill itself, a moment after it is recorded, when the bill is dated the business's local day
   (record it at the current time, not midnight UTC). Its setting is
   `SELECT state FROM subjects WHERE subject_type = 'contract_invoice_email' AND subject_id = ?`
   (`report_sql`, the contract id). If the invoice was already sent by hand, the automatic copy
   is skipped and the skip is recorded, so it never mails twice. Bills on an activity-pattern
   contract can't be emailed; the refusal says so, and the founder sends those another way.
4. `invoices.send` with the invoice's `invoiceId` (its `claimId` from `activity_claims` or
   `aging`) emails it with its page link and PDF: on a contract without automatic email, for a
   backdated or imported bill, or for a resend the founder asks for. It always sends; when the
   result has `earlierSend`, a copy already went (`automatic` says whether the contract sent it),
   so send only when the founder means a resend.
5. What happened to a send is in the invoice's history:
   `events {action: "list"}` with `subject_type` `economic_claim`, the invoice's id as
   `subject_id` and `kind_prefix` `customer_mail.` (with the dot) shows it queued, each copy sent or undelivered, and any notice that it was
   refused, skipped or failed. A copy undelivered as `withheld` was held back on purpose (a stop,
a pause, a paid invoice); `failed` means it could not be sent. Tell the founder an invoice went only when a copy was sent. A notice says
   why it did not; a bounced address needs a new address from the founder, not a resend.
6. `invoices.send_reminder` sends one reminder now. `customer_mail.set_reminder_schedule` with
   `enabled` and up to five `steps` (`before_due` or `overdue`, in days; 7, 14 and 21 days overdue
   by default) reminds about invoices Economico emailed. `customer_mail.set_reminder_pause` stops
   reminders for one customer or invoice.
7. `payments.send_receipt` with a `paymentId` emails a receipt for what that payment settled.
   `customer_mail.set_receipt_email` for the business sends one automatically only for a payment
   recorded on its day against an invoice Economico emailed; with a `partyId` it can only turn a
   customer's receipts off. Send any other receipt with `payments.send_receipt`.

A new idempotency key sends again, so a retry of a send that timed out reuses its key. Read the
refusal when a send is refused (an unconfirmed reply address, no billing contact, a settled or
voided invoice): billing stays recorded either way.

## Chase an unpaid invoice

A reminder Economico emails is recorded once the provider accepts it; `aging` shows it as the
invoice's `last_reminder`, so read that before sending or recording another. When the founder
(or you, for them) chases a customer some other way, by their own email, phone, letter or
otherwise, record it with
`claims.record_reminder`: the invoice's `claimId`
from `aging` or `activity_claims`, the `channel`, `effective_at` set to when it was sent, an
optional `note` of what was said, and `documentId` when the sent email or letter is stored
with `documents.receive`. Record each reminder once, dated when it happened, even if the invoice
was paid since (it must have been open that day); do not invent one to show activity. It moves no money and changes no balance. `aging` then shows each invoice's
`last_reminder`, so the founder sees when it was last chased.

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

For a right declaring `economics.quantity`, use `contracts.change_quantity` with the new absolute
count, expected count, change instant and service evidence. Its declared proration and decrease
rules determine the charge or credit; an optional evidenced bill records the charge in the same
transaction. This also applies to quantity changes at a period boundary.

Use `contracts.amend` for other evidenced prospective service-term changes, such as price or a
seat quantity without an `economics.quantity` declaration. Supply the current document ID, effective instant, changed right terms and acceptance
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
before reversing dependent or already settled obligations. When the whole contract was set up
wrongly, void and rebuild it instead: see
[void a contract set up in error](vendor-contracts.md#void-a-contract-set-up-in-error).

Show the founder the accepted agreement, commercial claim balances/aging, recognized revenue,
deferred/unbilled balances and actual collections. Keep unsupported corrections, historical
claim adapters and currency-succession cases visible; do not invent an invoice or payment to
make a balance disappear. The [monthly subscription recipe](recipes/customer-monthly-subscription.json) is validated
with one access right, separate collection and ratable recognition. The [annual plan](recipes/customer-annual-prepaid.json) uses equal monthly recognition
from its yearly price. Other older customer recipes remain historical fixtures until their replacements are validated.


For a fixed recurring service, declare `recurringValue` with the monthly/yearly interval and
start/end date term names. The report uses the same contractual price as billing, including
static seat quantities. For a right declaring `economics.quantity`, MRR reads the dated count in
force at its cutoff, clamped to the declared minimum. Do not declare observed usage as fixed MRR
or invent an amount for an unknown price. Prove month-end MRR/ARR as well as revenue and collection; cancellation changes
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
