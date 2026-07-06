# GAAP audit check catalog

Every check below names the **accounts** it touches (codes from Economico's
chart — confirm against `list_chart_of_accounts`), the **read tool** that
surfaces it, the **GAAP principle**, and the **correcting entry** to recommend.
Work in **minor units** (cents) throughout; convert only for the writeup. You
*recommend* corrections — the money-loop skills post them.

Run the checks per currency. The accounting equation and every tie-out hold
*within* a single-currency ledger, never across currencies (there is no FX
conversion in Economico).

## Normal balances & contra accounts (reference)

`list_chart_of_accounts` is authoritative. A balance whose sign is opposite its
normal balance is an anomaly to trace, not to assume away.

| Family | Codes | Normal balance | Contra accounts (carry the *opposite* sign) |
|---|---|---|---|
| Assets | 1xxx | Debit | 1130 Allowance for Doubtful Accounts; 1220 Accumulated Depreciation; 1240 Accumulated Amortization |
| Liabilities | 2xxx | Credit | — |
| Equity | 3xxx | Credit | 3300 Dividends (debit); 3200 Retained Earnings rolls up net income |
| Revenue | 4xxx, 7200/7400/7600/7800 | Credit | — |
| Expense / COGS | 5xxx, 6xxx, 7100/7300/7500/7700/7900 | Debit | — |

Note the 7xxx range is **mixed**: gains/other income (7200, 7400, 7600, 7800)
are revenue-type; losses/expense (7100, 7300, 7500, 7700, 7900) are
expense-type. Don't lump the whole range into one side.

---

## §1 Foundational integrity (do the books even tie?)

Run these first. If any fail, that's the headline finding — resolve it before
finer tests, because everything downstream inherits the error.

1. **Accounting equation.** `Assets = Liabilities + Equity` on
   `get_balance_sheet(currency)`, per currency. The report is constructed to
   balance, so a break means a report/ledger inconsistency worth escalating.
   *Severity: Critical if it doesn't tie.*
2. **Net-income roll-forward.** `get_income_statement` Net income should equal
   `Revenue − Expenses`, and that figure is what links the income statement into
   Equity (Retained Earnings 3200). A balance sheet that only balances because
   net income is forced in is a red flag — confirm the income statement stands
   on its own.
3. **Sign / contra checks.** Scan `get_balances`. Flag any normally-debit
   account (assets, expenses) sitting credit, or normally-credit account
   (liabilities, equity, revenue) sitting debit — *except* the contra accounts
   above, which are *supposed* to carry the opposite sign (1130, 1220, 1240
   reduce assets; 3300 reduces equity). A contra account at **zero** when its
   companion asset is large is itself a finding (see §2/§3).
4. **Per-journal balance.** Economico backstops debits = credits with a deferred
   DB constraint, so unbalanced posted journals shouldn't exist — but when you
   drill into a suspicious `get_journal(id)`, still confirm the lines net to
   zero and hit sensible accounts. The constraint guarantees arithmetic, not
   *correct* accounts.

---

## §2 Recognition & matching (accrual GAAP — ASC 606, matching principle)

Every Economico business is on the accrual basis, so these are live findings.
The ledger posts AR/AP on invoice-sent / bill-approved, but the period-end
judgment entries below (deferring unearned revenue, amortizing prepaids,
accruing incurred-but-unbilled cost, depreciation) it won't post on its own —
their absence is the misstatement to catch.

1. **Revenue recognized when earned, not when collected (ASC 606).** Compare
   `summarize_revenue(window)` *invoiced* vs *recognized*. A large
   `invoiced − recognized` gap means cash/billings ran ahead of delivery — that
   unearned portion belongs in **Unearned Revenue 2150**, not revenue.
   - *Symptom:* customer prepaid an annual plan, but the whole amount hit 4xxx
     revenue on invoice. *Impact:* revenue & equity overstated, liabilities
     understated. *Correcting entry:* `Dr 4xxx Revenue / Cr 2150 Unearned
     Revenue` for the unearned portion, then recognize ratably as earned.
2. **Deferred revenue actually releases.** If 2150 carries a balance, confirm it
   *decreases* over time (revenue gets recognized). A static 2150 means earned
   revenue is stuck as a liability — revenue & equity understated.
3. **Prepaid expenses amortize (matching).** A balance in **Prepaid Expenses
   1150** should draw down as the benefit is consumed (insurance, annual SaaS).
   - *Symptom A:* an annual cost was expensed in full on payment — expense
     overstated this period, understated later. *Correcting entry:* `Dr 1150
     Prepaid / Cr <expense>` for the unexpired portion.
   - *Symptom B:* 1150 sits flat for months with no amortization — expense
     understated. *Correcting entry:* `Dr <expense> / Cr 1150` for the consumed
     portion.
4. **Accrue incurred-but-unbilled costs (matching, completeness).** Expenses
   incurred in the period with no bill yet (e.g. usage-based infra at month-end)
   belong in **Accrued Expenses 2120**. If 2120 is always zero on books that
   clearly accrue elsewhere, expenses/liabilities may be understated at period
   ends. *Correcting entry:* `Dr <expense> / Cr 2120` at close, reversing when
   the bill posts.
5. **Depreciation & amortization (ASC 360 / 350).** If **PP&E 1210** or
   **Intangible Assets 1230** carry balances, expect non-zero **Accumulated
   Depreciation 1220** + **Depreciation Expense 6450** (and **1240** +
   **Amortization Expense 6460** for intangibles). Capitalized assets with zero
   accumulated depreciation are overstated and expense is understated.
   *Correcting entry:* `Dr 6450 Depreciation Expense / Cr 1220 Accumulated
   Depreciation` per the asset's useful life.
6. **Capitalize vs expense.** Inspect large one-off costs in 6xxx (esp. 6420
   Equipment and Hardware): a long-lived asset expensed in full should have been
   capitalized to 1210/1230 and depreciated. Conversely, routine costs parked in
   1210 that aren't long-lived assets overstate assets — they belong in expense.

---

## §3 Asset realizability & valuation (existence, valuation)

1. **Allowance for Doubtful Accounts (ASC 326 / net realizable value).** Age AR
   with `get_invoices(status, due_until)` (see §5 aging). If material receivables
   are well past due (90+) and **Allowance for Doubtful Accounts 1130** is zero,
   AR is carried above net realizable value — assets & equity overstated.
   *Correcting entry:* `Dr 6xxx Bad Debt Expense / Cr 1130 Allowance` for the
   estimated uncollectible portion (1130 is contra-asset, so it carries a credit
   balance reducing AR). For a confirmed-uncollectible invoice, recommend a
   write-off rather than a reserve.
2. **Negative / abnormal balances.** Economico has **no general overdraft
   check** (only the payment path is guarded), so accounts *can* go negative.
   A negative **Cash 1110**, negative AR, or any asset below zero is a true
   signal — trace it to the journals that drove it; it's often a
   double-posted payment, a misapplied receipt, or a missing entry.
3. **Digital assets at fair value (ASU 2023-08).** If **Crypto Holdings 1310**
   carries a balance, GAAP now measures in-scope crypto at **fair value** each
   period, with changes in **Unrealized Gain/Loss 7800/7900** and realized
   results in **7600/7700** on disposal. Crypto sitting at historical cost with
   7800/7900 untouched across a period where prices moved is a measurement
   departure — and a likely *missing-disclosure* note. Flag it; the remeasure
   itself is a management estimate, not something you post from this skill.
4. **Inventory (lower of cost and NRV, ASC 330).** If **Inventory 1140** is
   used, confirm it isn't carried above net realizable value and that COGS moves
   when inventory is sold (matching). A static 1140 alongside product revenue is
   suspicious.

---

## §4 Classification & presentation (classification, ASC 220)

1. **COGS vs Operating Expense.** Cost of Revenue lives in **5xxx**; operating
   overhead in **6xxx**. Misclassification distorts **gross margin** without
   touching net income, so it hides in the totals — check it deliberately.
   - Overhead (rent, G&A salaries, marketing) sitting in 5xxx *understates* COGS
     classification and *overstates* gross margin.
   - Direct delivery cost (hosting 5300, inference 5400, third-party API 5600
     that directly serves customers) parked in 6xxx *overstates* gross margin.
   - The spine helps: a vendor obligation with a `source_obligation_id` pointing
     at a customer-revenue obligation is *cost of revenue* by definition —
     confirm it's in 5xxx.
   - Infra vendors serving multiple environments (prod + dev/staging) booked to
     a single code: the production footprint belongs in 5300 (COGS), dev/staging
     in 6130 (R&D opex). One blended line on either side misstates gross margin —
     expect separate per-environment obligations on the vendor's contract.
2. **Current vs non-current.** Confirm assets/liabilities land in the right
   current (1100/2100) vs non-current (1200/2200) parent. A long-term note in
   2130 (short-term) or a current obligation in 2210 misstates working capital
   and liquidity ratios.
3. **Contra accounts presented as reductions.** 1130, 1220, 1240 must reduce
   their companion asset on the balance sheet (they carry credit balances). 3300
   Dividends reduces equity. Confirm the report nets them rather than showing
   them as positives.
4. **Gains/losses not netted into revenue.** 7xxx gains/losses (asset sales,
   crypto realized/unrealized) belong in *Other Income and Expenses*, below
   operating results — not inside 4xxx operating revenue. Netting a one-off gain
   into revenue inflates the top line.

---

## §5 Completeness & cutoff (completeness, cutoff)

1. **AR / AP aging (cutoff & realizability input).** Bucket `get_invoices` by
   `due_until`: current, 1–30, 31–60, 61–90, 90+ (overdue = `due_date < today`
   and status ≠ `paid`/`voided`). Feeds the ADA test (§3.1) and surfaces stale
   items that may need write-off. Mirror for AP with `get_bills` by `party_id`.
2. **Sales tax completeness.** If the business makes taxable sales but **Sales
   Tax Payable 2160** is always zero, collected/owed tax may be missing —
   liabilities understated. Confirm whether the business is even tax-registered
   before flagging (don't invent a liability that doesn't apply).
3. **Period cutoff.** Using `summarize_revenue` windows, watch for revenue or
   expense recorded in the wrong period — entries dated just before/after a
   close, or backdated `external_timestamp`. Drill into the specific journals
   around the boundary. Voided journals are excluded from balances automatically,
   but a void *and re-post* across a period boundary can shift income between
   periods — check both sides.
4. **Unrecorded liabilities.** Bills approved but never posted, or known
   recurring vendor obligations (`list_obligations` `role=vendor` recurring) with
   no corresponding posted expense for the period, suggest completeness gaps —
   the classic "search for unrecorded liabilities".

---

## §6 Consistency & disclosure (comparability, full disclosure)

1. **Consistent treatment period to period.** Compare `summarize_revenue` and
   account mix across windows: a revenue stream that recognized on delivery last
   quarter and on invoice this quarter, or a cost that moved from 5xxx to 6xxx,
   breaks comparability. Flag method changes — GAAP allows them but they must be
   deliberate and disclosed, not accidental drift.
2. **Disclosure gaps (full disclosure principle).** Note where GAAP statements
   would require a disclosure the ledger alone can't carry: crypto fair-value
   policy and holdings (ASU 2023-08), revenue recognition policy, significant
   concentrations (one customer >~30–40% of AR/revenue — build from
   `get_invoices` grouped by `party_id`), related-party transactions, and going
   concern if runway is short. These are *Observations* for the report, not
   posting corrections.

---

## Materiality & severity (how to rank)

Don't drown the report. Set a threshold up front (default ~1% of total assets or
total revenue — state the number) and rank:

- **Critical** — books don't tie (§1), or a misstatement large enough to flip a
  conclusion (profitable↔loss, solvent↔insolvent).
- **Material** — a real GAAP departure above the threshold (unrecognized
  deferred revenue, AR with no allowance, capital asset expensed, COGS/OpEx
  misclassification that materially moves gross margin).
- **Minor** — below threshold, or a presentation/classification nit with no
  bottom-line impact.
- **Observation** — a control, process, or disclosure note with no current
  misstatement (e.g. "2120 always zero — confirm month-end accrual process").

Rank by impact, not count. Lead with what moves a conclusion; don't let a pile
of immaterial classification nits bury the one finding that flips profit to loss
or overstates assets above the threshold.
