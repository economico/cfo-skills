# Choosing accounts

Every business shares one code-owned chart. An effect pattern fixes the account pair; you choose
only the revenue or expense leaf and, for expenses, the function. There is no chart-listing read:
this page is the working subset. Codes remembered from older Economico material (5300 hosting,
5400 inference, 6xxx operating expenses) have changed meaning or no longer exist; use these.

## Revenue

| Code | Account | Use for | Category that selects it |
|---|---|---|---|
| `4110` | Subscription revenue | Recurring plan fees, seats | `subscription` |
| `4120` | Usage-based revenue | Metered usage, overage, credits consumed | `usage` |
| `4100` | Product sales revenue | One-off product sales | `services` (the default) |
| `4200` | Service revenue | Implementation, onboarding, setup fees | set `revenueAccount` |
| `4210` | Consulting revenue | Hourly, retainer and fixed-fee consulting | `consulting` |
| `4130` | Licensing revenue | Licences | set `revenueAccount` |
| `4220` | Platform and marketplace revenue | Take rate on a marketplace | set `revenueAccount` |
| `4500` | Grant and award income | Grants | `grant` |

Deferred revenue is `2150`, receivables `1120`, unbilled usage `1125`; the patterns post those.

## Expenses: nature plus function

An expense account says what was bought; the `expenseFunction` says what it was for:
`cost_of_revenue`, `research_and_development`, `sales_and_marketing`,
`general_and_administrative`, `other`. Gross margin reads cost of revenue, so this choice matters
more than the account.

| Typical vendor | Code | Account | Usual function |
|---|---|---|---|
| AWS, GCP, Azure, Cloudflare, Render, Fly, Vercel, Neon, Supabase (production) | `5210` | Hosting and cloud infrastructure | `cost_of_revenue` |
| The same, for staging, CI or development | `5210` | Hosting and cloud infrastructure | `research_and_development` |
| OpenAI, Anthropic, model APIs called by the product | `5220` | AI and model inference | `cost_of_revenue` |
| Stripe, Paddle, PayPal fees | `5230` | Payment processing fees | `cost_of_revenue` |
| Twilio, Resend, Postmark, Algolia, data vendors used by the product | `5240` | Third-party APIs and data | `cost_of_revenue` |
| Notion, Google Workspace, Slack, Linear, 1Password | `5310` | SaaS and productivity tools | `general_and_administrative` |
| GitHub, Sentry, Cursor, Claude or ChatGPT subscriptions for the team | `5320` | AI and developer tools | `research_and_development` |
| Contractors, employer-of-record | `5140` | Contractors and EOR services | by the work: `research_and_development` for engineering |
| Customer support tools and outsourced support | `5150` | Customer support services | `cost_of_revenue` |
| Mailroom, virtual office and registered-address service | `5510` | Rent and facilities | `general_and_administrative` |
| Video, design and content-creation software | `5310` | SaaS and productivity tools | `sales_and_marketing` for marketing content; otherwise by purpose |
| Outsourced video production and content creation | `5430` | Content and creative services | `sales_and_marketing` |
| Ads, sponsorships | `5410` | Advertising and paid media | `sales_and_marketing` |
| Conferences | `5420` | Events and conferences | `sales_and_marketing` |
| Lawyers, incorporation | `5620` | Legal and corporate fees | `general_and_administrative` |
| Accountants, tax preparers | `5630` | Accounting and tax services | `general_and_administrative` |
| Insurance | `5530` | Insurance | `general_and_administrative` |
| Domains, registrars | `5310` | SaaS and productivity tools | `general_and_administrative` |
| Travel | `5710` | Travel | by purpose |
| Bank fees | `5910` | Bank charges and fees | `general_and_administrative` |
| Anything else | `5990` | Other operating expense | ask |

Set both in the effect: `"expenseAccount": 5220, "expenseFunction": "cost_of_revenue"`. Without
them the effect's `category` picks a default (`subscription` → 5310 general and administrative,
`usage` → 5220 cost of revenue, `consulting` → 5140 cost of revenue, `services` → 5990 cost of
revenue), which is rarely what a specific vendor needs.

Classify by environment and purpose, not by vendor name: the same cloud bill can serve production
(cost of revenue) and development (research and development); split it into two activities when
the invoice separates them. When a choice changes gross margin materially and the evidence is
ambiguous, ask the founder and explain the effect.

## Balance-sheet accounts the recipes touch

`1110` cash (per registered bank account), `2190` credit card payable (per registered card),
`2110` payables (per vendor), `2135` due to related parties (per founder), `1150` prepaid
expenses, `1170` recoverable input tax, `2160` output tax, `2240` SAFE and other financing,
`3100`/`3110` share capital and premium.


## Company cards and received issuer statements

A company card is a vendor contract with its issuer. Register its account as `kind: "card"`
(on 2190), with `providerPartyId` and `providerContractId` naming the issuer and its active
agreement. The issuer agreement can exist without processing-fee schedules.

Set `paymentTerms` through `contracts.create` or `contracts.amend`: `payingAccountId` is the
registered account used by default; `statementClosingDay` and `paymentDueDay` are
calendar days 1–31; set `autopay: true` only when that instruction is evidenced. Supply the full
replacement block on amendment. These are contract terms, never fields on the card account.

When purchases have already been booked per charge (Dr expense / Cr 2190), receive the issuer's
PDF with `documents.receive`, then call `card_statements.record` with `financialAccountId`,
`contractId`, `closingDate`, `statementBalance`, `minimumDue`, `dueDate`, `documentId` and the
issuer's unique `sourceFactId`. Amounts are minor-unit strings in the card currency. This records
a bill over existing debt, without repeating purchase expenses. Optional `interest` and `fees`
are only charges not already booked: interest posts to 7100, issuer fees to 5910, both against
2190. Check the result's `difference` (statement less booked closing balance, including these
items). A mismatch is accepted; research missing charges or timing rather than inventing an
expense to make it match.

Record a completed repayment with `card_statements.pay`: `statementId`, actual
`financialAccountId`, `amount`, `effectiveAt`, fresh `sourceFactId` and payment `evidence`. It
posts Dr 2190 / Cr cash and no expense. Partial repayment is supported. The account overview and
card page show the latest statement only after one is recorded; aging counts the latest bill
per card so overlapping statements do not double-count debt. An open bill's **Pay from** uses
the issuer contract; a settled bill uses the actual payment account. This records autopay
instructions and payments; it does not send money.

To correct statement facts or duplicated interest/fees, use `card_statements.void` with
`statementId`, `reason`, `effectiveAt`, a fresh `sourceFactId`, and correction `evidence`,
then record the correct statement with `card_statements.record` and a new `sourceFactId`.
When replacing a statement, date its void at the end of its original closing date:
`<closingDate>T23:59:59.999Z`. Otherwise the old interest and fees remain in the historical
closing balance and a correct replacement can show a negative `difference` equal to those
removed charges. A later-dated void reduces spending in the void's calendar month; if that is
a later month, the original closing month retains the old charges. Do not treat that
date-driven difference alone as a missing purchase.
An incorrectly future-dated statement can also be voided today; a void may precede the
mistyped closing date. Such an early void reduces spending and debt in its own period while
the original future-dated charges retain their recorded dates.
The same closing date can be reused after voiding. This reverses only the statement's own
interest and fees, preserves purchases and history, and removes its bill from aging and the
card page; the earlier unvoided statement becomes latest. If the statement has payments,
reverse them first with `card_statements.reverse_payment` using `paymentId`, `effectiveAt`,
a fresh `sourceFactId`, and correction `evidence`. It reverses the whole payment (Dr cash /
Cr 2190) and restores the unpaid amount. Record a replacement payment against the correct
statement or cash account. Corrections work across incorporation through ledger carrying.
Keep the replacement payment's true date even if its reversal is later-dated. The payment
guard counts back later reversals only for payments already effective by that true date, and
the replacement consumes the restored debt so it cannot be used twice.
Both commands return `reversalId`; retry with the same idempotency key. They do not move funds.

For monthly-only bookkeeping on a card in the business's functional currency, first record
its evidenced purchases with `journal.post`, one balanced entry per category within the
statement period (for example, on the closing date). Receive the PDF first, attach it as the
command's `source_document_id`, and set the command envelope's `effective_at` to that date.
Use the functional `currency` and minor-unit strings. Debit the expense leaf with
`subKey: {kind: 6, expenseFunction: "general_and_administrative"}` (choose the actual function),
and credit 2190 with `subKey: {kind: 4, financialAccountId: "the-registered-card-id"}`.
For example, 7,000 of software (5310) and 3,000 of hosting (5210, `cost_of_revenue`) leave
10,000 on the card before the 10,000 statement is recorded; paying it clears 2190 against cash.
This journal path works on fresh books. Its currency must match the business's functional
currency; do not invent an exchange amount to fit it.

Omit purchases already booked individually. If supplier bills already record the expenses,
use `payments.record` funded by the registered card to move their 2110 payable onto the card's
2190 before recording the issuer statement; do not repeat those expenses in a journal. Direct
`card_expense` contract effects are available only on existing legacy agreements, not through
fresh contract authoring. A journal backdated across incorporation keeps its card debt and
original evidence, but the card spending figure does not attribute it back to its original month.

## Starting with an existing account balance

When registering a company bank account, card or wallet that already has a balance, supply
`openingBalance` as an exact signed minor-unit string and `openingDate` as `YYYY-MM-DD`:

```json
{"financialAccountId":"operating-bank","name":"Operating bank","currency":"USD","kind":"checking","openingBalance":"125000","openingDate":"2026-11-01"}
```

This starts the books with $1,250 in the bank. Use the statement's balance at the start of the
opening date: it stands for everything before that date. Do not also record earlier transactions;
payments and journals before that date are refused. Transactions on the opening date are accepted.
For a card, a positive opening balance is the amount owed (credit `2190`, debit equity); negative
means a credit owed by the issuer. For a bank or wallet, positive is money held and negative is an
overdraft. Zero is valid and still establishes the start date.

The other side defaults to `3900` Opening balance equity. Use `openingEquityAccountCode` if the
business chooses another postable equity account. This records the starting position, not income
or a fresh capital contribution. Do not invent the balance or date; obtain them from the business
or its statement. Personal accounts registered with `ownerPartyId` do not take opening balances:
the books track only what the business owes the owner.

For an account registered earlier without an opening, `accounts.set_opening_balance` accepts
`financialAccountId` and the same opening fields, once, while it has no postings. For an account
already in use, reconcile the evidenced difference with `journal.post` instead. An unused legacy
card recorded on cash must be archived and registered correctly on `2190` before adding its opening.
