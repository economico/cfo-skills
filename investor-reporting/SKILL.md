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

- `get_saas_metrics(year?, month?, metrics?, currency?, trend?)` **first** for
  the standard operating metrics — it computes them deterministically from the
  ledger and snapshot history so you never hand-assemble them: point-in-time
  (`arr`, `mrr`, `acv`, `run_rate`, `gross_margin`, `burn`, `runway`, `cac`)
  and month-over-month (`net_new_arr` — the waterfall decomposed into
  new / resurrected / expansion / contraction / churn with per-contract
  drilldown — plus `nrr`, `grr`, `logo_churn`, `burn_multiple`, `rule_of_40`,
  `ltv`). Pass `trend: true` for 12-month sparkline series. Every metric
  carries its basis and confidence notes — quote them, don't strip them. A
  month with no snapshot returns null values with a gap note (history is
  forward-only and never backfilled); present the gap honestly.
- `summarize_revenue(period_start, period_end)` for period revenue.
- `get_income_statement(currency)`, `get_balance_sheet(currency)`, and
  `get_balances(currency)` for current books and burn.
- `get_invoices(status?, party_id?, due_from?, due_until?)` for AR, ACV, and
  customer-level revenue.
- `get_bills(status?, party_id?)` for payables and vendor cost context.
- `list_contracts`, `list_obligations`, and `list_parties` for customer,
  contract, obligation, SKU, and pricing-model context.
- `get_usage(obligation_id?, from?, until?)` for metered-usage revenue, usage
  growth, and the unit-economics rollup — metered cost tied to the revenue it
  serves via `source_obligation_id` (true gross margin per SKU/customer, not
  blended).
- `get_cap_table` for ownership by holder/class, authorized-vs-issued, and
  outstanding SAFEs — the dilution picture investors ask for alongside the
  operating metrics.
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

Prefer the figures `get_saas_metrics` returns over recomputing these by hand —
the definitions below are what it implements (so you can explain them), not a
worksheet:

- MRR: monthly recurring obligation revenue from active customer contracts.
- ARR: MRR * 12, plus annualized committed subscription contracts when present.
- ACV: annualized contract value for a customer contract; exclude one-time fees.
- NDR/NRR (`nrr`): starting recurring revenue + expansion - contraction -
  churn, divided by starting recurring revenue — month-over-month at contract
  level from the snapshot diff. GRR (`grr`) is the same without expansion.
- Burn rate (`burn`): operating burn on an accrual basis (COGS + opex −
  revenue) — labeled, not true cash burn.
- Burn multiple (`burn_multiple`): monthly operating burn divided by net new
  ARR for the same month; null against shrinkage.
- Runway (`runway`): cash balance divided by monthly burn; null when not
  burning.
- Rule of 40 (`rule_of_40`): growth% + operating margin%; growth is YoY once
  12 months of snapshots exist, else labeled month-over-month annualized.
- LTV (`ltv`): coarse blended — ARPA × gross-margin ratio ÷ monthly
  revenue-churn rate; null while no churn has been observed.
- Default alive: cash plus expected gross profit covers expected burn before
  the business reaches breakeven; state assumptions explicitly (not computed
  by the tool).

## Output

Investor reports should include: headline metrics, movement since the prior
period (lead with the net-new-ARR waterfall — name the customers behind each
bucket from the drilldown), what changed, risks, and the exact data gaps. If
the ledger does not contain usage ingestion, cohort history, or cash-flow
timing, say so and show the best defensible proxy rather than pretending the
metric is exact.

To hand the numbers to an investor directly, mint a live link instead of a
stale PDF: `create_report_share(reports: ["saas_metrics"], description: ...)`
returns an unguessable URL rendering the dashboard (headline figures, the
waterfall, supporting metrics with their confidence notes). It can be scoped
to a planning scenario, given an expiry, and revoked any time with
`revoke_report_share`.
