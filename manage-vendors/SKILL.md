---
name: manage-vendors
description: "Run procurement and vendor spend in Economico: track vendor purchasing pipelines; look up and adopt vendor catalog plans or model agreements; record email receipts, vendor bills, company-card and founder-paid expenses; review subscriptions and tool usage."
---

# Manage vendors and procurement

Follow [economico](../economico/SKILL.md) for ground rules (`catalog {action: "describe", name}`
before executing; fetch recipes via `catalog {action: "recipe", name}`; confirm the target business
before writing). Economico's own plan contract is already in the books — never model or import an
Economico invoice; pay one that is owed via [payments](../economico/references/payments.md).

## The procurement-to-pay loop

1. **Evaluate and negotiate (pipeline):** Track tools, subcontractors and suppliers under evaluation
   in the `vendor` pipeline (`opportunities.open`, `opportunities.stage`, `opportunities.link`) and
   capture vendor calls or quotes with [conversation intake](../economico/references/conversation-intake.md).
   See [pipelines](../economico/references/pipelines.md).
2. **Adopt or model the agreement:** Check the public vendor catalog first (`vendor_catalog` →
   `contracts.adopt`) before modelling by hand. Read
   [vendor contracts](references/vendor-contracts.md).
   - For GitHub Actions amount vs. metered minutes, read [GitHub Actions](references/github-actions.md).
   - For Stripe Billing/Invoicing/Tax/Radar fees deducted from the Stripe balance, read
     [Stripe balance fees](references/stripe-balance-fees.md).
3. **Record receipts, bills and payments:** Find receipts and invoices in email via
   [explore email](references/explore-email.md), classify expense accounts and functions via
   [accounts](../economico/references/accounts.md), record bills/receipts against the vendor
   contract, and settle open payables or founder reimbursements separately via
   [payments](../economico/references/payments.md).
4. **Track usage, spend budgets and cut waste:** Record seat and platform usage and run
   subscription reviews via [vendor usage](references/vendor-usage.md) (`vendor_utilization`,
   `aging` with `direction: "payable"`, `payment_routes`). Economico has no budget-approval
   workflow, so when a founder wants monthly or quarterly vendor/category spend budgets or burn
   caps, track them as live `at_most` targets in [plan-and-track](../plan-and-track/SKILL.md).

## References and recipes

- [Vendor contracts](references/vendor-contracts.md) — adopt-first, hand-modeling, tax, corrections, voids
- [GitHub Actions](references/github-actions.md) — amount vs. metered minutes and `invoice_overlap`
- [Stripe balance fees](references/stripe-balance-fees.md) — `txn_…` deductions and coverage windows
- [Explore email](references/explore-email.md) — finding and classifying emailed receipts and bills
- [Vendor usage](references/vendor-usage.md) — seat/platform utilization and subscription reviews
- Shared foundations in `economico`: [pipelines](../economico/references/pipelines.md),
  [conversation intake](../economico/references/conversation-intake.md),
  [accounts](../economico/references/accounts.md), [payments](../economico/references/payments.md),
  [evidence](../economico/references/evidence.md), [verify](../economico/references/verify.md)
- Fetch recipes via `catalog {action: "recipe", name}`: `vendor-monthly-subscription`,
  `vendor-annual-prepay`, `vendor-bill-paid-from-bank`, `vendor-receipt-paid-from-bank`,
  `vendor-receipt-company-card`, `founder-paid-expense`, `founder-card-foreign-bill`,
  `founder-card-foreign-refund`, `founder-payment-before-bill`, `founder-refund-after-reimbursement`
