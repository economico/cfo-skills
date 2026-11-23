# Run the whole business on the books

A founder can run their go-to-market, delivery, product, receivables, payables, finance and people
work from one system of record instead of a CRM, an ERP and a billing tool that have to be kept
in step. Economico is that record; you build the rest of the business system around it. Work from
the job the founder needs done ("who do we chase this week", "which deals close this quarter",
"what does this client cost us"), never from a module name. A request to "set up a CRM" is a
request to record the pipeline below and build the views the founder asked for.

## The split: economic facts and working state

Sort every piece of information before recording it:

- An **economic fact** creates, changes or settles an obligation, or estimates one: who the
  business trades with, a deal in the pipeline, what it sells and at what price, an agreement and
  its lifecycle, what was delivered or used, what was billed, owed, paid, earned or incurred, who
  works for it and who owns it. Economic facts go in the books, with their evidence.
- **Context** explains a record without changing it: a call note, an email thread, a proposal, a
  page in the founder's notes. Attach it to the party, deal or contract it concerns, as
  [record evidence](record-evidence.md) describes, so the next agent finds it from that record.
- **Working state** has no economic consequence of its own: tasks, project boards, support
  tickets, roadmap items, extra attributes the founder tracks about a customer. Keep it in the
  founder's own workspace (their repository, notes or existing tools), keyed by the exact
  `partyId`, `contractId` or opportunity id it concerns, so it joins back to the books; a
  [record link](record-links.md) in a note points straight at the record.

Read the books whenever you need their numbers, and build views over them (below); a copy kept
elsewhere goes stale the day it is made. When a piece of working state turns into an obligation
(a ticket becomes a paid change request, a timesheet is approved), record that fact here.

## Jobs to be done and where they live

Each row names the records the job needs; the linked reference owns the procedure. Describe every
command with `catalog` before using it.

| Area | Job | Record or read |
|---|---|---|
| Go-to-market | Know who we trade with and whom to write to | `parties.create` (a prospect with no categories until an agreement, a signed counterparty with its role); `parties.add_contact` (role `billing` for whoever receives invoices); `parties.set_address` |
| Go-to-market | Track every deal from first conversation to signature, and the vendors and investors being courted | [The pipelines](#the-pipelines) |
| Go-to-market | Make an offer from the price book or a bespoke scope | A draft contract ([customer contracts](customer-contracts.md#prepare-a-bespoke-draft)); `contracts.offer` when it is sent, `contracts.accept` on signature |
| Go-to-market | Keep and grow accounts | Renewal and expansion opportunities; contract lifecycle terms, `contract_deadlines`, `contracts.renew`, `contracts.give_notice`; `contracts.change_quantity` for seats |
| Go-to-market | Know what each customer is worth | `party_contract_values`, `saas_metrics`, `ledgers {action: "footprint", party_id}` |
| Product | Define what we sell and its prices | Rights and plans ([modeling products](modeling-products.md), [patterns](business-model-patterns.md)); the `on_sale` price book |
| Product | Know what a unit costs to serve | `unit_economics`, `usage_cost`; vendor usage ([vendor usage](vendor-usage.md)) |
| Delivery | Know what we owe each client and prove it was delivered | The contract's rights; `delivery` and `usage` phases with their evidence (an accepted readout, an approved timesheet, a meter reading) |
| Delivery | Track prepaid credits, allowances and capacity | Capacity rights and lots; `activity_capacity` |
| Delivery | Know what a project costs | Name the subcontractor's vendor right for the client work it serves; a [custom report](verify.md#a-question-no-report-answers) sets revenue on that contract against those costs. Attribute only costs the evidence ties to the work |
| Receivables | Bill, send, chase and collect | [Customer contracts](customer-contracts.md): billing, [email the invoice](customer-contracts.md#email-the-invoice-and-reminders), reminders, `payments.record`, receipts; `aging` |
| Payables | Know what we owe vendors and record what we paid | [Vendor contracts](vendor-contracts.md), [explore email](explore-email.md), [payments](payments.md); `aging` with `direction` `payable`; `payment_routes` for how to pay |
| Payables | Cut tools nobody uses | [Vendor usage](vendor-usage.md); `vendor_utilization` |
| Finance | Statements, cash, runway, ownership | `income_statement`, `balance_sheet`, `default_alive`, `cap_table`; [show the founder](show-the-founder.md), [investor update](investor-update.md) |
| Planning | Hold a pipeline, person or the company to dated targets | `measures.define`, `targets.define`, `targets.label`, `targets.activate`, `targets.archive`, `measures`, `targets`; [targets](targets.md) |
| People | Who works here, on what terms, at what cost | [People and payroll](people-and-payroll.md); `people_costs` |

## The pipelines

An opportunity is one relationship the founder is pursuing — a prospective customer, vendor,
investor, partner, advisor or hire — with an estimated value (or `null`), an expected close date,
a stage and optional `labels`. Its `pipelineKey` says which pipeline it is in. Three pipelines
are built in and work with no setup:

| Built-in pipeline | `pipelineKey` | The party becomes | Won by accepting |
|---|---|---|---|
| Sales | `customer` | a customer | the customer contract |
| Purchasing (tools, subcontractors, suppliers being evaluated) | `vendor` | a vendor | the vendor contract |
| Fundraising | `financing` | an investor | the SAFE, note or round agreement |

Any business can define its own pipelines with `pipelines.define`: a stable `pipelineKey` (such as
`enterprise`, `partners`, `advisors` or `hiring`), a display `name`, its ordered `stages` with
default `weightBps`, and optional `settlementKinds` (`contract`,
`people_arrangement`, or both). You can also call `pipelines.define` on a built-in key to give
Sales, Purchasing or Fundraising your own stage list. Each definition is a content-addressed
versioned document (`subjects {action: "list", type: "pipeline"}`), and every pursuit pins the
pipeline version it opened under; `pipelines.archive` stops new pursuits without disturbing open
ones.

Estimates never post to the books; a pursuit is won only when the agreement or People arrangement
it was for is accepted. The [pipeline recipe](recipes/customer-pipeline-to-contract.json) replays
the contract path.

1. Record the prospect as a party as soon as there is a real conversation, with no `categories`:
   it is not a customer, vendor, investor or worker until an agreement or arrangement says so. The
   workspace party list groups uncategorised prospects by their open pipelines instead of under
   "Other". Declare `partyCategories` on the template or contract (`{"customer": ["customer"]}`,
   `{"vendor": ["vendor"]}`, `{"investor": ["investor"]}`) so accepting the agreement gives the
   party its role automatically; use `parties.update` (keeping any existing categories) only for an
   agreement recorded without it. A lost prospect keeps no category. Add the contact you will write
   to with `parties.add_contact`.
2. `opportunities.open` with a stable `opportunityKey` (the founder's or their CRM's deal id when
   one exists), `pipelineKey`, `kind` `new`, the
   `currency`, `expectedValue` in minor units only when the founder or a proposal states it
   (`null` keeps it unknown), optional `expectedCloseOn`, optional `stage` (defaults to the
   pipeline's first stage), optional `weightBps` override (defaults to the stage's weight), and
   optional `labels` (up to 20 free-form string key/value pairs such as `segment`, `owner`,
   `source` or `role`). The same party can be pursued in several pipelines at once.
3. Move the pursuit with `opportunities.stage`. Stages are validated against the pipeline version
   the pursuit pinned when it opened; omitting `weightBps` inherits the new stage's default weight,
   or pass `weightBps` to override it. Update or clear labels on an open pursuit with
   `opportunities.label` (pass `null` for a key to remove it, or `replace: true` to replace the
   whole map). Each change takes the opportunity's current `expectedRevision`; linked contract or
   People events also move it, so read it with `subjects {action: "get", type: "opportunity", id}`
   before a change.
4. Settle the pursuit when the agreement or arrangement is ready:
   - **Align the signed amount first:** when a customer contract, SAFE or round agreement is signed
     at a final price or investment amount that differs from the pursuit's initial `expectedValue`,
     call `opportunities.stage` with `expectedValue` set to the signed amount before accepting so
     won value in the pipeline equals what was signed.
   - **Customer and vendor contracts:** create the draft contract (with `partyCategories` on the
     template or contract so acceptance adds the role automatically) and `opportunities.link` it
     (`contractId`, `contractDocumentId`; a historical activity-model contract also needs the
     acceptance's `activityKey` and `occurrenceKey`). Accepting that contract marks the pursuit
     won. `contracts.cancel` of the draft or offer marks it lost; `contracts.discard` and
     `contracts.void` leave it open, so link the replacement contract.
   - **Fundraising close (one motion):** when a SAFE or round agreement is signed, do the whole
     close in one sitting — update `expectedValue` if the check changed, link and accept the
     contract (declaring `partyCategories: {"investor": ["investor"]}`, which wins the pursuit and
     gives the party the `investor` role), and immediately record the commitment (`contracts.record`
     on the `exercise` phase, which is what puts the SAFE on the cap table; acceptance alone
     records no obligation). Record the wire with `payments.record` when the bank shows it. See the
     [fundraise recipe](recipes/targets-fundraise.json).
   - **Hiring close (one motion):** on a pipeline whose `settlementKinds` includes
     `people_arrangement`, when a candidate accepts, record both `people.record_arrangement` (direct
     employee, EOR, PEO or contractor — automatically wins a single open unlinked pursuit for that
     person) and `people.set_membership` (makes the person count in "people in place") together in
     one sitting. When a person has several open pursuits, link the specific arrangement explicitly
     with `opportunities.link` (`arrangementId`). Employment history that ended before the pursuit
     opened never wins it; link it explicitly if it is the one. Retracting an arrangement
     (`through <= from`) reopens the linked pursuit, and recording the right arrangement then wins
     it. See the [hiring plan recipe](recipes/targets-hiring-plan.json).
5. A pursuit that dies without an agreement gets `opportunities.lose` with the founder's `reason`.
6. A renewal or expansion is its own opportunity with `kind` `renewal` or `expansion` and the
   `predecessorContractId` of the agreement it extends, opened before the renewal is recorded.
   A renewal of the same agreement is `contracts.renew` on that contract; link the renewed
   version (`contractId` and the renewal's `documentId`). New scope on a new agreement links the
   new contract. The predecessor's own acceptance never wins it.

Every party, contact, stage change, label update and customer email is an event against the plan's
allowance, like any other write: record a stage change when the pursuit moves, not on every
conversation.

Read pursuits with `subjects {action: "list", type: "opportunity"}` (filterable by `pipeline`,
`party_id` and `status`), pipeline definitions with `subjects {action: "list", type: "pipeline"}`,
and bounded totals with `reports run pipeline` (filterable by `pipeline`, `stage`, `party_ids`,
and paged with `limit` and `after`): open known value, weighted value and a count of pursuits
whose value is unknown, by pipeline, stage and currency, from running totals that never scan
won or lost deals. Present weighted value as an estimate beside, never inside, revenue, bookings
or cash. In `report_sql`, query the `pipelines` and `opportunities` views, `open_opportunities`,
`pipeline_stage_totals`, `opportunity_labels` (for filtering and grouping by label) and
`opportunity_stage_history` (for stage conversion and time in stage).

**A weighted pipeline the founder can open.** [The weighted pipeline report](reports/weighted-pipeline.json)
is a ready definition: open pursuits by pipeline and stage with value and weighted value, weighted
value by expected close month as a chart, and the open pursuits themselves, with a `pipeline`
parameter to show one pipeline (add the business's own custom pipeline keys to its options when
publishing). Check it with `report_format`, preview it, show the founder, and publish it with
`report_definitions.publish` when they agree, as [verify](verify.md#an-answer-the-founder-wants-to-keep)
describes. Its close-month chart draws sales, purchasing and fundraising as separate lines and
your own pipelines together as a fourth. Each stage rounds on its own, so a pipeline's stages match
`reports run pipeline` for that pipeline and currency to within one minor unit per stage.

## The views: the founder's data platform

The books are queryable. Build each view the founder works from as a custom report, following
[verify](verify.md#a-question-no-report-answers): `report_sql` against the documented views, then
`report_definitions.publish` with their agreement, pinned where they want it. A published view is
a saved query, so in the workspace it always shows the books as they are now. Share one with
`shared_views.create` and `mode` `live` when someone else needs the current numbers; the default
`frozen` link keeps the numbers of the day it was made. Either link opens without a login and
never expires unless you set `expiresAt`, so set one and say who will hold the link
([verify](verify.md#an-answer-the-founder-wants-to-keep)). Good first views, one per job the founder
named:

- **Pipeline**: open deals by stage, expected close and weighted value.
- **Collections**: open invoices by customer, due date and last reminder.
- **Renewals**: contracts by their next lifecycle date (`contract_deadlines`).
- **Delivery due**: obligations billed but not yet delivered, and delivered but not yet billed.
- **Spend**: costs by vendor and expense function, with unused seats.
- **Customer economics**: revenue, collections and directly attributed cost per customer.

## Business shapes

| Business | Pipeline | Agreement | Delivery evidence | Receivables | Payables |
|---|---|---|---|---|---|
| Consulting or agency | Deals per engagement | Retainer, hourly and fixed-fee rights per client | Approved timesheets, accepted deliverables | Arrears or on acceptance; reminders | Subcontractors named to client work |
| Self-serve SaaS | Rarely; plans sell themselves | Price book templates; Stripe checkout as acceptance | Elapsed service time | Card collections via Stripe | Hosting and tools |
| Usage or API business | Larger accounts only | Base plus metered rights | Meter readings as `usage` facts | Usage billed in arrears | Infrastructure priced by use |
| Credits or prepaid AI | Top-ups as expansion deals | Credit lots with expiry | Consumption draws from named lots | Billed in advance, deferred until used | Model inference |
| Annual enterprise SaaS | Every deal, renewals above all | Annual price with installment billing and lifecycle terms | Service windows, implementation acceptance | Installments with due dates | Audits and cloud |
| Grant-funded research | Grants and retainers as deals | Tranches tied to accepted deliverables | Acceptance of each deliverable | Tranches on acceptance | Research subcontracts |

## Limits to state plainly

Say which of these the founder's request touches, then keep that part in their workspace:

- Parties and contracts take the fields their schemas declare and nothing more; pursuits carry
  free-form `labels` (up to 20 string key/value pairs) for attributes like segment, owner or
  source. Keep extra attributes on parties and contracts in the workspace keyed by the record's id,
  never folded into a name or a contract term.
- Tasks, project plans, time entry, support tickets, inventory and purchase orders have no
  records; approved time and accepted work enter the books as delivery evidence.
- There are no budgets, approval workflows or bad-debt write-offs.
- Economico emails invoices, reminders and receipts on the business's behalf when the founder asks
  or turns it on; it never moves money or signs anything.
