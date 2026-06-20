---
name: invoicing
description: >
  Create, send, void, and reconcile customer invoices in Economico. Use when
  asked to bill a customer, create an invoice, send an invoice, bill usage,
  invoice a retainer, milestone, subscription, grant tranche, onboarding fee,
  or record customer payment. Always prefer billing from an active contract and
  order form whose obligations can be referenced on invoice lines. Hand off to
  creating-contracts when the contract spine is missing and pricing when the
  charge structure is unclear.
---

# Invoicing

Prefer contract-backed invoicing: customer party -> active customer contract ->
obligations -> invoice lines -> send -> payment.

## Workflow

1. Identify the billing event: subscription period, usage period, retainer,
   hourly work, milestone, setup fee, grant tranche, or agent-native per-call
   settlement.
2. `list_parties`, `list_contracts(party_id)`, and `list_obligations(party_id)`.
   Use `creating-contracts` if there is no active contract or no matching
   obligation.
3. Build invoice lines from obligations. Use `quantity_micros` (`1_000_000` =
   1.0) and `unit_price_minor`; the invoice `amount` must equal line totals.
4. `create_invoice(party_id, contract_id, amount, currency, due_date, memo, lines)`.
5. Show the draft invoice and ask before external delivery unless the user has
   explicitly told you to send it.
6. `send_invoice(id, channel)` posts AR/revenue and sends through `flow`,
   `email`, or `link`.
7. Use `record_payment` only when the user provides a real payment event.
   Use `get_invoices` before reconciling or voiding; use `void_invoice` for
   corrections with a reason.

## Line Mapping

- Subscription plan: line points to `recurring` obligation, usually `4110`.
- Usage or overage: line points to `usage` obligation, usually `4120`.
- Setup/onboarding: line points to `one_off` service obligation, `4200`.
- Consulting retainer: line points to `recurring` consulting obligation, `4210`.
- Milestone/hourly/bounty: line points to `one_off` consulting obligation, `4210`.
- Grant tranche: line points to `one_off` grant obligation, `4500`.

For usage prices below one cent, invoice in billing units the obligation can
represent precisely, such as "1,000 API calls" rather than one call.

## Model Notes

- SaaS monthly and annual plans are usually billed in advance.
- Usage and hourly work are usually billed in arrears.
- Annual upfront invoices may create deferred revenue; mention the accounting
  implication even though `send_invoice` posts revenue when sent today.
- x402/MPP per-call revenue may arrive as settlement data rather than a normal
  invoice; if the user asks for settlement accounting, verify the current
  Economico tool surface before inventing a workflow.

## Hand Offs

Use `pricing` to define a new charge model. Use `creating-contracts` to create
the order form and obligations. Use `investor-reporting` after billing to report
MRR, ARR, ACV, retention, burn, and default-alive style metrics.
