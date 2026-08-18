---
name: pricing
description: >
  Customer usage: `4120`, never `4110`; every billable usage line needs a
  non-zero unit price. Prepaid packs and AI credits: one `usage`/`4120`/`prepaid`
  obligation plus `grant_usage_credits` — never a `one_off`, never instantiate
  a usage line via `create_contract_from_plan` (plan lines cannot set prepaid). Help a SaaS,
  usage-based, AI, consulting, services, or grant-funded business turn its
  business model into a customer-facing pricing.md and the matching Economico
  setup. Use when asked to design pricing, write pricing.md, map pricing to
  obligations, choose revenue accounts, model subscriptions, usage, credits,
  retainers, milestones, grants, or preview Economico platform fees. Hand off
  to creating-contracts for order forms, invoicing for bills to customers,
  setup-economico if the ledger is not connected, and investor-reporting or
  financial-analyst for analysis.
---

# Pricing

Create a plain-language `pricing.md` first, then set up the billable spine in
Economico. The output should be something a customer can read and something the
ledger can bill.

## Non-negotiable Customer Usage Setup

Before calling `create_obligation`, apply this account mapping exactly:

- Customer platform or access fee: `type="recurring"` and `account_code=4110`.
- Customer-metered API, credit consumption, or per-call charge:
  `type="usage"` and `account_code=4120`. You MUST use `4120` Usage-Based
  Revenue for this line, never `4110`, even when the same contract also has a
  subscription fee.

Every billable `usage` obligation needs a non-zero `price_per_unit_minor`.
Zero-priced usage records consumption but produces neither a billable amount nor
revenue. Define the metric at the unit you quote, then divide the pack price in
minor units by the number of those units. For example, a $40 pack for 500 renders
is $0.08 per render: `4000 / 500 = 8` minor units, so use
`metric_name="renders"` and `price_per_unit_minor=8`. For a sub-cent rate, keep
the natural metric and express the rate per N units with `units_per_price`. For
example, $0.004 per inference call is exactly `price_per_unit_minor=400` and
`units_per_price=1000` with `metric_name="inference_calls"`.

### Prepaid credit packs

For AI credits or any prepaid pack, a recurring subscription fee can remain a
`4110` line, but the consumption pool is a separate customer `usage` obligation
booked to `4120`, never `4110`:

1. Create **exactly one** `type="usage"`, `billing_mode="prepaid"`,
   `account_code=4120` obligation for that customer's credit pool, with a metric
   name and a non-zero unit price.
2. After the customer contract exists, call `grant_usage_credits` for every
   included or purchased credit grant — always on **that same** `obligation_id`.
   For a $25 pack of 100 credits, the unit price is `2500 / 100 = 25` minor units
   per credit. If they also bought a top-up of 100 more, grant those 100 on the
   same obligation; do not call `create_obligation` a second time. Never set
   `price_per_unit_minor=0` for a billable prepaid credit obligation.

Never represent a credit pack or top-up as a `one_off` obligation: that does not
create the prepaid pool and cannot be drawn down by metering. For one customer's
credit program, create one recurring subscription obligation and one prepaid
`usage` obligation; put included credits and top-ups into grants on that single
usage pool rather than creating another contract, plan, or usage obligation. A
top-up that refills the same credit pool is not a second plan and not a second
usage obligation — it is another `grant_usage_credits` call on the existing pool.

Plan lines cannot encode `billing_mode` today, so `create_contract_from_plan` on
a usage line always materializes **arrears**. Do not instantiate an arrears
usage line and then add a prepaid sibling — that double-charges.

- Subscription plus credits (a published Pro plan with a credit pool): put only
  the `recurring` / `4110` fee on the reusable plan. Instantiate the customer
  contract from that plan, then add exactly one prepaid `usage` / `4120`
  obligation before granting credits.
- Packs only, same price for everyone: create one reusable plan so the public
  catalog is on file — that plan's only line is `usage` / `4120` (never a
  `one_off` pack-purchase line). **Do not** call `create_contract_from_plan` for
  that customer. Create the customer contract directly with exactly one `usage` /
  `4120` obligation using `billing_mode="prepaid"`.

The pack purchase is an invoice plus `grant_usage_credits` on that one pool. Add
overage only when the customer actually agreed to it; every later top-up is
another invoice and grant on the existing obligation.

## Workflow

1. Identify the model: monthly SaaS, committed SaaS, usage API, AI credits,
   agent-native per-call, solo consulting, agency delivery, or grants.
2. Draft `pricing.md` with customer-facing sections: who it is for, plans or
   packages, billing cadence, included usage, overage, setup fees, payment terms,
   and cancellation/renewal terms.
3. Add a final "How this is recorded in Economico" section mapping every charge
   to an obligation `type`, account code, SKU, and billing trigger.
4. In Economico, read before writing: `get_parties`, `get_contracts`,
   `get_obligations`, `get_plans`, `list_chart_of_accounts`.
5. First decide whether this is a shared catalog or a bespoke deal. For a standard
   catalog many customers share, define reusable plans (`create_plan`) and
   instantiate each customer's contract from the selected plan
   (`create_contract_from_plan`) — see "Reusable Plans". Prepaid usage is the
   exception: never call `create_contract_from_plan` on a plan that includes a
   usage line (plans cannot encode `billing_mode`; follow "Prepaid credit packs").
   Do not create a reusable plan for bespoke, negotiated terms; use
   `creating-contracts` to create the party, order form, contract, and
   obligations directly.
6. Complete the customer contract lifecycle. When the user asks to onboard the
   customer, make the terms live/billable, finish the setup, or proceed without
   waiting, activate the contract and run `get_contracts(id)` for an active-status
   read-back. A `draft` or `offer` contract cannot bill, so do not report the
   pricing setup complete until the read-back says `active`. Follow
   `creating-contracts` for the activation rules and completion check.
7. For Economico's own cost to the business, call `preview_platform_fees` for
   the relevant month; do not mix this with the customer's product pricing.

## Obligation Map

Use these defaults unless the user's facts say otherwise:

| Model | Obligation setup |
| --- | --- |
| Monthly SaaS plans | One `recurring` monthly plan obligation, `4110` Subscription Revenue; optional onboarding as `one_off`, `4200` Service Revenue |
| Annual or quarterly SaaS | `recurring` subscription, `4110`; implementation as `one_off`, `4200`; mention deferred revenue (`2150`) for upfront annual billing |
| Metered API | Platform fee as `recurring`, `4110`; per-unit meter as `usage`, `4120`, with `metric_name`, `price_per_unit_minor`, and `units_per_price` when quoted per N |
| AI credits | Subscription as `recurring`, `4110` (plan line only); consumption as one prepaid `usage` / `4120` added on the customer contract (not via `create_contract_from_plan`); included credits and top-ups are two grants on that one pool. Never a `one_off`, never a second usage obligation or top-up plan |
| Prepaid packs (no subscription) | Catalog: one reusable plan whose only line is `usage` / `4120`. Customer: create the contract directly with one `usage` / `4120` / `prepaid` obligation and `grant_usage_credits`. Never `one_off`, never `4110`, never `create_contract_from_plan` |
| x402 / MPP per-call | Per-call `usage`, `4120`; settlement rail determines the cash/payment-fee leg |
| Solo consulting | Retainer as `recurring`, hourly/project/milestones as `one_off`, all usually `4210` Consulting Revenue |
| Consulting agency | Client retainer/milestones/rebilled specialists as `4210`; vendor subcontractor costs belong to `expense-tracking` under `5200` or another expense account |
| Grant-funded studio | Grant tranches as `one_off`, `4500` Grant and Award Revenue; protocol retainers/bounties as `4210` |

## Usage & Prepaid Credits

A `usage` obligation only *prices* the unit (`metric_name` +
`price_per_unit_minor`, optionally per `units_per_price` measured units);
metering records what's consumed and derives the bill.
Pick a `billing_mode` when you define it:

- **Arrears** (default) — each consumption event posts its P&L leg immediately
  and accrues the offset in `1125` Unbilled Receivable until the period is
  invoiced from the `get_usage` rollup.
- **Prepaid** — grant a credit pool up front with `grant_usage_credits`; usage
  draws it down FIFO-by-expiry and `list_usage_credits` shows what's left. This
  is the AI-credit / prepaid-pack model — use it instead of a one-off "top-up"
  revenue line.

`pricing` just sets the obligation up to meter; the metering verbs
(`record_usage`, `grant_usage_credits`, `get_usage`) run in the `invoicing`
money-loop.

## Reusable Plans

A **plan** is a reusable template of obligation lines — a named price list you
define once and instantiate onto many customer contracts. Prefer plans when the
pricing is a standard catalog (tiers every customer shares, e.g. Starter / Pro /
Team, a metered plan, an AI-credit plan); use direct obligations (below) for
one-off, bespoke terms. Prepaid usage is not instantiated from a plan — record
the catalog with `create_plan`, then create the customer's prepaid obligation
directly (see "Prepaid credit packs"). Do not create a separate plan for a
top-up that refills an existing credit pool. Create one plan for each purchasable tier or package:
its lines instantiate together, so do not put mutually exclusive tiers in the
same plan. A standard catalog can therefore contain several plans.

- `get_plans()` / `get_plans(id)` — read existing plans before creating a duplicate.
- `create_plan(name, sku?, description?, lines)` — define the template. Each line
  is `{type, name, sku?, amount_minor_units?, interval?, metric_name?,
  price_per_unit_minor?, units_per_price?, account_code}`, the same shape as an obligation. Record
  bundled allotments (e.g. "100 agentic credits included") in `description`;
  bundled use is not a separate billable line.
- `update_plan(id, name, lines)` — replace the plan's lines. Existing contracts
  are unaffected; each snapshots the plan at the moment it was instantiated.
- `create_contract_from_plan(plan_id, party_id, role="customer", currency,
  msa_url, order_form_url?, status?, overrides?)` — materialize a contract and
  its obligations from the plan in one call. Pass `status="active"` to bill
  immediately, then verify the returned contract with `get_contracts(id)`; use
  `overrides` (target a line by `sku` or `line_index`) to drop a line or adjust
  its amount/quantity/price for this customer without touching the shared plan.

## Economico Setup

For a standard price list, define the plan(s) first, then
`create_contract_from_plan` once per customer. For bespoke, one-off terms, create
the contract and each distinct billable obligation directly:

- `create_contract(role="customer", currency, msa_url, order_form_url?, term_length?, payment_terms?, services_scope?)`
- `update_contract_status(id, "active")` after acceptance, or when the user has
  authorized a completed live/billable setup or told you to proceed without
  waiting; then `get_contracts(id)` and verify `status="active"`.
- `create_obligation(contract_id, type, name, sku?, amount_minor_units?, interval?, metric_name?, price_per_unit_minor?, units_per_price?, billing_mode?, account_code)`
  — `billing_mode` (`arrears` / `prepaid`) applies to `usage` obligations; set
  `auto_invoice` on a `recurring` customer obligation to have the scheduled
  sweep bill and send it each period.

Use minor units: `$49.00` is `4900`. For usage prices below one cent, quote an
exact per-N rate: `$0.004/call` is `price_per_unit_minor=400` with
`units_per_price=1000`.

## Hand Offs

- Need a legal/order-form workflow: use `creating-contracts`, including its
  activation and active-status read-back before calling the setup complete.
- Need to invoice a period or customer: use `invoicing`.
- Changing a live customer's plan (upsell, downgrade, renewal): don't edit the
  obligations in place — `amend_contract` records a new dated version and
  `replace_contract` re-papers it, keeping prior versions for point-in-time
  history. See `creating-contracts`.
- Need investor metrics from the resulting books: use `investor-reporting`.
