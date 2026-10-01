# The business model brief

Write this file as `business-model.md` at the root of the founder's repository (or wherever they
ask). Keep it to what a founder reads in five minutes. Every number has a source; every judgment
is labeled as one. Amounts in dollars, not minor units.

It is also the skill's memory between sessions: the last section, "For the next session", is
what a later session reads first so it resumes instead of asking again
([a later session](first-session.md#a-later-session)).

```markdown
# <Company>: how it makes and spends money

_As of <date>. Built by <agent> from <sources read>. Economico business: `<slug>` on
<host>._

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

## Ownership

| Holder | Instrument | Units | Ownership | Capital put in | Source |
|---|---|---|---|---|---|
| Ada Park | common stock | 8,000,000 | 100% | $800 at par | stock purchase agreement, certificate of incorporation |

<Legal form and jurisdiction, and any SAFE or note outstanding. "None found" is not an answer
for a company: say which document is missing and ask for it.>

## Run rate today

| | Monthly |
|---|---|
| Subscription revenue (MRR) | $… |
| Committed vendor spend | $… |
| Usage-driven revenue and cost | varies: <basis> |

_Run rate from recorded terms, not a forecast._

## Judgments and gaps

- <Each judgment that changes a number, who made it and why: "Confirmed by Ada, 2026-09-26:
  Umbrella churned in August" or "Taken on Ada's behalf, 2026-09-26: Figma is founder-paid,
  paid on the personal Mastercard".>
- <Each source you could not read, and what it would add.>

## What was recorded

<Counts: parties, templates, contracts, occurrences, documents. What was left for the founder.>

## What the books say now

<Headline figures from the verify step, with their as-of date and the report they came from.>

## For the next session

**Standing answers** (<date>): business `<slug>` on <host>; may read <this repository, Stripe,
the mailbox>; history since <date>; records <after review | without asking again>.

**Rules the founder set**

- <A classification to apply the same way next time: "The Mastercard ending 1881 is Ada's
  personal card: receipts on it are founder-paid (owed to Ada)"; "Anthropic is cost of revenue".>

**Open questions**

- <A decision taken on the founder's behalf and still waiting for them, or a gap only they can
  fill, each with the option you recommend and why: "Recommended: record the Visa payoff when
  the statement arrives, since no payment is in the mailbox yet".>
```

Take the run-rate rows from `model_preview` (below), not from your head. If there is nothing to
put in a section, say "none found" and why.

## Preview the model before recording it

Once the tables are written, run `reports {action: "run", kind: "model_preview", model}` with
them as data: `plans` (key, name, price in minor units, cadence `month`, `year`, `one_time` or
`usage`), `customers` (name, plan key, since, price only when it differs from the list, status
`ended` for a churned one) and `vendors` (name, what it is for, amount and cadence, the account
and expense function from [accounts](accounts.md), and `drivers`: the product actions that make
it cost money, from [the codebase](explore-codebase.md)). It writes nothing. It refuses a
customer on a plan the price book does not have, an amount that is not minor units, or an account
that is not an expense account: fix the brief, not the call. Its `run_rate` is the "Run rate
today" table, and in hosts that render Economico's app the result draws the proposal inline, the
view the founder reviews ([show the founder](show-the-founder.md)).

## What "For the next session" holds, and what it never holds

It holds what the ledger cannot say: which business these books are, the founder's standing
answers, the rules they set, and what is still waiting for them. It never holds a secret (the
CLI keeps credentials in the gitignored `.economico/config.json`) or a copy of what the ledger
already knows: what was recorded is read back by its `externalId` and `sourceFactId`, never
from a list here. Update the section at the end of every session: move an answered question into
"Judgments and gaps" as confirmed, add the rules the founder set, and drop what no longer holds.
