# Forecasting playbook

Reference for the `forecasting` skill: how to derive forward entries from the
contract/obligation spine, the runway and cash-flow formulas, and the draft-entry
table format. Work in **minor units (cents)** throughout — `4999` is `$49.99` —
and convert to currency only for the final writeup. Never sum across currencies;
forecast each currency separately.

## Deriving forward entries from obligations

The contract → obligation spine encodes the future before any invoice or bill
exists. Each `list_obligations` row carries an `account_code`, an optional `sku`,
a cadence (`recurring` + `interval`, `usage`, or `one_off`), and an amount. Walk
it to emit one draft entry per obligation per period:

| Obligation | Cadence | Draft entry (per period) | Sign on cash |
|---|---|---|---|
| Customer (role `customer`) | `recurring` | `Dr 1120 Accounts Receivable / Cr <revenue code>` | inflow when collected |
| Vendor (role `vendor`) | `recurring` | `Dr <expense/COGS code> / Cr 2110 Accounts Payable` | outflow when paid |
| Either | `one_off` | one entry at its single due date | — |
| Either | `usage` | not committed — estimate from history or mark speculative | — |

Account codes are per-business — resolve them with `list_chart_of_accounts`,
don't hardcode. The codes in the examples (`1110` cash, `1120` AR, `2110` AP,
`4110` subscription revenue, `5400`/`6350` cost lines) are illustrative.

**Posting mechanics.** Post each forward entry *inside the scenario* with its
`external_timestamp` set to the expected book date (the entry's effective time,
distinct from when you post it), so it lands on the projected timeline rather than
today. Tag each with a stable reference like `forecast:<obligation>:<period>` so
re-running the projection is idempotent and never duplicates. Use `create_invoice`
when you want the invoice object (it posts the AR/revenue journal under the
scenario and, even if "sent", never delivers), or `create_journal` for a bare
forward AR/AP entry when you don't need a document.

**Collection vs accrual timing.** A recurring customer obligation accrues revenue
and AR on its invoice date, but cash arrives later (net-30, etc.). For a *cash*
runway forecast, model the payment leg too — a forward `record_payment` /
`Dr 1110 Cash / Cr 1120 AR` dated at expected-collection — or apply a collection
lag assumption and state it. For a *P&L / accrual* forecast, the AR/revenue entry
alone is enough.

## Modeling the what-ifs

- **Price change.** Re-derive the customer obligations' future entries at the new
  amount (e.g. ×1.20 for +20%). Optionally branch a `clone_scenario` so you can
  compare old vs new pricing side by side.
- **New deal.** Add the counterparty (`create_party`), its `create_contract` and
  `create_obligation`, then the future invoices the obligation implies — all under
  the scenario.
- **Churn.** Stop a customer's future entries from the churn date forward (simply
  don't emit them past that period); leave the entries before churn intact.
- **Hire / one-off cost.** A forward expense entry (`Dr <expense> / Cr 1110 Cash`
  or `/ Cr 2110 AP`) dated when it hits.

## Formulas

- **Monthly burn** = monthly operating outflows − monthly operating inflows
  (negative burn = profitable). For a cash forecast use modeled cash legs; if you
  only have accrual entries, use net income as a proxy and **label it a proxy**.
- **Ending cash (month _n_)** = starting cash + Σ(modeled inflows − outflows)
  through month _n_. Report the *path*, and flag the trough, not just the endpoint.
- **Runway (months)** = current cash ÷ average monthly net burn. If the modeled
  change turns burn negative (profitable), runway is unbounded — say so.
- **Cash-zero month** = the first projected month where ending cash < 0. This is
  the headline of a runway forecast.
- **MRR** = sum of recurring customer obligation amounts normalized to a month.
  **ARR** = MRR × 12 (+ annualized committed subscription contracts).
- **MRR/ARR delta** = scenario MRR/ARR − baseline MRR/ARR — the recurring-revenue
  impact of the what-if.
- **Gross margin (projected)** = (projected revenue − projected COGS) ÷ projected
  revenue, using the obligation `source_obligation_id` link to tie each cost back
  to the revenue line it serves where present.

## Draft / scheduled-entry table format

Mirror the month-by-month projection style so a human can read the forward book at
a glance. Two sections per snapshot: the committed draft schedule, then the
resulting statements under the scenario.

```
## Scheduled draft entries (scenario: <name>)

Forward entries are planning-only — posted inside the scenario, excluded from the
real ledger, discarded when the actual invoice/bill is created.

| Scheduled  | Flow             | Party    | Reference                 | Debit                       | Credit                  | Amount    |
| ---------- | ---------------- | -------- | ------------------------- | --------------------------- | ----------------------- | --------: |
| 2026-02-05 | future AR invoice | Acme    | forecast:acme-plan:2026-02 | 1120 Accounts Receivable    | 4110 Subscription Rev   | $3,850.00 |
| 2026-02-23 | future AP bill    | OpenAI  | forecast:llm:2026-02       | 5400 AI and LLM Inference   | 2110 Accounts Payable   | $2,050.00 |
```

Then the projected **Income Statement**, **Balance Sheet**, and a **cash path**
(cash by month with the trough and any cash-zero month flagged), each pulled with
the `scenario` parameter so they reflect `base ∪ scenario`.

## Scenario hygiene

- Default throwaway math to the seeded `test` sandbox; name anything you'll
  compare or revisit.
- `clone_scenario` to branch variants (`base` → `bull` / `bear`) from one start.
- `reset_scenario` to re-run a named case from scratch without losing the name.
- `delete_scenario` when done; if you leave a scenario in place, say so in the
  output so it isn't mistaken for reality.
- The real ledger is `scenario_id IS NULL` — omitting the `scenario` parameter
  always means reality. When in doubt, omit means *real*, so be explicit.
