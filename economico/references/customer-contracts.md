# Customer contracts

A customer relationship is a contract bound to a plan's template, with the customer as the
`customer` party. It records what was agreed; recordings on it are the invoices, usage and
payments that happened under it.

## Create one

1. **Party**: `parties.create` with `categories: ["customer"]`. Use a stable id from the source
   (`cus_…` from Stripe, or `cus_<slug>`). Add a billing contact with `parties.add_contact` if
   the founder invoices by email.
2. **Source**: `documents.receive` the signed order form, the accepted checkout (the Stripe
   checkout session or subscription as text), or the published terms the customer clicked
   through. See [evidence](evidence.md).
3. **Contract**: `contracts.create` with the plan's `templateDocumentId`, `parties: { "customer":
   "<partyId>" }`, per-activity `terms` for this customer (start and end dates, any negotiated
   price), `ledgerBindings: { "money": "<primaryLedgerId>" }`, and `sourceDocumentIds`.
4. **Acceptance**: `contracts.record` the `accept` activity with
   `evidence.acceptance: [{ "kind": "signed_doc", "ref": "<document id>" }]` and
   `effectiveAt` = the date the customer accepted. The contract becomes `active`.
5. **Read it back**: `subjects {action: "get", type: "contract", id}` shows `status`, the
   resolved terms and the ledgers it opened; `reports run activity_plan` shows what the timer
   will record.

The [monthly subscription recipe](recipes/customer-monthly-subscription.json) is this sequence
end to end.

## Agreement paper

For a SaaS, API or AI product with no agreement yet, suggest the Common Paper Cloud Service
Agreement (`https://commonpaper.com/standards/cloud-service-agreement/`) with an order form per
customer: it is a free, standard, lawyer-drafted agreement, and its order form carries exactly
the terms a template needs (subscription period, fees, payment terms, usage limits). For
consulting, a services MSA plus a statement of work per engagement. For self-serve checkout, the
product's own terms of service plus the checkout record are the agreement.

Economico stores the agreement and records its terms; it does not draft or sign. Never state
that a customer agreed to something no document shows.

## Recording under it

| Event | Activity | Facts | Evidence |
|---|---|---|---|
| A billing period starts | the scheduled `bill` (recorded by the timer) | — | — |
| A month of service completes | the scheduled `service` (recorded by the timer) | — | service, declared as `elapsed_time` |
| The customer pays | `receipt` | `amount`, `bank`, `billed` (the invoice's occurrence key) | payment: the Stripe charge (`stripe_object`) or the bank line |
| Usage for a period is measured | `usage` | the quantity | service: the usage rollup that produced the number |
| Usage is invoiced | `usage-bill` | `amount`, quantity, `due` | — |

A scheduled bill's occurrence key is `schedule:<period start>:<period end>`; a payment names it
in `billed`. Each recording returns an immutable `activity_statement` document: the invoice or
receipt, readable with `documents {action: "get"}` and shown as a labeled document in the
workspace.

Customers who pay an invoice rather than a card need a due date on the claim, or `aging` lists
the invoice under "unknown due date": give the billing effect `"dueDate": "fact:due"` (a `due`
date fact on a recorded bill) or `"dueDate": "term:due"` (a `due` date term on a scheduled one,
set per contract in `contracts.create` terms).

## Changes over time

- **Price change, upgrade, added seats**: `contracts.amend` with the new terms from an effective
  date, `expectedDocumentId` (the contract's current document) and acceptance evidence. Earlier
  periods keep their terms and their MRR.
- **Cancellation**: record the template's `active → terminated` transition at the effective
  date. Keep history; do not archive the party.
- **Wrong recording**: record again with `correctsOccurrenceId` on the latest occurrence; the
  original stays in the history.
- **Refund of unearned value**: a declared `refund_deferred` or `refund_payable` activity, not a
  negative invoice.

## Pitfalls

- A draft contract refuses money: accept first.
- `billed` must carry the invoice's occurrence key, not its occurrence id.
- A customer on a negotiated price is the same template with a term override, not a new plan.
- Do not create one contract per invoice. One agreement, many recordings.
