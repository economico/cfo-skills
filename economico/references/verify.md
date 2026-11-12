# Verify the books

After recording, prove the model with reads. Each figure you report should trace to rows in the
business model brief, and each row to a source. `reports {action: "list"}` gives every kind's
parameters and a runnable example; dates are `YYYY-MM-DD` or UTC instants. End a period with the
next period's first instant as a UTC timestamp (`until`, `as_of` or `period_end`
`2026-10-01T00:00:00Z` for September); `through` includes the instant it is given.

## The checks

| Question | Read | Expect |
|---|---|---|
| Is every contract live? | `subjects {action: "list", type: "contract"}` | Customer and vendor contracts `active`; none left `draft` unless the founder is still negotiating |
| Is the price book right? | `subjects {action: "list", type: "template", plan: "on_sale"}` | One template per purchasable plan in the brief |
| Does actual service match the agreement? | Returned phase statement documents and `ledgers` footprint/statement reads | Evidenced delivery/usage and recognition match the requested period; no inferred fulfillment |
| What did the business earn and spend? | `reports run income_statement` with `from` and `until` | Revenue per account and expenses per nature and function match recorded occurrences |
| Who owns the company? | `reports run cap_table`, and `capital_accounts` for an LLC, partnership or project | Every owner from the formation documents with their units; never empty for a company that has owners |
| Where is the money? | `reports run balance_sheet` with `as_of` | Cash per account matches the bank and card statements at that date, where history was recorded |
| Who owes what? | `reports run aging` with `direction` `receivable` or `payable` and `as_of` | Outstanding 1120 receivables and 2110 and 2170-2172 payables; paid vendor claims are zero. Owner/card debts use claim and native-balance reads below |
| What does each wallet hold? | `accounts {action: "overview", id}` for each wallet | `holdings` lists each token in its own unit and `decimals` and matches the wallet or explorer; a wallet's `balance` counts only its currency ledger |
| What is owed to owners or card issuers? | `reports run owner_balance` with `party_id` and `as_of` for an owner (2135 only), and `activity_claims` for the originating agreement, plus party footprint/native balance reads for 2190 card issuers | Identified 2135/2190 funding claims and reimbursements, per native currency; these are not included in aging |
| How did each month go? | `reports run income_statement_periods` with `through_month` | Revenue, expenses and net income for the trailing months; the current month marked as in progress |
| What does each unit earn and cost? | `reports run unit_economics` with `from` and `until` | Units each customer activity was paid on, revenue per unit, and its own posted cost; unattributed cost reported apart, never allocated |
| What is one contract's total? | `reports run contract_books` with `contract_id`, for a priced-rights contract | Its payments per account, and delivery, recognition and usage posted, from its own events |
| What is in the pipeline? | `reports run pipeline` | Open opportunities, weighted and unweighted, by direction and currency; never revenue or cash |
| What did discounts cost? | `reports run concessions` with `from_month` and `through_month` | Each offer's recognised discount on 4140 and the gross revenue of the contracts that took it, by purpose |
| Do the wallet and the books agree? | `accounts {action: "wallet_monitors", id: financialAccountId}`, and `wallet_movements` with the same `id` | `difference` is `"0"`; if not, run `accounts.check_wallet_monitor`, read again, and never journal the difference away. Movements with `status` `unclassified`, `conflict` or `needs_canonical_review` are listed for a decision, none classified on a guess |
| What will Economico charge? | `reports run platform_fees` with `period` (`YYYY-MM`) | A preview of the plan fee and the revenue fee; nothing accrues from it |
| What is open on one contract? | `reports run activity_claims` with `contract_id` | Original/outstanding per claim component, with its kind; `"0"` once settled or converted |
| What remains deferred or prepaid? | `reports run balance_sheet`, supported actual `recognition`/`amortization` detail | Billed value less evidenced consumption; payment alone never proves earning or expense |
| A question no report answers? | `report_sql {action: "query"}`, after `report_sql {action: "schema"}` | See "A question no report answers" below |
| Is the history intact? | `events {action: "head"}` and, over REST, `POST /v1/events/verify` | Verification ok. While derived contract values are still catching up, `ok` is `false` with empty `diffs` and a `pending` list: the core state matched. A non-empty `diffs` is a real difference |

Priced ownership vesting supports `activity_plan` and timers for its declared grant schedule;
service forecast and automatic recognition/backfill remain limited. Recurring metrics and
revenue summaries include supported priced services and actual independent settlements.
Do not use an empty legacy occurrence list or zero legacy metric as proof that nothing happened.
Retain returned statement IDs and compare claims and native actual postings against source
truth. For unsupported ownership families or reports, disclose the gap instead of manufacturing
records to populate a view. Existing historical books retain their historical read paths.

## Reading the numbers

- Amounts are minor-unit strings. Read the ledger's `decimals` with
  `ledgers {action: "get", ledger_id}` and scale by `10^decimals` exactly once. USD has 2
  decimals (`"4900"` is $49.00); JPY has 0 (`"4900"` is ¥4,900).
- Statements are cumulative unless windowed. `income_statement` without `from`/`until` is the
  whole book.
- Balances are native: never add a USD figure to a EUR figure or a usage count.
- A negative cash balance usually means payments were recorded without the funding that preceded
  them (the founder's investment, a SAFE, an opening balance). Say so rather than hiding it.
- If a figure surprises you, drill down: `ledgers {action: "balances", ledger_id}` for the
  account (add `as_of` for the balances as they stood at a date, with each sub-key such as the
  expense function kept separate), then `ledgers {action: "statement", ledger_id, account_code}` for the postings, then
  the occurrence and its statement document.

## Report back

Put the headline figures into the brief under "What the books say now", with the date they are
as of, and show the statements to the founder as in [show the founder](show-the-founder.md). Give the founder a link to the workspace (`https://<host>/e/<slug>/app`), where every
document, contract and report is readable.

## A question no report answers

When the founder asks something the report kinds do not cover ("spend by vendor against last
year", "which accounts moved most this quarter", "revenue by month for these three
customers"), answer it from the books with one read-only SQL query. Do not chain reads and add
the numbers up yourself, and do not export to a spreadsheet.

1. `report_sql {action: "schema"}` once: it lists the views, every table with its columns, and the
   conventions. Prefer the views (`journal_lines`, `accounts`, `contracts`, `claims`,
   `payments`, `events`, `commands`, `ingestions`, and `monthly_metrics` for recurring revenue,
   customers, burn, cash and runway by month, which are the built-in reports' own figures and
   leave out a wallet's tokens); a
   table is the books as stored and can change. If
   `schema` is refused, this connection may not query: use the report kinds.
2. `report_sql {action: "query", sql, params}` with one `SELECT`. Bind values with `?` and
   `params`; a number binds as a real, so write `CAST(? AS INTEGER)` for a `LIMIT`.

Rules that keep the answer right:

- Money is a column named `*_minor`, in minor units of the row's `currency`. Name your result
  column `*_minor` and select `currency` beside it; never add amounts across currencies.
- `signed_minor` is positive for a debit, so negate the sum for revenue, liabilities and equity.
- `journal_lines` holds every ledger. Before reading revenue, expenses or balances by account,
  join `ledgers` on `ledger_id` and keep `account_template = 'gaap_startup'`: an account code
  means what `accounts` says only there.
- Filter dates as `julianday(effective_at) >= julianday(?)`. `journal_lines` and `events` have
  `month` to group by; elsewhere use `strftime('%Y-%m', effective_at)`.
- To total by account name or category, total `journal_lines` by `account_code` first, then
  join `accounts`.
- `claims` has every invoice and bill with `billed_minor` and `open_minor`, as of now.
  `in_aging = 1` marks what the aging report counts.
- `payments` keeps a payment that was later reversed, with `reversed = 1`: leave those out of a
  total.
- An answer over 1,000 rows or 256 KiB is refused: aggregate in SQL, and select only the
  columns you need.

## An answer the founder wants to keep

When the founder will want the answer again (a monthly update, figures the team watches,
something to send), save it as a custom report instead of answering once: a page of figures,
charts, tables and text that Economico draws from a definition you write, run again for another
month or date.

1. `report_format {action: "guide"}` first: the format's reference, with two worked
   definitions. Do not write a definition from memory. If it is refused, this connection may not
   query, and cannot save a report.
2. `report_sql {action: "schema"}` for the columns, then `report_format {action: "check",
   definition}` until it passes, then `reports {action: "preview", definition}`. Read every
   block of the preview and show it to the founder.
3. With the founder's agreement, publish with `report_definitions.publish`. Then offer
   `report_definitions.pin`, which shows it to every member in the workspace, and a link to
   send, as in [investor update](investor-update.md).

**Opening a saved report again.** `reports {action: "definitions"}` lists the business's
reports; `reports {action: "run_definition", id, params}` runs one, with `params` naming its
parameters (`{month: "2026-10"}`), or with none for its defaults. Leave out `version` for the
latest. A value out of bounds runs the default instead, and that parameter's `note` in the
answer says so: read it before quoting a figure. To change a report, read it back with
`report_format {action: "source", id}`, edit that, check and preview it, and publish it with
the same `id`; `report_format {action: "versions", id}` lists what was published before.

**How current a run is.** An answer may be reused for up to an hour while the books are
unchanged, and `computed_at` says when it was worked out. Any write to the books gives a new
answer at once. Pass `fresh: true` only when the report reads the clock or the founder asks
for a new run.

**Which link to send.** `shared_views.create` takes `mode`:
- `frozen` (the default) captures the figures once, for the `params` you name, and never
  changes: for something sent and kept, such as a monthly update.
- `live` reruns the report against the books each time it is opened, and its reader may change
  the report's parameters within their bounds: for a page someone checks, such as a board
  member's dashboard. It shows today's books to anyone holding the link, so say so to the
  founder before creating one. Pass no `params` for a live link. A link keeps the version it
  was made with: after publishing a new version, make a new link and revoke the old one with
  `shared_views.revoke`. Built-in `reports` on any link are always live. No link expires
  unless you set `expiresAt`; offer one for a live link.

A report kind is the reference wherever one covers the figure: a period's revenue and expenses
are `income_statement`, and what a customer owes is `aging`, `activity_claims` or the `claims`
view's `open_minor` (the same figures, claim by claim), never a sum of journal lines you write. Use SQL to cut a figure a way no report does (by customer, by vendor, by month), and
check that your cut adds up to the report's total for the same window before you report it.
Say which read each figure came from.
