# Pipelines and business context

Economico is one system of record for economic facts, attached context, and pipelines across sales,
purchasing, hiring and fundraising. Work from the job the founder needs done ("who do we chase this
week", "which deals close this quarter", "what does this client cost us"), never from a module name.

## The split: economic facts and working state

Sort every piece of information before recording it:

- An **economic fact** creates, changes or settles an obligation, or estimates one: who the
  business trades with, a deal in the pipeline, what it sells and at what price, an agreement and
  its lifecycle, what was delivered or used, what was billed, owed, paid, earned or incurred, who
  works for it and who owns it. Economic facts go in the books, with their evidence.
- **Context** explains a record without changing it: a call note, an email thread, a proposal, a
  page in the founder's notes. Attach it to the party, deal or contract it concerns, as
  [record evidence](record-evidence.md) and [conversation intake](conversation-intake.md) describe,
  so the next agent finds it from that record.
- **Working state** has no economic consequence of its own: tasks, project boards, support
  tickets, roadmap items, extra attributes the founder tracks about a customer. Keep it in the
  founder's own workspace (their repository, notes or existing tools), keyed by the exact
  `partyId`, `contractId` or opportunity id it concerns, so it joins back to the books; a
  [record link](record-links.md) in a note points straight at the record.

Read the books whenever you need their numbers, and build views over them; a copy kept elsewhere
goes stale the day it is made. When a piece of working state turns into an obligation (a ticket
becomes a paid change request, a timesheet is approved), record that fact here.

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
default `weightBps`, and optional `settlementKinds` (`contract`, `people_arrangement`, or both).
You can also call `pipelines.define` on a built-in key to give Sales, Purchasing or Fundraising your
own stage list. Each definition is a content-addressed versioned document
(`subjects {action: "list", type: "pipeline"}`), and every pursuit pins the pipeline version it
opened under; `pipelines.archive` stops new pursuits without disturbing open ones.

Estimates never post to the books; a pursuit is won only when the agreement or People arrangement
it was for is accepted. Fetch `customer-pipeline-to-contract` (`catalog {action: "recipe", name:
"customer-pipeline-to-contract"}`) to replay the contract path.

1. Record the prospect as a party as soon as there is a real conversation, with no `categories`:
   it is not a customer, vendor, investor or worker until an agreement or arrangement says so. The
   workspace party list groups uncategorised prospects by their open pipelines instead of under
   "Other". Declare `partyCategories` on the template or contract (`{"customer": ["customer"]}`,
   `{"vendor": ["vendor"]}`, `{"investor": ["investor"]}`) so accepting the agreement gives the
   party its role automatically; use `parties.update` (keeping any existing categories) only for an
   agreement recorded without it. A lost prospect keeps no category. Add the contact you will write
   to with `parties.add_contact`.
2. `opportunities.open` with a stable `opportunityKey` (the founder's or their CRM's deal id when
   one exists), `pipelineKey`, `kind` `new`, the `currency`, `expectedValue` in minor units only
   when the founder or a proposal states it (`null` keeps it unknown), optional `expectedCloseOn`,
   optional `stage` (defaults to the pipeline's first stage), optional `weightBps` override
   (defaults to the stage's weight), and optional `labels` (up to 20 free-form string key/value
   pairs such as `segment`, `owner`, `source` or `role`). The same party can be pursued in several
   pipelines at once.
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
   - **Fundraising close (one motion):** update `expectedValue` if the check changed, link and
     accept the contract (`partyCategories: {"investor": ["investor"]}`), and immediately record
     the commitment (`contracts.record` on `exercise`). See [fundraise](../../fundraise/SKILL.md)
     and `catalog {action: "recipe", name: "targets-fundraise"}`.
   - **Hiring close (one motion):** on a pipeline whose `settlementKinds` includes
     `people_arrangement`, record both `people.record_arrangement` and `people.set_membership` in
     one sitting (linking explicitly with `opportunities.link` when a person has multiple open
     pursuits). See [hiring](../../hiring/SKILL.md) and `catalog {action: "recipe", name:
     "targets-hiring-plan"}`.
5. A pursuit that dies without an agreement gets `opportunities.lose` with the founder's `reason`.
6. A renewal or expansion is its own opportunity with `kind` `renewal` or `expansion` and the
   `predecessorContractId` of the agreement it extends, opened before the renewal is recorded.
   A renewal of the same agreement is `contracts.renew` on that contract; link the renewed
   version (`contractId` and the renewal's `documentId`). New scope on a new agreement links the
   new contract. The predecessor's own acceptance never wins it.

Every party, contact, stage change, label update and customer email is an event against the plan's
allowance, like any other write: record a stage change when the pursuit moves, not on every
conversation.

## Reading pipelines

Read pursuits with `subjects {action: "list", type: "opportunity"}` (filterable by `pipeline`,
`party_id` and `status`), pipeline definitions with `subjects {action: "list", type: "pipeline"}`,
and bounded totals with `reports run pipeline` (filterable by `pipeline`, `stage`, `party_ids`,
and paged with `limit` and `after`): open known value, weighted value and a count of pursuits
whose value is unknown, by pipeline, stage and currency, from running totals that never scan
won or lost deals. Present weighted value as an estimate beside, never inside, revenue, bookings
or cash. In `report_sql`, query the `pipelines` and `opportunities` views, `open_opportunities`,
`pipeline_stage_totals`, `opportunity_labels` (for filtering and grouping by label) and
`opportunity_stage_history` (for stage conversion and time in stage).

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
  or turns it on; apart from accepting and paying for an Economico plan the founder asks for, it
  never moves money or signs anything.
