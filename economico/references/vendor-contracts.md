# Vendor contracts

Every vendor the company pays gets the same spine as a customer: a party, a contract that says
what was agreed, and recordings for each bill, receipt and payment. The contract is what makes a
receipt books-grade: it fixes the account and expense function once, so every later receipt from
that vendor lands in the same place, and it keeps the vendor's terms and pricing as evidence.

## Build the spine once per vendor

1. **Party**: `parties.create` with `categories: ["vendor"]`, id `ven_<domain slug>`, and `iri`
   set to the vendor's website.
2. **Terms**: `documents.receive` the vendor's terms of service and pricing page as text (or the
   signed order form for an enterprise contract), `externalId` `<vendor>:terms:<date>`. The
   receipt email's footer usually links both. Spend research effort in proportion to the spend:
   a $12/month tool needs its receipt and its terms URL, not an afternoon.
3. **Activities and template**: one template per vendor with `accept` plus the activities its
   receipts need, each with its account and function fixed in the effect:
   `"expenseAccount": 5210, "expenseFunction": "cost_of_revenue"`. Pick them with
   [accounts](accounts.md). A vendor whose invoice has lines for different purposes (production
   and staging, API usage and seats) gets one activity per purpose.
4. **Contract**: `contracts.create` with `parties: { "vendor": "<partyId>" }`, the ledger
   binding and the terms document; then record `accept` with acceptance evidence (the terms
   document, or `{ "kind": "other", "ref": "<terms URL>" }` for click-through terms) dated when
   the company signed up.

## Pick the recipe by how the vendor is paid

| Situation | Effects | Recipe |
|---|---|---|
| Invoice on terms, paid later | `accrue_expense` with `dueDate`, then `pay` against the bill's key | [vendor-bill-paid-from-bank](recipes/vendor-bill-paid-from-bank.json) |
| Auto-debited or paid at purchase from the bank | `accrue_expense` and `pay` in one activity | [vendor-receipt-paid-from-bank](recipes/vendor-receipt-paid-from-bank.json) |
| Charged to the company card | `card_expense`; the card statement later `pay_card` | [vendor-receipt-company-card](recipes/vendor-receipt-company-card.json) |
| Paid on the founder's personal card | `related_expense` on a founder reimbursement contract; `reimburse_related` when paid back | [founder-paid-expense](recipes/founder-paid-expense.json) |
| Annual plan paid upfront | `prepay_expense` into 1150, then a monthly `expense` activity | `catalog describe activities.create` |
| Vendor credit on an open bill | `credit_expense` against the bill's claim | — |
| Startup credits (cloud promotional credit) | A promotional allowance consumed before the bill | `catalog describe templates.create` |

A founder-paid expense is owed to the founder, so its contract's party is the founder, not the
store; the store stays visible through the receipt document.

## Recording each receipt

```json
{
  "name": "contracts.record",
  "idempotency_key": "email:render:INV-2026-06-0042",
  "effective_at": "2026-07-01T00:00:00Z",
  "source_document_id": "<receipt document id>",
  "input": {
    "contractId": "<vendor contract id>",
    "activityKey": "bill",
    "occurrenceKey": "INV-2026-06-0042",
    "sourceFactId": "render:invoice:INV-2026-06-0042",
    "effectiveAt": "2026-07-01T00:00:00Z",
    "facts": { "amount": "68000", "due": "2026-07-31" },
    "evidence": { "service": [{ "kind": "other", "ref": "<receipt document id>" }] }
  }
}
```

The occurrence key is the vendor's invoice or receipt number; the `sourceFactId` is
`<vendor>:<kind>:<number>`. The effective date is the invoice date (or the charge date on a
receipt). A receipt that also records payment needs `payment` evidence too.

## Tax on vendor bills

Recoverable GST, HST or VAT on a bill is `accrue_input_tax` with the registration's input tax
code, on the same claim as the bill. Tax the company cannot recover (most US sales tax on
purchases, provincial PST) is part of the cost: include it in the expense amount, or use
`accrue_nonrecoverable_tax`. A foreign service with reverse charge is `self_assess_tax`. If the
company has no tax registration, tax on purchases is part of the cost.

## Pitfalls

- Do not create a vendor contract per receipt. One vendor, one contract (or one per distinct
  agreement), many recordings.
- Do not record a card-charged receipt as a bill to pay: nothing is owed to the vendor, and
  paying it again from the bank double-counts cash.
- A receipt with no company card match, no bank match and no founder answer is a gap: store the
  document and list it in the brief rather than guessing the payment method.
- Receipts arrive in date order per vendor; recordings on one contract must too.
