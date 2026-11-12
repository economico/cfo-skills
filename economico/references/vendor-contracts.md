# Vendor contracts

Model what the company receives under each vendor agreement. A hosting service, API allowance
or domain registration is a right; an invoice, bank payment or personal-card charge is not a
separate right. Payment method must not change the vendor's agreement or price.

Economico itself is the one vendor never to model: its plan is already a contract in the books,
and its invoices arrive there by themselves. Do not record or import an Economico invoice; pay one
that is owed with `payments.record`.

## GitHub Actions: amount or meter for one charge

The `team` template includes `actions-amount` for an evidenced net charge and `actions-linux`
for priced Linux minutes. Agent decision: choose one representation for each evidenced charge,
including when detailed usage arrives after an amount was recorded. Before **either** recording,
read the vendor's existing contracts and occurrences for that account and billing month,
including historical standalone Actions contracts. Read `usage_estimates` for each matching
contract with `contract_id`, `from` and `until`, paging its rows and `unbilledEstimates`;
`activity_claims` alone misses observed-only usage. Match invoice/report identities, periods,
SKUs and any amount already recorded. Different rights, source IDs or contract IDs do not prove
separate charges. The kernel raises `invoice_overlap` when an amount bill and observed usage share a charge
source ID, or share invoice/report evidence and matching period totals with the bill contained
in one usage period
on the same expense account, vendor and monetary ledger. It checks both recording orders and
separate contracts. Different source IDs without shared evidence can still describe one charge;
check those and statements spanning several usage periods manually. Shared-report evidence
needs a retained nonzero cost contribution on that monetary ledger; a free measurement alone
does not establish duplication. `contracts.observe_usage` alone
posts the priced expense and accrual before any invoice or claim exists: a recorded 600-cent
amount followed by a 600-cent observation produces 1,200 cents of expense, in either order.

If `actions-amount` already covers the charge, retain the later usage report with
`documents.receive`; do not send those minutes to `contracts.observe_usage` or record them as
another economic usage, bill or recognition. Link the report using `contracts.reconcile_usage`:
mark the already recorded charge's line `handledElsewhere: true`, give `invoiceTotal`, set
`source.documentId` to the retained document, and include it as
`{ kind: "other", ref: <documentId> }` in `evidence.billing`.
If the meter already covers it, reconcile its first invoice with `contracts.reconcile_usage`
against the observed period and quantity, which clears the accrual rather than adding the
expense again. If that period is already billed, retain a duplicate receipt with a
`handledElsewhere` line; record an evidenced difference as a signed delta on a metered line
with `correction: true`. A +12-cent correction to the billed 600-cent period makes expense
612 cents. Never bill the amount right as well. For unresolved overlap, retain the documents
and raise it for the
business's attention before adding a second expense. Do not silently reverse a valid bill,
invent an allocation or infer minutes by dividing money by a rate. Distinct, evidenced charges
may use separate rights, including in one month; a shared month alone is not a refusal rule.

The amount is already net of included compute and consumes no compute-credit lot. Keep its
billing, usage and recognition on the same obligation with the same evidenced period and amount;
settle the existing claim. This procedure is agent guidance, not a kernel duplicate guard.

## Stripe fees deducted from the balance

Stripe Billing, Invoicing, Tax and Radar fees can settle from the Stripe balance. A fee report
and its balance transaction evidence the fee and deduction; a payout transfers remaining cash.

**Before billing**, match the evidence to existing bills, claims, deductions and imported
journals, across hand-recorded and adopted contracts, including those pinned to an older plan.
Match historical aliases such as `txn_1:tax` too: switching to bare `txn_1` can double-post the
same fee even on the same old contract. The system-company standalone-fee cron recognizes
these contract aliases (including the historical `stripe:balance_transaction:` prefix) before
journalling, and contract fee phases refuse existing standalone
journal receipts (including hashed system-company keys). Keep those journals and reconcile
the fee rather than billing it again. Between contracts, the kernel compares exact strings. The source-ID rule
below is guidance, not enforcement; `txn_1:tax` and `txn_1:tax_fee` can both post.

For a balance deduction, use **one obligation per `balance_transaction_id` (`txn_…`)**, copied
unchanged into billing `sourceFactId`. Bill the complete evidenced total once through
`facts.billedAmount`, keeping every itemized row, quantity and separately rounded amount as
evidence. Two rows on one `txn_…` are parts of that obligation, not two bills. Do not bill only
row one and then try row two, invent component suffixes, or silently drop a component. If the
breakdown spans products, activities or classifications, a business-selected aggregate fee right
owns the complete total; the row split remains evidence, not separate same-ID claims. Do not silently
classify it all as one product. Keep an existing aggregate bill where present. If no suitable
aggregate right or bill is available, record the cash deduction as unapplied vendor funds and
raise attention for classification before billing the complete total; never leave it part-billed.
`incurred_by` names the originating charge or invoice, not the containing balance transaction.

For **Radar Pro screened and abuse rows on one `txn_…`**, the business can explicitly select
`screened` as the aggregate activity when both rows share its expense classification. Bill their
complete total there with `facts.billedAmount`, using only the screened transaction count as
`quantity`. Keep the abuse event count and amount as separate evidence; do not add event and
transaction counts together. Usage and recognition belong only to the selected activity, with
that same screened count and period; settle its single billing claim. Do not bill `abuse` again
on the same ID. If this classification is unsuitable or no aggregate choice is made, keep an
existing aggregate bill or record unapplied vendor funds and raise attention for an appropriate
aggregate right before billing the complete total.

An **invoiced period bill keeps its vendor invoice/source identity**, for example
`stripe:invoice:<in_id>`. Match deductions to its existing claim; do not rename the invoice or
create `txn_…` bills for the same fees. A period-less `txn_…` fee whose effective date falls
within a bill on the same right posts with `invoice_overlap` attention, in either recording
order and even with different invoice IDs. The agent receives at most one pending item per
right and bill period, naming the other charge in `relatedOccurrenceId`; further fees do not
add review events while it is held. Match rows and coverage before billing. Read the returned
`usageReviewItems` and the contract's `attention.usageReview`, then record the business's decision
with `contracts.resolve_usage_review`. A period-less overlap remains pending after either
charge is reversed, since other fees may still overlap; resolve it explicitly after checking
the period. An existing expense stays in the books and is matched as `handledElsewhere` on invoice reconciliation;
an arbitrary imported journal is also outside the source-fact guard.

Fee granularity and cash cadence come from the evidence. For an unscheduled fee right, supply
`period: { from, through }` when the fee-report row has `activity_start_time` and
`activity_end_time`; omit coverage only when the report supplies no period. When several rows
share one deduction, use its explicit aggregate period if supplied, otherwise their common
period when all agree. Every `contracts.record` phase of the obligation, including billing,
usage and recognition, must carry that **identical period**. Omit it on all phases only when
coverage is absent from the evidence. Phase suffixes alone do not connect period-less service
to a period bill. Keep conflicting row windows as evidence and raise attention for an aggregate
period; do not invent an envelope. Distinct legitimate deductions with overlapping reported periods can also hit the existing coverage
refusal on one right. Preserve the dates and raise attention rather than omit coverage to bypass
it. This is another existing limit, not an instruction to refuse a business's fees.

Two distinct `txn_…` IDs with an **identical window** on one right can report
`economic phase already recorded with different facts`: the window identifies the same phase,
which does not prove the second deduction is a duplicate or already booked. Preserve both IDs
and windows, reconcile the prior bill and raise attention for this kernel limit. Do not rename
IDs or drop or shift dates. A zero-length or reversed window reports
`period through must follow from`; it is not valid coverage. Retain the raw timestamps and
seek an evidenced positive-length period. Do not invent an interval or omit supplied coverage
to force a bill through. While either refusal is unresolved, record an actual unbooked cash
deduction as unapplied vendor funds, allocating it only when a correct bill exists. Unapplied funds do not recognize the pending fee
expense; keep that missing expense visible in attention until coverage or classification is resolved.

Preserve each row's rounded fee when finding the deduction total. Do not price combined monthly volume to
replace reported fees. The generic standalone-fee page does not establish Billing or Radar's
account-specific cadence. Quoted subscriptions and included-volume overages retain their
agreed periods and allowances. Fractional-cent abuse events use the report's event count and
amount; half-up rounding is an illustrative assumption, not a promise that each event rounds
up. Tax on Stripe's fees is separate from tax calculated for the business's customers.

Record the total's billing before usage and recognition, so both follow its reported amount.
Usage and recognition use `<txn_id>:usage` and `<txn_id>:recognition` on the same activity,
with the identical period used for billing (or all omit it when the evidence supplies none).
Settle that billing claim with `payments.record` allocations from the Stripe balance, using
`<txn_id>:payment`; this adds no expense. `payments.record` has no period field: its allocation
links the period's claim, and its `effectiveAt` retains the actual cash date. Retry with the
original command idempotency key. If cash is deducted before a period bill exists, record it with `allocations: []`, then use `payments.allocate` after billing;
keep the actual deduction date and do not move or repeat the cash movement. These rights have
no usage meter, so `contracts.reconcile_usage` cannot price their lines.

Keep processing fees separate: `payments.process` already books and settles its provider fee.
The system company's separate standalone-fee reader also posts expenses outside the contract
source guard. Match its existing expenses and ownership of future fees before billing there;
merely adopting a product plan does not disable that reader. See the catalog's Stripe README
for the control-plane implementation, and [Stripe exploration](explore-stripe.md#cash-gross-or-net)
for gross receipts and net payouts.

Corrections do not rewrite an adopted contract. `contracts.amend` changes terms within its
pinned right definition; it cannot remove that definition's monthly billing schedule. To use
the corrected unscheduled definition, propose `contracts.adopt` with `replaces: [<old id>]`,
starting after existing billed coverage through the ordinary replacement workflow below.
It preserves the old bills and refuses a start inside their coverage. Before every subsequent
bill, still match historical aliases and invoices across both contracts. Do not reverse valid
history, omit a required period or rename a source fact to work around a refusal.

## Recognition follows the service evidence

When the source requires recognition on completed service, including completion at a service
month-end, declare `recognition: {"timing": "delivery"}` and record the evidenced completion.
Do not add a ratable recognition schedule because the bill recurs monthly or is paid in advance.
Use `ratable` only when the source establishes continuous service and elapsed-time recognition.
The source's completion rule takes priority over a recipe for a different service.

## Look up and adopt first

Preserve invoices, receipts and order forms with `documents.receive`. Vendor bills don't say
which product they paid for reliably: identify the actual plan, account, service periods and
seller from the vendor platform and agreement, not just the card descriptor or invoice total.
An import of a vendor's history follows steps 1 to 8 in order; steps 1 and 8 are what keep a
plan change inside one contract.

1. **Read every invoice the request covers before recording any.** List each one's date, plan,
   price, quantities and service period. A line for another plan or price of the same
   subscription (an upgrade, a downgrade, a credit for unused time) is a plan change: note its
   change date and each period it cuts across (one that starts before the change and ends after
   it). A vendor rename alone is not a plan change: an adopted Max 20x subscription receiving a
   $200 Ultra receipt continues on that same contract. The vendor moves existing subscribers
   automatically at the same price; record the receipt there, without a second adoption or
   contract. Ask the founder about every actual plan change, as described under "An invoice
   that does not match" below, before you record that contract's history; one question can name them all.
2. Look every vendor up in `vendor_catalog` (no `business` argument) before modelling it:
   `find` with exactly one of `q` (a name), `domain`, `sender` or `descriptor`; then `plans`
   with the vendor's `slug`, and `get` with a version's `cid`. If a historical receipt names a
   retired plan, read `plans` with `include: "not_offered"`, then its `history`. Adoption uses
   the contract's evidenced `effectiveAt` before the returned sale cutoff; a billing or meter
   start term is not the contract start. Never invent an earlier date to pass that gate.
   `plans` lists only each plan's latest version; for a start before its
   `effectiveFrom`, call `history` with the `slug`, the `plan` key and `effectiveOn` set to the
   start, and adopt the version it returns (none returned: see the earliest-later rule below).
   Match the plan and version to the evidence. The version is the one in force at the
   contract's start (the first invoice's service period): the latest dated on or before it,
   and of versions sharing that date, the most recently published (the highest revision,
   a correction; `history` returns it). Model by hand as below only when the catalog has no
   match or no fitting version, or as a fallback when the connection lists no `vendor_catalog`
   tool.
   Read prices, terms and seller-and-tax rows from that version only; a later version's row
   (another seller for your country from a later date) says nothing about an earlier invoice.
   When the invoice's seller is the seller of the row that version selects for the buyer,
   including its default row, the seller fits. Let adoption select the row rather than
   reading it yourself: adopt without `acceptance` and without `seller`, giving the billing
   country, and compare the returned `seller`'s legal name (`sellerAndTax.sellers` in the
   version) with the invoice's seller. If they agree, accept the draft; if not, discard it and
   model by hand. A version's date is when the catalog first evidenced it, not a proven change
   day: when no version is dated by the start, the one in force cannot be adopted (no
   seller-and-tax rows, or no template for a line the evidence needs), or the evidence includes
   a seat change the in-force version can't record, adopt the earliest later
   version whose price, seller and rights match the invoice and, for that seat change, whose
   right declares `economics.quantity`; say which version you selected and why. Check its required
   terms, seller-and-tax rows, agreement versions, usage meters, lifecycle and classification
   suggestions. Fill each required term from the evidence its `requiredTerms` entry names
   (GitHub's `actions-linux.meterStart` is the month of the first Actions usage line), not from
   the contract start; when adoption answers a missing term, read that entry again. A receipt
   line the plan cannot price yet (a usage total without the meter's quantity) does not make
   the plan wrong: adopt it, keep that line as an evidence gap or unmatched reconciliation line,
   and request the vendor's usage report.
   When the version supplies a recording-only amount right, use it for that line on the
   existing contract (GitHub's `team` template includes `actions-amount`). Historical versions
   may instead have a standalone `team-actions-record` template; check existing coverage before
   adopting it beside another contract. Never hand-model the whole vendor
   to fit one line, and never back a quantity out of a dollar total by dividing by the rate.
   Adopt with `seller` only from the row's candidates: when the invoice names a seller entity
   the matched row does not list (another country's entity), the plan does not fit; model that
   agreement by hand with the invoice's entity, never relabel the party to cover the mismatch.
3. Read existing vendor parties and contracts before writing. A business may have a hand-recorded
   subscription and a second measured-usage contract beside it. Show the founder which agreements
   overlap and offer **replace** versus **beside**. The business and its agent decide; similar
   vendor names do not prove a duplicate. Reuse the party only if it represents the actual seller
   entity: Anthropic Ireland and Anthropic, PBC are different counterparties.
4. Show the plan/version, seller, billing country, tax outcome, prices, start, quantities and
   per-right expense account/function. The catalog's classification is a suggestion: choose each
   right's account and function from [accounts](accounts.md) and what the business says the
   service is for. Ask only for evidence that is missing or choices that are ambiguous. Billing country is the address or payment country the vendor uses, defaulting to
   the business address; incorporation jurisdiction does not settle VAT. Registration in another
   country does not make this VAT recoverable. Supply registration evidence where the books have
   no tax regime, and create an evidenced tax code if adoption asks for one.
5. Describe `contracts.adopt`, then call it with `vendorPlan: { cid, template }`, the vendor
   `partyId` or `create: { partyId }`, terms (calendar dates as `YYYY-MM-DD`), ledger bindings
   (`{ "money": <the business's primaryLedgerId> }`), source documents and any per-right
   `classification` choices. Optional `acceptance: { sourceFactId, evidence: { acceptance } }`
   accepts in that same transaction; otherwise it creates a draft. Keep adoption as a draft until
   the contract start date and the billing-country seller row are confirmed from evidence. Read
   back the draft's seller before accepting or recording any bill. Prefer `vendor: { create }`,
   which names the party after the selected seller entity; a party you create yourself needs
   `legalName` set to the invoice's seller entity. Set the call's envelope `effective_at` to the
   contract's start (the first invoice's service start; for a usage line, the start of the
   usage period it bills), never today. For a replacement, use the first period after existing
   bills' coverage as described in step 6. The contract version dates from this start, and its
   acceptance, bills and later amendments cannot precede it. Adopted today, an invoice from last month can only be dated wrongly, and an
   amendment from the invoice's change date is refused. A separate real agreement can
   reuse the template with its own classification; each adoption creates another contract.
6. For replacement, pass **all** the hand-recorded contract IDs being superseded in `replaces`.
   The adopted contract starts at the envelope's `effective_at`; the replaced contracts end there
   through the scheduled-end path, with links both ways and their history intact. If obligations
   extend beyond that start, read the refusal and choose an evidenced later start. Bills already
   recorded stay on the old contract, with their claims, payments and recognition history; the
   replacement covers periods after them. Do not re-record an existing invoice on the replacement,
   rename its source fact or vendor invoice ID, or reverse valid bills to make replacement fit.
   A billed source fact or posted invoice line is refused anywhere in the replacement chain,
   in either direction. Invoice references ignore ASCII case and surrounding whitespace; ingestion's
   external invoice id also survives a seller-party change. An invoice covering both plans can be
   reconciled on both contracts with distinct vendor `lineKey` values and separate source facts;
   mark the other contract's lines `handledElsewhere`. An ordinary bill and reconciled usage can
   share the invoice when their activities differ or service periods do not overlap. Different
   seller parties conflict only when line amount and currency match as well; ordinary bills
   must also overlap in service coverage (unknown coverage counts as overlap). The invoice guard
   accepts disjoint ordinary periods: keep the invoice number in `invoiceReference` on
   `contracts.record`, and identify each evidenced recording with its own `sourceFactId`.
   Exact source facts remain reserved after reversal. On the same contract, different ordinary
   activities are distinct rights; reconciliation evaluates a previously recorded seat count and
   flags a different count for review. A shared-reference reconciliation that overlaps a partial
   period or quantity-adjustment bill posts with `invoice_overlap`, as does an equal-money
   ordinary bill on another right with overlapping coverage. A period-less fee on the same right
   also raises `invoice_overlap` when its effective date falls within a bill's coverage; both
   charges post. One pending item covers that right and bill period and names the other charge
   in `relatedOccurrenceId`. Read `attention.usageReview` and the result's `usageReviewItems`
   (`fee.usageReviewItems` for `payments.process`, `processing.fee.usageReviewItems` for
   `receipts.record`; attention from the receipt's own bill is in `economic.usageReviewItems`),
   decide whether the charges stand, and use `contracts.resolve_usage_review`.
   Do not refuse a legitimate prorated add-on. A period-less overlap still needs that explicit
   resolution after either charge is reversed; it can represent further fees in the period.
   Across replaced contracts the family matching rule applies. An evidenced
   reversal or contract void releases the affected ordinary bill's reservation, not its history.
   The guard retains
   the relationship after an old contract's notice is withdrawn; a voided replacement is unlinked.
   Interim statements do not block a start within their period. Reconcile the later invoice on
   the contract carrying the statement to clear its accrual; billing only the replacement leaves
   the old accrual open and raises `accrual_overlap` on the replacement when its line overlaps
   that open accrual. Do not automatically release it: the new plan may bill only its own part.
   If the business determines the replacement invoice supersedes the old statement accrual,
   reconcile amount `"0"` on the old contract for that exact usage period, under another
   `lineKey` such as `clear-old-accrual`, with its own source fact and the original invoice id
   in `source.reference`. Do not mark this interim-period clearing line as a correction. It
   clears the old accrual and its expense without another payable (for a 3107 old statement followed by a 3107 replacement invoice,
   payable remains 3107;
   accrual becomes 0; expense returns from 6214 to 3107). Then use
   `contracts.resolve_usage_review` on the replacement to record the decision. If some old-plan
   service is still owed, retain that accrual or bill the evidenced remainder instead.
   Omitting `replaces` records beside them and can flag a possible
   duplicate vendor contract. Never delete or reverse old books just to adopt a plan. Discarding
   or rejecting an unused replacement draft automatically withdraws ends still owned by that
   draft in the same transaction. Independent end notices and completed terminations remain;
   only their draft replacement link clears. Use `contracts.withdraw_notice` when the business
   explicitly wants to withdraw an old contract's end.
7. Read back the contract, source version, agreements, seller, classification and replacement
   links. Research a newer version before proposing a change. Amendments change supported
   terms and dates; they cannot install a newer activity definition or add a missing quantity
   rule. For unrecorded history with an evidenced seat change, use step 2's earliest-later
   exception and adopt from the original evidenced start. For an already-recorded contract
   without a quantity rule, propose a dated seat-term amendment and obtain the founder's
   approval as described below. Record the original period's bill first. A mid-cycle
   `contracts.amend` needs explicit `proration` for that right: use `basis: "stated"`,
   `creditAmount` and `chargeAmount` from the evidence, including an evidenced zero credit,
   and the invoice's `sourceFactId`. The amendment records the charge; do not bill it again.
   Recognize its service separately with `adjustment: { amendment: { cid } }`, using the
   returned `prorations[right].cid`. Amend the seat term at the next period boundary for an
   evidenced removal effective then. Neither amendment installs `economics.quantity`;
   subsequent seat changes still need approved amendments. Follow step 8 for history that
   blocks an amendment. A later replacement must satisfy step 6's coverage and period-boundary
   constraints. Never reverse valid bills to acquire a seat rule. A no-longer-sold plan's
   existing contracts carry on.
8. **Record the history in date order, amending first.** Bill each invoice on its date. Once
   the founder has agreed to a plan change, amend the contract from its change date before
   recording anything dated after that change: a later bill, or the recognition of any period
   it cuts across, the old plan's period included; until they agree, record nothing dated after
   it. An amendment cannot precede history already recorded. For a change on 06-30 to a 06-29
   to 07-29 period: bill 06-29, amend from 06-30 with its proration, then recognize the old
   period and record the later bills. Correct an input a refusal names. If the refusal names
   only recognitions recorded too early, undo them as "Correct a recorded service" says
   (`recognitionOnly: true`), retry the amendment, then re-record the recognition. If it names
   any other record (a later bill, a contract version or lifecycle event), stop work on that
   contract: quote the refusal in the final message and keep the invoices it affects as open
   gaps. Beyond that recognition correction, never terminate, reverse, re-adopt or hand-model
   the contract, and never record a refund or payment no receipt shows, to work around a refusal.

Settle each right's expense account, function and tax code before the adopt call, and pass any
choice that differs from the template's default as `classification` in that call: a correction
after adoption costs a contract. `contracts.amend` changes terms and dates, not classification. A
draft that never recorded activity can be discarded (`contracts.discard`) and adopted again;
an accepted contract is corrected by `contracts.adopt` with `replaces` naming it, which ends it
where the replacement starts and keeps its history. Never adopt a second contract without
`replaces` just to fix classification: template reuse creates another contract and leaves the
first one running. `possible_duplicate_vendor_contract` asks you to review that overlap, not to
ignore the old one.

When correcting after bills exist, read those bills and their service periods first. Use
`replaces`, start the correction after their coverage, and record only subsequent invoices on the
replacement. Keep the vendor invoice ID unchanged as the source identity (`sourceFactId`, and
`source.reference` for usage reconciliation); a new contract or command key is not a new invoice.
An invoice reconciled on the old contract with every line unmatched or handled elsewhere posted
no usage bill there: reconcile its unmatched lines on the correction under the same
`source.reference` with a new source fact. Lines handled elsewhere keep the bill recorded for them.

If adoption returns `seller_legal_name_mismatch`, read the contract's
`attention.sellerLegalNameMismatch`: it names the party's legal name and the selected catalog
seller's legal name. Adoption records the business's choice but does not establish that those
entities are the same. Resolve the seller from the evidence and propose the correction. The
warning clears once the party's stated legal name agrees with the seller; never rename the party
merely to hide it. Adoption also binds that catalog seller to the party,
so later adoptions of the seller reuse it: if the seller belongs to another party, correct the
binding with `parties.rebind_vendor_identity` (frozen contracts keep their party). When unsure
which entity bills you, adopt without `acceptance` first so a wrong draft can still be discarded.

An invoice that does not match a recorded contract (another plan, price or seat count) is a
proposed amendment, not a licence to reverse, terminate or start contracts on your own. Ask the
founder, in one question, whether to amend that contract from the invoice's date (name the change
and the date, recommended option first); record the invoice only after they agree. The invoice
shows what the vendor billed, not that the business accepted new terms, so this question is
asked even when the founder says to ask only what the evidence can't settle. With the
founder away, record nothing for it and put the proposal in the final message. If the amendment
is refused, follow step 8: correct only early recognition it names; otherwise quote the
refusal and keep the invoice as an open gap.

For a mid-cycle change of the same subscription (for example a higher access plan), first ask
the founder to approve the amendment as above; an invoice showing the change is not that
approval. Once they agree, keep **one contract with a dated `contracts.amend`**. Read its
current document and right terms, record the original period's bill, then amend the plan/price
terms from the change instant before recording anything dated after it, including the old
period's recognition. If recognition recorded too early blocks the amendment, use the recovery
path below before retrying it; any other blocking record stops work as step 8 says.
Supply `proration: { "<right key>": { "basis": "actual_days", "sourceFactId": "<vendor invoice ID>:plan-change",
"creditAmount": "<unused old plan credit>", "chargeAmount": "<remaining new plan charge>" } }`.
Use the vendor's actual line amounts, on the right's price basis before reductions and tax,
and retain the invoice as acceptance evidence and a source document. Choose the evidenced basis:
`actual_days`, `elapsed`, `days_30` or `days_365`; use `stated` with both line amounts when the
vendor's basis is unknown. A calculated basis flags any difference while accepting the invoice's
figures. A fixed-day basis outside its matching billing cycle requires both vendor amounts and
raises `proration_basis_differs` review attention. If amounts are unknown, use `actual_days` or
obtain both figures; do not guess the annual price's unused portion with `days_30`.
This amendment records both lines atomically; do not bill them again. It retains the
original cycle anchor unless the vendor starts a new cycle: then add `resetBillingAnchor: true`
to the right's proration. That explicitly resets open-ended calendars on the change date and
charges the full new period; subsequent bills use that new anchor. Do not terminate,
re-adopt or create another contract to represent this plan change. A seat/count change instead
uses `contracts.change_quantity` under the right's quantity policy.

Keep the returned `prorations[right].cid`. Recognize the old period through its ordinary schedule,
which deducts the credited unused service, and the replacement with `contracts.record`,
`phase: "recognition"`, `adjustment: { "amendment": { "cid": "<returned cid>" } }`, and service
coverage clipped to the replacement's frozen service period (the new cycle when its anchor resets).
The credit is applied to the
replacement invoice in that same command, leaving its net payable (any excess remains receivable).
When the vendor refunds separately and charges the full replacement price, set
`creditDisposition: "refund"` in that right's `proration`, then record the evidenced refund and
full charge against their separate claims. The disposition is ignored for zero credit.
Otherwise read the net claim and record its evidenced payment; do not invent a cash refund for
an invoice credit. If the result names `reversedBillOccurrenceId`, the preceding replacement has
no live bill and predicts zero credit; record its corrected bill before requesting a positive
unused credit. This raises `amendment_bill_reversed` review attention. A corrected bill recorded
after a later change replaced its service remains posted as billed and raises
`amendment_service_overlap`, naming the replacing amendment, coverage, overlapping amount and
full billed consideration. Check both bills and credits with the vendor and retain the evidence.
Record the business's decision and reason through `contracts.resolve_usage_review`'s `items`.
Acknowledgement does not correct money; retain an open gap for an incorrect frozen credit until
an evidenced correction path is available. Fully discounted, zero-consideration bills predict
zero credit. `contracts.void` is refused once a mid-period plan change exists.
A reviewed rounding difference can be acknowledged with `contracts.resolve_usage_review`,
`priceDifferences: [{ "activityKey": "<right>", "basis": "amendment_credit", "latest": "<check latest>", "billed": "<check billed>", "bills": 1, "standing": [{ "effectiveAt": "<check latest>", "computed": "<computed>", "billed": "<check billed>", "facts": {} }] }]`,
a decision and a reason. Copy `latest`, `billed`, the actual `bills` count and the entire
`standing` list from the check, including occurrence IDs on charge bills;
correcting an older retained bill invalidates a cached acknowledgement too. This acknowledges the selected check's unresolved amounts, changes no
money and adds no matching tolerance. Resolve `proration_basis_differs` through the same command's
ordinary `items` selection. If the amendment refuses an
already-recognized or unsupported service, quote the reason and retain an open gap.

For a backdated contract, establish the plan and governing agreements in force at its start
(step 2 above). A current catalog version is not evidence of past terms. If a plan changed substantially,
research the vendor archive, version control or archived pages; model the historical agreement
by hand when the catalog cannot express those evidenced terms. A negotiated price, seat count
or start date is not a new shape: adopt the plan with `contracts.adopt` and put the agreed
numbers in its `terms` (they override the plan's defaults). For different commitment, renewal,
notice or cancellation terms, use `contracts.adopt` with a complete evidenced `lifecycle`;
do not derive a template for a lifecycle difference. Do not supply `lifecycle.changes`:
record governing change rules through `contracts.amend` with `agreements.changes`.
If a term has already ended at the adoption's effective day, supply a complete `lifecycle` for
the current term, or set `effective_at` at the commitment start when recording the historical
agreement. `lifecycleBinding` requires unpublished `/2` defaults and is unavailable for a
published plan.

Read the adoption receipt's flags. For `lifecycle_terms_differ`, compare the recorded lifecycle
with the signed order and retain the evidence for the agreed differences. Correct mistaken
renewal, notice or cancellation rules through an evidenced amendment. Amendment cannot rewrite
an existing term's start or end: retain an open correction gap for wrong bound dates and check
the existing discard/void eligibility before proposing a replacement. Do not use
`contracts.renew` as a date correction; it records an actual next term. For `lifecycle_terms_unknown`, retain an open gap, obtain
the actual commitment dates and sourced rules, then use `contracts.amend` to record the
complete lifecycle. Never infer them from acceptance or monthly billing. Both flags exist only
in the receipt, so keep them in the review notes; reading the contract does not return a durable
marker for either flag. Published catalog plans carry no reusable defaults while `/2`
publication is refused.

Derive only when the agreement's
shape differs (different rights or billing/recognition cadence): `templates.create` with
`vendorPlan: { cid, template, relation: "derived" }`, keeping the plan's activity keys. With no suitable vendor plan,
or when adoption answers `seller_and_tax_missing` (the version names no seller), model by hand
below. For `seller_and_tax_row_missing`, inspect the returned buyer facts, including state and
registration facts. If the invoice shows different facts, pass `billingCountry` or `buyerTax`
with evidence; otherwise model the agreement by hand from the invoice's seller and tax evidence.
Another version may cover those evidenced facts. Nothing about the business is published to the
vendor catalog automatically.

## Model an agreement by hand

Use one contract per real agreement, not one per vendor or receipt. Reuse a template when its
rights and timing fit. Distinct agreements remain separate; several rights can share an expense
classification. Keep the vendor's identity on the party and retain the evidence for every term.

Define activities with `model: "priced-rights-v1"`, their contractual names, terms/facts and
explicit price treatment. Bind them in a template with the same model. A received service uses
`economics.kind: "service"`, `direction: "receive"`, independent `billing` and `recognition`,
and the expense account/function in `classification`. Choose those with [accounts](accounts.md).
Create the contract with its actual start date, vendor party and source documents, then use
`contracts.accept` with the current document ID, expected draft status and acceptance evidence.
Do not create an `accept` activity or add the founder as a vendor party to encode who paid.

For a recurring vendor plan, declare `economics.billing.schedule` from the agreement's
evidenced cadence: `everyMonths: 1` for monthly or `everyMonths: 12` for annual billing,
with `startTerm` bound to the actual subscription start, `endTerm` only when an end date is
known, and `timing: "start"` for advance billing or `timing: "end"` for arrears. Set
`offsetDays` from the billing terms. Keep service recognition independent of billing; use
elapsed-time recognition only when the agreement establishes continuous service. Explicit
recognition on completed service uses `recognition.timing: "delivery"`, including completion
at a month-end; monthly billing alone does not authorize `ratable` recognition. See the
[monthly subscription recipe](recipes/vendor-monthly-subscription.json) for an open-ended,
USD 29 monthly plan. Vendor cadence belongs in the billing schedule; `recurringValue` is
only for customer service rights and is refused on vendor rights. If the cadence or anchor
is unknown, keep it as a gap and request the terms in the final message; one receipt is not a schedule.
A month label on an invoice does not establish a calendar-month billing anchor. Preserve the
actual invoice and payment dates; when no evidenced schedule fits them, use unscheduled billing
for those observed facts and keep future cadence unresolved. An inferred future period must not
make an already-issued, paid invoice disappear from the historical books.

## Record the economics, then the funding

Keep the supplied invoice ID exactly in ordinary billing's `invoiceReference` or usage
reconciliation's `source.reference`, and the supplied payment ID in the payment's `sourceFactId`.
A single billing can use its invoice ID as `sourceFactId`; when one invoice has distinct
recordings, each names its own evidenced source fact while retaining the same invoice reference.
Do not invent a new invoice number or line key to pass the guard.

When the vendor reissues a bill against a replacement plan, set `replacesBillId` on the new
`contracts.record` billing or invoice `contracts.reconcile_usage`: the old ordinary
`occurrenceId` or usage `reconciliationId`. Give the new recording a fresh source fact and
billing evidence, dated at or after the original phases. This reverses the old bill and all its
recorded lines, retires the usage estimates it cleared, and records the replacement in one
transaction. The replacement takes another right or a replacement contract; same-right
corrections use ordinary reversal or usage correction lines. The vendor party and currency
must match and the contracts must belong to the same replacement family. Without this explicit
reference, a possible equal-money overlap under a renamed right posts with `invoice_overlap`;
read the attention item and decide whether both charges stand. If one replaces the other, use
this explicit replacement, rather than resolving attention alone and leaving both payables.

For an ordinary bill, this also reverses that obligation's usage and recognition. The old expense
becomes zero until recognition is recorded on the new bill. Inspect
`result.replacement.reversedPhases`, then record the replacement's evidenced service phases.
Source retries retain that result, including after an event-only restore. A plan that would reverse a sibling bill on the same
obligation is refused; correct the whole obligation and all its bills explicitly.
`receipts.record` refuses replacement: use `contracts.record`, then record any new money moved
with `payments.record`. Do not void the replacement contract to remove its bill.

A payment stays in history and moves no cash: its settled value becomes a vendor credit and
is applied to the replacement bill. A smaller replacement leaves excess credit as a receivable
from the vendor; a larger one leaves only the difference payable. If its payment bounces,
`payments.reverse` removes that paid value from the replacement, keeping the old bill cancelled.
Unused excess credit is consumed first, and successive replacements lead to the live bill.
If the payment was matched later with `payments.allocate`, unapply that match with
`payments.unallocate` before reversing the payment; unallocation reopens the live replacement.
A payment or match dated after the replacement must be undone first: use `payments.unallocate`
for a later match, or `payments.reverse` for an initial allocation. Fully undone later legs
do not block replacement. Record or match the payment against the new bill at its evidenced date
after the replacement succeeds.
A foreign bill paid in the functional currency cannot be replaced at another booking rate:
reverse that payment first, replace the bill, then record the evidenced payment against the
replacement. A payment from an account in the bill's own currency remains supported.
Usage periods changed after
the original bill, or invoices that moved vendor credit lots, need their dependencies corrected
first. Replacement requires schema 34. Ordinary reversal still uses `contracts.reverse_economics`.

Identify the right and the obligation or service period. Record delivery/usage from source
facts; record recognition when the service is consumed and billing when invoiced. Never date
a phase after today: a service period that has not ended is not consumed yet and stays
prepaid (a recipe's period-end recognition waits until the period has ended), and a future-dated record blocks every later amendment or correction dated before it
(a plan change on the next invoice could not be recorded). For
`contracts.record`, supply that phase's date as `input.effectiveAt`; an envelope `effective_at`
does not supply this required input field. Follow the validated recipe's phase inputs and read
back the resulting statements. A bill can precede or follow recognition. An annual payment does not prove twelve months of service:
retain prepaid value and recognize the evidenced portions under the right's ratable schedule.

For a quantity right billed on each change (`economics.quantity.billedOn.on: "change"`),
a seat-addition charge is a separate obligation. For GitHub Team, supply the evidenced
prorated amount **before tax** in `contracts.change_quantity.bill.invoicedAmount`, not a taxed
receipt's gross total, and state the receipt's tax separately in `bill.taxAmount` (minor units,
including zero). A stated supplier tax figure is kept even when it differs from the code's
rate; inspect the command's `usageReviewItems` (`bill.usageReviewItems` on a quantity change)
and `attention.usageReview` for `tax_differs`. A received exclusive-tax seat-change charge with
a stated invoice amount, positive supplier tax and no stated tax raises `tax_amount_unconfirmed`
when it matches the calculated price plus tax within one minor unit. GitHub Team uses stated
proration without an independent price prediction, so a stated amount raises the warning until
tax is supplied (orchestrating agent decision). Plan-change charges are excluded. Check the receipt's before-tax amount and tax
before treating the payable as reconciled. Adoption leaves GitHub's tax outcome unknown; establish the tax code
and treatment from the receipt before billing. Retain the returned `changeId` and recognize
that charge with `contracts.record`, `phase: "recognition"` and
`adjustment: { "changeId": "qch_..." }`, using service evidence and a completed
recognition-schedule window clipped to the adjustment's coverage. Ordinary period recognition
does not consume the addition's prepaid balance. Do not bill it again when
`change_quantity.bill` already recorded its bill. For rights billed at a true-up, such as
GitLab and Notion, recognize the true-up window; recognition by `changeId` is refused.

A paid receipt can use `receipts.record` for one new billing phase and its full same-time
payment. Otherwise record billing, read `activity_claims`, then use `payments.record` with the
returned claim identity. See [payments](payments.md) for account ownership, reimbursements,
unapplied funds, transfers, provider fees and corrections.

A vendor priced in another currency than the business's functional one keeps the bill's
payable in that currency, and its expense is booked in the functional currency at the rate of
the date the obligation is first booked (the bill, or the first recognition of an accrued
service). The rate comes from the shared rate store; when a command answers `missing_rates`,
ask for the rate the evidence states and pass it as `exchangeRate`, never a guessed one. See
[payments](payments.md) for the gain or loss realized when the bill is paid.

| Evidence-backed situation | Executable example |
|---|---|
| Hosting bill, paid later by bank | [vendor bill](recipes/vendor-bill-paid-from-bank.json) |
| Delivered API service, receipt already paid by bank | [bank receipt](recipes/vendor-receipt-paid-from-bank.json) |
| Delivered service charged to company card, issuer paid later | [company-card receipt](recipes/vendor-receipt-company-card.json) |
| Vendor service paid personally, owner reimbursed later | [founder payment](recipes/founder-paid-expense.json) |
| EUR bill charged to the founder's own card, reimbursed in USD | [founder card, foreign bill](recipes/founder-card-foreign-bill.json) |
| EUR bill on the founder's own card, credited and refunded by the vendor | [founder card, foreign refund](recipes/founder-card-foreign-refund.json) |

These examples freeze one known charge from their synthetic source. Do not infer a perpetual
price from a one-off receipt. Bind the real agreement's rate/tier/fixed terms, or preserve an
unknown price and request the missing evidence. The same service template can be paid through
any supported account without adding payment activities.

Register personal accounts with `ownerPartyId`; register company cards with their issuer's
`providerPartyId`. A founder payment initially creates an owner claim. Reimbursement settles
that claim with company cash and no expense. Conversion to capital requires separate evidence
and an ownership right, never a silent choice or founder-expenses contract.

For PEO/EOR invoices, ordinary payroll or direct founder/team-contractor costs, first read
[People and provider payroll](people-and-payroll.md). A provider's service fee is one right;
worker cost, creditors and funding must not be collapsed into that fee.

## Fetch and record detailed vendor usage

How often to record, seat and tool use, and the subscription review are in [vendor usage](vendor-usage.md).

Backfill detailed past usage from the vendor platform, including zero-priced or included usage,
not from invoice totals. Vendor data expires: preserve the report with `documents.receive`
as soon as it is fetched. Convert the quantity to the meter's unit, then give each observation the `unit` of the quantity you send; a unit that differs from the meter's is recorded and flagged for review. Before observing, apply the GitHub Actions amount-versus-meter check
above: when an amount already records that charge, retain and link the report as
`handledElsewhere` rather than observe it. Otherwise record the fetched usage promptly.
For a shared account, state the
organization, workspace, project, user or tag filter and period that isolate this business;
record that filter with the report. Never treat the whole shared account as this business's use.

Use **one convention**: `contracts.observe_usage` on the adopted meter. Each observation names
`producer`, stable vendor `sourceId`, `revision`, `effectiveAt`, `bucket`, integer `quantity`
and evidence; `observedAmount` is the vendor's cost in minor units scaled by the meter's
`amountDecimals`, when its price treatment requires it. Prefer the vendor's bucket or event ID
as `sourceId`; if none exists, use a stable composite of account/filter, bucket boundaries and
model/meter. Never include API secrets. Keep the same producer and source ID when fetching again;
raise the revision only for an evidenced restatement. The observation identity hashes producer,
a zero byte and source ID, so refetching is safe even under a new command idempotency key.
Use a stable `usage-measure:<vendor>:<account>:<meter>:<batch>:<revision>` command key.

Read the adopted meter's quantity fact and its stated `unit` before supplying `quantity`.
Quantities are integers with no implicit decimal scale; `amountDecimals` scales money only.
Preserve the vendor's source quantity and unit in the report evidence, and convert explicitly:
7 GB is `"7000"` MB, 5.5 GB is `"5500"` MB; 1 GiB is 1024 MiB. Convert neither GB to GiB nor an
ambiguous unit without evidence. Check the proposed integer against that source conversion
before recording. Flag an implausibly small quantity deterministically when the units prove
the mismatch: `7` MB for an evidenced 7 GB is a scale error, and `7` nanogigabytes on an older
contract is not 7 GB. Use a corrected human-scale plan for a new adoption; an existing contract
needs an evidenced amendment or the explicitly converted old unit, never a silent unit change.
If the source is finer than the meter supports, request its rounding rule or state the precision
gap; don't truncate or invent subunits. If evidence supplies no source unit or conversion,
flag that uncertainty instead of guessing. A small quantity or $0 inside an allowance alone
does not establish a scale error; record legitimate included and zero usage.

This replaces an earlier convention some books still show: do not create another
measured-usage contract or write `contracts.record` with occurrence keys
`usage:<from>..<until>[:<model>]`. Vendor usage IDs become observation `sourceId`, not invented
`sourceFactId` values. Invoice reconciliation uses `contracts.reconcile_usage`, with the exact
vendor invoice ID in `source.reference`. Use that ID as `sourceFactId` for a single recording;
when one invoice spans contracts or includes an ordinary bill, give each reconciliation its own
source fact while keeping the same original `source.reference`. Preserve the vendor's distinct
line keys and mark lines recorded on another contract `handledElsewhere`. Read the returned
unestimated/unmatched lines and differences.

For named seats, use the dated People roster and scoped holder mappings in
[People and provider payroll](people-and-payroll.md#5-verify-and-connect-vendor-usage).
Read `people` `identities` using the vendor, account, holder and review date. Record a supported
match with `people.link_identity`; retain the original observation's opaque `holder` unchanged.
Active/former membership is separate from seat assignment, billed quantity and observed use.
Keep external, shared, agent and unresolved holders distinct. Flag a former person's paid seat
for review without treating the difference as authorization to cancel or remove access.

Attribute observations to the evidenced customer, holder, project and team. A team carries its
stated expense `function`: developer seats can go to research and development, sales seats to
sales and marketing, and product-serving use to cost of revenue. The meter and reconciliation
allocate by those shares; do not create duplicate vendor contracts to split one expense.
If the hand-recorded right lacks a usage meter, adopt or derive the evidenced right before
backfill, choosing replacement or beside explicitly and preventing overlapping charges.
An invoice reconciliation taxes usage with the right's tax code and treatment, so give
`invoiceTotal` including that tax. When the vendor charges a different tax figure, give each
line's `taxAmount` (including zero) rather than repeating tax as a separate line. For tax rounded
on the invoice total, allocate the stated tax and residual cent across the commercial lines.
The invoice's figure is recorded; `tax_differs` attention retains it and the derived figure for
review. Reverse-charge and imported-services tax is the business's own computation: the vendor's
stated figure (usually zero) is kept as evidence and never replaces it. After a final credit-paid
statement, record later tax with an `amount: "0"` correction and its `taxAmount` (for self-assessed
tax, the figure the business assessed); never send one for a period whose tax is already recorded. Statements post no tax. Tax-inclusive credit pools and tax-paying (`includes_tax`) lots on taxed rights are
refused: explain that and keep the evidence; never omit an evidenced tax code to make adoption
succeed. When adoption asks for `buyerTax.registrationProvided`, answer from evidence whether
the business gave this vendor its tax registration; holding one is not enough.

A granted startup credit, such as a YC cloud grant, is a **granted credit lot**, with its face,
coverage, validity, expiry and eligible rights from the grant evidence. It is not a purchased
prepayment, bank deposit or negative invoice, and it never replaces the vendor contract. Adoption
refuses to replace a contract that still holds credit, naming the balance; pass `retireCredit: true`
only when the vendor confirms that balance is gone. Amounts and dates vary by batch, so take
them from the founder's own grant email or the vendor's credits page, not from a program's public
page.

1. Preserve the grant evidence with `documents.receive`, and read the amount, grant date, expiry and
   what it covers. Ask for a screenshot of the vendor's credits page when the email states no expiry.
2. Adopt the vendor's plan if there is no contract yet. Then call `contracts.add_granted_credit` with
   the rights it can pay. Leave out what the grant excludes, such as support or marketplace charges;
   give `coverageBps` when it pays only a share of each charge. Metered and billed rights take one
   call each, because they draw credit at different times; record any bill for cost already
   recognized first.
3. Record the grant with `contracts.record` on the new right: phase `exercise`, facts `face` and
   `expiresAt`, with `effectiveAt` the grant date. A later tranche is another `exercise` when it
   arrives; never record one that has not been released.
4. Record usage as usual. Usage before the grant date stays an expense. When a grant is recorded
   after its usage, reconcile the vendor's statement with its `credit` on each line, which draws the
   lot and clears the accrual.
5. Reconcile consumption with the vendor's statements, and tell the founder the balance left, its
   expiry, and what each service costs without the credit. Record expiry when it comes (the `expire`
   consequence on the right), which adds no expense.

Keep paid and granted lots distinct. Where the plan already has a pool, its draw order stands. A
monthly included credit is recorded the same way, once a month. If a grant covers a right whose
price has a tax basis, or pays tax on a taxed right, the command explains the refusal: keep the
evidence and tell the founder.

## Finish the receipt review

Read back the recorded contracts, expense totals, vendor claims and owner/issuer balances.
In the final response, summarize what was recorded and what was skipped, naming each reviewed
zero-charge/free-tier notice and why it produces no entry. Also explain excluded customer mail
and duplicates when found. Use only notices actually read; do not invent a skipped item or
claim the mailbox is complete after a narrow search.

## Tax and missing evidence

Use the applicable registered tax code in the right's classification. Supported billing freezes
output, recoverable input, reverse-charge or imported-service tax outcomes; payment settles the
gross claim. For priced service rights, declare `classification.taxTreatment` as `inclusive`
when the evidence states a tax-inclusive total, or `exclusive` for a before-tax price. With a
`non_recoverable` code, this makes tax part of the service cost, including prepayments and
accruals; recoverable input tax stays separate. Recognition uses the first evidenced tax basis,
so a later changed rate requires correction rather than silently changing the cost. Explicit
treatment currently excludes recurring-value metrics, capacity and exercise consequences;
self-assessed tax uses the full supplier price as its base even when labelled inclusive.
Adoption accepts that combination and flags `self_assessed_inclusive_price` for review. A minimum may sit on an `exclusive` right:
its net shortfall is taxed as its own line item, and tax you cannot reclaim (a
`non_recoverable` code) is added to the shortfall's cost instead. State it on an inclusive
right and authoring refuses it. Do not add hand-written
tax posting activities. A taxed cancellation refund must name the original bill through the
consequence's `taxClaimFact` (a typed claim fact containing claim ID and component). State the
unearned service consideration, net of recoverable tax or including non-recoverable tax; the
kernel derives the tax adjustment from that bill and creates a separate refund claim. Keep
the original payment and settle the refund independently. Earned service, tax-only bills and
minimum-closed periods need a different explicit correction; do not approximate those refunds.

If the receipt does not establish who paid, preserve it and the payable rather than guessing an
account. If evidence does not establish consumption, do not recognize the expense just because
money moved. Verify claims, expense and account/owner balances after recording.

The [annual prepayment recipe](recipes/vendor-annual-prepay.json) uses one insurance
coverage right, separate billing and bank payment, and explicit equal monthly recognition.


## Correct a recorded service

If an earlier plan amendment is blocked by recognition you already entered, keep the same
contract. The refusal names blocking events, occurrence IDs, right keys, dates and the correction
command (up to ten records, newest first; retry to reveal more). For each blocking recognition,
in that order, call
`contracts.reverse_economics` with `contractId`, `activityKey`, its `occurrenceId`,
`recognitionOnly: true`, `effectiveAt` equal to the original recognition instant, a fresh
`sourceFactId` and correction `evidence`. This appends the inverse recognition and retains the
original bill, its payments and other service windows. Do not reverse the paid bill merely to
undo recognition. Retry the founder-approved `contracts.amend` at the plan-change date with the
invoice's proration lines. Re-record old-plan recognition and amendment recognition separately
with fresh source facts at or after the reversal date; use the amendment's returned `cid` for
the replacement. Check old billed value minus the unused credit, replacement recognition, the
net invoice payable and prepaid balance to the cent. All original and correcting records remain
in the audit trail. Corrected economics and reversal events do not themselves block amendment;
governing contract versions and lifecycle decisions still constrain its date.

For an independent service obligation, use `contracts.reverse_economics` with the original
occurrence ID and correction evidence, then record the corrected phases with new source facts.
This reverses the whole obligation at the correction date; earlier reports retain the original
activity. Undo dependent payment allocations or payments first, using the payment correction
commands and their evidence requirements. Check that the original claim is cancelled and that
replacement billing, recognition and quantities reconcile. Do not use this service path for
linked lifecycle, capacity, minimum or ownership consequences, or for prior-period restatement.

## Void a contract set up in error

When a whole contract was set up wrongly (the wrong rights, terms or usage metrics) and its
recorded history is wrong because of it, void it and rebuild it instead of correcting phase by
phase. If the history is right and only something is missing, such as a new right or metric
from a date, amend the contract (`contracts.amend`) instead.

1. Run `contracts.void` with `contractId`, `expectedDocumentId`, a `reason`, correction
   `evidence` and `preview: true`. Show the founder what it reverses (`reversedEvents`,
   `occurrenceIds`) and which payments it returns to unapplied funds (`releases`), and get their
   approval. It writes nothing.
2. Run it again without `preview`, with the preview's `planDigest` as `expectedPlanDigest`. A
   conflict means the contract changed since the preview: preview again.
3. A historical contract's own payment recordings (a founder paying personally, a card, a receipt
   collected) are taken back too and listed as `recordedPayments`: record each again, on the
   replacement or as an independent payment.
4. Create and accept the corrected contract, and record its facts with the same source fact IDs
   (new idempotency keys). Match each released payment to the new claims with
   `payments.allocate`, from the payment's unapplied funds.
5. Check that the voided contract reads `voided`, its claims `cancelled`, and that the
   replacement's claims, revenue or expense and the party's unapplied funds reconcile.

The void takes everything back at each fact's own date, so earlier periods read as if the
contract never existed. It refuses capacity, ownership, minimum and lifecycle consequences,
reversals that owe refunds and cross-currency settlements, naming what to undo first.

## Set the account bills are paid from

Record the paying account as an agreement term, not a fact on each bill. Supply
`paymentTerms: { payingAccountId: "mercury" }` on `contracts.create`, using an existing registered
financial account. To change it, use `contracts.amend` with the current `expectedDocumentId`,
`effectiveAt`, acceptance evidence and the replacement `paymentTerms`. `{}` clears the default.
The vendor page's **Pay from** uses this default for open bills and the actual payment account
for settled bills. It stays empty when neither is recorded. This records instructions; it does
not initiate a payment.

A replacement refuses if an unreversed payment or later match dated after the replacement
remains on the original claim, including a repayment after a bounce. Use `payments.unallocate`
for a later match, or `payments.reverse` for an initial allocation,
then record or match it against the replacement. Fully undone later payments and matches
do not block replacement or contribute retained credit. The replacement result’s
`remainingCredit` includes bounces already recorded in the command, even when their effective
date follows the replacement. Source retries retain that outcome. If a replacement was itself
credited with `refundSettled`, a bounce consumes its available refund credit; when a refund
payment consumed that credit, undo that payment with `payments.unallocate` or `payments.reverse`
first, then retry the original payment reversal.

For an initial refund-payment allocation, reverse the payment and record the actual cash refund
again with no allocations before retrying the bounce. This retains real cash as unapplied vendor
funds while freeing the refund claim. A later allocation can be unapplied without moving cash.
The same step applies when a cash refund consumed excess credit from a smaller replacement.
If the refund allocation was wholly or partly undone after the proposed bounce date, the reversal
refuses with that undo date. Date the payment correction at or after it and retain the original
bank bounce as evidence. This prevents simultaneously reopening payable and restoring vendor credit.
