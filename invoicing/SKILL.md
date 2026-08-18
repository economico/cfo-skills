---
name: invoicing
description: >
  Already billed: write nothing; the final answer must state total open AR across all
  customers in dollars. Before billing, list all invoices and inspect matching lines.
  Ad-hoc consulting or security review: append exactly one 4210 obligation with
  create_obligation, never amend_contract or 4200. Before record_payment, get financial
  accounts: pass the matching existing financial_account_id, or do not record payment
  when none exists. Create/send/void customer invoices; reconcile USD or stablecoin
  settlement. Use the seeded test scenario for practice. Hand off to creating-contracts
  if no contract, and pricing if unclear.
---

# Invoicing

Prefer contract-backed invoicing: customer party -> active customer contract ->
obligations -> invoice lines -> send -> payment.

On a practice or first-run dry run, always pass `scenario: "test"` on every
write. That seeded sandbox already exists; omitting the parameter writes the
real books.

For every payment request, use this exact order:

1. `get_invoices` to identify the sent invoice and its unpaid balance.
2. `get_financial_accounts`; read the result before proceeding.
3. Treat the user's direct statement that the customer paid as the real payment
   event. If a matching account exists, copy its returned
   `financial_account_id` into `record_payment`; do not substitute a payment
   receipt or seek separate email or bank confirmation.
4. If no matching account exists, stop. Do not call `record_payment` or create
   an account: leave the invoice sent, report the payment as unrecorded, and ask
   the user to add the bank account or wallet first.

Never omit `financial_account_id`: the fallback creates a default cash account
on the real books, including inside `test` and on receipt/settlement paths.

## Workflow

1. Identify the billing event: subscription period, usage period, retainer,
   hourly work, milestone, setup fee, grant tranche, or agent-native per-call
   settlement.
2. Before any billing write, call `get_invoices()` without a party filter. Use
   `get_invoices(id)` on possible matches to inspect their lines; list summaries
   alone may not include line details. If the same period, milestone, or usage
   window is already invoiced, do not create, send, or amend anything. Report
   the existing invoice and total open AR across all customers in dollars,
   computed from the unfiltered result. Then read `get_parties`,
   `get_contracts(party_id)`, and `get_obligations(party_id)` only as needed.
3. Build every invoice line with its matching `obligation_id`. If an active
   contract is missing, use `creating-contracts`. If an active contract exists
   but agreed work is outside its current obligations, call `create_obligation`
   exactly once to append only that term. Never call `amend_contract` for this
   path: it reissues the whole obligation set and duplicates unrelated grant or
   retainer versions. Professional reviews, audits, and other consulting use
   `4210`; `4200` is for setup/onboarding services, and `4500` is only for grants.
   Use `quantity_micros` (`1_000_000` = 1.0) and `unit_price_minor`; the invoice
   `amount` must equal line totals. Draft-only invoices still require
   obligation-linked lines.
4. `create_invoice(party_id, contract_id, amount, currency, due_date, memo, lines)`.
5. Show the draft invoice and ask before external delivery unless the user has
   explicitly told you to send it.
6. `send_invoice(id, channel="email")` posts AR and revenue and sends the invoice
   by email — but annual-plan lines defer to Unearned Revenue instead of
   recognizing on send (see Model Notes).
7. Use `record_payment` only when the user provides a real payment event **and**
   `get_financial_accounts` already returns an account. Pass that
   `financial_account_id` every time — without it the call creates cash on the
   real books. If the collection rail charged a processor fee, pass it —
   `record_payment(..., payment_endpoint_id?, fee_amount?, fee_account_code?)` —
   so the invoice still settles at **gross** while cash lands **net**
   (`amount − fee`); the fee posts as its own leg (default `5500` Payment
   Processing Fees). Use `get_invoices` before reconciling or voiding; use
   `void_invoice` for corrections with a reason. Cash that arrived before you
   know which invoice it settles is `record_unapplied_cash`, not
   `record_payment`.

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
the order form and obligations. Use `payment-rails` to register payment
instructions, compare routes, and model collection fees. Use
`investor-reporting` after billing to report MRR, ARR, ACV, retention, burn, and
default-alive style metrics.
