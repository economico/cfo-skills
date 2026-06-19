# Analyses: formulas, ratios, and how to read the ledger

Work in **minor units** (cents) throughout; convert to the display currency only
in the final number. Every figure below comes from the read tools listed in
`SKILL.md` — no posting.

## Debit/credit cheat sheet

Economico is double-entry. Each account has a *normal balance*; knowing it tells
you whether a balance is "good high" or "wrong sign". `list_chart_of_accounts`
is the authoritative map (codes → ledger account ids); the families:

| Family | Code range | Normal balance | A debit… | A credit… |
|---|---|---|---|---|
| Assets (cash, AR 1120, crypto receivable 1320) | 1xxx | Debit | increases | decreases |
| Liabilities (AP 2110) | 2xxx | Credit | decreases | increases |
| Equity | 3xxx | Credit | decreases | increases |
| Revenue | 4xxx | Credit | decreases | increases |
| Expenses / COGS | 5xxx | Debit | increases | decreases |

Sanity identities (should always hold; if they don't, dig in before trusting the
report):

- **Accounting equation:** `Assets = Liabilities + Equity` (the balance sheet is
  constructed to balance).
- **Net income:** `Revenue − Expenses`, and it rolls into Equity.
- A line's `direction` + the account family tells you the economic effect; a
  negative *balance* on a normally-positive account is an anomaly to flag.

## Profitability (from `get_income_statement`)

All current cumulative for the chosen currency. For period-specific margins, pair
with `summarize_revenue` windows.

- **Gross margin** = `(Revenue − COGS) / Revenue`. COGS = the 5xxx cost-of-sales
  accounts (vs operating expenses).
- **Net margin** = `Net income / Revenue`.
- **Operating margin** = `(Revenue − Operating expenses) / Revenue`.
- **Expense ratio** for any account = `Account expense / Revenue` — use to spot
  which line moved when net income drops.

## Revenue trend & quality (from `summarize_revenue`)

`summarize_revenue(period_start, period_end)` returns, per currency: *invoiced*,
*paid*, *outstanding*, *recognized*.

- **MoM / QoQ growth** = `(thisWindow.recognized − priorWindow.recognized) /
  priorWindow.recognized`. Call the tool once per window; don't infer a period
  from the cumulative income statement.
- **Collection rate** = `paid / invoiced` for the window — how much of what you
  billed actually came in.
- **Recognition gap** = `invoiced − recognized` — deferred/unearned revenue
  signal.
- **Revenue concentration** = top customer's invoiced ÷ total invoiced. Build it
  from `get_invoices` grouped by `party_id` (cross-reference `list_parties` for
  names). >30–40% from one customer is a concentration risk worth naming.

## Liquidity, burn & runway

- **Cash position** = sum of cash/bank asset balances from `get_balances` (the
  liquid 1xxx accounts), per currency.
- **Net burn (monthly)** = average monthly `(operating cash out − operating cash
  in)`. Approximate from `summarize_revenue` collections vs expense outflows over
  N recent months; state the window you used.
- **Runway (months)** = `Cash position / Net monthly burn`. If burn is negative
  (cash-flow positive), say "profitable / runway not cash-constrained" rather
  than printing a misleading number.

## Receivables (AR) health (from `get_invoices`)

- **Outstanding AR** = invoices with status `sent` or `partially_paid`.
- **Aging buckets:** filter by `due_until` to bucket by days overdue —
  current (not yet due), 1–30, 31–60, 61–90, 90+. Overdue = `due_date < today`
  and status ≠ `paid`/`voided`.
- **Overdue exposure** = total of all past-due invoices — the at-risk cash.
- **DSO (days sales outstanding)** ≈ `(Outstanding AR / Revenue over period) ×
  days in period`. Use the income statement revenue (or a `summarize_revenue`
  window) for the denominator; state which.

## Payables (AP) health (from `get_bills`)

- **Outstanding AP** = bills with an unpaid/partially-paid status.
- **Upcoming payables** = bills due in the near term — the cash you owe soon.
- **AP aging** mirrors AR aging by vendor (`party_id`).

## Anomaly & integrity pass

- **Negative balances:** scan `get_balances` for any normally-positive account
  (assets, especially cash) sitting negative — Economico has no general overdraft
  check, so this is a true signal, not a glitch.
- **Single-entry drill-down:** when a balance looks wrong, pull the specific
  `get_journal(id)` (id from the related invoice/bill/payment) and read its lines
  — confirm debits = credits and the accounts hit make sense.
- **Cross-report tie-out:** `summarize_revenue` *recognized* for a window vs the
  income-statement revenue movement should roughly agree; a large divergence
  means deferred revenue, voids, or backdated entries — investigate before
  reporting a number you can't defend.

## Spine analyses (parties, contracts, obligations)

These read commitments and structure the P&L can't show. Source: `list_contracts`
(each row has `role` customer/vendor), `list_obligations` (each has `account_code`,
`sku`, `type`, `interval`, `amount_minor_units` / `price_per_unit_minor`,
`source_obligation_id`), and `list_parties`.

### Committed run-rate (forward spend & revenue)

Normalize every **recurring** obligation to a monthly figure: monthly →
`amount_minor_units`; yearly → `amount_minor_units / 12`. Then:

- **Committed monthly spend** = sum over `role=vendor` recurring obligations. This
  is your locked-in burn *before* any bill posts — compare it to actual posted
  expense to spot under/over-running categories.
- **MRR** = sum over `role=customer` recurring obligations; **ARR** = MRR × 12.
- Bucket either side by `account_code` (spend) or `sku` (revenue) to see the mix.
- `usage` obligations are pricing definitions, not committed amounts — report them
  as rate cards (`metric_name` @ `price_per_unit_minor`), don't sum them into the
  run-rate.

### Unit economics / gross margin by customer or SKU

The `source_obligation_id` on a vendor obligation points at the customer revenue
obligation it serves. To compute true margin:

1. Group customer revenue obligations by customer (or `sku`).
2. For each, gather the vendor obligations whose `source_obligation_id` points to
   it — those are the directly-attributable costs.
3. **Contribution margin** = `revenue − attributable cost`; **margin %** =
   `contribution / revenue`. This beats blended gross margin because it ties cost
   to the revenue it produced.
Costs with no `source_obligation_id` are shared/overhead — report them separately,
don't force-allocate.

### Concentration & dependency

- **Revenue concentration**: top customer's MRR ÷ total MRR (from customer
  obligations), cross-checked against `get_invoices` actuals.
- **Vendor dependency**: top vendor's committed spend ÷ total committed spend.
- Flag either when one name exceeds ~30–40% — it's a continuity risk.
