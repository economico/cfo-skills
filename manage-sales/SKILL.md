---
name: manage-sales
description: "Run the customer and sales loop in Economico: design and run sales, renewal and expansion pipelines matched to your sales motion; send offers and sign customer contracts; bill, email and chase invoices; reconcile Stripe charges, fees and payouts; collect customer payments."
---

# Manage sales and customers

Follow [economico](../economico/SKILL.md) for ground rules (`catalog {action: "describe", name}`
before executing; fetch recipes via `catalog {action: "recipe", name}`; confirm the target business
before writing; never invent figures).

## 1. Research the sales motion and recommend the pipeline

Before opening deals or defaulting to the generic built-in 3-stage `customer` pipeline
(`open`/`qualified`/`proposed`), check existing pipelines
(`subjects {action: "list", type: "pipeline"}`) and research how this business actually sells.
Read `business-model.md`, the product codebase, plans in the price book
(`subjects {action: "list", type: "template"}`), existing contracts, Stripe (checkout/subscriptions
vs quotes/invoices), and mailbox/notes to identify the sales motion:

| Sales motion | Pipeline recommendation | Suggested stages (default `weightBps`) |
|---|---|---|
| **Pure self-serve / PLG / fully metered** (self-sign-up, card or API usage, no negotiated contracts) | **Do not force a per-user sales pipeline.** Record sign-ups directly as contracts and track growth in `saas_metrics` and [plan-and-track](../plan-and-track/SKILL.md). Add a pipeline only when a human-assisted motion exists alongside self-serve. | None (or an `expansion` pipeline only for sales-assisted upgrades) |
| **PLG + sales-assist / expansion** (self-serve entry, human close for team/annual/enterprise) | Customize `customer` (or define `expansion`) for qualified accounts moving to negotiated or annual terms; use `kind: "expansion"` with `predecessorContractId` for upgrades. | `pql` (1500), `discovery` (2500), `proposal` (5000), `security_legal` (7500), `commit` (9000) |
| **SMB / mid-market sales-led** (standard quotes/order forms, default sales pipeline) | Customize `customer` via `pipelines.define`. | `lead` (1000), `qualified` (2500), `demo` (4000), `proposal` (6000), `negotiation` (8000) |
| **Enterprise sales** (multi-stakeholder, POC/pilot, security & legal review, bespoke MSA/order forms) | Customize `customer` or define a separate `enterprise` pipeline alongside `customer`; use `labels` for `segment`, `owner`, `source`, `acv_band`. | `discovery` (1000), `qualification` (2000), `evaluation` (4000), `proposal` (6000), `security_legal` (7500), `negotiation` (8500) |
| **Consulting / services / agency** (scoped SOWs, retainers, milestone contracts) | Customize `customer` via `pipelines.define`. | `inquiry` (1000), `scoping` (3000), `proposal` (5500), `sow_review` (8000) |

- If a custom sales pipeline is already defined in Economico, use its existing stages unless the
  founder asks to revise them.
- If no custom sales pipeline exists yet, recommend the pipeline structure that matches the
  researched motion (or recommend skipping a pipeline for pure self-serve), explain why, and
  confirm with the founder via [asking the founder](../economico/references/asking-the-founder.md)
  before running `pipelines.define`. When the founder gave explicit instructions or is away, apply
  the best-fit recommendation and list the chosen pipeline and stages in the final summary.

## 2. The sales-to-cash loop

1. **Track the deal (pipeline):** Open and stage prospects in the agreed sales pipeline, and
   capture sales calls or proposals as context. Follow
   [pipelines](../economico/references/pipelines.md) and
   [conversation intake](../economico/references/conversation-intake.md).
2. **Make an offer and sign:** Bind a reusable plan from the price book (see
   [product-economics](../product-economics/SKILL.md)) or draft a bespoke contract, link it with
   `opportunities.link`, align `expectedValue` to the signed amount if it changed, and accept with
   `contracts.accept` (declaring `partyCategories: {"customer": ["customer"]}`). Read
   [customer contracts](references/customer-contracts.md).
3. **Bill, email and collect:** Record delivery/usage, recognition and billing independently,
   email invoices and reminders when asked, and settle open claims through
   [payments](../economico/references/payments.md). For Stripe subscriptions, charges, fees and
   net payouts, read [explore Stripe](references/explore-stripe.md).
4. **Renew, expand, set targets and measure:** Track renewals and expansions as linked opportunities
   (`predecessorContractId`), watch `contract_deadlines`, and review `party_contract_values`,
   `saas_metrics`, `aging` (`direction: "receivable"`) and `ledgers {action: "footprint", party_id}`.
   When setting up a sales motion, also recommend pairing it with live sales goals in
   [plan-and-track](../plan-and-track/SKILL.md) — won-value targets on the sales pipeline, per-rep
   quotas (`holder_kind: "party"` keyed on `owner` labels; see `targets-sales-quarter-quotas`), or
   MRR/revenue targets for self-serve motions — recorded proposed-first and confirmed before
   activating.

## References and recipes

- [Customer contracts](references/customer-contracts.md) — bespoke drafts, acceptance, billing,
  invoicing, email delivery, reminders, amendments and consulting
- [Explore Stripe](references/explore-stripe.md) — Stripe subscriptions, invoices, fees and payouts
- Shared foundations in `economico`: [pipelines](../economico/references/pipelines.md),
  [conversation intake](../economico/references/conversation-intake.md),
  [payments](../economico/references/payments.md), [evidence](../economico/references/evidence.md),
  [verify](../economico/references/verify.md)
- Fetch recipes via `catalog {action: "recipe", name}`: `customer-pipeline-to-contract`,
  `targets-sales-quarter-quotas`, `customer-monthly-subscription`, `customer-annual-prepaid`,
  `customer-per-seat-subscription`, `customer-consulting-hourly-milestone`,
  `customer-platform-fee-plus-usage`, `customer-prepaid-credits`,
  `customer-included-usage-overage`, `stripe-gross-net-payout`
