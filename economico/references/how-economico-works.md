# How Economico works

Economico records contractual rights and their economic consequences separately from payments.
An activity is a right a party gets under an agreement: access to software, measured API use,
consulting work, a domain registration or an ownership subscription. The right carries its
price and independent billing and recognition timing. A bank or card payment is a separate fact.

Discover `activities.create`, `contracts.record` and `payments.record` through `catalog` before
writing. These instructions use `model: "priced-rights-v1"`. If the connected deployment does
not expose this model, report the capability gap; do not invent equivalent posting activities.
Existing effect-based histories still replay, but their shapes are not the model for new work.

## The pieces

- **Activity:** one reusable right, its agreed terms, observed facts, price treatment and
  economics. `activities.create` saves an immutable definition.
- **Template:** the rights under one agreement, bound to party roles and ledger aliases.
  Reuse it across equivalent agreements; a price-only difference is a term override.
- **Contract:** the template bound to real parties, dates, terms and ledgers. Preserve the
  actual agreement boundary: several agreements with the same party are valid.
- **Lifecycle:** `contracts.accept`, `contracts.cancel` and `contracts.terminate` are evidenced
  state changes. Do not define an acceptance or cancellation activity. A separately priced
  exit right may produce economics on a transition; cancellation itself does not pay a refund.
- **Economic occurrence:** `contracts.record` records a phase of a right for an identified
  obligation or period. Billing, delivery/usage and recognition can happen at different times.
- **Claim:** a named receivable or payable. Commercial billing produces a claim; investments,
  owner funding and other obligations need not have a commercial invoice.
- **Payment:** `payments.record` identifies money, its account, counterparty and claim
  allocations. It contains no contract or activity input. Account ownership decides who funded it.
- **Document:** immutable source bytes or a generated statement, with provenance to the command
  and event. **Reports** derive balances and claims from the recorded facts.

## The envelope

MCP `commands`, REST `POST /v1/commands` and the CLI's `commands execute` use the same registry.
For example:

```json
{
  "business": "acme-sandbox-3f2a",
  "action": "execute",
  "name": "parties.create",
  "input": { "partyId": "cus_globex", "name": "Globex", "categories": ["customer"] },
  "idempotency_key": "model:party:cus_globex",
  "effective_at": "2026-09-01T00:00:00Z"
}
```

The receipt is `{commandId, replayed, result, events}`. Retry identical input with the same
idempotency key. Reusing it with different input conflicts. Read a domain `no_change` response
before assuming `result` exists. `source_document_id`, when supported by the described command,
links the source document. Money is a string of minor units; instants are UTC with `Z`.

## Terms, facts and price

Terms state the agreement: price, rate, currency, dates, limits. Facts state what happened:
delivered work, measured usage, or an identified claim/grant where that right requires one.
A financial account is a payment input, never a right's pricing fact.

Use `price.treatment` explicitly: `priced`, `included`, `free`, `promotional` or `unknown`.
Included and free rights still exist. Unknown pricing is a gap, not zero and not permission to
invent a formula. Fixed, rate and tiered calculations refer to integer terms or facts. For
example a fixed price uses `calculation: {type: "fixed", amount: "term:price"}` and
`currencyTerm: "currency"`. A rate needs its measured quantity, rate and units-per-price.

Classification belongs in `economics.classification`: the monetary ledger, revenue category
or expense account/function, product and applicable tax code. It does not create another right.
Separate rights only when what the party receives differs, even if two rights post alike.

## Recording rules that matter

1. Date `contracts.create` at the actual agreement start and accept with evidence of acceptance,
   its current document ID and expected status. A source terms URL proves the terms' location,
   not that someone accepted them; keep evidence of signup, signature or use as appropriate.
2. Record a stable `obligationKey` for a one-off item or the same `period` for all phases of a
   recurring obligation. Give each phase its own occurrence and source identity.
3. `delivery` or `usage` records fulfillment. `recognition` consumes that entitlement; it does
   not bill. Ratable recognition needs evidenced elapsed coverage under the frozen schedule.
4. `billing` records the commercial claim and optional `dueDate`. Advance billing stays deferred
   or prepaid until earned/consumed. Recognition before billing accrues independently; later
   billing clears the accrual rather than recording the income or expense twice.
5. Payment settles the claim independently. Read `activity_claims` and copy its `claimId` and
   full `component`, not an activity name, invoice text or counterparty total. See
   [payments](payments.md). Outstanding claims can still be settled after cancellation.
6. Retain immutable evidence and retry identities. Never re-record old history to convert its
   model. Payment correction uses reversal/unallocation; service correction uses
   `contracts.reverse_economics`. Supported historical claims settle through independent
   [payments](payments.md) without reissuing their invoices.

Do not assume a declared schedule has executed itself. Read returned phase statements, claims
and balances. The new model's forecast/scheduling surface is not yet equivalent to the historical
one; do not promise an `activity_plan` forecast or automatic recognition without confirming the
connected catalog's supported behavior and the recorded result.

## Preserve source identities

Copy a supplied source-fact identity exactly into `sourceFactId`, and an agreement's supplied
external identity into its source document's `externalId`. Do not reorder segments, rename it,
or add a phase suffix: `invoice:vendor:october` must not become `vendor:invoice:october:bill`.
A later recognition, acceptance or other distinct fact needs its own distinct identity; it does
not rename the original invoice or payment. Contract source documents may include supporting
evidence; keep the actual agreement linked and identifiable among them.

## Reading back

Use `subjects` to inspect the contract and `documents` for source and generated statement
bytes, retaining the document IDs returned by phase commands. New economic phases are not
listed under the historical `activity_occurrence` subject type. Follow actual footprint entries
to their statement documents instead. `reports run activity_claims` returns commercial and
noncommercial claim identities, original/outstanding amounts and their current components.
`aging`, `income_statement` and `balance_sheet` prove what is owed, earned and held.
`ledgers {action: "footprint", party_id}` shows the counterparty's actual postings, including
unapplied funds. Read the live report schema; reports do not all accept the same parameters.

For evidenced financing, distributions and draws, follow [company setup](company-setup.md).
For currency succession, valued settlement, owner/card advances and refunds after
reimbursement, follow [payments](payments.md). These supported paths do not imply every
admitted economic shape is executable: keep unsupported facts and questions visible.

`reports {action: "list"}` names each report kind's parameters and gives a runnable example;
a parameter another kind takes is refused.

## Telling Economico

When something is broken, unclear or missing, tell the Economico team in-band with
`messages.send`, leaving out `to`: `topic` is `support` or `feature_suggestion`, then a
`subject`, a `body` in your own words, and optional `context` such as the command you tried.
Tell the founder you did, once its record says `delivered` (it is `sent` while queued, and
`undeliverable` with the reason if it could not arrive). Messages go only to a business this one
has an agreement with in both books (Economico), or has connected with: ask with
`connections.request` (`to` its exact slug, `partyId` the party in these books it is), and the
other business accepts with `connections.accept`; then `messages.send` with `to` that party. Answer a message you received with `inReplyTo` (its id) and a
`body`, and name the documents a message concerns, such as an invoice Economico delivered,
with `about` (their document ids).
The message is kept permanently in this business's history and in Economico's own books, so
never include secrets or credentials. Sent and received messages are both `message` subjects.
