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
6. `send_invoice(id, channel)` posts AR and revenue and sends through `flow`,
   `email`, or `link` — but annual-plan lines defer to Unearned Revenue instead
   of recognizing on send (see Model Notes).
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

## Metered Usage

Don't invoice usage from a guess — record the consumption as it happens and bill
the rollup:

- `record_usage(obligation_id, quantity_micros, idempotency_key, recorded_at?)`
  logs each consumption event against its `usage` obligation and posts the P&L
  leg immediately. An **arrears** obligation parks the offset in `1125` Unbilled
  Receivable; a **prepaid** one draws down its credit pool. The idempotency key
  makes re-sending an event a no-op — never double-count.
- Prepaid plans: seed the pool with `grant_usage_credits(obligation_id,
  quantity_micros, expires_at?)` and watch remaining with `list_usage_credits`.
- When you bill the period, `get_usage(obligation_id, from, until)` returns the
  metered total — build the usage invoice line from it, reclassing `1125` to AR.

## Model Notes

- SaaS monthly and annual plans are usually billed in advance.
- Usage and hourly work are usually billed in arrears.
- Annual upfront invoices **auto-defer**: when `send_invoice` bills a line whose
  obligation is `recurring` + yearly, it credits `2150` Unearned Revenue
  (party-keyed) instead of the revenue account, and the recognition sweep
  releases it to revenue straight-line over 12 months. Read the schedule with
  `get_revenue_recognition`; post any due month on demand with
  `run_revenue_recognition`. Monthly/quarterly plans and one-offs still
  recognize on send.
- x402/MPP per-call revenue may arrive as settlement data rather than a normal
  invoice; if the user asks for settlement accounting, verify the current
  Economico tool surface before inventing a workflow.

## Hand Offs

Use `pricing` to define a new charge model. Use `creating-contracts` to create
the order form and obligations. Use `investor-reporting` after billing to report
MRR, ARR, ACV, retention, burn, and default-alive style metrics.
