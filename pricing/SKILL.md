---
name: pricing
description: >
  Help a SaaS, usage-based, AI, consulting, services, or grant-funded business
  turn its business model into a customer-facing pricing.md and the matching
  Economico setup. Use when asked to design pricing, write pricing.md, map
  pricing to obligations, choose revenue accounts, model subscriptions, usage,
  credits, retainers, milestones, grants, or preview Economico platform fees.
  Hand off to creating-contracts for order forms, invoicing for bills to
  customers, setup-economico if the ledger is not connected, and
  investor-reporting or financial-analyst for analysis.
---

# Pricing

Create a plain-language `pricing.md` first, then set up the billable spine in
Economico. The output should be something a customer can read and something the
ledger can bill.

## Workflow

1. Identify the model: monthly SaaS, committed SaaS, usage API, AI credits,
   agent-native per-call, solo consulting, agency delivery, or grants.
2. Draft `pricing.md` with customer-facing sections: who it is for, plans or
   packages, billing cadence, included usage, overage, setup fees, payment terms,
   and cancellation/renewal terms.
3. Add a final "How this is recorded in Economico" section mapping every charge
   to an obligation `type`, account code, SKU, and billing trigger.
4. In Economico, read before writing: `list_parties`, `list_contracts`,
   `list_obligations`, `list_chart_of_accounts`.
5. If there is no active customer contract, use `creating-contracts` to create
   the customer party, order form, contract, and obligations.
6. For Economico's own cost to the business, call `preview_platform_fees` for
   the relevant month; do not mix this with the customer's product pricing.

## Obligation Map

Use these defaults unless the user's facts say otherwise:

| Model | Obligation setup |
| --- | --- |
| Monthly SaaS plans | One `recurring` monthly plan obligation, `4110` Subscription Revenue; optional onboarding as `one_off`, `4200` Service Revenue |
| Annual or quarterly SaaS | `recurring` subscription, `4110`; implementation as `one_off`, `4200`; mention deferred revenue (`2150`) for upfront annual billing |
| Metered API | Platform fee as `recurring`, `4110`; per-unit meter as `usage`, `4120`, with `metric_name` and `price_per_unit_minor` |
| AI credits | Plan as `recurring`, `4110`; credit top-up as prepaid `one_off` with revenue recognized to `4120` as consumed; token usage is the metric |
| x402 / MPP per-call | Per-call `usage`, `4120`; settlement rail determines the cash/payment-fee leg |
| Solo consulting | Retainer as `recurring`, hourly/project/milestones as `one_off`, all usually `4210` Consulting Revenue |
| Consulting agency | Client retainer/milestones/rebilled specialists as `4210`; vendor subcontractor costs belong to `expense-tracking` under `5200` or another expense account |
| Grant-funded studio | Grant tranches as `one_off`, `4500` Grant and Award Revenue; protocol retainers/bounties as `4210` |

## Economico Setup

For each distinct billable term, create or confirm one customer obligation:

- `create_contract(role="customer", currency, msa_url, order_form_url?, term_length?, payment_terms?, services_scope?)`
- `update_contract_status(id, "active")` only after the user confirms the order form is accepted.
- `create_obligation(contract_id, type, name, sku?, amount_minor_units?, interval?, metric_name?, price_per_unit_minor?, account_code)`

Use minor units: `$49.00` is `4900`. Usage prices that are less than one cent
may need product-level pricing units, e.g. price per 1,000 calls, because
`price_per_unit_minor` is an integer.

## Hand Offs

- Need a legal/order-form workflow: use `creating-contracts`.
- Need to invoice a period or customer: use `invoicing`.
- Need investor metrics from the resulting books: use `investor-reporting`.
