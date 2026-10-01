# Modeling products and pricing

A product in Economico is not a catalog row. It is a **template**: the reusable bundle of
activities a customer contract is created from, with the price as a term. A template with
`plan: "on_sale"` is a price-book entry; `subjects {action: "list", type: "template", plan:
"on_sale"}` is the price book. Each customer contract binds a template to that customer, with
their own start date and any negotiated terms.

## Pick the shape

| The pricing says | Shape | Recipe |
|---|---|---|
| $X per month, billed monthly | Scheduled bill at period start, scheduled recognition at period end, `recurringValue` monthly | [customer-monthly-subscription](recipes/customer-monthly-subscription.json) |
| $Y per year, paid upfront | Scheduled annual bill, monthly recognition of Y/12 | [customer-annual-prepaid](recipes/customer-annual-prepaid.json) |
| Platform fee plus usage (per call, per GB, per token) | Fee as a subscription; usage as a recorded measurement with a `rate` calculation, earned when measured and invoiced after the period | [customer-platform-fee-plus-usage](recipes/customer-platform-fee-plus-usage.json) |
| Usage only, tiered | The usage activity alone, with a `tiered` calculation (`graduated` or `volume`) | the usage part of the fee-plus-usage recipe |
| $Z per seat per month | The subscription with the price computed as seats × price (`rate` calculation over a `seats` term); a seat change is a dated `contracts.amend` of the seat term | [customer-per-seat-subscription](recipes/customer-per-seat-subscription.json) |
| Included usage, then overage | An `included` allowance granted by the fee activity, which the usage activity draws before billing only the overage; one grant per period, and record the measurement before the lot expires | [customer-included-usage-overage](recipes/customer-included-usage-overage.json) |
| Prepaid credits or packs | A `paid` allowance granted and billed on purchase into deferred revenue, recognized as usage consumes it | [customer-prepaid-credits](recipes/customer-prepaid-credits.json) |
| Minimum commitment | A `minimum` calculation with a reconciliation activity | `catalog describe activities.create` |
| Hourly or fixed-fee services | A recorded activity with hours × rate, or a milestone amount, on `consulting` | the fee-plus-usage recipe with `consulting` as the category |
| Free tier | No template; note it in the brief | — |
| Free trial that converts | The subscription template with a later bill start term than the service start | subscription recipe |

## Rules that keep the numbers right

- **Terms hold the agreed price; facts hold what happened.** Put list prices as term defaults in
  the template. Override per customer in `contracts.create` `terms`, keyed by activity key:
  `{ "bill": { "price": "3900", "start": "2026-07-01", "end": "2027-07-01" } }`. Every activity
  that uses a term needs its own override (the bill and the recognition in the subscription
  recipe both carry `start` and `end`).
- **Bill and earn are different activities.** Subscriptions bill in advance (`bill` into
  deferred revenue) and earn as service elapses (`recognize`). Usage earns when measured
  (`accrue_revenue`) and is billed after (`bill_accrued`). Never book the whole year as revenue
  on the day an annual plan is paid.
- **MRR comes from `recurringValue`**, declared on the activity that earns the fixed fee. Usage,
  overage, one-off fees and allowances never contribute to MRR.
- **Category chooses the revenue account.** `subscription` → 4110, `usage` → 4120, `consulting`
  → 4210, `services` → 4100, `grant` → 4500. Set `revenueAccount` only to pick a different
  revenue leaf, for example 4200 for implementation fees (see [accounts](accounts.md)).
- **Tag the product.** Set the activity's `product` (for example `pro`, `api-calls`) so revenue
  is reported per product line; the name may not be a category word. Put the Stripe product and
  price ids in `externalReferences`.
- **Sub-cent prices.** Rates are integers in minor units over `unitsPerPrice`: $0.002 per call
  (0.2¢) is rate `200` per `1000` calls, and 0.002¢ a call (Stripe's `unit_amount_decimal`
  `"0.002"`, which is in cents) is rate `2` per `1000`. Rounding happens once per recording, not
  per unit.
- **Uneven annual splits.** $290 a year does not divide into twelve equal cents. Recognize the
  rounded-down monthly amount and record the remainder in the final month (a separate final
  activity or a corrected last period); never recognize more than was billed. Say so in the
  brief.
- **One template per purchasable plan.** Monthly and annual versions of the same plan are two
  templates. A negotiated deal that differs only in price is the same template with a term
  override; one that differs in structure is its own template with no `plan` flag.
- **Retire, do not delete.** A plan no longer sold is `templates.replace` with
  `plan: "off_sale"`; existing contracts keep their frozen terms.

## Build and check a template

1. `catalog {action: "describe", name: "activities.create"}` and `templates.create`, once per
   session, to confirm the current schema.
2. `activities.create` for each activity; keep the returned `documentId`s.
3. `templates.create` binding them under keys. The keys are what contracts, claims and
   `accountingKey`s refer to; choose short, stable ones (`accept`, `bill`, `service`,
   `receipt`, `usage`).
4. Bind a first contract, then preview before recording anything:
   `reports {action: "run", kind: "activity_effects", contract_id, activity_key, effective_at,
   facts}` and `reports {action: "run", kind: "activity_plan", contract_id, from, through}`.
   The plan shows every scheduled period the timer will record.

## Recipes

Each recipe is a complete, tested sequence of public commands (`steps`), run in order, where
`{"$ref": "step-key.field"}` means "the `field` of the result of step `step-key`" (usually a
`documentId` or a contract `id`). `setupAt` is the envelope `effective_at` for setup steps; a
step's `at` is its own. Steps with `read` or `report` are the checks, with the result expected
under `expect`. Replace the synthetic names, amounts and dates with the business's own; keep the
structure.

| Recipe | Models |
|---|---|
| [customer-monthly-subscription](recipes/customer-monthly-subscription.json) | Monthly plan as a price-book template; customer contract; acceptance, bill, Stripe payment, month of service; MRR |
| [customer-annual-prepaid](recipes/customer-annual-prepaid.json) | Annual plan paid upfront, recognized monthly |
| [customer-platform-fee-plus-usage](recipes/customer-platform-fee-plus-usage.json) | Monthly platform fee plus metered API usage invoiced in arrears |
| [customer-per-seat-subscription](recipes/customer-per-seat-subscription.json) | Per-seat monthly plan; a seat expansion amended from a date; MRR before and after |
| [customer-included-usage-overage](recipes/customer-included-usage-overage.json) | Monthly fee with included calls; usage draws the allowance, only the overage is billed; MRR excludes it |
| [customer-prepaid-credits](recipes/customer-prepaid-credits.json) | Credit pack paid upfront, deferred, recognized as credits are used |
| [stripe-gross-net-payout](recipes/stripe-gross-net-payout.json) | Card payments at gross into the Stripe balance, Stripe's fee on 5230, the net paid out to the bank |
| [vendor-bill-paid-from-bank](recipes/vendor-bill-paid-from-bank.json) | Vendor invoice on terms, paid later |
| [vendor-receipt-paid-from-bank](recipes/vendor-receipt-paid-from-bank.json) | Receipt already paid by bank debit |
| [vendor-receipt-company-card](recipes/vendor-receipt-company-card.json) | Receipt charged to the company card; card statement paid |
| [founder-paid-expense](recipes/founder-paid-expense.json) | Founder paid personally; company reimburses |
| [vendor-annual-prepay](recipes/vendor-annual-prepay.json) | Annual vendor plan paid upfront into prepaid expenses, released monthly |
| [corporation-founders](recipes/corporation-founders.json) | Founders buy restricted common stock: charter authorization, one contract per founder, cliff and monthly vesting; the cap table |
