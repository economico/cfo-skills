---
name: economico
description: "Set up, connect, model a whole business and run the books in Economico: first-session discovery into a full business model; resume sessions; explain how Economico works (accrual accounting, prepaid plans, recognition, payments, refusals); verify and show the books."
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
   Fetch recipes via `catalog` (`recipe`); schema wins.
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
| 3 | Discover | Notes on product, pricing, customers, costs, entity | [codebase](references/explore-codebase.md), [Stripe](../manage-sales/references/explore-stripe.md), [email](../manage-vendors/references/explore-email.md) |
| 4 | Propose | `business-model.md`, passing `model_preview`, then founder review of price book and costs | [first session](references/first-session.md), [brief template](references/business-model-brief.md) |
| 5 | Record | Legal form and owners, parties, accounts, templates, contracts, activity, evidence | [product-economics](../product-economics/SKILL.md), [manage-sales](../manage-sales/SKILL.md), [manage-vendors](../manage-vendors/SKILL.md), [setup-company](../setup-company/SKILL.md), [evidence](references/evidence.md), [accounts](references/accounts.md), [payments](references/payments.md) |
| 6 | Prove | Final message opening with views of the price book, contracts, statements and cap table | [verify](references/verify.md) |

Show [the founder views](references/show-the-founder.md) in phases 4–6: inline if supported, otherwise as text.

## Specialist skills and shared references

| The founder asks to… | Go to |
|---|---|
| connect an agent, log in, pick or create a business, run headless | [connect](references/connect.md) |
| how Economico works, or why something was refused | [how Economico works](references/how-economico-works.md) |
| design product pricing, reusable plans, patterns or unit economics | [product-economics](../product-economics/SKILL.md) |
| integrate a product backend over REST or MCP (sign-ups, contracts, usage) | [integrate-product](../integrate-product/SKILL.md) |
| track sales deals, add customers, bill, email invoices or reconcile Stripe | [manage-sales](../manage-sales/SKILL.md) |
| evaluate vendors, adopt plans, process email receipts, bills or tool usage | [manage-vendors](../manage-vendors/SKILL.md) |
| run a hiring pipeline, track candidates and close accepted offers | [hiring](../hiring/SKILL.md) |
| manage People arrangements, provider payroll runs or named service seats | [manage-people](../manage-people/SKILL.md) |
| run a fundraise pipeline, close SAFEs, notes or priced rounds | [fundraise](../fundraise/SKILL.md) |
| set up company profile, legal entity, cap table, bank accounts, cards or wallets | [setup-company](../setup-company/SKILL.md) |
| set goals, KPIs, OKRs, V2MOMs, quotas, operating plans or custom views | [plan-and-track](../plan-and-track/SKILL.md) |
| write and publish a monthly investor update | [investor-update](../investor-update/SKILL.md) |
| record payments, reimbursements, transfers or corrections | [payments](references/payments.md) |
| choose the account for a cost or revenue line | [accounts](references/accounts.md) |
| manage shared pipeline mechanics or capture conversations | [pipelines](references/pipelines.md), [intake](references/conversation-intake.md) |
| resolve links, page collections or find record/party context | [links](references/record-links.md), [evidence](references/record-evidence.md) |
| import existing books, check the books or show a statement | [import](references/import.md), [verify](references/verify.md), [show the founder](references/show-the-founder.md) |

## What is not available today

No standing Stripe sync, Economico email listener or bank feed; use your own access.
No planning scenarios, forecast overlays, stock options, tax returns or filings.
Economico moves money or signs only for its own plan the founder buys; it records what
happened elsewhere and emails customers only when asked.
