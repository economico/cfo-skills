---
name: creating-contracts
description: >
  Create customer contracts and order forms in Economico for SaaS, usage,
  AI-credit, consulting, services, and grant-funded businesses. Use when asked
  to create an order form, choose terms, set up a customer contract, map signed
  pricing into obligations, or prepare billing terms before invoicing. Prefer
  Common Paper Cloud Service Agreement for SaaS-like businesses; for consulting
  suggest a services MSA plus SOW and relevant OSCON/OWASP-style attachments.
  Hand off to pricing for pricing.md design and invoicing to bill active terms.
---

# Creating Contracts

Contracts are the agreement; obligations are the billable terms. Build both
before invoicing so every invoice line points back to an accepted order form.

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
2. `list_parties`; reuse the party if it exists, otherwise `create_party`.
3. `list_contracts(party_id)`; reuse an active matching customer contract only
   if the order form already covers the requested billing terms.
4. Draft the order form summary: scope, term, payment terms, line items, usage
   meters, included quantities, overage, cancellation/renewal, and acceptance.
5. `create_contract(role="customer", currency, msa_url, order_form_url?, term_length?, payment_terms?, services_scope?)`.
6. Create one `create_obligation` per billable line. Use revenue account codes:
   `4110` subscription, `4120` usage, `4200` setup/service, `4210` consulting,
   `4500` grants.
7. Move to `offer` while under review. Only `update_contract_status(id, "active")`
   when the user says the order form is accepted or signed.

## Obligation Patterns

- Monthly plan: `recurring`, `interval="monthly"`, `amount_minor_units`, `4110`.
- Annual plan: `recurring`, `interval="yearly"` or monthly billing of an annual
  commitment, `4110`; mention `2150` for deferred revenue if paid upfront.
- Usage meter: `usage`, `metric_name`, `price_per_unit_minor`, `4120`.
- Setup/onboarding: `one_off`, `4200`.
- Retainer: `recurring`, `4210`.
- Milestone, hourly invoice, bounty, grant tranche: `one_off`, usually `4210`
  for services or `4500` for grants.

## Guardrails

Never bill against a draft or offer contract. `invoicing` should use an active
contract and line-level `obligation_id`s. If the requested invoice has no
contract, stop and create or confirm the contract first.
