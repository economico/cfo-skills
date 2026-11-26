# Business views and reporting platform

The books and pipelines are queryable. Build each view the founder works from as a custom report,
following [verify](../../economico/references/verify.md#a-question-no-report-answers): `report_sql`
against the documented views, then `report_definitions.publish` with their agreement, pinned where
they want it. A published view is a saved query, so in the workspace it always shows the books as
they are now. Share one with `shared_views.create` and `mode` `live` when someone else needs the
current numbers; the default `frozen` link keeps the numbers of the day it was made. Either link
opens without a login and never expires unless you set `expiresAt`, so set one and say who will
hold the link ([verify](../../economico/references/verify.md#an-answer-the-founder-wants-to-keep)).

## Weighted pipeline report

The `weighted-pipeline` report template (`catalog {action: "recipe", name: "weighted-pipeline"}`)
is a ready definition: open pursuits by pipeline and stage with value and weighted value, weighted
value by expected close month as a chart, and the open pursuits themselves, with a `pipeline`
parameter to show one pipeline (add the business's own custom pipeline keys to its options when
publishing). Check it with `report_format`, preview it, show the founder, and publish it with
`report_definitions.publish` when they agree. Its close-month chart draws sales, purchasing and
fundraising as separate lines and your own pipelines together as a fourth. Each stage rounds on its
own, so a pipeline's stages match `reports run pipeline` for that pipeline and currency to within
one minor unit per stage.

## Good first views

Build one per job the founder named:

- **Pipeline**: open deals by stage, expected close and weighted value.
- **Collections**: open invoices by customer, due date and last reminder.
- **Renewals**: contracts by their next lifecycle date (`contract_deadlines`).
- **Delivery due**: obligations billed but not yet delivered, and delivered but not yet billed.
- **Spend**: costs by vendor and expense function, with unused seats.
- **Customer economics**: revenue, collections and directly attributed cost per customer.

## Jobs to be done across functions

| Area | Job | Skill / Read |
|---|---|---|
| Sales & CRM | Track deals, send offers, bill, email invoices, renewals | [manage-sales](../../manage-sales/SKILL.md), `party_contract_values`, `contract_deadlines`, `aging` |
| Product | Price book, unit cost to serve | [product-economics](../../product-economics/SKILL.md), `unit_economics`, `usage_cost` |
| Purchasing | Evaluate vendors, record bills, cut unused tools | [manage-vendors](../../manage-vendors/SKILL.md), `aging`, `payment_routes`, `vendor_utilization` |
| Hiring & People | Hiring pipeline, People arrangements, payroll cost | [hiring](../../hiring/SKILL.md), [manage-people](../../manage-people/SKILL.md), `people_costs` |
| Fundraising | Investor pipeline, SAFEs & rounds, updates | [fundraise](../../fundraise/SKILL.md), [investor-update](../../investor-update/SKILL.md), `cap_table` |
| Finance | Statements, cash, runway | [economico](../../economico/SKILL.md), `income_statement`, `balance_sheet`, `default_alive` |
| Planning | Goals, KPIs, OKRs, V2MOMs, quotas & operating plans | [targets](targets.md), `measures`, `targets` |

## Business shapes

| Business | Pipeline | Agreement | Delivery evidence | Receivables | Payables |
|---|---|---|---|---|---|
| Consulting or agency | Deals per engagement | Retainer, hourly and fixed-fee rights per client | Approved timesheets, accepted deliverables | Arrears or on acceptance; reminders | Subcontractors named to client work |
| Self-serve SaaS | Rarely; plans sell themselves | Price book templates; Stripe checkout as acceptance | Elapsed service time | Card collections via Stripe | Hosting and tools |
| Usage or API business | Larger accounts only | Base plus metered rights | Meter readings as `usage` facts | Usage billed in arrears | Infrastructure priced by use |
| Credits or prepaid AI | Top-ups as expansion deals | Credit lots with expiry | Consumption draws from named lots | Billed in advance, deferred until used | Model inference |
| Annual enterprise SaaS | Every deal, renewals above all | Annual price with installment billing and lifecycle terms | Service windows, implementation acceptance | Installments with due dates | Audits and cloud |
| Grant-funded research | Grants and retainers as deals | Tranches tied to accepted deliverables | Acceptance of each deliverable | Tranches on acceptance | Research subcontracts |
