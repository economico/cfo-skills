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
   price), `ledgerBindings: { "money": "<primaryLedgerId>" }`, and `sourceDocumentIds`. Date
   the envelope's `effective_at` at the start, never today: the contract cannot record anything
   before it. If you created one at the wrong date, `contracts.discard` the draft and create it
   again ([how Economico works](how-economico-works.md#recording-rules-that-matter)).
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

## Recording history

Backfilling past months is one timeline per contract, recorded oldest first. Put every event on
it at its own date, then record them in that order:

- each period's `bill`, occurrence key `schedule:<start>:<end>`, at the period start;
- each `receipt`, at its payment date, `billed` naming the key of the bill it paid;
- each period's `usage` and `usage-bill`;
- each completed period's `service`, same occurrence key as its bill, at the instant
  `activity_plan` gives for it (just before the period end), with the `elapsed_time` service
  evidence the plan shows.

A payment that comes late lands after the next period's bill, and that is right: the order is
the dates, not the periods. Three rules make it matter:

- **Recorded activities go forward.** A `receipt` or `usage` dated before the contract's latest
  recorded activity is refused. Recording every bill first, or finishing one period's payment
  before a later-dated one, leaves the earlier payments unrecordable.
- **A scheduled period needs its bill first.** A `service` before its `bill` is refused
  (`recognize exceeds its available balance`), and so is a `receipt` naming a bill that is not
  recorded yet.
- **The timer runs daily.** It records any scheduled period you leave, but not until its next
  run, so a completed period you do not record stays in deferred revenue for the rest of the
  session. Backfill a contract in one pass, right after accepting it: if the timer runs in
  between and records later periods, the payments dated before them are refused. Report that as
  a gap; do not work around it.

Check it with `reports run activity_plan` for the contract from its start through today: no
period due before today may be left unrecorded, whatever its status. `fixed` is one you skipped;
`blocked` and `contingent` give a `reason` (a missing bill, missing evidence) to fix. Deferred
revenue (2150) should hold only what is billed and not yet earned: the current month of each
monthly plan and the unearned rest of each annual one.

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
