---
name: economico
description: "Model a founder's business in Economico: connect the ledger, read the product's code, Stripe and email to learn what it sells, to whom and what it costs to run, record plans as rights templates, customer and vendor agreements as contracts, and evidence-backed economics and independent payments, then prove the books with reports. Use when someone wants to set up or connect Economico or their books, model their business, pick up the books where an earlier session left off, turn Stripe or pricing into contracts, set up vendors, record receipts or invoices, or asks how Economico works."
---

# Economico

Economico is a double-entry ledger that the founder's own agent operates. Every business is
one append-only, hash-chained history. Every write is a **command** from one catalog, run through
one door (`commands` over MCP, `POST /v1/commands` over REST, `economico commands execute` in the
CLI). Contracts describe priced rights and independently timed economic consequences.
Billing creates commercial claims; payments settle identified claims using a registered account.
Changing who pays does not change the underlying right. Lifecycle commands, invoices and
payments are not additional activities.

You gather source facts with the founder’s authorized access and turn them into reviewed commands.

## Ground rules

1. **Never touch a real business to experiment.** Rehearse on a disposable business. Before the
   first write, say which business slug you are writing to and why.
2. **Describe before you execute.** `catalog {action: "describe", name}` is the schema of record.
   The recipes in this skill are known-good shapes, not a substitute for the live schema.
3. **Propose, then record.** Discovery produces a written business model the founder reviews.
   Nothing that posts money is recorded until they agree, unless they told you to proceed.
   Ask for decisions, never for facts you can look up: a round of choices, your recommendation
   first, through the host's question tool when it has one
   ([asking the founder](references/asking-the-founder.md)).
4. **Money is a string of minor units.** When reporting, scale exactly once by `10^decimals`,
   using the ledger's `decimals` from `ledgers {action: "get", ledger_id}`. For USD (2 decimals),
   `"4900"` is $49.00; for JPY (0 decimals), `"4900"` is ¥4,900. Dates are `YYYY-MM-DD`;
   instants are UTC with `Z`.
5. **Preserve supplied IDs exactly.** Copy source IDs into `sourceFactId` and agreement IDs into
   document `externalId`; never reorder or suffix them. Create IDs only when none were supplied.
   Store evidence with `documents.receive`; use stable `idempotency_key`s.
6. **Never invent a number.** Prices, dates and amounts come from code, Stripe, a signed order form
   or a receipt. If none says it, ask, or record the gap in the business model.
7. **Refusals are information.** A refusal names the rule it applied. Read `code` and `message`,
   fix the input, and do not work around a refusal by changing the economics.
8. **Report what the books say, not what you did.** Show the founder the model as it takes shape
   and end with the reports that prove it, as views ([show the founder](references/show-the-founder.md)).

## The one-session run

When the founder says "set up Economico and model my business", run these phases in order.
Read each reference when you reach its phase. If the repository
already has `business-model.md`, read it first: it holds the business, the standing answers and
the open questions, so resume from it ([a later session](references/first-session.md#a-later-session)).

| # | Phase | Output | Read |
|---|---|---|---|
| 1 | Connect and choose the business | A verified connection and a named target business | [connect](references/connect.md) |
| 2 | Understand the machine | The mental model you will map the business onto | [how Economico works](references/how-economico-works.md) |
| 3 | Discover | Notes on product, pricing, customers, costs, entity | [codebase](references/explore-codebase.md), [Stripe](references/explore-stripe.md), [email](references/explore-email.md) |
| 4 | Propose | `business-model.md` in the founder's repo, always previewed with `model_preview`, then reviewed by them, its price book and costs shown in the review message | [first session](references/first-session.md), [brief template](references/business-model-brief.md) |
| 5 | Record | The legal form and the owners (the cap table, even for a sole owner), parties, accounts, templates, contracts, activity, evidence; the contracts shown as one table | [products](references/modeling-products.md), [customers](references/customer-contracts.md), [vendors](references/vendor-contracts.md), [evidence](references/evidence.md), [accounts](references/accounts.md), [payments](references/payments.md), [company](references/company-setup.md) |
| 6 | Prove | A final message that opens with the price book, the contracts, the income statement, the balance sheet and the cap table as views, then a short summary | [verify](references/verify.md) |

[first-session.md](references/first-session.md) is the playbook that strings these together,
including founder review and missing access. At the end of
phases 4, 5 and 6, show the founder their business as it takes shape:
[show the founder](references/show-the-founder.md) says which view, from which read, inline where
the host renders Economico's app and as text everywhere else.

## When the request is narrower

| The founder asks to… | Go to |
|---|---|
| connect an agent, log in, pick or create a business, run headless | [connect](references/connect.md) |
| understand how Economico works, or why something was refused | [how Economico works](references/how-economico-works.md) |
| turn pricing (code, Stripe or a pricing page) into plans | [modeling products](references/modeling-products.md) |
| add a customer, record a signed deal, bill or collect | [customer contracts](references/customer-contracts.md) |
| set up vendors, process receipts or invoices from email | [explore email](references/explore-email.md), [vendor contracts](references/vendor-contracts.md) |
| record payments, reimbursements, transfers or payment corrections | [payments](references/payments.md) |
| choose an account for a cost or a revenue line | [accounts](references/accounts.md) |
| set the company profile, bank accounts, cards, legal identity, founders' shares or the cap table | [company setup](references/company-setup.md) |
| check the books after a change | [verify](references/verify.md) |
| see a statement, the contracts or who owes what | [show the founder](references/show-the-founder.md) |
| write an investor update | [investor update](references/investor-update.md) |

## What is not available today

Say so plainly instead of approximating: there is no standing Stripe sync, no email listener and
no bank feed (you gather facts with your own access and record them); no planning scenarios or
forecast overlays; no stock options; no tax returns or filings. Economico never moves money, sends
an invoice or signs anything; it records what happened elsewhere.
