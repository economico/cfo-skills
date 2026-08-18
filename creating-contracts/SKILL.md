---
name: creating-contracts
description: >
  Create contracts in Economico. Exactly one obligation per line; never amend a
  new contract. Mixed SaaS: subscription 4110, implementation 4200. Vendor LLM
  usage: 5400, never hosting 5300. Use when asked
  to create an order form, choose terms, set up a customer contract, map signed
  pricing into obligations, or prepare billing terms before invoicing for SaaS,
  usage, AI-credit, consulting, services, or grants. Prefer
  Common Paper Cloud Service Agreement for SaaS-like businesses; for consulting
  suggest a services MSA plus SOW and relevant OSCON/OWASP-style attachments.
  Real setup omits scenario; only explicit dry runs use test.
  Hand off to pricing for pricing.md design and invoicing to bill active terms.
---

# Creating Contracts

Contracts are the agreement; obligations are the billable terms. Build both
before invoicing so every invoice line points back to an accepted order form.

Real contract setup is the default: omit `scenario` so the agreement lands in
the real ledger. Only when the user explicitly asks to practice, rehearse, or
dry-run should every write pass `scenario: "test"`; use the seeded sandbox and
do not create another scenario. "Go ahead without waiting" is permission to
complete the real setup, not a request for a rehearsal.

## Terms Selection

- SaaS, API, AI, and hosted software: recommend Common Paper Cloud Service
  Agreement 2.1: `https://commonpaper.com/standards/cloud-service-agreement/2.1/`.
- Consulting and agency work: recommend a services MSA plus a statement of work.
  Attach delivery-specific rules where useful: open-source contribution /
  conference-community style terms for open-source projects, OWASP-aligned
  testing rules for security work, and subcontractor/pass-through terms for
  agency delivery.
- Grants: use a grant agreement or award letter with milestone acceptance,
  tranche schedule, IP/open-source obligations, and payment rail.

Customer `create_contract` requires an allowlisted Common Paper `msa_url` today.
If the user needs non-SaaS paper, still capture the business terms in
`services_scope`, `term_length`, `payment_terms`, and `order_form_url`; note
that the legal paper may live outside the current allowlist.

## Workflow

1. Confirm the counterparty, currency, billing contact, term, payment terms, and
   pricing model. Use `pricing` first if the model is not clear.
2. `get_parties`; reuse the party if it exists, otherwise `create_party`.
3. `get_contracts(party_id)`; reuse an active matching customer contract only
   if the order form already covers the requested billing terms.
4. Draft the order form summary: scope, term, payment terms, line items, usage
   meters, included quantities, overage, cancellation/renewal, and acceptance.
5. `create_contract(role="customer" | "vendor", currency, msa_url?, order_form_url?, term_length?, payment_terms?, services_scope?)`.
   Customer contracts require an `msa_url`; vendor agreements may omit it.
6. Create exactly one `create_obligation` per agreed billable line and choose
   exactly one account by direction and line type. Map every line before calling
   the tool; do not copy another line's account:
   - Customer subscription -> `4110`.
   - Customer usage -> `4120`.
   - Customer setup, onboarding, or implementation -> `4200`, even when sold
     beside a subscription. A one-time implementation line must never use `4110`.
   - Customer consulting -> `4210`; grant -> `4500`.
   - Vendor spend -> a cost/expense account, never customer revenue. Hosting is
     `5300`; AI model or LLM inference usage is always `5400`, never `5300`.
7. Finish the lifecycle. A billable setup is not complete in `draft` or `offer`:
   - If the order form is accepted/signed, or the user asks to onboard, go live,
     make the contract billable, finish the setup, or proceed without waiting,
     call `update_contract_status(id, "active")` after creating the obligations.
   - Use `offer` only when the terms are explicitly still under review or awaiting
     acceptance. Say that it is non-billable and incomplete; never present an
     offer or draft as a finished contract setup.
8. Run `get_contracts(id)` to read the contract back and verify its status is
   exactly `active`. For a requested billable setup, do not report success until
   the read-back is active; if it is not, complete the transition or report the
   blocker explicitly.

## Obligation Patterns

- Monthly plan: `recurring`, `interval="monthly"`, `amount_minor_units`, `4110`.
- Annual plan: `recurring`, `interval="yearly"` or monthly billing of an annual
  commitment, `4110`; mention `2150` for deferred revenue if paid upfront.
- Usage meter: `usage`, `metric_name`, `price_per_unit_minor`, `4120`.
- Setup/onboarding: `one_off`, `4200`.
- Retainer: `recurring`, `4210`.
- Milestone, hourly invoice, bounty, grant tranche: `one_off`, usually `4210`
  for services or `4500` for grants.

## Amendments & Renewals

When a live deal changes, don't edit its obligations in place — contracts are
append-only and versioned so the recurring base is reconstructable at any past
date. Pick by whether the paper stays the same:

- **`amend_contract(id, effective_date, obligations)`** — same contract, new
  dated terms (an upsell, downgrade, added meter). `obligations` is the
  *complete* new set effective from that date (not a delta); the current set is
  retained as a closed prior version. Use for a mid-term change under the same
  order form.
- **`replace_contract(id, effective_date, obligations, msa_url?, order_form_url?, …)`**
  — the old contract is closed out and linked to a fresh superseding one. Use for
  a renewal or re-paper on new terms.

Both require an **active** contract. Prior versions stay queryable, and
`get_obligations(as_of=<date>)` returns the obligation set in force on that
date. Use these only to change an agreement that existed before the request.
Do not amend a contract you just created to correct wording or omitted fields:
validate the inputs before creation, and never duplicate its obligation set.

## Guardrails

Never bill against a draft or offer contract. `invoicing` should use an active
contract and line-level `obligation_id`s. If the requested invoice has no
contract, stop and create or confirm the contract first. Before handing off to
`invoicing`, use the workflow's `get_contracts` read-back rather than assuming a
successful create or status-update call left the contract active.
