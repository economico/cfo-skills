# Measures and targets

A **measure** is a named, versioned, read-only SQL query over the report schema that returns one
row of whole-number figures for a date window, such as pursuits won in a pipeline, people in place,
or ARR on a date. A **target** holds one pipeline, one person (a **quota**) or the whole company
to stated whole-number values against a pinned measure version, period by period. An
**operating plan** is the set of targets that share a `plan` label.

Recording, revising, labelling, activating, archiving or reading a measure or target never posts
to any ledger. Progress is always read from the books and open pipelines, never entered by hand.

## Tools, commands and permissions

Describe every command with `catalog` before calling it.

- **Measures:** `measures.define` records a new measure (`msr_` ID, version 1) or a new immutable
  version of an existing measure (pass `expectedVersion` when revising so a concurrent update fails
  with `conflict` naming the current version instead of silently overwriting it).
  `measures {action: "list"}` pages recorded measures (100/page); `measures {action: "get", id, version?}`
  reads one definition; `measures {action: "versions", id}` pages its versions (`list`, `get` and
  `versions` return `version_event_id`, `recorded_at` and `recorded_by`);
  `measures {action: "try", id? | definition?, version?, from, through, pipeline? | party?}` runs a
  draft or recorded measure for a window and holder without recording anything and refuses a holder
  that does not fit the measure.
- **Targets:** `targets.define` records a new target (`tgt_` ID, version 1) or revises an existing
  one; `reason` (up to 200 characters) is required on every call, including creating a new target.
  Pass `expectedVersion` (the version you read and edited) when revising so a concurrent change is
  refused with `conflict` naming the current version. `targets.label` sets or removes key/value
  labels outside the versioned definition. `targets.activate` moves `proposed` or `archived` to
  `active`; `targets.archive` moves `proposed` or `active` to `archived`.
- **Target reads:** `targets {action: "list", status?, holder_kind?, pipeline?, party?, label_key?, label_value?, cursor?}`
  lists active targets by default (100/page); `targets {action: "get", id, version?, as_of?}` reads
  a definition; `targets {action: "versions", id}` pages revisions;
  `targets {action: "read", id, version?, as_of?, cursor?, current_only?}` evaluates up to 12
  started periods against the pinned measure version (plus every upcoming `not_started` period on
  the first page). Pass `current_only: true` (refused with `cursor`) for a single-period summary
  table: it returns every period on one page (`next_cursor: null`), runs only the current (or latest
  started) period, and marks other started periods `"not_run"`.
- **Permissions:** Reading measures and targets requires whole-books `report_sql` read access.
  Recording, revising, labelling, activating or archiving also requires `reports.manage`.

### Key rules and limits

- **Parameters.** A measure's SQL may bind only `:from`, `:through`, `:pipeline` and `:party` (and
  never both `:pipeline` and `:party`). `:from` and `:through` are inclusive calendar dates
  (`YYYY-MM-DD`); a point-in-time measure ignores `:from`.
- **Columns and units.** 1 to 12 declared columns, each with `unit: "number"` (counts or ratios) or
  a 3-letter uppercase ISO 4217 currency code (minor units, e.g. `"USD"`). Every column returns an
  exact whole number (string on JSON surfaces). Keeping amounts in the declared currency is the
  query's job. A `"number"` column may declare optional `decimals` (`1` to `6`) when its figure has
  a fractional part: the query still returns a whole number scaled by `10^decimals`, and
  `measures try` and `targets read` return `decimals` beside each column, stated value and estimate.
  Omit `decimals` for whole numbers (`"decimals": 0` is refused), and never set `decimals` on a
  currency column (currency amounts are already in minor units).
- **Open-pipeline estimates.** A column may set `estimate_for` to another declared column of the
  same unit (the two may declare different `decimals`, e.g. a `decimals: 2` weighted count estimate
  for a whole-number count). A target cannot state a value on an estimate column; `targets read`
  returns it beside the stated column under `estimate`, and never counts it toward `actual`, `gap`
  or `status`.
- **Holder fit.** A measure using `:pipeline` requires `{ "kind": "pipeline", "pipeline": "<key>" }`
  on an existing pipeline; one using `:party` requires `{ "kind": "party", "party": "<party_id>" }`
  on an existing party; one using neither requires `{ "kind": "company" }`.
- **Periods and modes.** 1 to 52 explicit, non-overlapping periods in date order (`from <= through`;
  gaps and unequal lengths allowed). `"each_period"` (default) passes each period's own window;
  `"to_date"` passes the target's first `from` through that period's `through`. In `"to_date"` mode,
  stated values must be cumulative from the first period's `from` through each period's `through`
  (stating a single-month figure on every period makes later cumulative `at_least` periods trivially
  met and cumulative `at_most` periods trivially breached). When expanding a cadence such as
  "monthly for a year" into explicit periods, do not invent a split: record the period numbers the
  founder stated, or propose a derived split clearly labelled as derived and confirm every period's
  number before activating. For a goal "by" a date on a month-end or point-in-time figure (ARR,
  MRR, cash balance, people in place), record one `each_period` period per month end rather than
  one long period: a single long ARR or MRR period reads `cant_be_measured` until an event lands in
  its final month, and `"to_date"` means nothing for a point-in-time measure. Still follow
  proposed-first: use the founder's monthly numbers or a derived split clearly labelled as derived
  and confirm before activating.
- **Time zones and UTC bucketing.** Period status (`met`, `not_met`, `in_progress`, `not_started`,
  or `not_run` for non-current started periods under `current_only: true`) compares `period.from`
  and `period.through` against `today` in the business's configured local `time_zone`, while every
  starter recipe below buckets events and postings by UTC `effective_at` date. For non-UTC
  businesses, a period can read final up to a day before or after its UTC window closes, and a
  late-evening local entry can land in the next UTC period.
- **Stated values.** Each stated value names a non-estimate column, `comparison` (`"at_least"` or
  `"at_most"`), and a signed 64-bit whole-number decimal string in that column's scale: minor units
  for currency columns, whole counts for plain `"number"` columns, and scaled by `10^decimals` when
  a `"number"` column declares `decimals` (for example, on a `decimals: 2` column, `2.00` is
  recorded as `"200"`, so a burn multiple cap of `<= 2.00x` states `"200"` with `"at_most"`).
  `targets read` evaluates each started period through its `period.through` (even when
  `in_progress`) and returns `stated`, `actual`, `gap` (`actual - stated`), `decimals` (when
  declared), and `met`, `not_met`, `in_progress`, `cant_be_measured` (with `reason`), `not_run`
  (started period skipped under `current_only: true`, returning `stated` only), or `not_started`
  (future periods are never run).
- **Bounds.** Definition at most 64 KiB; `name` up to 120 characters; `reason` up to 200; up to 20
  labels (values up to 200 characters); lists page at 100; `targets read` runs at most 12 started
  periods per call (page older periods with `next_cursor` and the same `version` or `as_of`).

## Proposed-first convention

Record every new target as `proposed` (the default when `status` is omitted on `targets.define`)
unless the founder explicitly stated the exact period numbers and asked to make them active:

1. Check existing measures with `measures {action: "list"}` before defining a duplicate.
2. Draft the measure and test it with `measures {action: "try", definition, from, through}` against
   a real window before calling `measures.define`; pass `pipeline` if the query binds `:pipeline`,
   or `party` if it binds `:party`.
3. Call `targets.define` with a short `reason` (required on every `targets.define`, including new
   targets) and without `status` (or with `status: "proposed"`). If the founder gave only an annual
   or total figure rather than per-period numbers, record your derived split as `proposed` and label
   it clearly as derived. Proposed targets stay out of default `targets list` reads (pass
   `status: "proposed"` or `"all"` to list them) and remain readable with `targets {action: "read", id}`.
4. Show the founder the stated period numbers, the pinned measure, and a preview `targets read`
   output.
5. Confirm with the founder following [asking the founder](asking-the-founder.md) (recommended
   option first). Once the founder confirms the numbers (revising the proposed target first if they
   adjust any figure), call `targets.activate`.

Revising an active target (`targets.define` with `id`, `expectedVersion` and `reason`) takes effect
immediately as a new version while keeping every earlier version; passing `expectedVersion` (the
version you read and edited) ensures a concurrent revision fails with `conflict` naming the current
version rather than silently replacing someone else's edit. Confirm the founder's new numbers before
recording a revision. When re-forecasting mid-year, carry the earlier periods' stated numbers
unchanged and update only the future periods so past periods stay comparable against what was
agreed.

## Closing deals, hires and rounds

Keep pipeline actuals, roles and cap table in step in one sitting when a pursuit closes (full
mechanics in [business system](business-system.md#the-pipelines)):

- **Signed amount differs from estimate:** before accepting a customer contract, SAFE or round
  agreement, call `opportunities.stage` with `expectedValue` set to the signed amount so won value
  equals what was signed.
- **Automatic roles:** declare `partyCategories` on the template or contract (`{"customer":
  ["customer"]}`, `{"vendor": ["vendor"]}`, `{"investor": ["investor"]}`) so acceptance gives the
  party its role automatically; use `parties.update` (keeping existing categories) only for an
  agreement recorded without it.
- **Fundraising close (one motion):** link and accept the SAFE or round contract (which wins the
  pursuit and adds the `investor` role), and in the same sitting record the commitment
  (`contracts.record` on the `exercise` phase, which puts it on the cap table — acceptance alone
  records no obligation). Record the wire with `payments.record` once the bank shows it.
- **Hiring close (one motion):** record `people.record_arrangement` (wins the hiring pursuit) and
  `people.set_membership` (makes the person count in "people in place") together in one sitting.

## Starter measure recipes

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

## Operating-plan recipe

An **operating plan** is a set of targets sharing a `plan` label (such as `"2026-annual"` or
`"2026-q4"`).

1. **Record the plan's targets** with `targets.define` (passing `reason` and
   `labels: { "plan": "2026-annual" }` on each, or attaching the label later with `targets.label`).
   Start each target `proposed` unless the founder stated the exact period numbers and asked to
   activate them immediately.
2. **List the plan** with `targets {action: "list", label_key: "plan", label_value: "2026-annual"}`.
   Pass `status: "all"` when reviewing proposed or archived targets alongside active ones. Note
   that listing by label returns the plan's **current** members (labels are mutable outside target
   versions, so targets added or relabelled later change what `list` returns). If the founder wants
   a fixed original-plan comparison later, record the set of target IDs and adopted version numbers
   when the plan is activated.
3. **Compare original plan against current plan** after a mid-year revision:
   - Do not assume the original adopted plan is `version: 1`: under proposed-first, version 1 may
     be an unconfirmed draft and a pre-activation edit makes version 2 the adopted plan. Confirm
     each target's adopted version with `targets {action: "versions", id}` (or pass
     `as_of: "<YYYY-MM-DD>"`, which selects the highest version **recorded** on or before that UTC
     calendar date — `as_of` compares `recorded_on`, not an adoption date or `effective_at`).
     Prefer `version` over `as_of` for a revision made the same day: `as_of` goes by the UTC date
     a version was recorded, while `today` is in the business's configured time zone, so a revision
     recorded late or early in the local day can fall on an adjacent UTC date.
   - Read each target at its adopted revision (`targets {action: "read", id, version: <adopted>}`
     or `as_of: "<YYYY-MM-DD>"`) and again at its latest version (`targets {action: "read", id}`).
     Comparing a revision always requires `targets {action: "read"}` at **each** version — never
     `targets {action: "get"}` for one side — because only `targets read` computes that version's
     `gap`, `status` and pinned-measure `actual` against the books.
   - Both reads evaluate actuals against current books; only the stated values, pinned measure
     version, holder, mode and periods change between revisions.
4. **Query stated plan numbers in `report_sql`** when building a custom report over the plan:
   `targets`, `target_labels`, `target_versions` and the `target_period_values` view are readable
   in `report_sql`. `value_minor` on `target_period_values` is exact across the whole signed 64-bit
   range and never `NULL` for a valid target, so compare through `value_minor` in SQL (and select
   `value` to read back a stated value above 2^53 in JSON). `target_period_values` also carries
   `decimals` (`0` for whole-number or currency columns): a `"number"` column's stated value is
   scaled by `10^decimals`, so divide by `10^decimals` when showing it. Live actuals come only from
   `targets read`.

## One-table recipe

When the founder asks "how are we doing against our plan?" or wants several targets side by side:

1. List the set of targets with `targets {action: "list", label_key: "plan", label_value: "<plan>"}`
   (or filter by `holder_kind`, `pipeline`, `party`, or omit filters for all active targets).
2. Call `targets {action: "read", id}` for each target in the list (for a historical plan baseline,
   pass the adopted `version` or UTC `as_of` recorded date on each read).
3. `actual`, `gap`, `status` and `estimate` belong to each **stated column** inside
   `period.values[]` (not to the period as a whole; a target may state multiple columns per period).
   Select the matching period (typically the current `in_progress` period or the most recently
   ended period) and build one row per target, period and stated column:

| Target | Column | Holder | Period | Stated | Actual (through period end) | Gap (`actual - stated`) | Status | Open pipeline | Pace / Coverage (computed) |
|---|---|---|---|---|---|---|---|---|---|
| Q4 Hiring | `won` | pipeline: `hiring` | 2026-11 | >= 2 | 0 | -2 | in_progress | 1.20 (est.) | 0.60x cover |
| Q4 Won Value | `won_minor` | pipeline: `customer` | 2026-11 | >= $25,000 | $0 | -$25,000 | in_progress | $19,200 (est.) | 0.77x cover |
| Q4 ARR | `arr_minor` | company | 2026-11 | >= $15,000 | $12,000 | -$3,000 | in_progress | — | stock (no linear pace) |
| Q4 Net Burn | `net_burn_minor` | company | 2026-11 | <= $20,000 | $18,000 | -$2,000 | in_progress | — | 90% of cap used |

Gap sign rule: `gap = actual - stated`. A negative gap means **short of** an `at_least` target and
**under the cap** for an `at_most` target; a positive gap means **above** an `at_least` target and
**over the cap** for an `at_most` target.

Formatting rules:
- Present every number using its `decimals`: scale every currency column once by the currency's
  decimals (`10^2` for USD), and scale every `"number"` column or estimate that carries `decimals`
  once by `10^decimals`, placing each value's decimal point by its own `decimals` (for example,
  `open_weighted: "120"` with `decimals: 2` displays as `1.20`, and `burn_multiple: "11"` with
  `decimals: 2` displays as `0.11x`). Never sum across currencies or units.
- Note that `actual` for an `in_progress` period is evaluated as recorded through period end
  (`period.through`), including any future-dated starts, ends or postings already on the books.
- Keep `estimate` values clearly marked as open-pipeline estimates, never added into `Actual`.
- When a column returns `cant_be_measured`, show `can't be measured` and its `reason` instead of
  filling in zero.
- When comparing an original adopted revision against the current revision, call
  `targets {action: "read"}` at **each** version (never `targets {action: "get"}` for one side),
  because only `targets read` evaluates each version's `gap`, `status` and pinned-measure `actual`;
  show each revision's own `Actual` and `Gap` unless both revisions have the exact same pinned
  measure (`id` and `version`), `holder`, `column`, `mode` and `window`; only when all five match
  can you place `Original stated` and `Current stated` beside a single `Actual`.

## Pace and coverage formulas

`targets read` returns no pace, coverage ratio or cross-period total because each depends on an
assumption Economico cannot verify. Compute them from the read response when helpful, and state the
assumption beside the figure.

### Linear time pace (flow measures, `in_progress` periods)

Use only for flow measures where uniform daily progress is a reasonable benchmark (for example,
steady monthly revenue or burn). Do not compute linear daily pace for point-in-time stock measures
(cash balance, people in place, ARR, MRR) or lumpy enterprise deals and single-seat hires.

Using the read's `today` (in the business's `time_zone`) and the period's evaluated inclusive
`period.window.from` and `period.window.through` (which equals `period.from..period.through` in
`each_period` mode and the cumulative `periods[0].from..period.through` window in `to_date` mode,
where `stated` and `actual` are both cumulative):

```text
elapsed_days     = (today - period.window.from) + 1
total_days       = (period.window.through - period.window.from) + 1
expected_to_date = stated * (elapsed_days / total_days)
pace_ratio       = actual / expected_to_date          (when expected_to_date != 0)
projected_end    = actual * (total_days / elapsed_days)
```

- Because `targets read` evaluates `actual` through `period.window.through` rather than `today`,
  `actual` is not a true "through today" figure if future-dated postings already exist inside the
  current period. When future-dated postings exist, run
  `measures {action: "try", id, version, from: period.window.from, through: today}` (with the
  target's `pipeline` or `party` if any) for a true to-today figure before computing pace.
- For `comparison: "at_most"` with `stated > 0` (such as a positive burn or payroll cap), report
  `actual / stated` as share of cap used and compare it to `elapsed_days / total_days` rather than
  calling higher numbers "ahead". When `stated <= 0` (for example, a break-even `0` cap or a
  negative net-burn operating-profit cap), do not divide by `stated`: show `actual`, `stated`
  and `gap` directly and state whether `actual` is at or below the cap.
- Always note: "assuming linear pace across calendar days."

### Open-pipeline coverage (`estimate_for` columns, `at_least` targets)

Compute coverage and `implied_total` **only** for `in_progress` periods, **only** when the stated
column has `comparison: "at_least"` and a measured `actual` (a read can return `estimate.ok: true`
while the stated column is `cant_be_measured` with no `actual` or `gap`), and **only** when
`estimate.ok` is `true`:

```text
actual_scaled   = actual / 10^(stated.decimals ?? 0)
stated_scaled   = stated / 10^(stated.decimals ?? 0)
estimate_scaled = estimate.value / 10^(estimate.decimals ?? 0)
remaining_gap   = max(0, stated_scaled - actual_scaled)
coverage_ratio  = estimate_scaled / remaining_gap       (when remaining_gap > 0)
implied_total   = actual_scaled + estimate_scaled
```

- When `remaining_gap == 0`, the period target is already met and requires no open pipeline cover.
- **Unavailable estimate caused by unsized deals (`estimate.ok` is `false`):** never treat an
  unmeasurable estimate as zero. Query `open_opportunities` (joined with `parties` and, for a
  person quota, `opportunity_labels`) via `report_sql` to: (1) name every unsized deal
  (`expected_value IS NULL`), (2) compute the weighted total of the sized deals only and show it
  clearly labelled as **partial (excluding unsized deals)** — never present that partial figure as
  the estimate — and (3) ask the founder for the missing values.
- **Period that has not started (`not_started`):** `targets read` does not run future periods. To
  check open-pipeline coverage for a period that has not started yet, run
  `measures {action: "try", id, version, from, through}` (with the target's `pipeline` or `party`)
  for that period's window and label the result plainly as an estimate for a period that has not
  started.
- Note that an open pursuit whose `expected_close_on` fell in an earlier period is not included in
  the current period's `estimate`.
- Always label `implied_total` as actual plus weighted open-pipeline estimate — never as booked
  revenue, cash or confirmed hires.

## Checking the plan (session routine)

When a business has active targets, check the plan at the start of a working session (and whenever
asked):

1. List active targets with `targets {action: "list"}` (or filter by `label_key: "plan"`).
2. Read each target with `targets {action: "read", id}`, compute pace and open-pipeline coverage
   per the formulas above, and break out any unsized deals separately as a partial figure.
3. **Lead with what is short and by how much**, followed by whether the open pipeline covers each
   gap, then the targets that are on track or met.

State plainly to the founder that Economico does not send alerts or notify anyone on its own when a
target falls behind. If your host supports scheduled tasks, offer to set up a recurring plan check
and say what it needs (the business slug, the schedule, and read access to the books).

## Plan against actual as a saved custom report

When the founder wants a workspace page or shareable view showing stated plan numbers beside live
actuals, start from [the plan-vs-actual report](reports/plan-vs-actual.json). It reads stated
period values from `target_period_values` (joined with `targets` and `target_labels`) and computes
live actuals using the exact same SQL queries as the starter measures. Before validating or
publishing:

1. Fill in the business's recorded measure IDs and pinned versions in the parameter defaults and
   options (`won_value_measure_id` / `won_value_measure_version`, `cash_measure_id` /
   `cash_measure_version`, and `net_burn_measure_id` / `net_burn_measure_version`) from
   `measures {action: "list"}` so every block restricts itself strictly to targets pinned to the
   measure ID and version whose SQL that block implements. Never drop the `t.measure_id` and
   `t.measure_version` filters: selecting by column name alone would compute the wrong actual for a
   target pinned to a different measure or version. Set each version parameter's `min`, `max` and
   `default` to the same published version, so a viewer cannot select targets pinned to another
   version while the block still runs the SQL written for this one.
2. Add the business's own `plan` label values to the `plan` parameter's `options`.
3. Fill in `sales_target_id` with the target ID you want to plot so the chart tracks a single
   target. When a pipeline has multiple targets, add one chart block per target rather than
   combining two targets into one line.
4. Validate with `report_format {action: "check", definition}`, preview with
   `reports {action: "preview", definition}`, show the founder, and publish with
   `report_definitions.publish` when they agree.

A period that has not started yet shows its stated value with actual, gap and open weighted left
empty, as `targets read` does. A stated value whose magnitude is 2^53 or more cannot be returned as a
number, so the report leaves it empty too; `targets read` returns it exactly. The report counts a period as started from its first day in UTC,
while `targets read` uses the business's time zone, so the two can differ for a few hours around
midnight on a period's first day.

**Caveat:** a custom report cannot invoke a measure's stored SQL dynamically. Each block repeats
the measure's query and **must stay pinned to the exact measure ID and version each target uses**;
when a target is revised to a new measure version or uses a custom measure, update the block SQL and
its measure parameter and republish the report. `targets read` remains the authoritative read.

## Executable recipes

Each recipe below replays end to end against the kernel:

- [Sales quarter with quotas](recipes/targets-sales-quarter-quotas.json) — custom `sales` pipeline,
  three salespeople, won, lost, open and unsized deals, a pipeline target and three per-person
  quotas read with open-pipeline estimates.
- [Operating plan (proposed, activated, revised)](recipes/targets-operating-plan.json) — targets
  grouped under a `plan` label, recorded `proposed`, activated, revised mid-quarter with a reason,
  and read against both version 1 and the current version.
- [Hiring plan](recipes/targets-hiring-plan.json) — custom `hiring` pipeline settled by
  `people.record_arrangement`, `people.set_membership` for headcount, and `to_date` hires plus
  `each_period` people-in-place targets.
- [Fundraise](recipes/targets-fundraise.json) — `financing` seed-round pipeline with a signed and
  funded SAFE winning its pursuit, open and lost investors, and a raised-amount target with its
  weighted open estimate.

See [business system](business-system.md) for defining pipelines and settling pursuits, and
[verify](verify.md#a-question-no-report-answers) for `report_sql` rules.
