# Payments and corrections

Payments record money that moved elsewhere. They never initiate a transfer. A payment identifies
its registered account, direction, amount/currency, counterparty, effective instant, source fact,
evidence and allocations. Contracts determine what is owed; the account determines funding.

## Settle an identified claim

Describe `payments.record`, then read the claim from `activity_claims`. Omit `contract_id`
to find contractless unapplied balances or recoverable owner/card funds. Copy `claimId` and
`component: {originLedgerId, accountCode}` exactly. An allocation contains those fields and a
positive amount. Several claims may share one payment if their party, currency, direction and
ledger agree. Partial payment leaves the rest outstanding. Do not substitute invoice text or
an aggregate party balance for claim identity.

- Company bank/wallet: settles the claim against that account's cash.
- Company card: settles the vendor claim and creates an attributable claim owed to its issuer.
  Register the issuer as `providerPartyId` on the account.
- Personal card/account: settles the vendor claim and creates an attributable owner claim.
  Register `ownerPartyId`; never add that owner to the vendor agreement just to describe payment.
- Reimbursement/card-statement payment: use the company bank to settle the returned funding
  claim IDs/components. The counterparty is now the owner/issuer. No second expense is recorded.

An owner cash contribution first establishes an investment claim under an evidenced
`capital_contribution` exercise; its bank deposit is an independent incoming payment.
See [cash contributions](company-setup.md#cash-contributions-without-new-units). Converting an
existing owner claim to equity instead uses an evidenced `convert_claim` exercise, without
cash. Reimbursement pays the owner claim. Do not silently change one into another.

Keep a supplied payment source ID unchanged in `sourceFactId`; do not reorder its segments or
add a payment suffix. An occurrence key or idempotency key is a separate identity.

## Unapplied money and transfers

For money whose matching claim is unknown, `payments.record` can retain an
unapplied residual (`allocations: []`, or an allocated total below the payment). Preserve its
payment ID. Later `payments.allocate` consumes that payment's funds and names the claims; it
moves no cash. Owner/card advances also retain a separate owner/issuer funding claim for
the unapplied portion. Keep the returned funding claim ID and component for reimbursement;
it belongs to the payment, without an invented vendor agreement. Matching later attributes
the original funding to the matched service but never creates another reimbursement claim.
See [founder payment before the bill](recipes/founder-payment-before-bill.json).

`payments.transfer` records a movement between two identified company bank/wallet accounts. It
creates no invoice or service. Every amount lives only in the ledger of its own currency. For an
exchange between currencies, `amount`/`currency` is what left the source account and
`receivedAmount`/`receivedCurrency` is what arrived: both are evidenced facts, and no rate is
supplied. Exchanging USD 5,000 into CHF 4,500 posts USD 5,000 out of the bank against 3420
Currency exchange in the USD ledger, and CHF 4,500 into the franc account against 3420 in the CHF
ledger. No exchange gain or loss is posted. A transfer or payment that takes an account below
zero is recorded as an overdraft, not refused. Explicit fees remain separate evidenced economics.

To settle a claim in another currency than the payment, give the allocation two amounts:
`amount` is what it discharges in the claim's own currency (never more than the claim's balance)
and `cashAmount` is the payment-currency cash applied. A customer paying a CHF 2,000 invoice with
USD 2,200 states `amount` CHF 2,000 and `cashAmount` USD 2,200, each as a minor-unit string
(`"200000"` and `"220000"`). Do not supply `cashAmount` for a claim in the payment's own
currency. Owner/card funding claims and unapplied residuals stay in the payment currency, and
`payments.allocate`/`payments.unallocate` take the same two amounts for a cross-currency match.
Never infer one amount from the other with a rate: each must come from evidence.

When the claim is in another currency than the business's functional one, it was booked at
its transaction date's rate, and settling it with cash in another currency realizes the
difference in the same event: the bill or invoice at that booked rate against the cash in the
functional currency goes to 7150 Realized foreign-exchange loss or 7250 Realized
foreign-exchange gain, attributed to the claim. You record only the evidenced amounts; the
kernel derives the gain or loss. Paying from a third currency (a CHF card on a EUR bill) values
the cash at the payment date's rate from the shared rate store. If the command answers
`missing_rates`, the store had no rate for that date: ask for the bank statement's rate and
state it as `exchangeRate` (`base`, `quote`, `date`, `numerator`, `denominator`, `evidence`)
rather than inventing one. Undo a match that realized FX whole, not in part.
Claim reads retain the original `component`, `unitCode` and `original` amount while showing
`ledgerId`, `ledgerUnitCode` and `ledgerOutstanding` for the ledger that holds the claim.

## Paid receipts

`receipts.record` atomically combines one new commercial billing phase with its full payment at
the same instant. Pass `billing` (the right, obligation, facts and billing evidence) and `payment`
(the account and payment evidence, without allocations). It returns the economic statement and
payment/funding claims. Delivery/recognition still require their own supported evidence and
phases. For a bill already recorded, use `payments.record` against its existing claim instead.

Use the same idempotency key when retrying the receipt. Changing the key is not a retry strategy:
source identity prevents duplicate billing/payment. A receipt is not evidence that an annual
service was fully consumed on the purchase date.

## Provider services

A processor's fee is a priced service right under its own agreement. The account identifies the
provider party and accepted agreement. Route terms select `rail`, `direction` and `asset`; the
right's fixed/rate/tiered price determines the fee (a measured basis uses `grossAmount`).

`payments.process` takes the independent `payment`, rail, explicit `netAmount`, and service
evidence. It records provider usage, recognition, billing and separate fee settlement exactly
once. A paid receipt may include the same terms in `processing`. Do not also record the fee by
hand. Incoming net is gross minus fee; outgoing net is gross plus fee. Net is signed in the
primary direction, so a fee larger than a tiny incoming payment gives negative net. Provider
taxes and cross-currency fees are currently refused.

## Correct a recorded mistake

- Retry lost responses using the original key before attempting a correction.
- `payments.unallocate` undoes all or part of a **later** `payments.allocate` match. Name its
  allocation ID, claim components, amounts and correction evidence. Funds return to unapplied;
  no cash moves. Reapply them to the correct claims.
- `payments.reverse` reverses a whole independent payment with correction evidence. It restores
  claims and reverses funding. Consumed residual funds or reimbursed/converted funding claims
  must first be restored by supported dependent corrections. Initial allocations made inside
  `payments.record` use whole reversal followed by corrected recording.
- A real refund is a new opposite-direction payment against an evidenced refund claim, not a
  correction that erases the sale. Cancellation creates no cash movement. For an owner/card
  refund, set `refundsFundingClaim` to its original funding claim, use the same account,
  and fully allocate one evidenced refund claim (or the original unapplied advance).
  The returned `recoverableClaim`, when present, is money the reimbursed owner/issuer now
  owes the company. Record its return to the company bank separately. Follow the executable
  [refund after reimbursement](recipes/founder-refund-after-reimbursement.json) example.
  The refund cannot exceed original funding or offset unrelated debts. Reverse dependent
  refunds before their original payment or later-match corrections, and collections before
  their refund. For an advance later matched to a bill, the refund is limited to what was
  actually matched to that agreement/right, net of corrections; future matches do not count.

Read claims, party cash and the balance sheet after correction. The changed record remains in
history, and the correcting events explain the new balances.

## Settle an existing historical invoice

Read the existing contract's `activity_claims` report. Preserve its `claimId`, original
`component`, source identity and remaining balance; allocate a new independent payment to
that claim. Do not recreate the invoice or rewrite the old contract to change its payer.
The original currency ledger must still be active; currency succession remains unsupported.
Historical capital distributions, financing repayments and refund claims retain their original
meaning. An owner-funded payment creates a separately reimbursable claim on the same report.
A historical bill already referenced by an independent payment or allocation cannot be edited;
use an evidenced linked credit for the commercial correction.
