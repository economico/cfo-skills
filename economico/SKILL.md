---
name: economico
description: "Model a business in Economico: discover pricing and costs from code, Stripe and email; reuse rights and plans; record evidenced contracts, economics and payments; prove the books. Use for setup, AgentID sign-in, agreements, resuming the books, vendor invoices, People or provider payroll."
---

# Economico

Economico keeps one append-only, hash-chained double-entry history per business.
Contracts price rights with independent billing, recognition and payment. Billing creates claims;
registered-account payments settle them. Who pays never changes the right; lifecycle events
are not extra activities. Use only the founder's authorized source access.

## Ground rules

1. **Never touch a real business to experiment.** Rehearse on a disposable business. Before the
   first write, say which business slug you are writing to and why.
2. **Describe before you execute.** `catalog {action: "describe", name}` is the schema of record.
   Recipes give known-good shapes; the live schema wins.
3. **Propose, then record.** Discovery produces a written business model the founder reviews.
   Nothing that posts money is recorded until they agree, unless they told you to proceed.
   For setup, ask scope before discovery; the next question round is after the brief is written,
   `model_preview` passes and the model is shown. Keep missing facts as gaps. Ask only decisions,
   recommendation and reason first, through the host's question tool. Read
   [asking the founder](references/asking-the-founder.md) immediately before each round.
4. **Money is a string of minor units.** When reporting, scale exactly once by `10^decimals`,
   using the ledger's `decimals` from `ledgers {action: "get", ledger_id}`: `"4900"` is $49.00
   in USD (2 decimals), ¥4,900 in JPY (0). Dates are `YYYY-MM-DD`; instants are UTC with `Z`.
   Use the founder's stated date; [verify in the business's zone](references/verify.md).
5. **Preserve supplied IDs exactly.** Copy source IDs into `sourceFactId` and agreement IDs into
   document `externalId`; never reorder or suffix them. Create IDs only when none were supplied.
   Store evidence with `documents.receive`; use stable `idempotency_key`s.
6. **Never invent a number.** Prices, dates and amounts come from code, Stripe, a signed order form
   or a receipt. Missing facts stay gaps in the brief; request their evidence in the final message.
7. **Refusals are information.** A refusal names the rule it applied. Read `code` and `message`
   and fix the input; never change the economics to get around it.
8. **Report what the books say.** After verification, reread [show the founder](references/show-the-founder.md)
   right before the final response. Repeat the tables and decisions here; a brief link does not count.
9. **Model in this session**, without background agents or wakeups. Explicitly user-authorized
   agent-owned [scheduled capture](references/conversation-intake.md) may continue after chat.

## The one-session run

For setup, read [first session](references/first-session.md) before discovery or questions.
Resolve its unanswered scope choices first. If `business-model.md` already exists, read its
standing answers and gaps, then resume [a later session](references/first-session.md#a-later-session).

| # | Phase | Output | Read |
|---|---|---|---|
| 1 | Connect and settle scope | A verified target; scope choices resolved before source reads | [connect](references/connect.md), [scope](references/first-session.md#0-agree-the-scope-one-round-of-questions), [questions](references/asking-the-founder.md) |
| 2 | Understand the machine | The model you map the business onto | [how Economico works](references/how-economico-works.md) |
| 3 | Discover | Notes on product, pricing, customers, costs, entity | [codebase](references/explore-codebase.md), [Stripe](references/explore-stripe.md), [email](references/explore-email.md) |
| 4 | Propose | `business-model.md`, passing `model_preview`, then founder review of price book and costs | [first session](references/first-session.md), [brief template](references/business-model-brief.md) |
| 5 | Record | Legal form and owners (a cap table even for a sole owner), parties, accounts, templates, contracts, activity, evidence; contracts as one table | [products](references/modeling-products.md), [customers](references/customer-contracts.md), [vendors](references/vendor-contracts.md), [evidence](references/evidence.md), [accounts](references/accounts.md), [payments](references/payments.md), [company](references/company-setup.md) |
| 6 | Prove | A final message opening with views of the price book, contracts, income statement, balance sheet and cap table, then a summary | [verify](references/verify.md) |

Show [the founder views](references/show-the-founder.md) in phases 4–6: inline if supported, otherwise as text.

## When the request is narrower

| The founder asks to… | Go to |
|---|---|
| connect an agent, log in, pick or create a business, run headless | [connect](references/connect.md) |
| how Economico works, or why something was refused | [how Economico works](references/how-economico-works.md) |
| business model patterns, reuse rights, combine offerings or bespoke contracts | [business model patterns](references/business-model-patterns.md) |
| turn pricing (code, Stripe, pricing page) into plans, or publish | [modeling products](references/modeling-products.md) |
| run sales and delivery, track deals, build views | [business system](references/business-system.md) |
| measures, targets, quotas or plans | [targets](references/targets.md) |
| add a customer, record a deal, bill, email invoices or collect | [customer contracts](references/customer-contracts.md) |
| set up vendors, process receipts or invoices from email | [explore email](references/explore-email.md), [vendor contracts](references/vendor-contracts.md) |
| People, provider payroll or named service seats | [People and payroll](references/people-and-payroll.md) |
| review subscriptions, or record tool and seat use | [vendor usage](references/vendor-usage.md) |
| record payments, reimbursements, transfers or corrections | [payments](references/payments.md) |
| choose the account for a cost or revenue line | [accounts](references/accounts.md) |
| company profile, accounts, cards, wallets, legal identity or cap table | [company setup](references/company-setup.md) |
| capture conversations | [intake](references/conversation-intake.md) |
| resolve links, page collections or find record/party context | [links](references/record-links.md), [evidence](references/record-evidence.md) |
| import existing books | [import](references/import.md) |
| check the books after a change | [verify](references/verify.md) |
| see a statement, contracts or who owes what | [show the founder](references/show-the-founder.md) |
| write an investor update | [investor update](references/investor-update.md) |

## What is not available today

No standing Stripe sync, Economico-run email listener or bank feed: gather with your own access.
No planning scenarios or forecast overlays; no stock options; no tax returns or filings.
Economico never moves money or signs anything; it records what happened elsewhere and emails
customers only when asked.
