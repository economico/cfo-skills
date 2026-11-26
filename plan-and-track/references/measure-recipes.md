# Starter measure recipes

Tested SQL measure definitions for [targets and operating plans](targets.md).

Nothing is seeded into a business's books. Adopt only the measures the founder needs, adjust the
currency (`'USD'` and `"unit": "USD"`) and pipeline key (`'sales'` or `'customer'`) to the
business's own books, test each one with `measures {action: "try"}`, and then record it with
`measures.define`. Recipe 10 shows a per-person quota pattern keyed on an `owner` pursuit label;
splits, credit and commission rules stay with the business.

### 1. Pursuits won in a pipeline

Holder: pipeline (`:pipeline`, such as `"hiring"` or `"customer"`). Includes a weighted
open-pipeline count estimate (`open_weighted`, `decimals: 2`) for open pursuits expected to close
in the window.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Pursuits won in a pipeline",
  "description": "Count of pursuits won in a pipeline during the window, with a weighted open-pipeline count estimate to two decimal places.",
  "sql": "SELECT (SELECT count(*) FROM opportunities o JOIN events e ON e.event_id = o.won_event_id WHERE o.pipeline_key = :pipeline AND o.status = 'won' AND julianday(e.effective_at) >= julianday(:from) AND julianday(e.effective_at) < julianday(:through, '+1 day')) AS won, (SELECT (coalesce(sum(o.weight_bps), 0) + 50) / 100 FROM open_opportunities o WHERE o.pipeline_key = :pipeline AND o.expected_close_on BETWEEN :from AND :through) AS open_weighted",
  "columns": [
    { "name": "won", "unit": "number" },
    { "name": "open_weighted", "unit": "number", "decimals": 2, "estimate_for": "won" }
  ]
}
```

**Limits:** Counts pursuits whose current `status` is `'won'`, dated by the surviving winning
event's UTC `effective_at`, so voiding a settled contract removes an earlier win from historical
reads. Counts renewals and expansions alongside new business (`o.kind` in `'new'`, `'renewal'`,
`'expansion'`); to count new business only, add `AND o.kind = 'new'` to the won subquery, and in
the open subquery join `opportunities opp ON opp.opportunity_id = o.opportunity_id` with
`AND opp.kind = 'new'` (`open_opportunities` has no `kind` column). `open_weighted` reads
`open_opportunities` (excludes open pursuits with no `expected_close_on`, and any pursuit whose
`expected_close_on` fell in an earlier period) and carries two decimal places (`decimals: 2`,
rounded half-up from basis points: one pursuit at 80% and one at 40% return `"120"`, displayed as
`1.20` expected pursuits).

### 2. Won value in a pipeline

Holder: pipeline (`:pipeline`, such as `"customer"` or `"financing"`). Sums `expectedValue` on won
pursuits in one currency, with a weighted open-pipeline value estimate on the same scale.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Won value in a pipeline (USD)",
  "description": "USD expected value of pursuits won in a pipeline during the window, with a weighted open-pipeline estimate on the same scale.",
  "sql": "SELECT (SELECT CASE WHEN count(*) = 0 THEN 0 WHEN count(*) > count(o.expected_value_minor) THEN NULL ELSE sum(o.expected_value_minor) END FROM opportunities o JOIN events e ON e.event_id = o.won_event_id WHERE o.pipeline_key = :pipeline AND o.status = 'won' AND o.currency = 'USD' AND julianday(e.effective_at) >= julianday(:from) AND julianday(e.effective_at) < julianday(:through, '+1 day')) AS won_minor, (SELECT CASE WHEN count(*) = 0 THEN 0 WHEN count(*) > count(o.expected_value) OR max(length(o.expected_value)) > 18 THEN NULL ELSE sum((CAST(o.expected_value AS INTEGER) / 10000) * o.weight_bps) + (sum((CAST(o.expected_value AS INTEGER) % 10000) * o.weight_bps) + 5000) / 10000 END FROM open_opportunities o WHERE o.pipeline_key = :pipeline AND o.currency = 'USD' AND o.expected_close_on BETWEEN :from AND :through) AS open_weighted_minor",
  "columns": [
    { "name": "won_minor", "unit": "USD" },
    { "name": "open_weighted_minor", "unit": "USD", "estimate_for": "won_minor" }
  ]
}
```

**Limits:** Single currency (`'USD'`; replace both the SQL filter and column `unit` for another
currency); pursuits in other currencies are excluded. Sums each pursuit's `expectedValue`, not
invoiced or recognized revenue on the settled contract. Counts pursuits whose current `status` is
`'won'` dated by the surviving winning event (so voiding a contract removes an earlier win), and
includes renewals and expansions alongside new business (for new business only, add
`AND o.kind = 'new'` to the won subquery, and in the open subquery join
`opportunities opp ON opp.opportunity_id = o.opportunity_id` with `AND opp.kind = 'new'`, since
`open_opportunities` has no `kind` column). A `NULL` `expectedValue` on either side, or an open
pursuit with `expected_value` over 18 digits, returns `NULL` for that column alone so the two
columns fail independently. By contrast, a won pursuit with `expectedValue` over 18 digits triggers
an integer overflow in `expected_value_minor` on `opportunities` that fails the whole measure for
the period (`won_minor` reads `cant_be_measured` and `open_weighted_minor` has `estimate.ok: false`
with reason). Open pursuits without `expected_close_on`, or with `expected_close_on` in an earlier
period, are excluded from `open_weighted_minor`.

### 3. People in place

Holder: company. Point-in-time count of active team memberships on `:through` (ignores `:from`).

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "People in place",
  "description": "Active team memberships on the books at the end of :through (ignores :from).",
  "sql": "SELECT count(DISTINCT r.party_id) AS people FROM people_records r WHERE r.kind = 'membership' AND r.current = 1 AND r.valid_from < r.valid_through AND julianday(r.valid_from) < julianday(:through, '+1 day') AND julianday(r.valid_through) >= julianday(:through, '+1 day')",
  "columns": [
    { "name": "people", "unit": "number" }
  ]
}
```

**Limits:** Point-in-time headcount at the end of UTC day `:through` (ignores `:from`). For an
`in_progress` period, `targets read` evaluates through `period.through` rather than `today`, so
future-dated starts and end dates already recorded in `people_records` are counted as of period end
(for headcount as of `today`, run `measures {action: "try", id, from: today, through: today}`).
Counts recorded team memberships (`people.set_membership`) in `people_records`; workers with only
an arrangement (`people.record_arrangement`) or a vendor contract and no membership record are not
counted.

### 4. Payroll cost

Holder: company. Posted people and payroll expense across `5110` (salaries and wages), `5120`
(employer payroll taxes), `5130` (employee benefits), `5140` (contractors and EOR services) and
`5190` (other people costs).

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Payroll cost (USD)",
  "description": "Posted USD payroll and people expense (5110, 5120, 5130, 5140, 5190) during the window.",
  "sql": "SELECT coalesce(sum(jl.signed_minor), 0) AS payroll_cost_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code IN (5110, 5120, 5130, 5140, 5190) AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')",
  "columns": [
    { "name": "payroll_cost_minor", "unit": "USD" }
  ]
}
```

**Limits:** Single currency (`USD`) on UTC posting `effective_at` dates. Excludes `5150` (customer
support services) and payroll provider administration fees booked to `5640`. Because it reads
`journal_lines` by posting date, a provider invoice dated in a later month lands in the invoice's
month; if the business needs work-month attribution from `payroll.record_statement` and
`payroll.link_recorded_cost`, query `payroll_costs` by `month` instead (which is month-granular
and omits rows with `month IS NULL`).

### 5. ARR

Holder: company. Month-end annual recurring revenue from `monthly_metrics`, matching `saas_metrics`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "ARR (USD)",
  "description": "Month-end USD annual recurring revenue from monthly_metrics for the UTC month of :through.",
  "sql": "SELECT (SELECT m.arr_minor FROM monthly_metrics m WHERE m.month = substr(:through, 1, 7) AND m.currency = 'USD') AS arr_minor",
  "columns": [
    { "name": "arr_minor", "unit": "USD" }
  ]
}
```

**Limits:** Month-end figure for the UTC calendar month of `:through` (ignores `:from` and
mid-month cutoffs), in the books' functional currency (`USD`). `monthly_metrics` covers at most the
12 calendar months ending at the month of the books' latest event by `effective_at` (not
`recorded_at` — any recorded event, including non-posting events such as `targets.define`, sets the
window):
- A quarterly or annual period reads `cant_be_measured` until an event is effective-dated in its
  final month (`:through`'s month).
- One far-future-dated event shifts the 12-month window forward and pushes current months out
  (`cant_be_measured`).
- A historical period that once read `met` or `not_met` later reads `cant_be_measured` once the
  books advance more than 12 months past it.
- Books with no events, incorporation or functional-currency-change months, and months in a
  different functional currency have no matching row and return `NULL` (`cant_be_measured`).
- If any month in `monthly_metrics` contains an amount over 18 digits, the query fails with an
  integer-overflow error and marks the period `cant_be_measured`.

### 6. MRR

Holder: company. Month-end monthly recurring revenue from `monthly_metrics`, matching
`saas_metrics`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "MRR (USD)",
  "description": "Month-end USD monthly recurring revenue from monthly_metrics for the UTC month of :through.",
  "sql": "SELECT (SELECT m.mrr_minor FROM monthly_metrics m WHERE m.month = substr(:through, 1, 7) AND m.currency = 'USD') AS mrr_minor",
  "columns": [
    { "name": "mrr_minor", "unit": "USD" }
  ]
}
```

**Limits:** Same as ARR: month-end figure for the UTC calendar month of `:through` (ignores `:from`
and mid-month cutoffs), functional currency (`USD`), covering at most the 12 calendar months ending
at the latest event by `effective_at` (not `recorded_at`; any recorded event moves the window).
Quarterly or annual periods read `cant_be_measured` until an event is effective-dated in
`:through`'s month, a far-future-dated event pushes current months out, older periods fall out
after 12 months, missing or currency-change/incorporation months return `NULL`
(`cant_be_measured`), and an amount over 18 digits in any `monthly_metrics` month fails the query
with integer overflow (`cant_be_measured`).

### 7. Recognized revenue

Holder: company. Accrual-basis operating revenue under `4000` (`4100`–`4500`, net of `4140`
discounts and allowances) recognized in the window.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Recognized revenue (USD)",
  "description": "USD operating revenue (4000-4999, net of 4140 discounts) recognized during the window.",
  "sql": "SELECT coalesce(sum(-jl.signed_minor), 0) AS revenue_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code >= 4000 AND jl.account_code < 5000 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')",
  "columns": [
    { "name": "revenue_minor", "unit": "USD" }
  ]
}
```

**Limits:** Single currency (`USD`) on UTC `effective_at` dates; counts operating revenue under
`4000` (`4100`–`4500`, net of contra account `4140`) and excludes non-operating `7000`-series
income. Does not translate foreign-currency ledgers; if a business changed functional currency and
holds multiple `gaap_startup` ledgers in the same currency, filter to the relevant `ledger_id`.

### 8. Cash balance

Holder: company. Point-in-time USD bank cash (`1110`) and USD-ledger custodial stablecoins (`1410`)
on the primary USD ledger in force at the end of `:through` (ignores `:from`).

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Cash balance (USD)",
  "description": "USD bank cash (1110) and USD-ledger stablecoin balance (1410) on the primary USD ledger in force at the end of :through (ignores :from).",
  "sql": "WITH target_ledger AS (SELECT ledger_id FROM (SELECT f.ledger_id FROM functional_currency_periods f WHERE f.currency = 'USD' AND julianday(f.effective_from) < julianday(:through, '+1 day') AND (f.effective_until IS NULL OR julianday(:through, '+1 day') <= julianday(f.effective_until)) UNION ALL SELECT pm.value AS ledger_id FROM book_meta pm JOIN book_meta fc ON fc.key = 'functional_currency' AND fc.value = 'USD' WHERE NOT EXISTS (SELECT 1 FROM functional_currency_periods) AND pm.key = 'primary_ledger_id') LIMIT 1) SELECT CASE WHEN (SELECT ledger_id FROM target_ledger) IS NULL THEN NULL ELSE coalesce(sum(jl.signed_minor), 0) END AS cash_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code IN (1110, 1410) AND julianday(jl.effective_at) < julianday(:through, '+1 day') AND jl.ledger_id = (SELECT ledger_id FROM target_ledger)",
  "columns": [
    { "name": "cash_minor", "unit": "USD" }
  ]
}
```

**Limits:** Point-in-time balance through end of UTC day `:through` (ignores `:from`). For an
`in_progress` period, `targets read` evaluates through `period.through` rather than `today`, so
future-dated cash postings already recorded before period end are included (for cash as of `today`,
run `measures {action: "try", id, from: today, through: today}`). Selects the primary USD
`gaap_startup` ledger in force at the end of `:through` via `functional_currency_periods` (falling
back to `book_meta` `primary_ledger_id` when `functional_currency` is `'USD'` and no period history
row exists), so incorporating in place or changing functional currency reads the predecessor ledger
for earlier dates and the successor ledger afterwards without double-counting, and returns `NULL`
(`cant_be_measured`) on dates when the business's functional currency was not USD. Sums `1110` and
`1410` only; excludes `1310` (digital assets, which `saas_metrics` includes in cash),
foreign-currency bank accounts, secondary USD ledgers on non-USD-functional books, and self-held
token wallets kept in CAIP-19 token ledgers. If the business holds `1310` digital assets or
par-valued stablecoins in a self-held wallet and wants month-end cash matching `saas_metrics`
within its 12-month window, query `monthly_metrics` (`cash_minor`) instead and check
`excluded_token_units` separately.

### 9. Net burn

Holder: company. Accrual-basis operating burn: operating expenses under `5000` (`5110`–`5990`, net
of `5950` promotional credits) minus operating revenue under `4000` (`4100`–`4500`, net of `4140`
discounts). Positive when the business burned; negative when it made an operating profit. Target a
burn cap with `comparison: "at_most"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Net burn (USD)",
  "description": "Accrual-basis USD operating burn (5000-5999 operating expenses minus 4000-4999 operating revenue) during the window; negative means operating profit.",
  "sql": "SELECT coalesce(sum(jl.signed_minor), 0) AS net_burn_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code >= 4000 AND jl.account_code < 6000 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')",
  "columns": [
    { "name": "net_burn_minor", "unit": "USD" }
  ]
}
```

**Limits:** Accrual-basis operating burn in USD `gaap_startup` ledgers only on UTC `effective_at`
dates; it is not the change in cash. Excludes income tax (`6800`–`6830`), non-operating items
(`7000`-series) and foreign-currency ledgers. Can also differ from `saas_metrics` burn when
expenses post in USD-par token ledgers; for a month-end figure that matches `saas_metrics` (within
its 12-month window), query `monthly_metrics.net_burn_minor` instead.

### 10. Won value by deal owner

Holder: person (`:party`, a salesperson's `partyId`). Sums `expectedValue` on won pursuits in the
`'sales'` pipeline carrying the label `owner = :party`, with a weighted open-pipeline estimate for
the same owner on the same scale.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Won value by deal owner (USD)",
  "description": "USD expected value of sales pursuits won in the window whose owner label matches :party, with a weighted open-pipeline estimate for the same owner.",
  "sql": "SELECT (SELECT CASE WHEN count(*) = 0 THEN 0 WHEN count(*) > count(o.expected_value_minor) THEN NULL ELSE sum(o.expected_value_minor) END FROM opportunities o JOIN opportunity_labels ol ON ol.opportunity_id = o.opportunity_id AND ol.label_key = 'owner' AND ol.label_value = :party JOIN events e ON e.event_id = o.won_event_id WHERE o.pipeline_key = 'sales' AND o.status = 'won' AND o.currency = 'USD' AND julianday(e.effective_at) >= julianday(:from) AND julianday(e.effective_at) < julianday(:through, '+1 day')) AS won_minor, (SELECT CASE WHEN count(*) = 0 THEN 0 WHEN count(*) > count(o.expected_value) OR max(length(o.expected_value)) > 18 THEN NULL ELSE sum((CAST(o.expected_value AS INTEGER) / 10000) * o.weight_bps) + (sum((CAST(o.expected_value AS INTEGER) % 10000) * o.weight_bps) + 5000) / 10000 END FROM open_opportunities o JOIN opportunity_labels ol ON ol.opportunity_id = o.opportunity_id AND ol.label_key = 'owner' AND ol.label_value = :party WHERE o.pipeline_key = 'sales' AND o.currency = 'USD' AND o.expected_close_on BETWEEN :from AND :through) AS open_weighted_minor",
  "columns": [
    { "name": "won_minor", "unit": "USD" },
    { "name": "open_weighted_minor", "unit": "USD", "estimate_for": "won_minor" }
  ]
}
```

**Limits:** Hardcodes `o.pipeline_key = 'sales'` and `o.currency = 'USD'` (change `'sales'` to
`'customer'` or the business's sales pipeline key, and adjust the currency and column `unit` as
needed). Requires each pursuit to carry `labels: { "owner": "<partyId>" }`; pursuits without that
label are excluded, and a deal shared between two owners is not split. Counts pursuits whose current
`status` is `'won'` dated by the surviving winning event (so voiding a contract removes an earlier
win), and includes renewals and expansions alongside new business (for new business only, add
`AND o.kind = 'new'` to the won subquery and join `opportunities opp ON opp.opportunity_id =
o.opportunity_id` with `AND opp.kind = 'new'` in the open subquery). Because `targets read` only
evaluates started periods, a monthly quota does not show next month's open deals — use a
quarter-long period when checking quarterly pipeline cover. Open pursuits without
`expected_close_on`, or with `expected_close_on` in an earlier period, are excluded from
`open_weighted_minor`. Any open pursuit with `expectedValue: null` (or over 18 digits) expected to
close in the window makes that owner's entire `open_weighted_minor` estimate unmeasurable
(`ok: false`, never zero), while `won_minor` stays measurable. A won pursuit with `expectedValue:
null` makes `won_minor` `cant_be_measured`; a won pursuit with `expectedValue` over 18 digits
triggers integer overflow in `expected_value_minor` on `opportunities` and fails the whole measure for
the period.

### 11. Burn multiple

Holder: company. Accrual-basis USD operating burn divided by USD expected value of sales pursuits
won in the window, to two decimal places (`decimals: 2`, so `100` = `1.00x` and `200` = `2.00x`).
Target a ceiling such as 2.00x with `comparison: "at_most"` and value `"200"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Burn multiple",
  "description": "Accrual-basis USD operating burn divided by USD expected value of sales pursuits won in the window, to two decimal places (200 = 2.00x). Returns NULL when nothing was won or any won deal has no value.",
  "sql": "WITH b AS (SELECT coalesce(sum(jl.signed_minor), 0) AS burn FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code >= 4000 AND jl.account_code < 6000 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')), a AS (SELECT CASE WHEN count(*) = 0 THEN 0 WHEN count(*) > count(o.expected_value_minor) THEN NULL ELSE sum(o.expected_value_minor) END AS new_arr FROM opportunities o JOIN events e ON e.event_id = o.won_event_id WHERE o.pipeline_key = 'sales' AND o.status = 'won' AND o.currency = 'USD' AND julianday(e.effective_at) >= julianday(:from) AND julianday(e.effective_at) < julianday(:through, '+1 day')) SELECT CASE WHEN a.new_arr IS NULL OR a.new_arr <= 0 OR b.burn < -400000000000000 OR b.burn > 400000000000000 OR a.new_arr > 400000000000000 THEN NULL WHEN b.burn >= 0 THEN (b.burn / a.new_arr) * 100 + ((b.burn % a.new_arr) * 100 + a.new_arr / 2) / a.new_arr ELSE -((-b.burn / a.new_arr) * 100 + ((-b.burn % a.new_arr) * 100 + a.new_arr / 2) / a.new_arr) END AS burn_multiple FROM a, b",
  "columns": [
    { "name": "burn_multiple", "unit": "number", "decimals": 2 }
  ]
}
```

**Limits:** Uses the annual expected value of won pursuits in `'sales'` (change `'sales'` to
`'customer'` or the business's sales pipeline key, and adjust `'USD'` as needed) rather than
`monthly_metrics` net new ARR, because `monthly_metrics` cannot measure an in-progress quarter
before an event lands in its final month; it does not subtract churn or contraction. Returns `NULL`
(`cant_be_measured`) when no deals were won (`new_arr <= 0`), when any won deal in the window has
`expectedValue: null` — an unsized win makes burn multiple unmeasurable, never zero — or when
`|burn|` or `new_arr` exceeds `400000000000000` minor units (preventing 64-bit integer overflow into
floating point). Negative values mean the business made an operating profit while winning deals.
Shares net burn's accrual USD `gaap_startup` limits (excludes taxes, `7000`-series items and foreign
ledgers).

### 12. Deals won against the period before

Holder: pipeline (`:pipeline`). Counts pursuits won in the window, pursuits won in the comparison
window of the same number of calendar days immediately before `:from`, and the first as a ratio of
the second to two decimal places (`decimals: 2`, so `100` = `1.00` flat and `120` = `1.20` / 20%
more).

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Deals won against the period before",
  "description": "Pursuits won in the window, pursuits won in the window of the same length immediately before it, and the first as a ratio of the second to two decimal places (120 = 1.20, 20% more).",
  "sql": "WITH w AS (SELECT (SELECT count(*) FROM opportunities o JOIN events e ON e.event_id = o.won_event_id WHERE o.pipeline_key = :pipeline AND o.status = 'won' AND julianday(e.effective_at) >= julianday(:from) AND julianday(e.effective_at) < julianday(:through, '+1 day')) AS now_won, (SELECT count(*) FROM opportunities o JOIN events e ON e.event_id = o.won_event_id WHERE o.pipeline_key = :pipeline AND o.status = 'won' AND julianday(e.effective_at) >= julianday(date(:from, '-' || CAST(julianday(:through) - julianday(:from) + 1 AS INTEGER) || ' days')) AND julianday(e.effective_at) < julianday(:from)) AS before_won) SELECT now_won AS won, before_won AS won_before, CASE WHEN before_won = 0 THEN NULL ELSE (now_won * 100 + before_won / 2) / before_won END AS share_of_before FROM w",
  "columns": [
    { "name": "won", "unit": "number" },
    { "name": "won_before", "unit": "number" },
    { "name": "share_of_before", "unit": "number", "decimals": 2 }
  ]
}
```

**Limits:** Needs `each_period` mode; in `to_date` mode, the evaluated window grows from the
target's first `from` through `period.through`, so the comparison window shifts earlier than the
target's start instead of comparing consecutive periods. The comparison window is the **same number
of calendar days immediately before `:from`** (`date(:from, '-N days')` through `:from` exclusive),
so for adjacent months of different lengths a 30-day November compares against October 2–31 rather
than October 1–31. When every period is a full calendar month starting on the first of the month,
use the calendar-month variant for the `before_won` lower bound instead:

```sql
julianday(e.effective_at) >= julianday(date(:from, '-1 month'))
  AND julianday(e.effective_at) < julianday(:from)
```

Returns `NULL` (`cant_be_measured`) for `share_of_before` when `won_before = 0`, while `won`
and `won_before` still measure independently. Counts surviving `'won'` pursuits by UTC `effective_at`
of `won_event_id`.

### 13. Travel spend (static category cap)

Holder: company. Posted USD travel, meals and entertainment expense (`5710`–`5790`) during the
window. Target a static cap (such as `<= $10,000`) with `comparison: "at_most"` and value
`"1000000"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Travel spend (USD)",
  "description": "Posted USD travel, meals and entertainment expense (5710-5790) during the window.",
  "sql": "SELECT coalesce(sum(jl.signed_minor), 0) AS travel_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code BETWEEN 5710 AND 5790 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')",
  "columns": [
    { "name": "travel_minor", "unit": "USD" }
  ]
}
```

**Limits:** Single currency (`USD`) on `gaap_startup` ledgers by UTC `effective_at` date (if a
business incorporated in place or changed functional currency and holds multiple `gaap_startup`
ledgers in `USD`, filter to the relevant `ledger_id`). Sums `5710` (travel), `5720` (meals),
`5730` (entertainment) and `5790` (other employee travel-related costs). To cap travel alone
without meals or entertainment, narrow to `jl.account_code = 5710`; swap the account-code filter
for any other static category budget (such as `5410` for paid media or `5310`–`5390` for software
and technology tools).

### 14. Travel spend per employee

Holder: company. Posted USD travel, meals and entertainment expense (`5710`–`5790`) in the window
divided by active team memberships at the end of `:through`, returning `travel_minor` and `people`
as supporting `context` columns. Target a per-employee rate (such as `<= $1,000` per person) with
`comparison: "at_most"` and value `"100000"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Travel spend per employee (USD)",
  "description": "Posted USD travel, meals and entertainment expense (5710-5790) in the window divided by active team memberships at the end of :through, returning travel_minor and people as supporting context.",
  "sql": "WITH t AS (SELECT coalesce(sum(jl.signed_minor), 0) AS travel_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code BETWEEN 5710 AND 5790 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')), p AS (SELECT count(DISTINCT r.party_id) AS people FROM people_records r WHERE r.kind = 'membership' AND r.current = 1 AND r.valid_from < r.valid_through AND julianday(r.valid_from) < julianday(:through, '+1 day') AND julianday(r.valid_through) >= julianday(:through, '+1 day')) SELECT CASE WHEN p.people <= 0 THEN NULL WHEN t.travel_minor >= 0 THEN (t.travel_minor + p.people / 2) / p.people ELSE -((-t.travel_minor + p.people / 2) / p.people) END AS travel_per_person_minor, t.travel_minor, p.people FROM t, p",
  "columns": [
    { "name": "travel_per_person_minor", "unit": "USD" },
    { "name": "travel_minor", "unit": "USD" },
    { "name": "people", "unit": "number" }
  ]
}
```

**Limits:** Single currency (`USD`) over `5710`–`5790` on UTC `effective_at` dates (filter to the
relevant `ledger_id` if the business holds multiple USD `gaap_startup` ledgers after incorporating
in place or changing functional currency), divided by point-in-time `people` (`people_records` with
`kind = 'membership'`) at the end of UTC day `:through` (rounded half-up to the nearest minor
unit). Returns `NULL` (`cant_be_measured`) for `travel_per_person_minor` when `people = 0`, while
`travel_minor` and `people` still measure in `context`. During an `in_progress` period, `people`
includes recorded starts and exits through `period.through`; run `measures {action: "try", id,
from: period.window.from, through: today}` for headcount as of `today`. Swap `5710`–`5790` for
`5310`–`5390` for a SaaS/dev-tools-per-employee budget.

### 15. R&D AI and token spend % of Net New Revenue

Holder: company. Posted USD R&D AI and developer-tool spend (`5220` and `5320` with
`expense_function = 'research_and_development'`) as a percentage of USD won new and expansion sales
bookings during the window to two decimal places (`unit: "percent", decimals: 2`, so `"1500"` =
`15.00%`), with `rd_tokens_minor` and `net_new_revenue_minor` as supporting `context`. Target
`<= 15.00%` with `comparison: "at_most"` and value `"1500"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "R&D AI and token spend % of Net New Revenue",
  "description": "Posted USD R&D AI and developer-tool spend (5220, 5320 with expense_function research_and_development) as a percentage of USD won new and expansion sales bookings during the window to two decimal places (1500 = 15.00%), with rd_tokens_minor and net_new_revenue_minor as supporting context.",
  "sql": "WITH s AS (SELECT coalesce(sum(jl.signed_minor), 0) AS rd_tokens_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code IN (5220, 5320) AND json_extract(jl.sub_key, '$.expense_function') = 'research_and_development' AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')), r AS (SELECT CASE WHEN count(*) = 0 THEN 0 WHEN count(*) > count(o.expected_value_minor) THEN NULL ELSE sum(o.expected_value_minor) END AS net_new_revenue_minor FROM opportunities o JOIN events e ON e.event_id = o.won_event_id WHERE o.pipeline_key = 'sales' AND o.status = 'won' AND o.kind IN ('new', 'expansion') AND o.currency = 'USD' AND julianday(e.effective_at) >= julianday(:from) AND julianday(e.effective_at) < julianday(:through, '+1 day')) SELECT CASE WHEN r.net_new_revenue_minor IS NULL OR r.net_new_revenue_minor <= 0 OR s.rd_tokens_minor < -40000000000000 OR s.rd_tokens_minor > 40000000000000 OR r.net_new_revenue_minor > 40000000000000 THEN NULL WHEN s.rd_tokens_minor >= 0 THEN (s.rd_tokens_minor / r.net_new_revenue_minor) * 10000 + ((s.rd_tokens_minor % r.net_new_revenue_minor) * 10000 + r.net_new_revenue_minor / 2) / r.net_new_revenue_minor ELSE -((-s.rd_tokens_minor / r.net_new_revenue_minor) * 10000 + ((-s.rd_tokens_minor % r.net_new_revenue_minor) * 10000 + r.net_new_revenue_minor / 2) / r.net_new_revenue_minor) END AS rd_token_pct, s.rd_tokens_minor, r.net_new_revenue_minor FROM s, r",
  "columns": [
    { "name": "rd_token_pct", "unit": "percent", "decimals": 2 },
    { "name": "rd_tokens_minor", "unit": "USD" },
    { "name": "net_new_revenue_minor", "unit": "USD" }
  ]
}
```

**Limits:** Filters `5220` (AI and model inference) and `5320` (AI and developer tools) on
`json_extract(jl.sub_key, '$.expense_function') = 'research_and_development'` (filter to the
relevant `ledger_id` on multi-USD-ledger books), so customer-facing inference booked to
`cost_of_revenue` is excluded. Uses the USD expected value of won `'new'` and `'expansion'`
pursuits in `'sales'` (excluding `'renewal'` so flat renewals do not inflate net new bookings;
change `'sales'` to `'customer'` or the business's sales pipeline key, and adjust `'USD'` as
needed); does not subtract churn or contraction. Returns `NULL` (`cant_be_measured`) for
`rd_token_pct` when `net_new_revenue_minor <= 0` (for example, in week 1 of a quarter before any
deal closes), when any won deal has `expectedValue: null` (which also makes
`net_new_revenue_minor` unmeasurable in `context`), or when either amount exceeds `40000000000000`
minor units to guard 64-bit integer arithmetic; `rd_tokens_minor` still measures in `context`
whenever the query runs (SQLite `sum()` itself fails the whole query if aggregated postings exceed
64 bits). When same-period bookings are frequently `<= 0` at the start of a period, replace the
denominator's date filter with a rolling 90-day window or the immediately preceding period of equal
length:

```sql
julianday(e.effective_at) >= julianday(date(:through, '-89 days'))
  AND julianday(e.effective_at) < julianday(:through, '+1 day')
```

Or replace CTE `r` with recognized operating revenue under `4000` (`4100`–`4500`, net of `4140`
discounts, as in recipe 7) for a share-of-recognized-revenue cap (such as cloud/inference COGS
`<= 20%` of recognized revenue).

### 16. Payroll % of ARR

Holder: company. Annualized monthly USD payroll expense (`5110`, `5120`, `5130`, `5140`, `5190`
multiplied by 12) as a percentage of month-end USD ARR for `:through`'s UTC month to two decimal
places (`unit: "percent", decimals: 2`, so `"5000"` = `50.00%`), with `payroll_minor` and
`arr_minor` as supporting `context`. Use with monthly `each_period` periods and state a cap such
as `<= 50.00%` with `comparison: "at_most"` and value `"5000"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Payroll % of ARR",
  "description": "Annualized monthly USD payroll expense (5110, 5120, 5130, 5140, 5190 multiplied by 12) as a percentage of month-end USD ARR for :through's UTC month to two decimal places (5000 = 50.00%), with monthly payroll_minor and arr_minor as supporting context.",
  "sql": "WITH p AS (SELECT coalesce(sum(jl.signed_minor), 0) AS payroll_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code IN (5110, 5120, 5130, 5140, 5190) AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')), a AS (SELECT (SELECT m.arr_minor FROM monthly_metrics m WHERE m.month = substr(:through, 1, 7) AND m.currency = 'USD') AS arr_minor) SELECT CASE WHEN a.arr_minor IS NULL OR a.arr_minor <= 0 OR p.payroll_minor < -3000000000000 OR p.payroll_minor > 3000000000000 OR a.arr_minor > 40000000000000 THEN NULL WHEN p.payroll_minor >= 0 THEN ((p.payroll_minor * 12) / a.arr_minor) * 10000 + (((p.payroll_minor * 12) % a.arr_minor) * 10000 + a.arr_minor / 2) / a.arr_minor ELSE -(((-p.payroll_minor * 12) / a.arr_minor) * 10000 + (((-p.payroll_minor * 12) % a.arr_minor) * 10000 + a.arr_minor / 2) / a.arr_minor) END AS payroll_pct_of_arr, p.payroll_minor, a.arr_minor FROM p, a",
  "columns": [
    { "name": "payroll_pct_of_arr", "unit": "percent", "decimals": 2 },
    { "name": "payroll_minor", "unit": "USD" },
    { "name": "arr_minor", "unit": "USD" }
  ]
}
```

**Limits:** Requires monthly `each_period` periods because it annualizes the window's payroll by
`* 12` (for quarterly periods use `* 4`; for a full-year period drop `* 12`). Inherits the
single-currency (`USD`) posting-date limits of recipe 4 (`5110`, `5120`, `5130`, `5140`, `5190`;
excludes `5150` and `5640`; filter to the relevant `ledger_id` on multi-USD-ledger books) and the
12-month `monthly_metrics` window limits of recipe 5. Returns `NULL` (`cant_be_measured`) for
`payroll_pct_of_arr` when `arr_minor` is `NULL` or `<= 0` (or exceeds the 64-bit overflow guard
`3000000000000` monthly payroll / `40000000000000` ARR), while `payroll_minor` still measures in
`context`. To cap R&D payroll alone, add
`AND json_extract(jl.sub_key, '$.expense_function') = 'research_and_development'` to CTE `p`.

### 17. ARR per team member

Holder: company. Month-end USD ARR from `monthly_metrics` for `:through`'s UTC month divided by
active team memberships at the end of `:through`, with `arr_minor` and `people` as supporting
`context`. Target an efficiency floor (such as `>= $250,000` per team member) with
`comparison: "at_least"` and value `"25000000"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "ARR per team member (USD)",
  "description": "Month-end USD ARR from monthly_metrics for :through's UTC month divided by active team memberships at the end of :through, with arr_minor and people as supporting context.",
  "sql": "WITH a AS (SELECT (SELECT m.arr_minor FROM monthly_metrics m WHERE m.month = substr(:through, 1, 7) AND m.currency = 'USD') AS arr_minor), p AS (SELECT count(DISTINCT r.party_id) AS people FROM people_records r WHERE r.kind = 'membership' AND r.current = 1 AND r.valid_from < r.valid_through AND julianday(r.valid_from) < julianday(:through, '+1 day') AND julianday(r.valid_through) >= julianday(:through, '+1 day')) SELECT CASE WHEN a.arr_minor IS NULL OR p.people <= 0 THEN NULL WHEN a.arr_minor >= 0 THEN (a.arr_minor + p.people / 2) / p.people ELSE -((-a.arr_minor + p.people / 2) / p.people) END AS arr_per_person_minor, a.arr_minor, p.people FROM a, p",
  "columns": [
    { "name": "arr_per_person_minor", "unit": "USD" },
    { "name": "arr_minor", "unit": "USD" },
    { "name": "people", "unit": "number" }
  ]
}
```

**Limits:** Point-in-time month-end measure (`each_period` monthly periods, ignoring `:from`),
sharing `monthly_metrics` 12-month window limits (recipe 5) and `people_records` membership limits
(recipe 3). Returns `NULL` (`cant_be_measured`) for `arr_per_person_minor` when `people = 0` or
`arr_minor IS NULL`, while the measurable side still appears in `context`. `people_records` does
not store a job title or department column; for **ARR per engineer**, restrict `r.party_id` to
engineering party IDs (or to people with `expense_function = 'research_and_development'` in
`payroll_costs` when provider payroll statements are recorded).

### 18. Per-vendor spend per employee

Holder: party (`:party`, a vendor's `partyId`). Commercial USD trade payable bills
(`economic_claims` on `2110`, excluding reversed and voided bills) for `:party` during the window
divided by active team memberships at the end of `:through`, with `vendor_spend_minor` and `people`
as supporting `context`. Target a per-vendor cap (such as `<= $500` per employee on Anthropic) with
`holder: { "kind": "party", "party": "<vendor_party_id>" }`, `comparison: "at_most"` and value
`"50000"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Vendor spend per employee (USD)",
  "description": "Commercial USD trade payable bills (2110, excluding reversed and voided bills) for :party during the window divided by active team memberships at the end of :through, with vendor_spend_minor and people as supporting context.",
  "sql": "WITH v AS (SELECT CASE WHEN count(*) = 0 THEN 0 WHEN count(*) > count(CASE WHEN json_extract(ec.state, '$.unitCode') = l.unit_code AND length(ltrim(json_extract(ec.state, '$.original'), '-')) <= 18 THEN CAST(json_extract(ec.state, '$.original') AS INTEGER) END) THEN NULL ELSE sum(CAST(json_extract(ec.state, '$.original') AS INTEGER)) END AS vendor_spend_minor FROM economic_claims ec JOIN ledgers l ON l.ledger_id = ec.origin_ledger_id WHERE ec.party_id = :party AND ec.account_code = 2110 AND l.account_template = 'gaap_startup' AND l.unit_kind = 'currency' AND l.unit_code = 'USD' AND json_extract(ec.state, '$.direction') = 'payable' AND json_extract(ec.state, '$.kind') = 'commercial' AND julianday(ec.effective_at) >= julianday(:from) AND julianday(ec.effective_at) < julianday(:through, '+1 day') AND NOT EXISTS (SELECT 1 FROM economic_reversal_occurrences er WHERE er.original_source_fact_id = json_extract(ec.state, '$.origin.sourceFactId')) AND NOT EXISTS (SELECT 1 FROM postings p JOIN events ev ON ev.event_id = p.event_id WHERE p.external_id = ec.claim_id AND p.ledger_id = ec.origin_ledger_id AND p.account_code = ec.account_code AND ev.kind IN ('contract.void_reversed', 'contract.vendor_usage_bill_reversed'))), p AS (SELECT count(DISTINCT r.party_id) AS people FROM people_records r WHERE r.kind = 'membership' AND r.current = 1 AND r.valid_from < r.valid_through AND julianday(r.valid_from) < julianday(:through, '+1 day') AND julianday(r.valid_through) >= julianday(:through, '+1 day')) SELECT CASE WHEN v.vendor_spend_minor IS NULL OR p.people <= 0 THEN NULL WHEN v.vendor_spend_minor >= 0 THEN (v.vendor_spend_minor + p.people / 2) / p.people ELSE -((-v.vendor_spend_minor + p.people / 2) / p.people) END AS vendor_spend_per_person_minor, v.vendor_spend_minor, p.people FROM v, p",
  "columns": [
    { "name": "vendor_spend_per_person_minor", "unit": "USD" },
    { "name": "vendor_spend_minor", "unit": "USD" },
    { "name": "people", "unit": "number" }
  ]
}
```

**Limits:** Reads commercial USD trade payable bills directly from `economic_claims` (indexed by
`party_id`, so it stays bounded with no 2,000-claim view cap) where `account_code = 2110`,
`direction = 'payable'`, and `kind = 'commercial'` (excluding refunds, owner/distribution claims,
and card-issuer funding claims on `2190`), excluding bills reversed via
`economic_reversal_occurrences` or voided/reversed via `contract.void_reversed` and
`contract.vendor_usage_bill_reversed`. Covers both invoices on terms (`contracts.record` billing
phase) and card/bank receipts (`receipts.record`). Measures gross billed original amounts (gross of
recoverable input tax and before partial post-billing credits that leave the bill active) rather
than amortized prepaid expense recognition or raw cash outflows (an annual prepay bill lands in its
billing window), and does not see direct `journal.post` expense lines or legacy pre-priced-rights
`activity_occurrence` claims (on older books with legacy activity claims under 2,000 total claims,
query `claims` with `coalesce(c.kind, 'commercial') = 'commercial'`). A bill on a USD ledger
invoiced in a foreign currency (`unitCode != 'USD'`) or over 18 digits makes `vendor_spend_minor`
and `vendor_spend_per_person_minor` unmeasurable (`NULL`) rather than silently understating spend.
Returns `NULL` (`cant_be_measured`) for `vendor_spend_per_person_minor` when `people = 0`, while
`vendor_spend_minor` and `people` still measure in `context`.

### 19. Cash runway in months

Holder: company. Primary-ledger USD cash balance (`1110`, `1410`) at the end of `:through` divided
by accrual USD operating net burn (`5000`-series operating expenses minus `4000`-series operating
revenue on the same primary USD ledger) during the window to one decimal place
(`unit: "number", decimals: 1`, so `"120"` = `12.0` months), with `cash_minor` and
`monthly_burn_minor` as supporting `context`. Use with monthly `each_period` periods and state a
runway floor such as `>= 12.0` months with `comparison: "at_least"` and value `"120"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Cash runway (months)",
  "description": "Primary-ledger USD cash balance (1110, 1410) at the end of :through divided by accrual USD operating net burn (4000-5999) on that ledger during the window to one decimal place (120 = 12.0 months), with cash_minor and monthly_burn_minor as supporting context.",
  "sql": "WITH target_ledger AS (SELECT ledger_id FROM (SELECT f.ledger_id FROM functional_currency_periods f WHERE f.currency = 'USD' AND julianday(f.effective_from) < julianday(:through, '+1 day') AND (f.effective_until IS NULL OR julianday(:through, '+1 day') <= julianday(f.effective_until)) UNION ALL SELECT pm.value AS ledger_id FROM book_meta pm JOIN book_meta fc ON fc.key = 'functional_currency' AND fc.value = 'USD' WHERE NOT EXISTS (SELECT 1 FROM functional_currency_periods) AND pm.key = 'primary_ledger_id') LIMIT 1), c AS (SELECT CASE WHEN (SELECT ledger_id FROM target_ledger) IS NULL THEN NULL ELSE coalesce(sum(jl.signed_minor), 0) END AS cash_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code IN (1110, 1410) AND julianday(jl.effective_at) < julianday(:through, '+1 day') AND jl.ledger_id = (SELECT ledger_id FROM target_ledger)), b AS (SELECT CASE WHEN (SELECT ledger_id FROM target_ledger) IS NULL THEN NULL ELSE coalesce(sum(jl.signed_minor), 0) END AS monthly_burn_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code >= 4000 AND jl.account_code < 6000 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day') AND jl.ledger_id = (SELECT ledger_id FROM target_ledger)) SELECT CASE WHEN c.cash_minor IS NULL OR b.monthly_burn_minor IS NULL OR b.monthly_burn_minor <= 0 OR c.cash_minor > 4000000000000000 OR b.monthly_burn_minor > 4000000000000000 THEN NULL WHEN c.cash_minor <= 0 THEN 0 ELSE (c.cash_minor / b.monthly_burn_minor) * 10 + ((c.cash_minor % b.monthly_burn_minor) * 10 + b.monthly_burn_minor / 2) / b.monthly_burn_minor END AS runway_months, c.cash_minor, b.monthly_burn_minor FROM c, b",
  "columns": [
    { "name": "runway_months", "unit": "number", "decimals": 1 },
    { "name": "cash_minor", "unit": "USD" },
    { "name": "monthly_burn_minor", "unit": "USD" }
  ]
}
```

**Limits:** Designed for monthly `each_period` periods so the window's net burn equals one month's
operating burn; both `cash_minor` and `monthly_burn_minor` scope to the primary USD `gaap_startup`
ledger in force at the end of `:through`, inheriting recipe 8's cash limits (`1110` and `1410`
only, excluding `1310` digital assets and foreign accounts) and recipe 9's accrual operating burn
limits (`5110`–`5990` expenses minus `4100`–`4500` revenue). Returns `NULL` (`cant_be_measured`)
for `runway_months` when the business broke even or made an operating profit
(`monthly_burn_minor <= 0`, where runway is unbounded), when the business was not USD-functional on
`:through`, or above the 64-bit overflow guard (`4000000000000000` minor units), while `cash_minor`
and `monthly_burn_minor` still measure in `context`. Returns `0` (`0.0` months) when
`cash_minor <= 0` and burn is positive. To smooth lumpy monthly bills over a trailing 3-month
average burn on month-end periods (`:through` on the last day of a calendar month), replace CTE
`b`'s lower bound with
`julianday(jl.effective_at) >= julianday(date(:through, '+1 day', '-3 months'))` and divide the
signed 3-month total `s` with sign-aware rounding
(`CASE WHEN s >= 0 THEN (s + 1) / 3 ELSE -((-s + 1) / 3) END`). To match `saas_metrics` runway
(including `1310` digital assets) within its 12-month window — or to avoid scanning lifetime cash
postings on high-volume books — compute integer tenths (`decimals: 1`) from
`monthly_metrics.cash_minor` and `monthly_metrics.net_burn_minor` (never select
`monthly_metrics.runway_months` directly in a measure, because it is `REAL` and measures reject
fractional numbers).

### 20. Monthly net burn (static budget with context)

Holder: company. Accrual-basis USD operating net burn (`5000`-series operating expenses minus
`4000`-series operating revenue) during the window, returning `operating_expenses_minor` and
`operating_revenue_minor` as supporting `context` columns. Target a monthly burn cap (such as
`<= $30,000`) with `comparison: "at_most"` and value `"3000000"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Monthly net burn (USD)",
  "description": "Accrual-basis USD operating net burn (5000-5999 operating expenses minus 4000-4999 operating revenue) during the window, with operating_expenses_minor and operating_revenue_minor as supporting context.",
  "sql": "WITH l AS (SELECT coalesce(sum(CASE WHEN jl.account_code >= 5000 AND jl.account_code < 6000 THEN jl.signed_minor ELSE 0 END), 0) AS operating_expenses_minor, coalesce(sum(CASE WHEN jl.account_code >= 4000 AND jl.account_code < 5000 THEN -jl.signed_minor ELSE 0 END), 0) AS operating_revenue_minor FROM journal_lines jl JOIN ledgers lg ON lg.ledger_id = jl.ledger_id WHERE lg.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code >= 4000 AND jl.account_code < 6000 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')) SELECT operating_expenses_minor - operating_revenue_minor AS net_burn_minor, operating_expenses_minor, operating_revenue_minor FROM l",
  "columns": [
    { "name": "net_burn_minor", "unit": "USD" },
    { "name": "operating_expenses_minor", "unit": "USD" },
    { "name": "operating_revenue_minor", "unit": "USD" }
  ]
}
```

**Limits:** Positive when the business burned; negative when it made an operating profit (which
stays under a positive `at_most` cap). Shares the single-currency USD `gaap_startup` accrual limits
of recipe 9 (UTC `effective_at` dates; excludes taxes, `7000`-series non-operating items, and
foreign-currency or token ledgers; filter to the relevant `ledger_id` if the business holds
multiple USD `gaap_startup` ledgers after incorporating in place or changing functional currency).
Do not approximate operating cash burn by subtracting raw `payments` `in` from `out`: `payments`
includes financing/equity wires and owner contributions as `in` and both corporate-card purchases
and card-bill settlements as `out`; use `reports run default_alive` for cash runway analysis or
filter out financing, transfer, owner and card-settlement movements explicitly.

### 21. Burn multiple (dynamic budget with context)

Holder: company. Accrual-basis USD operating net burn divided by USD expected value of won new and
expansion sales pursuits during the window to two decimal places (`unit: "number", decimals: 2`, so
`"150"` = `1.50x`), returning `net_burn_minor` and `net_new_arr_minor` as supporting `context`
columns. Target a burn-multiple ceiling (such as `<= 1.50x`) with `comparison: "at_most"` and value
`"150"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Burn multiple (dynamic budget)",
  "description": "Accrual-basis USD operating net burn divided by USD won new and expansion sales bookings during the window to two decimal places (150 = 1.50x), with net_burn_minor and net_new_arr_minor as supporting context.",
  "sql": "WITH b AS (SELECT coalesce(sum(jl.signed_minor), 0) AS net_burn_minor FROM journal_lines jl JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code >= 4000 AND jl.account_code < 6000 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')), a AS (SELECT CASE WHEN count(*) = 0 THEN 0 WHEN count(*) > count(o.expected_value_minor) THEN NULL ELSE sum(o.expected_value_minor) END AS net_new_arr_minor FROM opportunities o JOIN events e ON e.event_id = o.won_event_id WHERE o.pipeline_key = 'sales' AND o.status = 'won' AND o.kind IN ('new', 'expansion') AND o.currency = 'USD' AND julianday(e.effective_at) >= julianday(:from) AND julianday(e.effective_at) < julianday(:through, '+1 day')) SELECT CASE WHEN a.net_new_arr_minor IS NULL OR a.net_new_arr_minor <= 0 OR b.net_burn_minor < -400000000000000 OR b.net_burn_minor > 400000000000000 OR a.net_new_arr_minor > 400000000000000 THEN NULL WHEN b.net_burn_minor >= 0 THEN (b.net_burn_minor / a.net_new_arr_minor) * 100 + ((b.net_burn_minor % a.net_new_arr_minor) * 100 + a.net_new_arr_minor / 2) / a.net_new_arr_minor ELSE -((-b.net_burn_minor / a.net_new_arr_minor) * 100 + ((-b.net_burn_minor % a.net_new_arr_minor) * 100 + a.net_new_arr_minor / 2) / a.net_new_arr_minor) END AS burn_multiple, b.net_burn_minor, a.net_new_arr_minor FROM a, b",
  "columns": [
    { "name": "burn_multiple", "unit": "number", "decimals": 2 },
    { "name": "net_burn_minor", "unit": "USD" },
    { "name": "net_new_arr_minor", "unit": "USD" }
  ]
}
```

**Limits:** Returns `NULL` (`cant_be_measured`) for `burn_multiple` when `net_new_arr_minor <= 0`
(no won new or expansion deals in the window), when any won deal has `expectedValue: null` (which
also makes `net_new_arr_minor` unmeasurable in `context`), or above the 64-bit overflow guard
(`400000000000000` minor units), while `net_burn_minor` (and `net_new_arr_minor = 0` when nothing
closed) still measure in `context`. Negative values mean the business made an operating profit
while winning deals. Counts `'new'` and `'expansion'` won pursuits (excluding `'renewal'`); change
`'sales'` to `'customer'` or the business's sales pipeline key, filter to the relevant `ledger_id`
on multi-USD-ledger books, and adjust `'USD'` as needed. Won bookings do not subtract churn or
contraction; to measure net ARR change including churn across month-end snapshots within the
12-month `monthly_metrics` window, replace CTE `a` with `m_end.arr_minor - m_prev.arr_minor`
(leaving a missing prior month as `NULL` / `cant_be_measured` instead of coalescing to zero).

### 22. Key-account customer expansion

Holder: party (`:party`, a strategic customer's `partyId`). Accrual-basis USD operating revenue
under `4000` (`4100`–`4500`, net of `4140` discounts, reversals and voided-contract reversals)
recognized for customer `:party` during the window, with `active_contracts` and `invoices`
(commercial USD trade receivable invoices billed to `:party` in the window, excluding reversed and
voided invoices) as supporting `context` columns. Target a key-account expansion floor (such as
`>= $10,000` recognized in the month) with `holder: { "kind": "party", "party":
"<customer_party_id>" }`, `comparison: "at_least"` and value `"1000000"`.

```json
{
  "spec": "economico.measure/1",
  "report_schema": 1,
  "name": "Key-account customer revenue (USD)",
  "description": "USD operating revenue (4000-4999, net of reversals and voids) recognized on customer :party's contracts during the window, with active_contracts and commercial USD invoice count as supporting context.",
  "sql": "SELECT (SELECT coalesce(sum(-jl.signed_minor), 0) FROM subject_index si JOIN contracts c ON c.contract_id = si.subject_id JOIN events e ON e.subject_type = 'contract' AND e.subject_id = c.contract_id JOIN journal_lines jl ON jl.event_id = e.event_id JOIN ledgers l ON l.ledger_id = jl.ledger_id WHERE si.subject_type = 'contract' AND si.key_name = 'party' AND si.key_value = :party AND json_extract(c.parties, '$.customer') = :party AND l.account_template = 'gaap_startup' AND jl.currency = 'USD' AND jl.account_code >= 4000 AND jl.account_code < 5000 AND julianday(jl.effective_at) >= julianday(:from) AND julianday(jl.effective_at) < julianday(:through, '+1 day')) AS recognized_revenue_minor, (SELECT count(*) FROM subject_index si JOIN contracts c ON c.contract_id = si.subject_id WHERE si.subject_type = 'contract' AND si.key_name = 'live_party' AND si.key_value = :party AND c.status = 'active' AND c.replaced_by IS NULL AND json_extract(c.parties, '$.customer') = :party) AS active_contracts, (SELECT count(*) FROM economic_claims ec JOIN ledgers l ON l.ledger_id = ec.origin_ledger_id WHERE ec.party_id = :party AND ec.account_code = 1120 AND l.account_template = 'gaap_startup' AND l.unit_kind = 'currency' AND l.unit_code = 'USD' AND json_extract(ec.state, '$.direction') = 'receivable' AND json_extract(ec.state, '$.kind') = 'commercial' AND julianday(ec.effective_at) >= julianday(:from) AND julianday(ec.effective_at) < julianday(:through, '+1 day') AND NOT EXISTS (SELECT 1 FROM economic_reversal_occurrences er WHERE er.original_source_fact_id = json_extract(ec.state, '$.origin.sourceFactId')) AND NOT EXISTS (SELECT 1 FROM postings p JOIN events ev ON ev.event_id = p.event_id WHERE p.external_id = ec.claim_id AND p.ledger_id = ec.origin_ledger_id AND p.account_code = ec.account_code AND ev.kind = 'contract.void_reversed')) AS invoices",
  "columns": [
    { "name": "recognized_revenue_minor", "unit": "USD" },
    { "name": "active_contracts", "unit": "number" },
    { "name": "invoices", "unit": "number" }
  ]
}
```

**Limits:** Single currency (`USD`) on `gaap_startup` ledgers by UTC `effective_at` date (filter to
the relevant `ledger_id` on multi-USD-ledger books). Scopes revenue through `subject_index` and
`contracts` (`json_extract(c.parties, '$.customer') = :party`) into contract events and postings so
work stays bounded to `:party`'s customer contracts and nets `contract.economics_recorded`,
`contract.economics_reversed`, `contract.void_reversed` and cancellation revenue postings accurately;
direct `journal.post` revenue entries not tied to a customer contract are excluded.
`active_contracts` counts currently active, unreplaced customer contracts referencing `:party` at
read time rather than historical contract count on `:through`, while `recognized_revenue_minor` and
`invoices` are strictly windowed to `:from..:through`. `invoices` reads commercial `1120`
receivables directly from `economic_claims` (indexed by `party_id`, avoiding the 2,000-claim view
cap and excluding refunds/financing claims as well as reversed or voided invoices; omits legacy
pre-priced-rights `activity_occurrence` claims).
