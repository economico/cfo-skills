# Measures and targets

A **measure** is a named, versioned, read-only SQL query over the report schema that returns one
row of whole-number figures for a date window, such as pursuits won in a pipeline, people in place,
travel spend per employee, or ARR on a date. A **target** holds one pipeline, one person (a
**quota**), a vendor or customer party, or the whole company to stated whole-number values against a pinned
measure version, period by period. A **budget** is the founder word for an `at_most` target —
either a static spending cap (such as at most `$10,000` on travel) or a dynamic operating rate
(such as at most `$1,000` per employee on travel, or `15%` of Net New Revenue on R&D tokens). An
**operating plan** is the set of targets that share a `plan` label.

Recording, revising, labelling, activating, archiving or reading a measure or target never posts
to any ledger and never blocks recording a real expense that already happened. Progress is always
read from the books and open pipelines, never entered by hand.

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
- **Columns and units.** 1 to 12 declared columns, each with `unit: "number"` (counts or ratios),
  `unit: "percent"` (percentages), or a 3-letter uppercase ISO 4217 currency code (minor units,
  e.g. `"USD"`). Every column returns an exact whole number (string on JSON surfaces). Keeping
  amounts in the declared currency is the query's job. A `"number"` or `"percent"` column may
  declare optional `decimals` (`1` to `6`) when its figure has a fractional part: the query still
  returns a whole number scaled by `10^decimals`, and `measures try` and `targets read` return
  `decimals` beside each column, stated value, estimate and context value (for example, on a
  `unit: "percent", decimals: 2` column, `"1500"` displays as `15.00%`; without `decimals`, `"15"`
  displays as `15%`). Omit `decimals` for whole numbers (`"decimals": 0` is refused), and never set
  `decimals` on a currency column (currency amounts are already in minor units).
- **Open-pipeline estimates.** A column may set `estimate_for` to another declared column of the
  same unit (the two may declare different `decimals`, e.g. a `decimals: 2` weighted count estimate
  for a whole-number count). A target cannot state a value on an estimate column; `targets read`
  returns it beside the stated column under `estimate`, and never counts it toward `actual`, `gap`
  or `status`.
- **Supporting context columns.** Any declared non-estimate column that a period does not state
  (such as `travel_minor` and `people` beside `travel_per_person_minor`, or `rd_tokens_minor` and
  `net_new_revenue_minor` beside `rd_token_pct`) is returned in declared order on every measured
  started period under `period.context[]` (`{ column, unit, decimals?, ok: true, value }` or
  `{ column, unit, decimals?, ok: false, reason }`) from the same measure execution with zero extra
  SQL queries. One column returning `NULL` never fails the stated column or other context columns
  (though a query-level or view-binding error — such as an 18-digit integer overflow in
  `monthly_metrics` or `claims` exceeding its 2,000-row cap — fails the whole period), and when a
  ratio's denominator is zero (`cant_be_measured`), `context` still shows the real spend and
  denominator. `context` is omitted when every non-estimate column is stated, and on
  `not_started` or `not_run` periods.
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
  for currency columns, whole counts or whole percentages for unscaled `"number"` / `"percent"`
  columns, and scaled by `10^decimals` when a `"number"` or `"percent"` column declares `decimals`
  (for example, on a `unit: "number", decimals: 2` column, `2.00` is recorded as `"200"`, so a burn
  multiple cap of `<= 2.00x` states `"200"` with `"at_most"`; on a `unit: "percent", decimals: 2`
  column, a `<= 15.00%` budget states `"1500"` with `"at_most"`).
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
5. Confirm with the founder following [asking the founder](../../economico/references/asking-the-founder.md) (recommended
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
mechanics in [business system](../../economico/references/pipelines.md#the-pipelines)):

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

Nothing is seeded into a business's books. Read [starter measure recipes](measure-recipes.md) for
the 22 tested SQL measure definitions (pursuits, sales bookings, MRR/ARR, cash, burn, runway,
headcount, per-person quotas, dynamic spend budgets, payroll % of ARR, and key-account revenue).
Adopt only what the founder needs, adjust currency and pipeline keys to the business's books,
test with `measures {action: "try"}`, and record with `measures.define`.

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
   `decimals` (`0` for whole-number or currency columns): a `"number"` or `"percent"` column's
   stated value is scaled by `10^decimals`, so divide by `10^decimals` when showing it. Live
   actuals come only from `targets read`.

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

| Target | Column | Holder | Period | Stated | Actual (through period end) | Gap (`actual - stated`) | Status | Context / Open pipeline | Pace / Coverage (computed) |
|---|---|---|---|---|---|---|---|---|---|
| Q4 Hiring | `won` | pipeline: `hiring` | 2026-11 | >= 2 | 0 | -2 | in_progress | 1.20 (est.) | 0.60x cover |
| Q4 Won Value | `won_minor` | pipeline: `customer` | 2026-11 | >= $25,000 | $0 | -$25,000 | in_progress | $19,200 (est.) | 0.77x cover |
| Q4 ARR | `arr_minor` | company | 2026-11 | >= $15,000 | $12,000 | -$3,000 | in_progress | — | stock (no linear pace) |
| Q4 Travel Budget | `travel_per_person_minor` | company | 2026-11 | <= $1,000 | $708.33 | -$291.67 | in_progress | $8,500 spent · 12 people | $3,500 headroom |
| Q4 R&D Tokens | `rd_token_pct` | company | 2026-11 | <= 15.00% | 11.00% | -4.00% | in_progress | $11,000 spend · $100,000 NNR | $4,000 headroom |

Gap sign rule: `gap = actual - stated`. A negative gap means **short of** an `at_least` target and
**under the cap** for an `at_most` target; a positive gap means **above** an `at_least` target and
**over the cap** for an `at_most` target. For an `in_progress` `at_most` budget where `actual >
stated` (positive gap), call out plainly that it is **over budget** while the period is still open.

Formatting rules:
- Present every number using its `unit` and `decimals`: scale every currency column once by the
  currency's decimals (`10^2` for USD), scale every `"number"` or `"percent"` column, estimate or
  context value that carries `decimals` once by `10^decimals`, and append `%` for `unit: "percent"`
  (for example, `open_weighted: "120"` with `unit: "number", decimals: 2` displays as `1.20`,
  `burn_multiple: "11"` with `unit: "number", decimals: 2` displays as `0.11x`, and `rd_token_pct:
  "1500"` with `unit: "percent", decimals: 2` displays as `15.00%`). Never sum across currencies or
  units.
- Show supporting `period.context[]` values right beside ratio and rate targets so the founder sees
  the real dollars and driver (such as `$8,500` travel across `12` people behind `$708.33 / <=
  $1,000.00`, or `$4,000` R&D tokens on `$0` Net New Revenue when `rd_token_pct` is
  `cant_be_measured`). Never issue extra SQL queries when `context` already carries those figures.
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

## Dynamic budgets and founder-led planning

Keep target stated numbers static (`$1,000`/employee, `15.00%`, `50.00%`, `$250,000`/person) and
put the driver math in the SQL measure. That keeps every budget rate a clean, immutable version on
the target while `period.context[]` delivers the live underlying dollars and headcount on every
`targets read`.

### Pre-spend and pre-hire headroom check (advisory only)

When the founder asks *"can we afford this trip / vendor charge / hire?"*:

1. Read the active budget targets (`targets {action: "read", id, current_only: true}`). Verify that
   the evaluated period has `status: "in_progress"` (since `current_only: true` falls back to the
   latest ended period when no period is open today) and that every needed `period.context[]` entry
   has `ok: true` before computing spendable headroom; for headcount as of `today` when future-dated
   starts or exits exist inside the period, run `measures {action: "try", id, version, from:
   period.window.from, through: today}` first.
2. Compute the current earned envelope and remaining headroom directly from `stated` and
   `period.context[]`:
   - **Per-employee cap** (`travel_per_person_minor <= stated`, context `travel_minor` & `people`):
     `earned_envelope = stated * people`, `remaining_headroom = earned_envelope - travel_minor`.
   - **Percentage-of-driver cap** (`rd_token_pct <= stated`, context `rd_tokens_minor` &
     `net_new_revenue_minor`): `earned_envelope = net_new_revenue_minor * stated / 10^(2 + (decimals ?? 0))`,
     `remaining_headroom = earned_envelope - rd_tokens_minor`.
3. Show the rate, the underlying actuals, the earned envelope, and what the proposed spend does to
   headroom. State clearly that the check is **advisory only**: Economico never blocks recording a
   real expense or hire that the business actually makes.

### Hire and replace payroll-envelope check

When the founder wants to hire a replacement or overlap an outgoing role without stalling behind a
rigid headcount slot:

1. Read the active `Payroll % of ARR` budget (`targets {action: "read", id}`) and inspect `context`
   (`payroll_minor`, `arr_minor`, confirming both have `ok: true`) plus any recorded `valid_through`
   end dates in `people_records` and `payroll_costs` if needed.
2. Compute the monthly payroll envelope `arr_minor * (stated / 10^(2 + (decimals ?? 0))) / 12` and
   show **both**:
   - **Overlap month(s):** current monthly payroll + new hire's monthly loaded cost vs. the
     envelope (noting whether the overlap temporarily pushes payroll above the cap).
   - **Post-exit steady state:** after the departing team member exits (their `people_records`
     `valid_through` end date drops them from headcount-based measures automatically at period end,
     while `Payroll % of ARR` reflects posted `journal_lines` once final pay and severance postings
     conclude), current payroll − departing cost + new hire cost vs. the envelope.
3. State any assumptions about loaded cost (payroll taxes, benefits) and confirm before revising a
   target or recording an arrangement.

### What-if revenue scaling ("If we double revenue this year, what can we hire and spend?")

Because operating rates live cleanly on active targets (`at_most` `50%` of ARR on payroll,
`at_least` `$250,000` ARR per team member, `at_most` `15%` of Net New Revenue on R&D tokens,
`at_most` `$1,000` travel per employee), answer scaling questions directly from live books and the
agreed operating plan without building a parallel spreadsheet:

1. Read the active plan (`targets {action: "list", label_key: "plan", label_value: "<plan>"}` and
   `targets {action: "read", id, current_only: true}` on each target) to get the agreed rates and
   current baseline (`arr_minor`, `people`, `payroll_minor`, spend actuals from `context`).
2. Apply the founder's scenario driver (for example, `2x` current ARR and the implied Net New ARR
   `scenario_arr - current_arr`) to each active target's stated rate:
   - **Headcount capacity** from `ARR per team member`: `floor(scenario_arr / stated_arr_per_person)`
     total team → net new hires `total - current_people`.
   - **Payroll envelope** from `Payroll % of ARR`: `scenario_arr * stated_payroll_pct` annualized
     (divide by 12 for monthly payroll) → incremental payroll capacity above current run-rate.
   - **Variable spend pools** from per-employee and `%`-of-revenue caps: e.g., `travel_cap *
     projected_people` and `rd_token_pct * projected_net_new_revenue`.
3. Label every projected figure plainly as a **what-if projection from active target rates** (never
   posted to any ledger and never recorded as a target unless the founder asks to adopt it).

### What-if projected P&L from targets and dynamic budgets

When the founder asks *"what will our P&L look like at the end of 2027 if we hit our targets?"* (or
asks for a multi-quarter or multi-year operating forecast from their current plan):

1. **Pull the live baseline from the books:**
   - Run `reports run income_statement` over the last 3 closed months (month by month, scaling minor
     units once by the ledger's `decimals`) to establish trailing revenue, COGS, payroll, R&D AI /
     tokens, travel, and fixed opex run-rates.
   - Run `reports run saas_metrics` and `reports run default_alive` for the latest closed month to get
     current MRR, ARR, cash balance, net burn, and any `excluded_token_units`.
   - Query active team memberships as of the baseline cutoff date (`people_records` with
     `kind = 'membership'`, `current = 1`, `valid_from < valid_through`, and the validity window
     covering the cutoff date — never `current = 1` alone, which includes expired and future-dated
     memberships) and active recurring commitments (such as vendor subscriptions in `contracts` or
     trailing recurring opex lines) via `report_sql` so baseline headcount and fixed commitments
     match the books.
2. **Read every active target and dynamic budget in the operating plan:**
   - Call `targets {action: "list"}` (or filter by `label_key: "plan", label_value: "<plan>"`).
   - For each target, call `targets {action: "get", id}` and `targets {action: "read", id}` (to read
     its stated values and live `period.context[]`), and `measures {action: "get", id, version}` for
     the pinned measure's exact SQL denominator, currency, and account-code scope.
3. **Project forward period by period** (for example quarterly or annually through the founder's
   target horizon such as end of 2027) under **"Hit targets" (100%)** — and optional sensitivity
   cases when useful (for example **80% / 100% / 150%** of revenue target):
   - **Revenue** follows the active ARR, recognized-revenue, or won-bookings targets (state
     explicitly how ARR or bookings translate into period recognized revenue when only ARR or
     bookings targets exist).
   - **Dynamic budgets** scale automatically from each target's stated rate and driver:
     - **R&D AI / tokens** at its stated `%` of Net New Revenue (or recognized revenue if that is
       the measure's denominator), e.g. `<= 15.00%` of Net New Revenue.
     - **Payroll / People** at its stated `%` of ARR (e.g. `<= 50.00%` of ARR, divided by 4 per
       quarter or 12 per month), cross-checked against projected headcount from `ARR per team member`
       (`floor(projected_arr / stated_arr_per_person)`).
     - **Per-employee variable spend** (such as travel at `<= $1,000`/employee or per-vendor software
       caps) scales with projected headcount.
     - **COGS** scales at its active COGS-`%`-of-revenue target if recorded, or at the trailing
       3-month COGS `%` of recognized revenue stated as an explicit assumption.
   - **Fixed baseline opex lines** (such as rent, insurance, legal, accounting, and non-rate
     software) carry forward at their trailing 3-month average unless the founder names a step
     change.
4. **Present a clear projected P&L** — **Revenue**, **COGS**, **Gross Margin**, **Payroll /
   People**, **R&D AI / Tokens**, **Travel & Other Opex**, **Net Burn / Operating Profit**, and
   **Ending Cash / Runway (accrual-based proxy)** (starting cash from `default_alive` minus
   cumulative projected accrual net burn, plus any named financing inflows — labelled explicitly as
   an accrual proxy assuming cash collections and payments track accrual burn, ignoring
   working-capital timing and capex unless timed cash flows are modeled separately).
   - **State every assumption explicitly** right beneath the table (trailing baseline months,
     ARR-to-recognized-revenue assumption, fixed opex carry-forward, whether dynamic caps are
     modeled at 100% utilization or current actual utilization, hiring ramp timing, and any excluded
     foreign-currency or token holdings).
   - Label the projection prominently as **Illustrative projection · unposted**. Never post
     hypothetical numbers to the ledger, never invent missing facts into the books, and never record
     projected periods as active targets unless the founder asks to adopt them.

### Board-approved operating rules

Keep board governance and founder operating adjustments in one record instead of diverging
spreadsheets:

1. Agree on the operating rules once (for example, `<= 15.00%` R&D tokens, `<= $1,000`/employee
   travel, `<= 50.00%` payroll of ARR), confirm them with the founder, and activate the targets
   with a clear board reason (`"Adopted Q4 board operating rules"`). Note each target's adopted
   version number (check `targets {action: "versions", id}`).
2. When operating conditions change, revise the target rate via `targets.define` with `id`,
   `expectedVersion` and a clear `reason` — with zero SQL edits on the measure — carrying past
   periods unchanged so historical periods remain comparable.
3. For board reporting, read each target at **both** its adopted board version (`version: <board>`)
   and its latest operating version so the board sees actuals against the original board rules and
   against the current operating revision side by side.

### Exporting a formal board budget spreadsheet on demand

Founders do not have to live inside a static budget spreadsheet all year. Whenever an investor,
board member, or lender asks for a formal budget file, generate it on demand from live books plus
the active targets and dynamic budgets:

1. **Gather live actuals and plan targets:**
   - Pull closed-period actuals (`reports run income_statement` by month, `reports run saas_metrics`,
     and `reports run default_alive`).
   - List the operating plan (`targets {action: "list", label_key: "plan", label_value: "<plan>"}`)
     and confirm each target's original board-approved version via `targets {action: "versions", id}`.
2. **Read both the board baseline and the current operating plan:**
   - For each target, call `targets {action: "read", id, version: <board>}` for the original
     board-approved rules and `targets {action: "read", id}` for the current operating revision,
     capturing stated values, actuals, gaps, and supporting `period.context[]` dollars and drivers.
3. **Assemble the four-view board table** across closed and upcoming periods:
   - **Actual:** closed-period figures directly from the books (never mixed with open-pipeline
     estimates or projections).
   - **Board Baseline (v1 / adopted version):** original board-approved target numbers and implied
     dollar envelopes.
   - **Current Dynamic Budget:** latest active operating rates and earned dollar envelopes evaluated
     at actual headcount, ARR, or Net New Revenue.
   - **Projected:** future periods projected from the active targets and dynamic rates using the
     what-if projected P&L workflow above, clearly labelled **Illustrative projection · unposted**.
   - Keep currencies and units separate, and place any open-pipeline `estimate` in its own column.
4. **Write the spreadsheet file on demand:**
   - Export the structured tables as **CSV**, **XLSX**, or a **Google Sheet** using your host's
     file-writing or spreadsheet tools (Economico serves the verified reads and does not generate
     XLSX files or push to Google Sheets natively). Include an **Operating rules & assumptions**
     tab or section naming each target ID, board vs. current version, pinned measure, and projection
     assumption so every number traces back to the books.

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
actuals, fetch the `plan-vs-actual` report template (`catalog {action: "recipe", name: "plan-vs-actual"}`). It reads stated
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

**Native targets blocks vs SQL blocks:** to embed an operating plan or explicit targets directly in
an `economico.report/2` report without duplicating measure SQL, use a native `targets` block
(`title`, `plan` or `target_ids`, and optional `version` or `as_of`): it evaluates each target's
pinned measure for its current period with percent dynamic budgets, supporting context columns and
open-pipeline estimates, and freezes safely inside `economico.report-snapshot/2` shared views. When
a custom SQL `query` or `chart` block needs bespoke joins over target rows, each SQL block repeats
the measure's query and **must stay pinned to the exact measure ID and version each target uses**;
when a target is revised to a new measure version or uses a custom measure, update the block SQL and
its measure parameter and republish the report. `targets read` remains the authoritative read.

## Executable recipes

Fetch each tested recipe via `catalog {action: "recipe", name}`:

- `targets-sales-quarter-quotas` — custom `sales` pipeline,
  three salespeople, won, lost, open and unsized deals, a pipeline target and three per-person
  quotas read with open-pipeline estimates.
- `targets-operating-plan` — targets
  grouped under a `plan` label, recorded `proposed`, activated, revised mid-quarter with a reason,
  and read against both version 1 and the current version.
- `targets-hiring-plan` — custom `hiring` pipeline settled by
  `people.record_arrangement`, `people.set_membership` for headcount, and `to_date` hires plus
  `each_period` people-in-place targets.
- `targets-fundraise` — `financing` seed-round pipeline with a signed and
  funded SAFE winning its pursuit, open and lost investors, and a raised-amount target with its
  weighted open estimate.
- `targets-dynamic-budgets` — headcount-scaled travel budget,
  `percent`-unit R&D AI token budget, monthly net burn budget, burn multiple dynamic budget, and
  key-account customer expansion (`:party` customer holder) in one operating plan, returning live
  `context` columns, handling a zero-denominator month cleanly, and revising an operating rate while
  keeping the board-approved version readable.

See [business system](../../economico/references/pipelines.md) for defining pipelines and settling pursuits, and
[verify](../../economico/references/verify.md#a-question-no-report-answers) for `report_sql` rules.
