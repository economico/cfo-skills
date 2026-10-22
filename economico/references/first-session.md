# First session: model the business in one go

The goal of the first session is a live economic model of the founder's company, built from its
own evidence: what it sells and at what price, who buys it, what it costs to run and who it pays,
recorded in Economico so every future invoice, receipt and payment lands on terms that already
exist. The founder should finish the session able to see their business on one page and in the
books, with every number traceable to a source.

Do not install a generic chart of products. Shape the model around what this company actually
does. Use the session's stated business date for the model and dated reads; the host's wall
clock does not change that date.

The question rounds are scope before discovery, then review after the brief is written,
`model_preview` passes and the founder has seen the model. During discovery, keep gaps and
judgments in the brief rather than inserting another question round.

## 0. Agree the scope (one round of questions)

First look for `business-model.md` in the founder's repository. If it is there, an earlier
session wrote it: this is [a later session](#a-later-session), which starts from it.

Read [asking the founder](asking-the-founder.md) before the first question-tool call.
Extract the scope already settled by the request before composing any questions. A named
business settles the target; "you can read" a source grants that access. Use those answers
without reconfirming or offering to narrow them. Ask only the missing choices below. When the
business and source access are given, the scope round needs only history and recording choices
that remain unanswered.

Read `businesses {action: "list"}` to verify the target and note which integrations you can reach.
Do not read Stripe or mail just to find a date for the history question: offer "from today"
versus "since the evidence begins". Ask the remaining scope questions in one round, then work
uninterrupted until the review point:

| Header | Decision | Usually recommend |
|---|---|---|
| `Business` | Which business to write to (see [connect](connect.md)) | A sandbox, when the only business is the real one |
| `Access` | Only additional sources whose access the request has not granted; name what extra access is needed | The additional source needed to resolve an evidenced gap |
| `History` | How far back to record: "from today" (just the model), or "since <date>" (the model plus past invoices, receipts and payments; history dated before the business joined is a free [import](import.md)) | From today |
| `Recording` | Whether you may record after review without asking again | Record after I approve the model |

Missing access is not a blocker: work with what you have and list the gaps in the brief.

## 1. Discover (read-only)

Run the three discoveries in this session. Parallelize independent tool reads, rather than
spawning background agents or scheduling follow-ups.

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

- **Code vs Stripe.** The code enforces limits (included usage, seat caps); Stripe holds its
  configured prices. With no Stripe price, use complete prices evidenced in code or published
  pricing rather than assuming the plan is free. Save prelaunch plans as reusable off-sale
  templates; no customers or checkout yet does not justify omitting their evidenced terms.
  A Stripe price with no plan in code is legacy or custom: make it a question at the review.
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

Before asking for the founder's review, show the preview in an assistant message: the actual
price-book table, cost table and one run-rate line per native currency, as in
[show the founder](show-the-founder.md). Writing the brief or calling the preview tool alone
does not show these views in a terminal. The founder must see what they are deciding before
the question tool is called.

Then ask the founder to review the material modeling judgments you made, such as a vendor's
expense function, how a legacy plan is represented or a customer's negotiated terms, together
with unresolved source disagreements. Exclude choices the founder already settled. Use one
round of up to four questions, each with its recommended reading first ([asking the founder](asking-the-founder.md)).
A missing document or fact is a disclosed gap, not a decision: missing ownership evidence never
justifies recommending a guessed owner, share count, par value or investment amount. If the model required no material
judgment, ask whether to record the displayed model with those gaps or revise it. When already
told to proceed without review, take your recommendations, open the final message with these
views and list the decisions you took.

## 4. Record, in this order

Each step reads back before the next. Use `model:<step>:<id>` idempotency keys, so a rerun of the
session replays instead of duplicating.

1. **Company basics.** Profile description and URL; the legal form and jurisdiction when
   `business get` does not have them yet, evidenced by the formation document; financial accounts
   for each real bank account, card and wallet. See [company setup](company-setup.md).
2. **Ownership.** Establish the actual owners and units from formation documents or the founder,
   never from cash received. Discover supported priced ownership rights and evidenced
   authorization/subscription/vesting or conversion consequences. The ownership recipe links
   returned by `business get` use priced rights and separate payments. Record unsupported ownership
   families and missing quantities as explicit gaps. See [company setup](company-setup.md#ownership).
3. **Parties.** One per customer, vendor and founder. Stable ids from the source: `cus_<stripe id
   or slug>`, `ven_<domain slug>`, `founder_<name>`.
4. **Source documents.** Every order form, terms page, receipt and invoice you will cite, with
   `externalId` set to its source identity. See [evidence](evidence.md).
5. **Rights and templates.** Save the source-backed price book with `templates.create`,
   including prelaunch plans with `definition.plan: "off_sale"`. Read each saved template back and match its returned
   ID, price and sale status to the brief before proceeding. Reuse a template across equivalent
   agreements. Follow [modeling guidance](modeling-products.md#recipes) for the command shape;
   name rights by what the party receives and separate billing, recognition and payment.
6. **Contracts.** Bind each real agreement at its actual start date, with source documents and
   its party roles. Accept through `contracts.accept` with evidence. A wrongly dated unused
   draft can be discarded and recreated; accepted history is preserved.
7. **History** (if in scope). Record evidenced fulfillment, billing and recognition at their own
   dates and against consistent obligation/period identities. Settle the resulting claims with
   independent [payments](payments.md), not payment activities. Verify recorded phases instead
   of assuming a timer has run. Supported historical claims and claims carried through currency
   succession use the payment guide's explicit identities and amounts. Preserve and report
   unsupported facts rather than re-recording existing history. See [customer contracts](customer-contracts.md).


Once the contracts are recorded, show them in a message: one table, not one view per command
(see [show the founder](show-the-founder.md)).

If a command is refused, fix the input and retry with a new idempotency key only if the input
changed. Never loosen the economics to get past a refusal (for example, never drop a `service`
effect because service evidence is missing: ask for the evidence or record the gap).

## 5. Prove

Run the reads in [verify](verify.md) and put the results in the brief under "What the books say
now": actual revenue and expenses, open receivables and payables, cash per account,
and who owns the company (`cap_table`, and `capital_accounts` for an LLC, partnership or project).
Label any MRR/ARR computed from agreement prices as an estimate, not a ledger metric;
disclose unsupported new-model metric coverage. Every figure should be explainable from rows in the brief. A figure you cannot explain is a
modeling error; find it before you finish. Show the income statement, the balance sheet, the
aging and the cap table to the founder as you read them, inline or as text ([show the founder](show-the-founder.md)).

## Finish

Complete the foreground work and verification, then reread
[show the founder](show-the-founder.md) immediately before composing the final response.
Build the response from the verified reads and brief. Before sending, check it contains:

- Price-book and contract tables, with prices, billing, parties and status.
- Income-statement flow and table, balance-sheet table and balancing check, with period and currency.
- One ownership table with each holder's instrument, share or unit quantity, ownership
  percentage and vested units together in that holder's row; applicable capital balances.
  Build it from `cap_table`, with the [table shape](show-the-founder.md#the-text-versions).
  Explain missing or inapplicable reports.
- Open receivables and payables, material decisions and remaining evidence gaps.

Earlier messages and the brief do not replace these final views. Update "For the next session"
in the brief with standing answers, founder rules and open questions ([template](business-model-brief.md)).
End with a short paragraph naming the business, recorded work and the founder's next action.

## A later session

A founder who comes back should not be asked again what they already answered. When
`business-model.md` exists:

1. **Read it first**, "For the next session" before anything else. Load the complete brief
   with the host's file-reading tool (`Read` in Claude Code), including its standing answers.
   When no file-reading tool is offered, read the complete file with the available method.
   Use the business date stated
   in this request for recording and reports: "Today is <date>" pins that business cutoff, even
   when the host date is later. Do not advance an explicit historical cutoff from the host clock,
   record catch-ups after it or call future invoices overdue. Its business slug is the one to
   write to (still check `businesses` lists it; if it does not, ask). Its standing answers
   are this session's scope: ask the scope round only for what they do not cover or what the
   founder's request changes.
2. **Keep confirmed decisions.** A judgment the founder confirmed stands, and a rule they set
   applies to everything new. Change one only when the founder asks or new evidence contradicts
   it; then say which and why.
3. **Review unconfirmed decisions.** When the returning founder asks what needs deciding, put
   material modeling judgments made in their absence through one recommended-choice round
   ([asking the founder](asking-the-founder.md)). Tie each choice and reason to the specific
   plan, customer, vendor or payment in the brief. An autonomous judgment is not a standing answer
   confirmed by the founder. Otherwise ask only genuinely open choices; reuse confirmed answers.
   When they are away, keep open decisions in the brief. Missing documents, identities and
   quantities remain evidence gaps; request them in the final message, without question-tool
   options or a recorded default.
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
