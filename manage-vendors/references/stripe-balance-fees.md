# Stripe fees deducted from the balance

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
for the control-plane implementation, and [Stripe exploration](../../manage-sales/references/explore-stripe.md#cash-gross-or-net)
for gross receipts and net payouts.

Corrections do not rewrite an adopted contract. `contracts.amend` changes terms within its
pinned right definition; it cannot remove that definition's monthly billing schedule. To use
the corrected unscheduled definition, propose `contracts.adopt` with `replaces: [<old id>]`,
starting after existing billed coverage through the ordinary replacement workflow below.
It preserves the old bills and refuses a start inside their coverage. Before every subsequent
bill, still match historical aliases and invoices across both contracts. Do not reverse valid
history, omit a required period or rename a source fact to work around a refusal.
