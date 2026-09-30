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
| What is the recurring revenue? | `reports run saas_metrics` with `year` and `month` | MRR equals the sum of fixed monthly fees in the brief; usage adds nothing |
| What did the business earn and spend? | `reports run income_statement` with `from` and `until` | Revenue per account and expenses per nature and function match recorded occurrences |
| Where is the money? | `reports run balance_sheet` with `as_of` | Cash per account matches the bank and card statements at that date, where history was recorded |
| Who owes what? | `reports run aging` with `direction` `receivable` or `payable` and `as_of` | Open invoices and bills only; card and paid-at-purchase receipts never appear |
| What is open on one contract? | `reports run activity_claims` with `contract_id` | `outstanding` per invoice; `"0"` once paid |
| What happens next? | `reports run activity_plan` with `contract_id`, `from`, `through` | The next scheduled periods; usage shows as contingent until measured |
| Billing and cash for a period | `reports run revenue_summary` with `period_start`, `period_end` | Invoiced, paid, outstanding and recognized per currency |
| Is the history intact? | `events {action: "head"}` and, over REST, `POST /v1/events/verify` | Verification ok |

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
