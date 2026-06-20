---
name: investor-reporting
description: >
  Produce investor-facing reports from Economico for SaaS, usage-based, AI,
  consulting, services, and grant-funded businesses. Use when asked for
  investor updates, board metrics, MRR, ARR, ACV, NDR, churn, burn rate, burn
  multiple, runway, default alive, gross margin, revenue mix, concentration, or
  SaaS/consulting/grant KPIs. Read-only: pull reports and invoices, compute the
  metrics, and hand off to financial-analyst for deeper accounting analysis.
---

# Investor Reporting

This is read-only. Do not post invoices, bills, payments, journals, contracts,
or obligations from this skill. Pull the books and explain the metrics.

## Data Sources

Use:

- `summarize_revenue(period_start, period_end)` for period revenue.
- `get_income_statement(currency)`, `get_balance_sheet(currency)`, and
  `get_balances(currency)` for current books and burn.
- `get_invoices(status?, party_id?, due_from?, due_until?)` for AR, ACV, and
  customer-level revenue.
- `get_bills(status?, party_id?)` for payables and vendor cost context.
- `list_contracts`, `list_obligations`, and `list_parties` for customer,
  contract, obligation, SKU, and pricing-model context.
- `list_chart_of_accounts(currency)` to verify account names and codes.

Amounts are minor units. Do not sum across currencies; report USD and USDC
separately unless the user explicitly provides a conversion policy.

## Metrics by Model

| Model | Investor metrics |
| --- | --- |
| Monthly SaaS | MRR, ARR, active customers, plan mix, churn, expansion/contraction, NDR/GRR, AR aging |
| Annual/quarterly SaaS | ARR, ACV, committed ARR, implementation revenue, billings vs revenue, renewal dates |
| Metered API | Subscription vs usage revenue, usage growth, gross margin, revenue concentration, net retention |
| AI credits | Plan MRR, credit top-up revenue, token usage, inference cost, gross margin, credit balance/risk if available |
| x402/MPP API | Per-call revenue, paid calls, rail mix, payment fees, gross margin per request |
| Solo consulting | Bookings, recognized consulting revenue, utilization proxy, retainer base, customer concentration, AR aging |
| Consulting agency | Revenue, subcontractor cost, gross margin by engagement, backlog, AP exposure |
| Grant-funded studio | Grant runway, tranche schedule, grant vs earned services mix, milestone risk, stablecoin cash |

## Core Formulas

- MRR: monthly recurring obligation revenue from active customer contracts.
- ARR: MRR * 12, plus annualized committed subscription contracts when present.
- ACV: annualized contract value for a customer contract; exclude one-time fees.
- NDR: starting recurring revenue + expansion - contraction - churn, divided by
  starting recurring revenue.
- Burn rate: monthly cash operating loss. Use income statement as a proxy if
  cash-flow data is unavailable and label it as a proxy.
- Burn multiple: net burn divided by net new ARR for the same period.
- Runway: cash balance divided by monthly burn.
- Default alive: cash plus expected gross profit covers expected burn before
  the business reaches breakeven; state assumptions explicitly.

## Output

Investor reports should include: headline metrics, movement since the prior
period, what changed, risks, and the exact data gaps. If the ledger does not
contain churn, usage ingestion, cohort history, or cash-flow timing, say so and
show the best defensible proxy rather than pretending the metric is exact.
