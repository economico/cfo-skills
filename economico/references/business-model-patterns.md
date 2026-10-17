# Business model patterns

Business model patterns are composable pricing ideas for the company's offerings or a bespoke
client contract. This catalog is bundled with the skill; it is not a list of records in the
connected business. Use the business's `subjects` and `documents` reads to find saved rights and
plans, then the existing activity, template and contract commands to save the agreed result.

## Find a pattern

List these choices when asked to explore business models. Match the founder's service, pricing
method or unit to the examples; a combination expands into independently configurable rights.
The IDs match the workspace's Business model patterns explorer, not activity or document IDs.

| Pattern ID | Pattern | Components and search examples |
|---|---|---|
| fixed | Fixed price | One agreed amount: setup, milestone, subscription, availability retainer |
| unit | Per unit | Uniform, graduated or volume rates × quantity: seats, hours, API calls, transactions |
| percentage | Percentage of value | Share of an evidenced base: take rate, commission, revenue share |
| spread | Exchange spread | Margin between reference and customer exchange rates; support must be checked |
| setup | Setup fee | Fixed price for onboarding or implementation, with delivery criteria |
| access | Recurring access | Fixed price with configurable service and billing cadence |
| access-usage | Access plus usage | Fixed access + per-unit usage |
| setup-access | Setup plus ongoing access | Fixed setup + separate fixed recurring access |
| processing | Processing fee plus take rate | Per-transaction fee + percentage of the same evidenced value |
| time-services | Hourly or daily services | Per-unit time: consulting, freelance work, day rate, billable hours |
| retainer | Monthly retainer | Fixed recurring service; resolve availability versus prepaid work |
| milestones | Milestone-based project | Separate fixed fees for agreed deliverables and completion criteria |
| bespoke | Bespoke client services | Setup + billable hours + retainer + software access |

A block can appear several times. Setup and access remain different rights even if both have
fixed prices. Monthly, annual and other cadences are terms, not different pricing primitives.
A selection or copied brief is a modeling request, never evidence of a customer agreement.

For executable shapes and required facts, read the relevant recipes in
[modeling products](modeling-products.md#recipes): monthly/annual access, seats, access plus usage,
hours plus a milestone, or prepaid/included capacity. Adapt their definitions, never their
example amounts, parties, dates, evidence or transactions. Percentage and spread ideas are not
validated recipes: check the live calculation and currency semantics before promising support.

For tiered components or Stripe tiered Prices, read [Stripe tiered pricing](stripe-tiered-pricing.md)
before mapping the schedule. Tier flat fees stay within their pricing component.

## Find saved rights and plans

1. Verify the connection and target business using [connect](connect.md). Call
   `subjects {action: "types"}` to learn supported filters.
2. Read `subjects {action: "list", type: "activity"}` and
   `subjects {action: "list", type: "template"}`. Follow each returned `nextCursor` with `after`
   until exhausted. Do not restrict discovery to on-sale plans: an unmarked template may be
   valid custom terms. There is no assumed name-search endpoint; match the returned definitions
   locally by what is provided, pricing method, units, timing and evidence requirements.
3. Read candidates with `subjects {action: "get", type: "activity", id}` or
   `subjects {action: "get", type: "template", id}`. Inspect `state.definition`, status and
   `state.documentId`. For each template binding, read its frozen `activityDocumentId` through
   `documents {action: "get", id}`; the activity's latest version may differ from that binding.
4. Compare economic meaning, direction, price treatment/calculation, units/scales, service,
   billing and recognition timing, required terms/facts and ledger requirements. Customer
   services use `economics.direction: "provide"`; a vendor cost or ownership right is not a
   matching customer service. Names or matching account codes alone do not establish equivalence.
   Reuse suitable immutable document versions; don't mutate a shared right to fit a new deal.

Discovery is complete when every requested component has a proposed reused document ID or a
reason it needs a new definition, with missing terms and evidence listed. Preserve separate
agreements with the same client and several offerings in the same company.

## Resolve the selection

Use source facts already available, then ask for unresolved decisions. For every component,
resolve currency, price, quantity source and scale, service dates, billing cadence, recognition
basis and evidence. [Modeling products](modeling-products.md) owns the definition rules.

- Time-based services: agree hours versus days, billable day length, fractional time and work approval evidence.
- Milestones: name each deliverable and its fixed fee, completion/acceptance evidence, billing trigger and recognition basis. Keep each milestone a separate right.
- Retainer: clarify availability versus prepaid hours/capacity, expiry, rollover and overage.
  Prepaid consumption must not invoice the original purchase again.
- Percentage: agree the gross/net base, refunds, currency, exact numerator/denominator and
  rounding. Verify a source-backed sample against the live rate calculation.
- Exchange spread: resolve both currencies, reference-rate source and margin method. A fee
  percentage is not a substitute for exchanging currencies. If unsupported, preserve the gap.
- Recurrence: distinguish service period, billing and collection; upfront billing does not mean
  immediate recognition. Keep measured usage separate from fixed recurring revenue.

## Save only the requested destination

Describe each live command before use. Keep exact amounts and quantities in declared integer
units and bind returned immutable document IDs, never pattern IDs or guessed IDs.

**Reusable company plan.** Reuse/create the agreed rights with `activities.create`; compose a
`priced-rights-v1` template with `templates.create`. Resolve whether it is on sale; don't infer
that from creating it. To extend an existing plan, read that plan and create a new reusable
plan with its suitable rights and the additions, preserving the original and existing customer
contracts. Read back the saved activities and template. Stop here: no customer or contract is
required to define the company's offerings.

**Bespoke client contract.** Follow [customer contracts](customer-contracts.md). Resolve the real
client, agreement dates, source documents, per-right terms and native ledger bindings. Reuse a
suitable template via `templateDocumentId`, or pass an inline template as `definition` to
`contracts.create`—exactly one, never both. An inline template binds saved immutable activity
document IDs; it does not require a saved reusable plan. Create/reuse only the rights needed.
Read back the resulting draft with `subjects {action: "get", type: "contract", id}` and stop.
Acceptance, billing, recognition and payments require their separate instructions and evidence;
a request to prepare a contract does not authorize those steps.

Show the resulting plan or draft contract, component names and IDs, what was reused, and any
unresolved terms or evidence. Do not report an unsupported component as configured. Browsing or
listing alone performs no writes; a request to save follows the skill's existing authority rules.

## Preserve the visual pattern

When saving a right from an agreed pattern, put its catalog ID in
`definition.externalReferences.economicoPattern`. This is a display reference, never pricing
logic or evidence. Keep it when reusing the right; change it if the service meaning changes.
For combinations, label the individual rights: `setup-access` becomes `setup` and `access`;
`access-usage` becomes `access` and `unit`; `processing` becomes `unit` and `percentage`;
`bespoke` becomes `setup`, `time-services`, `retainer`, and `access`. Each milestone uses
`milestones`. A single building block keeps its own ID. Read back the reference with the
saved definition. Existing rights without a reference use an icon inferred from their pricing.
