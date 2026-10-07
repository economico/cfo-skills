# Vendor contracts

Model what the company receives under each vendor agreement. A hosting service, API allowance
or domain registration is a right; an invoice, bank payment or personal-card charge is not a
separate right. Payment method must not change the vendor's agreement or price.

## Establish the agreement

Create/reuse the vendor party and preserve source terms, order forms and receipts with
`documents.receive`. Use one contract per real agreement, not automatically one per vendor or
one per receipt. Reuse a template when the same rights and timing recur. Distinct agreements
with the same vendor remain separate; several rights can have the same expense classification.

Define activities with `model: "priced-rights-v1"`, their contractual names, terms/facts and
explicit price treatment. Bind them in a template with the same model. A received service uses
`economics.kind: "service"`, `direction: "receive"`, independent `billing` and `recognition`,
and the expense account/function in `classification`. Choose those with [accounts](accounts.md).
Create the contract with its actual start date, vendor party and source documents, then use
`contracts.accept` with the current document ID, expected draft status and acceptance evidence.
Do not create an `accept` activity or add the founder as a vendor party to encode who paid.

## Record the economics, then the funding

Copy the supplied invoice ID exactly into billing's `sourceFactId`, and the supplied payment
ID exactly into the payment's `sourceFactId`. Do not rename them to match a recipe's conventions.
Identify the right and the obligation or service period. Record delivery/usage from source
facts; record recognition when the service is consumed and billing when invoiced. A bill can
precede or follow recognition. An annual payment does not prove twelve months of service:
retain prepaid value and recognize the evidenced portions under the right's ratable schedule.

A paid receipt can use `receipts.record` for one new billing phase and its full same-time
payment. Otherwise record billing, read `activity_claims`, then use `payments.record` with the
returned claim identity. See [payments](payments.md) for account ownership, reimbursements,
unapplied funds, transfers, provider fees and corrections.

| Evidence-backed situation | Executable example |
|---|---|
| Hosting bill, paid later by bank | [vendor bill](recipes/vendor-bill-paid-from-bank.json) |
| Delivered API service, receipt already paid by bank | [bank receipt](recipes/vendor-receipt-paid-from-bank.json) |
| Delivered service charged to company card, issuer paid later | [company-card receipt](recipes/vendor-receipt-company-card.json) |
| Vendor service paid personally, owner reimbursed later | [founder payment](recipes/founder-paid-expense.json) |

These examples freeze one known charge from their synthetic source. Do not infer a perpetual
price from a one-off receipt. Bind the real agreement's rate/tier/fixed terms, or preserve an
unknown price and request the missing evidence. The same service template can be paid through
any supported account without adding payment activities.

Register personal accounts with `ownerPartyId`; register company cards with their issuer's
`providerPartyId`. A founder payment initially creates an owner claim. Reimbursement settles
that claim with company cash and no expense. Conversion to capital requires separate evidence
and an ownership right, never a silent choice or founder-expenses contract.

## Tax and missing evidence

Use the applicable registered tax code in the right's classification. Supported billing freezes
output, recoverable input, reverse-charge or imported-service tax outcomes; payment settles the
gross claim. For priced service rights, declare `classification.taxTreatment` as `inclusive`
when the evidence states a tax-inclusive total, or `exclusive` for a before-tax price. With a
`non_recoverable` code, this makes tax part of the service cost, including prepayments and
accruals; recoverable input tax stays separate. Recognition uses the first evidenced tax basis,
so a later changed rate requires correction rather than silently changing the cost. Explicit
treatment currently excludes recurring-value metrics, capacity and exercise consequences;
inclusive treatment requires supplier-charged tax. A minimum may sit on an `exclusive` right:
its net shortfall is taxed as its own line item, and tax you cannot reclaim (a
`non_recoverable` code) is added to the shortfall's cost instead. State it on an inclusive
right and authoring refuses it. Do not add hand-written
tax posting activities. A taxed cancellation refund must name the original bill through the
consequence's `taxClaimFact` (a typed claim fact containing claim ID and component). State the
unearned service consideration, net of recoverable tax or including non-recoverable tax; the
kernel derives the tax adjustment from that bill and creates a separate refund claim. Keep
the original payment and settle the refund independently. Earned service, tax-only bills and
minimum-closed periods need a different explicit correction; do not approximate those refunds.

If the receipt does not establish who paid, preserve it and the payable rather than guessing an
account. If evidence does not establish consumption, do not recognize the expense just because
money moved. Verify claims, expense and account/owner balances after recording.

The [annual prepayment recipe](recipes/vendor-annual-prepay.json) uses one insurance
coverage right, separate billing and bank payment, and explicit equal monthly recognition.


## Correct a recorded service

For an independent service obligation, use `contracts.reverse_economics` with the original
occurrence ID and correction evidence, then record the corrected phases with new source facts.
This reverses the whole obligation at the correction date; earlier reports retain the original
activity. Undo dependent payment allocations or payments first, using the payment correction
commands and their evidence requirements. Check that the original claim is cancelled and that
replacement billing, recognition and quantities reconcile. Do not use this service path for
linked lifecycle, capacity, minimum or ownership consequences, or for prior-period restatement.
