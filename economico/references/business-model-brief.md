# The business model brief

Write this file as `business-model.md` at the root of the founder's repository (or wherever they
ask). Keep it to what a founder reads in five minutes. Every number has a source; every judgment
is labeled as one. Amounts in dollars, not minor units.

```markdown
# <Company>: how it makes and spends money

_As of <date>. Built by <agent> from <sources read>. Economico business: <slug>._

## What we sell

<Two or three sentences: the product, who pays, how it is delivered.>

## Price book

| Plan | Price | Billing | Includes | Overage | Source | Template |
|---|---|---|---|---|---|---|
| Pro monthly | $49 / month | monthly in advance | 20,000 generations | $1 per 1,000 | code `src/billing/plans.ts`, Stripe `price_…` | Pro monthly plan |

## Customers

| Customer | Plan | Since | Price (if not list) | Source |
|---|---|---|---|---|
| Acme Analytics | Pro monthly | 2026-07-01 | | Stripe `sub_…` |

## Costs

| Vendor | For | Amount | Cadence | Paid by | Account · function | Source |
|---|---|---|---|---|---|---|
| Render | production hosting | ~$680 | monthly, usage | bank ACH | 5210 · cost of revenue | email INV-2026-06-0042 |

## Run rate today

| | Monthly |
|---|---|
| Subscription revenue (MRR) | $… |
| Committed vendor spend | $… |
| Usage-driven revenue and cost | varies: <basis> |

_Run rate from recorded terms, not a forecast._

## Judgments and gaps

- <Each assumption that changes a number, and what would confirm it.>
- <Each source you could not read, and what it would add.>

## What was recorded

<Counts: parties, templates, contracts, occurrences, documents. What was left for the founder.>

## What the books say now

<Headline figures from the verify step, with their as-of date and the report they came from.>
```

Compute the run-rate rows in code from the tables above, not in your head. If there is nothing
to put in a section, say "none found" and why.
