# First session: model the business in one go

The goal of the first session is a live economic model of the founder's company, built from its
own evidence: what it sells and at what price, who buys it, what it costs to run and who it pays,
recorded in Economico so every future invoice, receipt and payment lands on terms that already
exist. The founder should finish the session able to see their business on one page and in the
books, with every number traceable to a source.

Do not install a generic chart of products. Shape the model around what this company actually
does.

## 0. Agree the scope (one round of questions)

First look for `business-model.md` in the founder's repository. If it is there, an earlier
session wrote it: this is [a later session](#a-later-session), which starts from it.

Read `businesses {action: "list"}` and note which integrations you can reach, then ask once, in
one round, as in [asking the founder](asking-the-founder.md), and work without further
interruption until the review point:

| Header | Decision | Usually recommend |
|---|---|---|
| `Business` | Which business to write to (see [connect](connect.md)) | A sandbox, when the only business is the real one |
| `Access` | What you may read: this repository, Stripe, the mailbox, anything else (Ramp, Mercury, a pricing page); multi-select, from what is connected | Everything connected |
| `History` | How far back to record: "from today" (just the model), or "since <date>" (the model plus past invoices, receipts and payments) | From today |
| `Recording` | Whether you may record after review without asking again | Record after I approve the model |

Skip any question the founder's request already answers. Missing access is not a blocker: work
with what you have and list the gaps in the brief.

## 1. Discover (read-only)

Run the three discoveries. They are independent; run them in parallel if you can.

| Source | Reference | You come back with |
|---|---|---|
| The codebase | [explore-codebase](explore-codebase.md) | What the product does, who pays, the plans and limits enforced in code, the usage the product meters, the billing integration, the services it runs on |
| Stripe | [explore-stripe](explore-stripe.md) | Products, prices, active subscriptions and customers, invoices and payouts, fees |
| Email | [explore-email](explore-email.md) | Vendors, what each charges and how it is paid, receipts and invoices with their Message-IDs, signed customer agreements |

Also read `business {action: "get"}` and `subjects {action: "list", type: "contract"}`: a business
that already has contracts is extended, never duplicated. Every business starts with one contract
already in place: its own service agreement with Economico (Economico is a vendor party). Leave it
as it is; it is not part of the model you are building.

Also establish **who owns the company**: the legal form and jurisdiction from `business get`, and
the owners with their shares, units or percentage from the formation documents (certificate of
incorporation or formation, stock purchase agreements, the operating agreement, a formation
service's confirmation such as Stripe Atlas) found in the repository or the mailbox. Read
`cap_table` too: a business whose cap table already lists its holders is extended, never
duplicated. Every company has owners, so the cap table is part of the model even when no money
came from them.

## 2. Reconcile

The sources disagree more often than not. Resolve each disagreement explicitly:

- **Code vs Stripe.** The code enforces limits (included usage, seat caps); Stripe holds prices.
  A plan in code with no Stripe price is either free or not launched. A Stripe price with no plan
  in code is legacy or custom: make it a question at the review.
- **Stripe vs signed terms.** A signed order form wins over a Stripe price for that customer.
  Stripe is evidence of what was charged, not of what was agreed.
- **Email vs code.** A vendor in the code (the hosting provider, the model API, the email API)
  serves the product and is usually cost of revenue. A vendor that bills you but is not in the
  code is overhead, and its function comes from what it is for, per [accounts](accounts.md):
  team development tools (GitHub, Cursor) are research and development, workspace tools
  (Notion) general and administrative.
- **One vendor, two roles.** Stripe is a billing provider (fees are 5230 payment processing) and
  also where customer money lands before payout.

## 3. Propose: write `business-model.md`

Write the brief into the founder's repository using [the template](business-model-brief.md). It
is the "wow" artifact: one page that explains how this company makes and spends money, with every
line traceable to its source and mapped to what will be recorded. It must contain:

1. What the company sells, to whom, and how it is delivered.
2. The price book: one row per plan with price, cadence, included usage, overage, and the
   template it becomes.
3. Customers: one row per paying customer with plan, start date, source (Stripe subscription id
   or signed document), and any negotiated difference.
4. Costs: one row per vendor with what it is for, amount and cadence, payment method, account and
   expense function, and the source.
5. Money in and out per month at today's run rate, computed in code from the rows above
   (subscription MRR, committed vendor spend), labeled as run rate, not forecast.
6. Assumptions and gaps: every value you could not source, and every judgment you made.
7. What will be recorded, in order.

Then preview it: `reports run model_preview` with the brief's tables as data
([how](business-model-brief.md#preview-the-model-before-recording-it)). Always, even when the
founder told you to proceed without review: the preview is the check before anything is recorded,
not a courtesy for the review. It checks the proposal and computes the run rate the brief reports;
a refusal is a mistake in the brief, so fix the brief and preview again. Record nothing until it
passes.

Stop here for the founder's review unless they told you to proceed. The review message shows
the model from the preview: the price book, the cost table and the run-rate line, as in
[show the founder](show-the-founder.md). If they told you to proceed, the price book opens the
final message instead. Keep the review short, then ask the three to five judgments that change the
numbers as one round of questions (a plan you treated as legacy, a vendor you classified as cost of
revenue, a customer whose price differs from the plan), each with the reading you recommend first
([asking the founder](asking-the-founder.md)). When told to proceed, take your recommendations and
list them in the final message.

## 4. Record, in this order

Each step reads back before the next. Use `model:<step>:<id>` idempotency keys, so a rerun of the
session replays instead of duplicating.

1. **Company basics.** Profile description and URL; the legal form and jurisdiction when
   `business get` does not have them yet, evidenced by the formation document; financial accounts
   for each real bank account, card and wallet. See [company setup](company-setup.md).
2. **Ownership.** One contract per holder from the recipe `business get` names under
   `ownership` (its recipes), so `cap_table` lists every owner with their units and
   `capital_accounts` (LLCs, partnerships, projects) names each one. A sole owner is recorded too:
   a single-member LLC's member holds all the units. Record it even when the owners put in no
   cash yet. The quantities come from a document or the founder, never from cash received; when
   neither says, ask in the scope round and list the gap. See
   [company setup](company-setup.md#ownership).
3. **Parties.** One per customer, vendor and founder. Stable ids from the source: `cus_<stripe id
   or slug>`, `ven_<domain slug>`, `founder_<name>`.
4. **Source documents.** Every order form, terms page, receipt and invoice you will cite, with
   `externalId` set to its source identity. See [evidence](evidence.md).
5. **Activities and templates.** One template per plan in the price book, one per vendor. Start
   from [the recipes](modeling-products.md#recipes). Preview each template's money effects with
   `activity_effects` before binding a customer to it.
6. **Contracts.** One per customer subscription or agreement, one per vendor relationship. Record
   acceptance on each, with the source that evidences it.
7. **History** (if in scope). Per contract, in date order: invoices, payments, receipts, usage.
   Scheduled periods record themselves daily; record past ones only if the founder wants the
   history on the books today.

Once the contracts are recorded, show them in a message: one table, not one view per command
(see [show the founder](show-the-founder.md)).

If a command is refused, fix the input and retry with a new idempotency key only if the input
changed. Never loosen the economics to get past a refusal (for example, never drop a `service`
effect because service evidence is missing: ask for the evidence or record the gap).

## 5. Prove

Run the reads in [verify](verify.md) and put the results in the brief under "What the books say
now": MRR and ARR, revenue and expenses to date, open receivables and payables, cash per account,
and who owns the company (`cap_table`, and `capital_accounts` for an LLC, partnership or project).
Every figure should be explainable from rows in the brief. A figure you cannot explain is a
modeling error; find it before you finish. Show the income statement, the balance sheet, the
aging and the cap table to the founder as you read them, inline or as text ([show the founder](show-the-founder.md)).

## Finish

Open the final message with the views: the price book table (plan, price, billing), the contracts
table (party, customer or vendor, template,
status), the income statement (its flow line and its table), the balance sheet table, the cap table as a table (one row per holder) and
the open items from the aging, inline or as text
([show the founder](show-the-founder.md)). A sentence that quotes the totals is not the view.
Before the final message, update "For the next session" in the brief: the standing answers,
the rules the founder set, and the questions still waiting for them
([template](business-model-brief.md)).

Then report in five lines or fewer: the business
written to, what was recorded (counts of templates, contracts, occurrences, documents), the
headline numbers, the open gaps, and the one thing the founder should do next (usually: grant email or Stripe access that was missing, or confirm a
judgment).

## A later session

A founder who comes back should not be asked again what they already answered. When
`business-model.md` exists:

1. **Read it first**, "For the next session" before anything else. Its business slug is the one
   to write to (still check `businesses` lists it; if it does not, ask). Its standing answers
   are this session's scope: ask the scope round only for what they do not cover or what the
   founder's request changes.
2. **Do not relitigate.** A judgment under "Judgments and gaps" stands, whoever made it, and a
   rule the founder set applies to everything new. Change one only when the founder asks, or when
   new evidence contradicts it; then say which and why.
3. **Ask what is waiting.** The open questions are the first round of questions when the
   founder is present ([asking the founder](asking-the-founder.md)); when they are away, keep
   them open. Every one has a recommended option first, a gap only the founder can fill
   included: recommend what you would record if they did not answer, and say why.
4. **Read the books for what exists**: `subjects`, `parties` and `documents` by `externalId`
   and `sourceFactId`, and extend them. Record only what is new since the last session, with the
   same idempotency keys, so a rerun replays instead of duplicating.
5. **Update the brief** at the end: its tables where the business changed, and "For the next
   session".

Say in the final message what you took from the brief and what, if anything, was new.

## When the founder has not incorporated

A developer exploring a project can model it before there is a company. Use a business with the
legal form `project`; everything above works the same. Plans with no customers yet are templates
with no contracts. Costs are still vendor contracts. When the project incorporates, the business
incorporates in place (`business.incorporate`, owner-only) and the history carries over.
