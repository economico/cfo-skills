# Investor update

A monthly update for the founder's investors, with every number read from the books. The books
supply the metrics; the founder supplies the story. Write it for the month that just closed
unless the founder names another.

## Read the month

| Section | Read | Use |
|---|---|---|
| Recurring revenue | `reports run saas_metrics` with `year` and `month`, for this month and the one before | MRR and ARR, and their change on last month |
| Revenue and spend | `reports run income_statement` with `from` the first of the month, `until` the first of the next | Revenue, cost of revenue, gross margin, operating expenses, net income or loss |
| Cash and runway | `reports run default_alive` with `period` `YYYY-MM` | Cash at month end, net burn, months of runway |
| Money owed to the company | `reports run aging` with `direction` `receivable` and `as_of` the month end | Anything overdue worth a line |

Scale every amount once by the ledger's `decimals`, as [verify](verify.md) says. Compare with
last month by reading last month the same way; never estimate a figure the reports did not
return. When the books for the month are incomplete (receipts not yet recorded, a Stripe payout
missing), say which, and offer to record them first: an update built on partial books understates
burn.

## Ask the founder for the story

In one round ([asking the founder](asking-the-founder.md)): the month's highlights, the lowlights,
and the asks (intros, hires, advice). Offer choices drawn from the books where they suggest one
(a new customer, a large contract, an overdue invoice), and leave room for their own words.

## Write it

Short, in the founder's voice, as Markdown they can paste into an email:

1. One line on the month.
2. Metrics: MRR and ARR with the change, revenue, net burn, cash, runway in months.
3. Highlights, lowlights, asks.

Save it as `investor-updates/<YYYY-MM>.md` in the founder's repository when there is one. It is a
draft: Economico does not send it, and nothing is recorded on the books.
