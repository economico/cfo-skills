# Verify the books

After recording, prove the model with reads. Each figure you report should trace to rows in the
business model brief, and each row to a source. `reports {action: "list"}` gives every kind's
parameters and a runnable example; dates are `YYYY-MM-DD` or UTC instants, `until` is exclusive,
`through` and `period_end` are inclusive.

## The checks

| Question | Read | Expect |
|---|---|---|
| Is every contract live? | `subjects {action: "list", type: "contract"}` | Customer and vendor contracts `active`; none left `draft` unless the founder is still negotiating |
| Is the price book right? | `subjects {action: "list", type: "template", plan: "on_sale"}` | One template per purchasable plan in the brief |
| Does actual service match the agreement? | Returned phase statement documents and `ledgers` footprint/statement reads | Evidenced delivery/usage and recognition match the requested period; no inferred fulfillment |
| What did the business earn and spend? | `reports run income_statement` with `from` and `until` | Revenue per account and expenses per nature and function match recorded occurrences |
| Who owns the company? | `reports run cap_table`, and `capital_accounts` for an LLC, partnership or project | Every owner from the formation documents with their units; never empty for a company that has owners |
| Where is the money? | `reports run balance_sheet` with `as_of` | Cash per account matches the bank and card statements at that date, where history was recorded |
| Who owes what? | `reports run aging` with `direction` `receivable` or `payable` and `as_of` | Outstanding 1120 receivables and 2110 payables; paid vendor claims are zero. Owner/card debts use claim and native-balance reads below |
| What is owed to owners or card issuers? | `activity_claims` for the originating agreement, plus party footprint/native balance reads | Identified 2135/2190 funding claims and reimbursements; these are not included in aging |
| What is open on one contract? | `reports run activity_claims` with `contract_id` | Original/outstanding per claim component, with its kind; `"0"` once settled or converted |
| What remains deferred or prepaid? | `reports run balance_sheet`, supported actual `recognition`/`amortization` detail | Billed value less evidenced consumption; payment alone never proves earning or expense |
| Is the history intact? | `events {action: "head"}` and, over REST, `POST /v1/events/verify` | Verification ok |

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
