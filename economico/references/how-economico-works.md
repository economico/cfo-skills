# How Economico works

Read this once per session before you design anything. It is the model you map a business onto.

## The pieces

| Piece | What it is | How you touch it |
|---|---|---|
| **Business** | One set of books with one append-only, hash-chained history. Addressed by a slug. | `businesses`, `business {action: "get"}` |
| **Command** | A persisted, authorized, idempotent intent. The only way anything is written. | `catalog` to discover, `commands` to execute |
| **Party** | A counterparty: a customer, vendor, founder, investor or employee. A contract gives it a role in that contract only; what it is to the business is its `categories`, set when you create it, and every list groups it by them. | `parties.create`, `parties` read |
| **Financial account** | A named bank account, wallet or company card, so cash and card balances are kept per account. Bookkeeping only; nothing is connected. | `accounts.register`, `accounts` read |
| **Document** | Immutable bytes addressed by content hash: an uploaded source (a PDF or text receipt, an order form), or a definition, contract or statement Economico produced. | `documents.receive`, `documents` read |
| **Activity** | A reusable, immutable definition of something that happens under an agreement: its trigger, the terms it is priced by, the facts observed when it happens, and the effects it posts. | `activities.create` |
| **Template** | A reusable bundle of activities for a set of party roles. A price plan is a template marked `plan: "on_sale"`; a vendor's standard terms are a template. | `templates.create` |
| **Contract** | A template bound to real parties, resolved terms and ledgers. Starts `draft`; money is refused until an acceptance transition makes it `active`. | `contracts.create`, `contracts.amend` |
| **Occurrence** | One recorded happening of a bound activity, with facts and evidence. It posts the declared effects atomically and returns an immutable `activity_statement` (the invoice, bill or receipt document). | `contracts.record` |
| **Ledger** | One independently balanced set of accounts in one unit. The GAAP books are one ledger per currency; usage meters, capacity pools and share classes are their own ledgers and never net with money. | `ledgers` read |
| **Report** | A calculation over ledgers: statements, aging, SaaS metrics, claims, plans. | `reports {action: "list" \| "run"}` |

The chain an agent builds is always the same:

```text
party ─┐
       ├─ contract (template + terms + ledgers) ── accept ── record occurrences ── reports
activity definitions ── template ─┘                          │
                                                   facts + evidence + source documents
```

## The envelope

Every write is this shape, on MCP `commands` (with `"action": "execute"`), REST
`POST /v1/commands`, or the CLI's `commands execute --args`:

```json
{
  "business": "acme-sandbox-3f2a",
  "action": "execute",
  "name": "parties.create",
  "input": { "partyId": "cus_globex", "name": "Globex Corp", "categories": ["customer"] },
  "idempotency_key": "model:party:cus_globex",
  "effective_at": "2026-09-01T00:00:00Z"
}
```

The receipt is `{ commandId, replayed, result, events }`. Replaying the same key returns the
original receipt with `replayed: true`; the same key with a different command is
`idempotency_conflict`. A domain no-op returns `{ code: "no_change" }` without an error, so read
`code` before `result`. `effective_at` is the book date. Some commands also take a
`source_document_id` on the envelope; `catalog describe` lists which document kinds each accepts
under `sourceDocumentKinds`.

## Terms, facts, calculations, effects

An activity definition separates what was agreed from what was observed:

- **terms** are agreed values: a price, a start and end date, an included quantity. Each has a
  type (`integer`, `text`, `date`, `instant`), an optional unit, and an optional default. A
  template supplies defaults; each contract can override them per activity key in
  `contracts.create` `terms`.
- **facts** are observed when the activity is recorded: an amount on a receipt, a usage count,
  the bank account the money landed in, the occurrence key of the invoice being paid. Facts never
  have defaults.
- **calculations** derive amounts: `rate` (quantity × rate ÷ unitsPerPrice), `tiered` (graduated
  or volume), `allowance`, `minimum`, `reprice`, `tax`. Their result is `calc:<key>:amount`.
- **effects** name a code-owned posting pattern, a ledger alias and an amount reference
  (`term:x`, `fact:x` or `calc:k:amount`). The pattern owns the account pair; you only choose
  the classification (category, revenue or expense account, expense function).

The trigger decides when an activity happens: `transition` (a lifecycle move such as
`draft → active`), `recorded` (you record it with facts), or `scheduled` (a date schedule over
start/end terms; the daily timer records fixed periods itself). Anything that happens more than
once must say `repeatable: true`.

## Effect patterns

The patterns you will use to model a business. Each debits and credits fixed accounts; the
evidence column is the evidence purpose `contracts.record` must carry when the effect posts a
non-zero amount.

| Pattern | Debit | Credit | Needs | Use for |
|---|---|---|---|---|
| `bill` | 1120 receivables | 2150 deferred revenue | — | Invoicing a customer ahead of or at delivery |
| `recognize` | 2150 deferred revenue | revenue | service | Earning billed service as it is delivered |
| `accrue_revenue` | 1125 unbilled | revenue | service | Earning usage or work before it is invoiced |
| `bill_accrued` | 1120 receivables | 1125 unbilled | — | Invoicing what was already earned |
| `collect` | 1110 cash | 1120 receivables | payment | A customer payment against a billed claim |
| `accrue_expense` | expense | 2110 payables | service | A vendor invoice for delivered service |
| `pay` | 2110 payables | 1110 cash | payment | Paying a vendor claim from a bank account |
| `card_expense` | expense | 2190 card payable | service | A purchase already charged to the company card |
| `pay_card` | 2190 card payable | 1110 cash | payment | Paying the card statement |
| `pay_related` | 2110 payables | 2135 due to related parties (`toPartyRole`) | payment | A founder paid a vendor's bill personally; now owed to them |
| `related_expense` | expense | 2135 due to related parties | payment | A founder cost with no vendor contract of its own (a per diem, mileage) |
| `reimburse_related` | 2135 | 1110 cash | payment | Paying the founder back |
| `prepay_expense` | 1150 prepaid | 2110 payables | — | A vendor charge for a future period (annual plans) |
| `expense` | expense | 1150 prepaid | service | Using up a prepaid vendor period |
| `credit_expense` | 2110 payables | expense | acceptance | A vendor credit against an open bill |
| `receive_unapplied` | 1110 cash | 2185 unapplied receipts | payment | Money in before you know what it pays |
| `apply_receipt` | 2185 | 1120 receivables | acceptance | Applying that money to an invoice later |
| `refund_deferred` | 2150 deferred revenue | 1110 cash | payment | Refunding an unearned customer balance |
| `bill_tax` | 1120 receivables | 2160 output tax | — | Sales tax, GST or VAT charged on an invoice |
| `accrue_input_tax` | 1170 recoverable tax | 2110 payables | — | Recoverable tax on a vendor bill |
| `observe` | meter | meter | — | Recording a measured quantity in its own usage ledger |
| `meter_billed` | meter | meter | — | Marking measured quantity as invoiced |
| `grant` | capacity | capacity | — | Granting seats, credits or included usage |
| `consume` | capacity | capacity | — | Drawing that capacity down |
| `commit` | contract flow | contract flow | acceptance | The accepted contract value, without cash or revenue |
| `fund_liability` | 1110 cash | 2240 financing | payment | SAFE or convertible money received |
| `fund_capital` | 1110 cash | 3100 legal capital | payment | Cash paid for shares at par |

`reports {action: "run", kind: "activity_effects"}` without a contract lists every pattern with
its accounts; with `contract_id`, `activity_key`, `effective_at` and `facts` it previews exactly
what a recording would post, without writing.

## Claims: how payments find their invoice

A billing effect opens a **claim** identified by contract, the billing activity's binding key and
the occurrence key. A settlement effect (`collect`, `pay`) names that claim:

- `accountingKey` on the settlement effect names the claim's key: the **binding key of the
  billing activity** in the template (for example `bill`), unless the billing effect carries its
  own `accountingKey`, which then wins. `bill_accrued` does: it bills what an earlier activity
  earned, so its claim is keyed by that earning activity (`usage`), and the payment must name
  `usage`, not the invoicing activity. When unsure, record the invoice, then read
  `reports run activity_claims` and copy the claim's key.
- `claimFact` names a text fact that will carry the **occurrence key** of the invoice being paid
  (not its occurrence id).
- `financialAccountFact` names a text fact that will carry a registered financial account id.

When the charge and the payment happen in one recording (a receipt that says "paid"), put both
effects in one activity and omit `claimFact`; the payment settles the claim this occurrence opens.

## Recording rules that matter

- **Acceptance first.** A draft contract refuses every money effect. Record the `draft → active`
  transition with `acceptance` evidence (the signed order form, the accepted terms of service).
- **Evidence is computed from the effects**: `service` for anything earned or incurred,
  `payment` for anything that moves cash, `acceptance` for activation and credits.
- **Keys are identities.** `occurrenceKey` is unique within the contract activity;
  `sourceFactId` is unique across the whole business. Identical input replays the original
  statement (`duplicate: true`); the same key with different facts is refused.
- **Dates go forward per contract.** A recording may not be in the future, nor earlier than the
  contract's latest recorded activity. Backfill history in date order, contract by contract.
- **Scheduled periods record themselves.** The daily timer records each fixed scheduled period
  (occurrence key `schedule:<start>:<end>`) once it is due, and skips any period already recorded.
  Record one by hand only when you need it on the books now; use that same occurrence key.
- **Corrections are new events.** Nothing is edited. A wrong occurrence is corrected with
  `correctsOccurrenceId`, and only the latest active occurrence of that activity can be
  corrected: fix mistakes as you go, not at the end. A contract change is `contracts.amend`; a
  wrong definition is `activities.replace`, which only future contracts use, because existing
  contracts keep their frozen version. The cheapest fix is prevention: preview with
  `activity_effects`, and record the first occurrence of every activity (including the payment)
  on one contract before binding the template to the rest.

## Reading back

| Question | Read |
|---|---|
| What types can I list, with which filters? | `subjects {action: "types"}` |
| Which plans are on sale? | `subjects {action: "list", type: "template", plan: "on_sale"}` |
| What state is this contract in? | `subjects {action: "get", type: "contract", id}` |
| What was recorded on it? | `subjects {action: "list", type: "activity_occurrence", contract_id}` |
| What is owed on it, per invoice? | `reports run activity_claims` with `contract_id` |
| What will it do next? | `reports run activity_plan` with `contract_id`, `from`, `through` |
| The source or statement bytes | `documents {action: "get", id}` |

`reports {action: "list"}` names each report kind's parameters and gives a runnable example;
a parameter another kind takes is refused.
